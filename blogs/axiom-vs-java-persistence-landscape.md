# Understanding Java Persistence: Where Axiom Sits Among Hibernate, jOOQ, Spring Data, MyBatis, Spring JDBC, EclipseLink, JDBI, and Querydsl

*A grounded comparison, built on the actual `axiom-spec` / `axiom-warp-jdbc` / `axiom-warp-reactive` source (Axiom **2.0.0**) and the well-documented behavior of the other eight tools.*

> **Ensemblu / Project Axiom:** Immutable data-oriented infrastructure for modern Java. Zero reflection magic, zero dependency bloat, zero framework control flow.

---

## 1. The real axis of comparison

Every Java persistence tool answers the same question differently:

> **Who is the authority on what a value’s real database type is, and how is that authority expressed?**

Three answers exist in this landscape:

1. **A Java-side type registry decides** (reflection- or annotation-driven).  
   Hibernate, EclipseLink, Spring Data JPA.

2. **A generated Java API, derived from the DB schema, decides.**  
   jOOQ, Querydsl.

3. **The SQL text itself decides, explicitly, at the point of use.**  
   MyBatis, Spring JDBC / `NamedParameterJdbcTemplate`, JDBI — **and Axiom**, which formalizes this into a strict, minimal, dual-engine contract (`AxiomProtocol` + `::cast`).

Axiom belongs to the third group philosophically. It is the only tool in that group that:

- has **zero** reflection / annotation dependency anywhere in the pipeline, and  
- shares **one identical contract** across both a blocking JDBC engine and a reactive Vert.x engine.

---

## 2. Per-tool profile

### Hibernate
Entities are annotated (`@Entity`, `@Column`, …) or XML-mapped. Reflection walks the class graph to generate SQL and marshal `ResultSet` columns into fields. The internal `org.hibernate.type` registry is the Java ↔ JDBC ↔ DDL authority. Types outside the default registry (`jsonb`, arrays, native enums, PostGIS) need a custom `UserType` or `AttributeConverter`.

- **Type authority:** Java class + Hibernate’s registry (reflection-driven).  
- **SQL visibility:** Hidden by default (HQL/Criteria); visible only if logged or dropped to native SQL.  
- **Casting:** Yours, via custom converters — not inline `::cast`.

### EclipseLink
The other major JPA provider. Same shape as Hibernate: annotation/XML mapping, reflection-based marshaling, `@Converter` for non-default types. Historically stronger in some JPA edge cases and multi-tenancy; smaller ecosystem.

- **Type authority:** Same category as Hibernate.  
- **Casting:** Custom converters.

### Spring Data (JPA)
A repository abstraction on top of JPA (almost always Hibernate). You write `interface UserRepo extends JpaRepository<User, Long>`; Spring Data generates the implementation via proxies and method-name parsing (`findByEmailAndActive(...)`). A second layer of reflection/proxy magic on top of Hibernate.

- **Type authority:** Delegates to the JPA provider + method-name inference.  
- **Risk:** Furthest from explicit SQL — a misnamed method silently changes query semantics with no SQL text to inspect.

### jOOQ
Inverts the direction: build-time codegen introspects the live schema and emits typed Java APIs. You write queries via a fluent DSL that mirrors SQL. Non-mapped types use a registered `Converter` / `Binding`.

- **Type authority:** Generated code from the DB schema — closest in spirit to “let Postgres be the source of truth,” but encoded as a large generated Java API.  
- **SQL visibility:** High.  
- **No runtime reflection**, but a **mandatory codegen step** on every migration.

### Querydsl
Fluent type-safe queries, but Java-source-first: Q-classes generated from JPA entity annotations (annotation processor), not from the live database. Historically paired with Hibernate; the standalone SQL module is less actively maintained.

- **Type authority:** Annotated entities → inherits the JPA provider’s type story + its own codegen.  
- **Health note:** Core project has had maintenance gaps; check current status before adopting.

### MyBatis
A SQL-mapper, not an ORM. You write SQL (XML or annotations) with `#{param}` placeholders; MyBatis binds and maps results. Closest mainstream tool to Axiom’s philosophy: SQL is explicit; you write the cast when Postgres needs one (`#{payload}::jsonb` works like Axiom’s `:java.payload::jsonb`).

- **Type authority:** The SQL you write.  
- **Reflection:** Used for POJO result mapping unless you hand-write a result handler — a difference from Axiom’s protocol-driven materialization into `PersistentMap`.

### Spring JDBC / `NamedParameterJdbcTemplate`
Thin wrapper over raw JDBC. Full SQL with `:namedParam`, bind a `Map` or `SqlParameterSource`, map rows yourself. No entity mapping, no codegen. Reflection only if you opt into `BeanPropertySqlParameterSource`.

- **Type authority:** SQL text — same category as Axiom and MyBatis.  
- **No closed type contract** like `AxiomProtocol`; null-vs-`jsonb` gotchas are yours to handle.  
- **Structurally nearest to Axiom on the JDBC side.** Axiom formalizes the “explicit SQL + explicit cast” convention into an enforced contract (`IngressIntegrity.verifyAlignment`, closed `AxiomProtocol` enum).

