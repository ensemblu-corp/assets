# Axiom Type Bridging: When You Must Cast, and When You Must Not

*A working reference for the JSON → Java → `AxiomProtocol` → Postgres pipeline across `axiom-spec`, `axiom-warp-jdbc`, and `axiom-warp-reactive` (Axiom **2.0.0**).*

---

## 1. The core mechanism, in one paragraph

JSON has 5 types. Postgres has 40+. Axiom does **not** try to close that gap in Java. It defines **seven carriers** in `AxiomProtocol` (`STRING`, `INTEGER`, `LONG`, `DOUBLE`, `BOOLEAN`, `TIMESTAMP`, and the catch-all `OPAQUE`).

`SqlParser.forge()` turns your `:java.name` placeholders into bare `?` (or `$1, $2…` via `Dialect` on the reactive engine), leaving every other character in the SQL template — including any `::cast` you wrote — **completely untouched**. The binder (`JdbcBinder` / `ReactiveBinder`) sends the Java value with a generic type tag. **Real type resolution happens on the Postgres side**, at bind time, against the destination column or the explicit `::cast` you wrote. Java never models Postgres’s type catalog — it delegates that job to Postgres itself.

---

## 2. Why this works: Postgres’s coercion graph

Postgres does not require the parameter’s declared type to exactly match the column. If the two types belong to the same **implicit / assignment-cast family**, Postgres coerces automatically. If they don’t, you get an error (or worse, a silent wrong value) unless you force it with `::type`.

### Families that coerce for free (no `::cast` needed)

| Carrier | Covers these Postgres types “for free” | Why |
|--------|----------------------------------------|-----|
| `STRING` | `varchar(n)`, `text`, `char(n)` / `bpchar`, `name`, domains over these | Built-in casts both ways. Length/padding rules are enforced by Postgres at assignment. |
| `INTEGER` / `LONG` / `DOUBLE` | `int2`, `int4`, `int8`, `numeric`, `real`, `double precision` | Strict hierarchy — see §2a. |
| `TIMESTAMP` | `date`, `timestamp`, `timestamptz` | Connected in Postgres’s temporal graph — **but see §4 for the real caveat**. |
| `BOOLEAN` | `bool` | Family of one. |

**Rule of thumb:** if the column is a plain member of one of these families, **do not write a `::cast`**. It’s noise — the carrier already routes correctly.

### 2a. The numeric family is a hierarchy

```text
int2 → int4 → int8 → numeric → real → double precision
```

- **Widening (moving right)** is an **implicit** cast — safe in expressions and binds.
- **Narrowing (moving left)** is **assignment-only** — fine for `INSERT`/`UPDATE` targets (what Axiom’s binders do), but would need an explicit cast in a bare expression.
- Range errors (`smallint out of range`, etc.) are thrown by Postgres at bind time.

**Precision caveat — `real` / `double precision` vs `numeric`:**  
binary floating-point vs exact decimal. The cast is legal and silent. A `double` carrying accumulated rounding error can land in a `numeric` column with those artifacts intact. For money or precise quantities, prefer exact decimal on the Java side and verify the value going in — do not trust the `DOUBLE` carrier by default.

---

## 3. Where you MUST cast: everything routed through `OPAQUE`

`OPAQUE` binds as a plain string (`bindString` → `setString` / `addString`). Postgres does **not** freely coerce a bare string into these targets. Skip the cast and you get errors like `column "x" is of type jsonb but expression is of type character varying` — or no error and the wrong result.

**Always write an explicit `::type` next to the placeholder for:**

| Target | Template fragment |
|--------|-------------------|
| `jsonb` / `json` | `:java.payload::jsonb` |
| `uuid` | `:java.id::uuid` |
| Custom enum | `:java.status::order_status` |
| Arrays | `:java.tags::text[]`, `:java.ids::int[]` |
| `interval` | `:java.duration::interval` (not covered by `TIMESTAMP`) |
| `numeric(p,s)` when you want a deliberate boundary error | `:java.amount::numeric(12,2)` |
| PostGIS / domains / extension types | `:java.geom::geometry`, etc. |

**Template pattern:**

```sql
INSERT INTO events (id, payload, tags, status)
VALUES (
  :java.id::uuid,
  :java.payload::jsonb,
  :java.tags::text[],
  :java.status::order_status
)
```

### Nulls through `OPAQUE`

- **JDBC:** `JdbcBinder.bindNull()` uses `Types.OTHER` for `OPAQUE` nulls so Postgres can infer the type (not `VARCHAR`).
- **Reactive:** `ReactiveBinder.bindNull()` sends an untyped null (`tuple.addValue(null)`). Both paths are safe for nulls; you do not need extra ceremony.

---

## 4. The carrier that isn’t as clean as it looks: `TIMESTAMP`

`date` / `timestamp` / `timestamptz` do not throw cast errors against each other — but “no error” ≠ “correct”:

| Column type | Safety |
|-------------|--------|
| **`timestamptz`** | Safe. Absolute instant. JDBC (`java.sql.Timestamp` from epoch millis) and reactive (`Instant`) agree. |
| **`timestamp` (naive)** | **Not safe by default.** Writing an absolute instant into a naive column requires a timezone for wall-clock digits. `JdbcBinder`’s `setTimestamp()` with no `Calendar` uses the **JVM default timezone**, silently. Real divergence between JDBC and reactive engines. |

**Guidance:**

- Prefer **`timestamptz`** only — no cast, no special handling.
- If you must use naive `timestamp`, do **not** trust the default binder. Control timezone explicitly at the JDBC layer (e.g. `setTimestamp(i, ts, Calendar.getInstance(TimeZone.getTimeZone("UTC")))`), or avoid naive columns entirely.

---

## 5. Quick reference

| Situation | Cast needed? |
|-----------|--------------|
| `text` / `varchar` / `char` / `name` | ❌ No |
| `int2` / `int4` / `int8` / `numeric` / `real` / `double precision` | ❌ No (assignment-cast) — but see §2a for `double` ↔ `numeric` precision |
| `bool` | ❌ No |
| `timestamptz` | ❌ No |
| Naive `timestamp` | ⚠️ No cast syntactically — verify timezone handling manually |
| `jsonb` / `json` | ✅ `:java.x::jsonb` / `::json` |
| `uuid` | ✅ `:java.x::uuid` |
| Custom enum | ✅ `:java.x::your_enum` |
| Array type | ✅ `:java.x::type[]` |
| `interval` | ✅ `:java.x::interval` |
| Any other extension / domain / custom type | ✅ Yes |

---

## 6. Portability note

This design is **Postgres-specific**:

1. `::type` is not portable to MySQL, SQL Server, or Oracle (`CAST(value AS type)` or vendor equivalents).
2. The free coercion families above are Postgres’s cast catalog. Other engines differ (sometimes dangerously — e.g. MySQL non-strict truncation).
3. `Dialect` in `axiom-warp-reactive` has a real implementation for **`POSTGRES`**; `GENERIC` is a pass-through. Coercion assumptions have **not** been re-validated for other databases.

**If Axiom ever targets another database, every rule in this document must be re-derived from that engine’s cast catalog. None of it transfers automatically.**

---

## Related reading

- [Axiom vs the Java persistence landscape](axiom-vs-java-persistence-landscape.md)
- Live demo: [axiom-strike-jdbc](https://github.com/ensemblu-corp/axiom-strike-jdbc)
- Modules: `axiom-spec`, `axiom-warp-jdbc`, `axiom-warp-reactive` (2.0.0)
