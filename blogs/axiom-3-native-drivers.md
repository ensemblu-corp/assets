# Why Axiom 3 Talks to Databases Natively

![Axiom](https://img.shields.io/badge/axiom-3.0.0-blue)
![Java](https://img.shields.io/badge/Java-26-orange)
![Panama](https://img.shields.io/badge/Project-Panama-informational)

*Five vendor C clients, one `Executioner` shape, and the trade-offs of connections with no pool. The examples in this article are written against the 3.0.0 sources of the `axiom-warp-*-driver` modules.*

---

## 1. The question 3.0 asked

Axiom 2.x executed SQL through JDBC. It did so carefully, with a closed type contract (`AxiomProtocol`), explicit `::cast` boundaries and no reflection in Axiom itself. But under that care was still the usual stack:

```
your code → AxiomWarp → JDBC API → connection pool → vendor JDBC driver → wire protocol
```

Each of those layers is reasonable on its own, but together they add up:

- **The JDBC driver re-implements the wire protocol in Java** and makes its own decisions about type inference, prepared-statement thresholds and batching.
- **The pool is a second lifecycle** with its own tuning surface (min, max, idle, leak detection) and its own failure modes.
- **The integration code** (`SovereignDataSource`, adapters, pool providers) exists only to connect the two.

Java 22 made the Foreign Function & Memory API final. That made a different question practical: *what if Axiom called the database vendor's own C client directly?* Every major database already ships one, maintained by the people who wrote the server.

```
3.0   your code → Executioner → Panama downcall → libpq / libmysqlclient / OCI / sqlite3 / ODBC → wire
```

---

## 2. One shape, five engines

Every driver, whether it's PostgreSQL, MySQL, Oracle, SQLite or SQL Server, exposes the same surface:

```java
Executioner db = PostgreSQL.connect(file("database.properties"))   // or MySQL, Oracle, SQLite, SqlServer
        .requiredHost().requiredPort().requiredDbName().requiredUser()
        .optionalPassword()
        .getPostgreSqlDriverOrThrow();

db.wield(conn -> …);                         // one native connection, always closed
db.transaction(conn, () -> …);               // atomic block → rollback on any throwable
db.wieldTransactional(() -> sql1, () -> sql2);
db.shot(sql).withConnection(conn);           // raw statement
db.dynamicQuery(sql).to(v).bind(T).execute(conn);
db.fromFile(csv).withConnection(conn).usingFileHeaders().withDataTypes(..).onTableName(t);
```

Every driver uses the same package layout too:

| Package | Responsibility |
|---|---|
| `api` | Static `connect(..)` entry point and the `Executioner` |
| `provision` | `RawProvisioner`, which validates the config and builds the executioner |
| `engine` | `Strike` (raw + parameterised execution) and `SqlParser` (marker rewriting) |
| `ingest` | `Hammer`, contract-verified bulk loading |
| `io` | `CsvEngine`, a streaming CSV reader (classpath first, then filesystem) |
| `scope` | `BoundScope`, which binds a confined `Arena` to the current call |
| `catalog` | The engine's type catalog plus status enums |

Learn one driver and you know all five. The differences are underneath, where each engine has its own idiomatic way of doing things.

---

## 3. Provisioning: a curried chain, not a builder

```java
public static RequiredHost basedOnConfigSource(ConfigSource config) {
    return () -> () -> () -> () -> () -> () -> materialize(config);
}
```

The chain `requiredHost() → requiredPort() → requiredDbName() → requiredUser() → optionalPassword() → get…OrThrow()` is a sequence of single-method interfaces. Two consequences:

1. **You can't forget a step.** Skipping `requiredPort()` isn't a runtime error; the next method doesn't exist yet, so the code doesn't compile.
2. **Validation happens once, at boot.** `materialize` reads every field through `Result`, validates the port range, and fails with `🚨 INFRASTRUCTURE BOOT FAILURE` if anything is wrong. A misconfigured service never starts accepting traffic.

The chain matches each engine's real requirements. Oracle asks for `requiredServiceName()` instead of a database name, and SQLite only asks for `requiredPath()`.

---

## 4. `wield`: connection ownership you can read

This is the PostgreSQL implementation, in full:

```java
public <T> Result<T> wield(ThrowingFunction<MemorySegment, Result<T>> action) {
    return BoundScope.use(Arena.ofConfined(), () -> {
        MemorySegment conn = engage();
        try {
            Result<T> result = action.apply(conn);
            disconnect(conn);
            return result;
        } catch (Throwable t) {
            disconnect(conn);
            throw t;
        }
    }).flatMap(res -> res);
}
```

And `BoundScope`:

```java
private static final ScopedValue<Arena> TX_ARENA = ScopedValue.newInstance();

public static <T> Result<T> use(Arena arena, ThrowingSupplier<T> op) {
    return Axiom.Check.attempt(() ->
            ScopedValue.where(TX_ARENA, arena).call(() -> {
                try (arena) { return op.get(); }
            }));
}
```

The whole connection lifecycle is in these two snippets:

- A **confined arena** owns every native allocation made for the call (connection strings, parameter arrays, ingest buffers).
- **`ScopedValue`** makes that arena visible to nested code (`BoundScope.currentArena()`) without thread-locals. That matters on virtual threads, which you'll use a lot (§8).
- The connection is **always** finished, and the arena is **always** closed, whether the lambda returns normally or throws.
- Any throwable becomes a failed `Result`. Nothing escapes unless you call `getOrThrow()`.

---

## 5. Parameters: `:java.name` in, native syntax out

You write one marker syntax for every engine:

```sql
SELECT * FROM accounts WHERE status = :java.status LIMIT :java.limit
```

Each driver's `SqlParser.forge(..)` scans the bytes once and rewrites every marker into the engine's own placeholder:

| Engine | Rewritten to | Bound with |
|---|---|---|
| PostgreSQL | `$1`, `$2`, … | `PQexecParams` with an explicit type OID per value |
| Oracle | `:1`, `:2`, … | `OCIBindByPos` |
| SQLite | positional | `sqlite3_bind_*` chosen by `SqliteType` |
| SQL Server | `?` | `SQLBindParameter` with C-type/SQL-type from `SqlServerType` |
| MySQL | `?` → spliced literal | `mysql_real_escape_string` + quoting (see below) |

Then you bind every marker **in order**, each with an explicit type from the engine's catalog:

```java
db.dynamicQuery(sql)
  .to("ACTIVE").bind(PostgresType.TEXT)
  .to(10).bind(PostgresType.INT8)
  .execute(conn);
```

Two rules are enforced at execution time: you can't bind more values than there are markers, and you can't execute with fewer. A statement can have at most **21** markers. That's a deliberate ceiling: a query that needs more parameters than that is usually a bulk operation that belongs in `fromFile(..)` or a transaction loop.

### The MySQL exception, stated plainly

The MySQL C API's text protocol has no out-of-band parameters. Its only native parameter path is the prepared-statement API, which needs per-type `MYSQL_BIND` structs and is much more fragile to drive through raw memory layouts. The 3.0 MySQL driver chooses the text protocol and makes the splice explicit:

```java
db.dynamicQuery("SELECT * FROM accounts WHERE status = :java.status AND balance > :java.min")
  .bindQuotedValue(userInput)   // escaped by libmysqlclient itself, then quoted
  .bindValue(1000)              // trusted numeric literal only
  .execute(conn);
```

Escaping is done by the vendor's own `mysql_real_escape_string`, using the connection's character set, which is the correct primitive for this protocol. But it's a different safety model from the other four engines, and the API names it so you can't mistake one for the other.

---

## 6. Bulk ingest: verify the contract, then move the bytes

Every driver's `Hammer` follows the same discipline:

1. **Resolve headers**, either from the file (`usingFileHeaders()`) or declared (`withHeaders(..)`).
2. **Verify the contract**: one declared type per header, matched against the live table catalog by exact column name and exact type. Every mismatch is collected and reported **before a single row is sent**.
3. **Stream in batches** of 1,000 rows using the engine's idiomatic bulk path.

A typical contract failure looks like this:

```
Contract Breach: declared mapping does not match table accounts:
  - column 'balance' declared VARCHAR (oid 1043) but the table has numeric(18,4) (oid 1700)
  table columns: [account_id, owner, balance, currency, tier, status, created_at]
```

And here's each engine's bulk path:

| Engine | Catalog check | Bulk path |
|---|---|---|
| PostgreSQL | `pg_attribute` via `::regclass` | `COPY t (cols) FROM STDIN WITH CSV`, streamed through `PQputCopyData` |
| MySQL | `information_schema.COLUMNS` | batched multi-row `INSERT … VALUES (…), (…)` via `mysql_real_query` (no native upcalls needed) |
| Oracle | `USER_TAB_COLUMNS` | prepared `INSERT`, bind buffers registered **once** (`SQLT_INT`/`FLT`/`STR`), `OCITransCommit` per batch |
| SQLite | `PRAGMA table_info` | `sqlite3_prepare_v2` once, then `bind → step → reset → clear_bindings` per row, batched transactions |
| SQL Server | `INFORMATION_SCHEMA.COLUMNS` | prepared `INSERT`, buffers + length indicators bound once with `SQLBindParameter`, overwritten per row |

PostgreSQL's `COPY` statement is built with an **explicit, quoted column list**, so column order comes from your declaration and never from the table's physical layout.

---

## 7. Results: immutable rows, honest text

Every query returns `Result<PersistentList<PersistentMap<String, Object>>>`. Column values are the **text the client library produced**: `"50000.0000"`, not a `BigDecimal`. That's intentional:

- The driver doesn't guess types. The database already formatted the value, and Axiom doesn't silently re-interpret it.
- Conversion happens at the edge, explicitly, where you know what you want:

```java
var balance = row.targetKey("balance").toBigDecimalResult();
var count   = row.targetKey("total_accounts").toLongVal();
```

Statements that return no rows (`INSERT`, DDL) yield a single `{status: "OK"}` row, so every call has the same result shape.

---

## 8. Concurrency without a pool

Since each `wield` owns an independent connection and a confined arena, concurrency needs no special API:

```java
try (var lanes = Executors.newVirtualThreadPerTaskExecutor()) {
    var count = lanes.submit(() -> db.wield(conn -> countQuery(conn)));
    var sum   = lanes.submit(() -> db.wield(conn -> sumQuery(conn)));
    var c = count.get();
    var s = sum.get();
    return c.flatMap(x -> s.map(y -> merge(x, y)));
}
```

The 2.x `warp.parallel(..)` API and the entire Vert.x engine were retired because, with virtual threads and per-call connections, they no longer covered anything that plain Java doesn't already do.

---

## 9. The honest trade-offs

**Connection cost.** Without a pool, every `wield` pays a connection handshake (TCP, plus TLS and authentication if configured). That's the price of simple ownership. To keep it low:

- Put related statements **inside one `wield`**, especially transactions, batches and read-then-write sequences.
- For very high request rates, measure against your latency budget before adopting the no-pool model wholesale.

**Native client installation.** Your deployment image now has to contain the vendor client (`libpq`, Instant Client, ODBC Driver 18…). That's an operational dependency rather than a Maven one, but it's still a dependency.

**Library resolution per OS.** Drivers look the client up by name. On Windows, a DLL on `PATH` is enough. On Linux and macOS, check that the lookup name resolves on your images (an exact soname, a symlink or `java.library.path`) before relying on it in CI.

**Preview flags.** The stack compiles with `--enable-preview` on Java 26, so your application runs with `--enable-preview`. Add `--enable-native-access=ALL-UNNAMED` (or the `Enable-Native-Access` manifest attribute) to silence restricted-method warnings.

---

## 10. When to choose what

| You need… | Choose |
|---|---|
| Explicit SQL, zero framework, every call auditable | Axiom 3 native drivers |
| An embedded database for tools, tests or the edge | `axiom-warp-sqlite-driver` (`engine.path=:memory:` for tests) |
| Several engines behind one API shape | Axiom 3: same `Executioner` everywhere |
| An ORM-managed object graph with lazy loading | Not Axiom. See [Axiom vs. the Java Persistence Landscape](axiom-vs-java-persistence-landscape.md) |
| Very high request rates on a pooled connection model | Benchmark first: Axiom 3 trades pooling for simple ownership |

---

## Related

- [Axiom 2.0.0 → 3.0.0: Ultimate Release Changelog](changelog-2.0.0-to-3.0.0.md)
- [A Black Box for Every Result: axiom-telemetry](axiom-telemetry-flight-recorder.md)
- [Hard Truths About Java Persistence](axiom-persistence-hard-truths.md)
- [Project Axiom homepage](https://ensemblu-corp.github.io/assets/index.html)