### JDBI
Similar territory to Spring JDBC, standalone (no Spring). Fluent API and optional annotated interfaces. Extensible via `ColumnMapper` / `ArgumentFactory`.

- **Type authority:** SQL text.  
- **Reflection:** Optional (annotated-interface style uses proxies).

---

## 3. Where Axiom differs from all eight

1. **Zero reflection anywhere, by design.**  
   Read path (`ResultRow` / navigators) and write path (`AxiomProtocol` enum-dispatched setters) are plain method dispatch. `axiom-warp-reactive` goes further and disables Vert.x’s reflection-based JSON codec (`AxiomGhostCodec`) so nothing reintroduces it.

2. **A closed, minimal type contract — not an open registry.**  
   `AxiomProtocol` is a **closed enum**: six concrete carriers + `OPAQUE`. Nothing is added per project. MyBatis/JDBI can grow a first-class `jsonb` handler; Axiom deliberately never will — non-primitives go through `OPAQUE` + `::cast`, permanently.

3. **One identical contract across blocking and reactive engines.**  
   Nothing else in this list does this. `AxiomBinder` has two production implementations (`JdbcBinder`, `ReactiveBinder`) behind the same `AxiomProtocol` / `SqlParser` contract. That is also why the `TIMESTAMP` / timezone seam between the two engines matters: it is a place where two engines that are *supposed* to behave identically can diverge (see the [type-bridging guide](axiom-type-bridging-guide.md)).

4. **No schema codegen step.**  
   `AxiomProtocol` is fixed at compile time. Correctness beyond the six primitives is enforced by you writing the right `::cast`, not by a generator. Faster to set up; no build dependency on a live database. **Schema drift is caught at runtime by Postgres**, not at compile time by a generator.

---

## 4. Comparison matrix

| Tool | Type authority | Reflection / proxies | SQL visibility | Non-primitives (json / uuid / enum / array) | Reactive engine | Schema codegen? |
|------|----------------|----------------------|----------------|---------------------------------------------|-----------------|-----------------|
| **Hibernate** | Java class + internal registry | Yes, extensively | Hidden unless logged | Custom `UserType` / converter | Bolted-on (Hibernate Reactive) | No |
| **EclipseLink** | Java class + internal registry | Yes | Hidden unless logged | Custom `@Converter` | No mainstream story | No |
| **Spring Data JPA** | JPA provider + method-name parsing | Yes (proxies + provider) | Hidden; query inferred | Same as provider | Separate R2DBC stack | No |
| **jOOQ** | DB schema via generated code | No runtime | High | Generated or custom `Binding` | No native reactive | **Yes** |
| **Querydsl** | Annotated entities → Q-classes | Build-time processor | Depends on backend | Inherits from JPA | No | **Yes** |
| **MyBatis** | SQL you write | For bean result mapping | High | Explicit `::cast` | No | No |
| **Spring JDBC** | SQL you write | Only if bean param source | High | Explicit `::cast`; no null-type safeguard | No | No |
| **JDBI** | SQL you write | Annotated-interface style | High | Explicit `::cast`; extensible factories | No | No |
| **Axiom** | SQL you write, enforced by closed contract | **None, by design** | High — literal templates + `:java.name` | Explicit `::cast` via `OPAQUE` | **Yes — shared contract JDBC + reactive** | No |

---

## 5. Honest trade-offs

### Correctly reframed as non-goals (not gaps)

- **No entity / POJO mapping layer** — rows are `PersistentMap` / `PersistentList`.  
- **No open type-handler registry** — closed `AxiomProtocol` is intentional.  
- **No cascade / lazy-load / dirty-checking** — framework control flow is rejected on purpose.  
- **No schema codegen** — runtime Postgres validation over compile-time generated types.

### Real seams to respect

- **Postgres-first** — `::cast` and coercion families are Postgres’s; other engines need their own rules.  
- **`TIMESTAMP` vs naive `timestamp`** — prefer `timestamptz`; do not trust default timezone behavior on naive columns (see type-bridging guide).  
- **Schema drift fails at run time** — there is no generator to catch a renamed column at build time.

---

## 6. When to choose what

| You want… | Lean toward |
|-----------|-------------|
| Full ORM, rich domain objects, cascading | Hibernate / EclipseLink / Spring Data JPA |
| Type-safe DSL + schema-driven Java API | jOOQ |
| Explicit SQL, flexible handlers, optional POJOs | MyBatis / JDBI / Spring JDBC |
| Explicit SQL, **zero reflection**, one contract for **JDBC + reactive**, immutable maps | **Axiom** |

Axiom is not a drop-in Hibernate replacement. It is a deliberate alternative for teams that treat **SQL as the contract**, **Postgres as the type authority**, and **immutable data** as the only representation worth keeping in memory.

---

## Related

- [Axiom Type Bridging Guide](axiom-type-bridging-guide.md) — when to write `::cast`  
- Live demo: [axiom-strike-jdbc](https://github.com/ensemblu-corp/axiom-strike-jdbc)  
- Modules (2.0.0): `axiom`, `axiom-spec`, `axiom-warp-jdbc`, `axiom-warp-reactive`
