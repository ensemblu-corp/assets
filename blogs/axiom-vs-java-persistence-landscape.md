# Understanding Java Persistence: Where Axiom Sits Among Hibernate, jOOQ, Spring Data, MyBatis, Spring JDBC, EclipseLink, JDBI, and Querydsl

*A grounded comparison, built on the actual `axiom-spec` / `axiom-warp-jdbc` / `axiom-warp-reactive` source and standard, well-documented behavior of the other eight tools.*

> **Ensemblu / Project Axiom:** *Immutable data-oriented infrastructure for modern Java. Zero reflection magic, zero dependency bloat, zero framework control flow.*

---

## 1. The real axis of comparison

Every Java persistence tool answers the same question differently: **"Who is the authority on what a value's real database type is, and how is that authority expressed?"**

Three answers exist in this landscape:

1. **A Java-side type registry decides** (reflection- or annotation-driven). Hibernate, EclipseLink, Spring Data JPA.
2. **A generated Java API, derived from the DB schema, decides.** jOOQ, Querydsl.
3. **The SQL text itself decides, explicitly, at the point of use.** MyBatis, Spring JDBC / `NamedParameterJdbcTemplate`, JDBI — and Axiom, which formalizes this into a strict, minimal, dual-engine contract (`AxiomProtocol` + `::cast` convention, covered in depth earlier).

Axiom belongs to the third group philosophically, but it's the only tool in that group that (a) has zero reflection/annotation dependency anywhere in the pipeline, and (b) shares one identical contract across both a blocking JDBC engine and a reactive Vert.x engine.

---

## 2. Per-tool profile

### Hibernate
The reference JPA implementation. Entities are Java classes annotated (`@Entity`, `@Column`, `@Id`...) or XML-mapped; at runtime, reflection walks the class graph to generate SQL and to marshal `ResultSet` columns back into object fields. Its internal `org.hibernate.type` registry is the Java-class ↔ JDBC-type ↔ dialect-DDL-type authority — the thing that decides `Integer` → `INTEGER`, `String` → `VARCHAR`, etc. Anything outside that default registry (Postgres `jsonb`, arrays, enums-as-native-enum, PostGIS) needs a custom `UserType` or `AttributeConverter` you write and register yourself.
- **Type authority:** Java class + Hibernate's internal registry (reflection-driven).
- **SQL visibility:** Hidden by default (HQL/Criteria generates it); visible only if you log it or drop to native queries.
- **Casting responsibility:** Yours, via custom `UserType`/converter classes — not inline `::cast` syntax.

### EclipseLink
The other major JPA provider (reference implementation for Java EE/Jakarta EE). Functionally similar shape to Hibernate — annotation/XML-driven entity mapping, reflection-based marshaling, a converter API (`@Converter`) for anything outside its default type mappings. Historically stronger in some JPA-spec edge cases and multi-tenancy; ecosystem and community are smaller than Hibernate's.
- **Type authority:** Same category as Hibernate — Java class + internal type system, reflection-driven.
- **Casting responsibility:** Custom converters, same pattern as Hibernate's `AttributeConverter`.

### Spring Data (JPA)
Not a persistence engine itself — a repository-abstraction layer sitting on top of JPA (almost always Hibernate underneath). You write an interface (`interface UserRepo extends JpaRepository<User, Long>`), and Spring Data generates the implementation via proxies, reflecting on method names (`findByEmailAndActive(...)`) to derive queries. This adds a second layer of reflection/proxy magic on top of Hibernate's own.
- **Type authority:** Delegates entirely to the underlying JPA provider (usually Hibernate) — same type-bridging story, plus method-name-parsing magic on top.
- **Distinctive risk:** The furthest from explicit SQL of anything in this list — a misnamed repository method silently changes query semantics with no SQL text to inspect.

### jOOQ
Inverts the direction: a build-time code generator introspects your live schema (`information_schema`/`pg_catalog`) and emits Java classes with fields already typed to match each column — precision/scale for `numeric(10,2)`, array component types, enum values, all generated, not hand-declared. You then write queries via a fluent, compile-time-checked DSL that mirrors SQL syntax. For types the generator has no built-in Java mapping for, you register a `Converter`/`Binding`.
- **Type authority:** Generated code, derived from the DB schema itself — closest in spirit to Axiom's "let Postgres be the source of truth," but jOOQ encodes that truth into a large generated Java API rather than leaving it in SQL text.
- **SQL visibility:** High — the DSL closely mirrors real SQL, and `.getSQL()` shows exactly what's sent.
- **No reflection at runtime**, but a mandatory **codegen step** tied to your schema, which needs to be rerun on every migration.

### Querydsl
Similar fluent, type-safe query-building idea to jOOQ, but Java-source-first rather than schema-first: it generates "Q-classes" from your JPA entity annotations (via an annotation processor) rather than from the live database. Historically paired with Hibernate/JPA as the query layer; the standalone SQL module exists but is far less actively maintained than the JPA integration.
- **Type authority:** Derived from your annotated entity classes — so it inherits whatever type-bridging story your underlying JPA provider already has, plus a codegen step of its own.
- **Project health note:** The core Querydsl project has had maintenance gaps; several forks/successors exist. Worth checking current status before adopting for new work.

### MyBatis
A SQL-mapper, not an ORM. You write SQL yourself (in XML or annotations), with `#{paramName}` placeholders; MyBatis handles the binding and result mapping, with a `TypeHandler` registry for non-trivial types. Structurally the closest existing mainstream tool to Axiom's philosophy — SQL is explicit and visible, you write the cast yourself when Postgres needs one (`#{payload}::jsonb` works exactly like Axiom's `:java.payload::jsonb`).
- **Type authority:** The SQL text you write, same principle as Axiom — but MyBatis's default `TypeHandler` set is broader than Axiom's 6-carrier `AxiomProtocol`, and it's extensible via more elaborate XML config than Axiom's single-enum contract.
- **Reflection:** Used for result-object population (mapping columns to POJO fields/setters) unless you hand-write a custom result handler — a difference from Axiom's protocol-driven materialization.

### Spring JDBC / `NamedParameterJdbcTemplate`
A thin wrapper over raw JDBC. You write full SQL with `:namedParam` placeholders, bind a `Map<String,Object>` or a `SqlParameterSource`, and get back `RowMapper`-driven results. No entity mapping, no codegen, no reflection in the SQL-binding path itself (though `BeanPropertySqlParameterSource`, if you opt into it, does use reflection).
- **Type authority:** The SQL text, exactly like Axiom and MyBatis. `NamedParameterJdbcTemplate` doesn't have anything resembling `AxiomProtocol` — no contract of allowed Java types at all; you're calling straight through to `PreparedStatement` semantics, so the same `varchar`-vs-`jsonb` null-binding gotcha we covered for `JdbcBinder` applies here too, without Axiom's `Types.OTHER` fix built in — you'd handle it yourself.
- **This is the tool Axiom is structurally nearest to on the JDBC side** — the difference is that Axiom formalizes the "explicit SQL + explicit cast" convention into an enforced contract (`IngressIntegrity.verifyAlignment`, the closed `AxiomProtocol` enum) rather than leaving it as an unstated convention per call site.

### JDBI
Similar territory to Spring JDBC's `NamedParameterJdbcTemplate`, but a standalone library (no Spring dependency) with a fluent fluent-builder API and optional `@SqlQuery`/`@SqlUpdate` annotated-interface style if you want it. Like MyBatis and Spring JDBC, you write real SQL and bind named/positional parameters; type-handling extensibility is via `ColumnMapper`/`ArgumentFactory` registration.
- **Type authority:** SQL text, same category as MyBatis/Spring JDBC/Axiom.
- **Reflection:** Optional — the annotated-interface style uses proxies/reflection; the fluent `Handle`-based style doesn't have to.

---

## 3. Where Axiom differs from all eight, structurally

1. **Zero reflection anywhere in the pipeline, by design, not by omission.** Hibernate, EclipseLink, Spring Data, Querydsl, and (if you opt into bean-based param sources or annotated interfaces) MyBatis/JDBI all use reflection somewhere. Axiom's read path (`ResultRow.navigate().toObjectVal()`) and write path (`AxiomProtocol`'s enum-dispatched setter) are both plain method dispatch — no `Class.getDeclaredFields()`, no proxy generation, anywhere. `axiom-warp-reactive` goes a step further and actively disables Vert.x's own internal reflection-based JSON codec (`AxiomGhostCodec`) so nothing accidentally reintroduces it.

2. **A closed, minimal type contract instead of an open, extensible one.** MyBatis's `TypeHandler` registry and JDBI's `ArgumentFactory`/`ColumnMapper` are both *extensible* — you can register a handler per type, growing the registry indefinitely. `AxiomProtocol` is a **closed enum**: 6 concrete carriers plus `OPAQUE`. Nothing gets added to it per-project. This is a real trade-off, not a pure win: MyBatis/JDBI can grow a first-class ergonomic handler for `jsonb` if you want one; Axiom deliberately never will — every non-primitive type goes through `OPAQUE` + `::cast`, permanently, by design.

3. **One identical contract shared across blocking and reactive engines.** Nothing else in this list does this. Hibernate Reactive exists but is a substantially different runtime from classic Hibernate with its own gaps and caveats. jOOQ, Querydsl, MyBatis, JDBI, and `NamedParameterJdbcTemplate` are all fundamentally built around blocking JDBC; reactive usage means going around them, not through them. Axiom's `AxiomBinder` interface has two production implementations (`JdbcBinder`, `ReactiveBinder`) behind the identical `AxiomProtocol`/`SqlParser` contract — which is also exactly why the `TIMESTAMP`/`Instant` vs `Timestamp` divergence we found earlier is worth caring about: it's a seam between two engines that are supposed to behave identically and, in that one case, don't quite.

4. **No schema codegen step.** jOOQ and Querydsl both require a build-time generation step tied to your current schema, rerun on every migration. Axiom has no equivalent — `AxiomProtocol` is fixed at compile time regardless of your schema, and correctness for anything beyond the 6 primitives is enforced by you writing the right `::cast`, not by a generator keeping Java types in sync with the database automatically. This is faster to set up and has no build-pipeline dependency on a live database, but it also means **schema drift is caught at runtime by Postgres, not at compile time by a generator** — a `::jsonb` cast against a column that got renamed or dropped will fail when the query runs, not when you build.

---

## 4. Comparison matrix

| Tool | Type authority | Reflection/proxies used? | SQL visibility | Non-primitive types (json/uuid/enum/array) | Reactive engine | Schema codegen required? |
|---|---|---|---|---|---|---|
| **Hibernate** | Java class + internal type registry | Yes, extensively | Hidden (HQL/Criteria) unless logged | Custom `UserType`/`AttributeConverter` | Bolted-on (Hibernate Reactive, separate runtime) | No |
| **EclipseLink** | Java class + internal type registry | Yes, extensively | Hidden unless logged | Custom `@Converter` | No mainstream reactive story | No |
| **Spring Data (JPA)** | Delegates to JPA provider + method-name parsing | Yes (proxies + provider's reflection) | Hidden, and query itself is inferred from method names | Same as underlying provider | Spring Data R2DBC exists as a separate, different stack | No |
| **jOOQ** | DB schema, via generated code | No runtime reflection | High — DSL mirrors SQL | Generated bindings, or custom `Converter`/`Binding` | No native reactive engine (sync JDBC-based) | **Yes**, build-time |
| **Querydsl** | Annotated entity classes, via generated Q-classes | Annotation processor (build-time), no runtime reflection in core path | Depends on backing provider | Inherits from JPA provider | No | **Yes**, build-time |
| **MyBatis** | SQL text you write | Only if using bean-based result mapping | High — you write the SQL | Explicit `::cast` in SQL, same convention as Axiom | No | No |
| **Spring JDBC / `NamedParameterJdbcTemplate`** | SQL text you write | Only if using `BeanPropertySqlParameterSource` | High | Explicit `::cast`, no built-in null-type safeguard | No | No |
| **JDBI** | SQL text you write | Only in annotated-interface style | High | Explicit `::cast`; extensible `ArgumentFactory` | No | No |
| **Axiom** | SQL text you write, enforced via a closed contract | **None, anywhere, by design** | High — templates are literal SQL with `:java.name` placeholders | Explicit `::cast` via `OPAQUE`, non-extensible by design | **Yes — shared contract across `axiom-warp-jdbc` and `axiom-warp-reactive`** | No |

---

## 5. Honest trade-offs — separating "deliberately not a goal" from "still a real gap"

This is where it matters to be precise rather than defensive, because two different kinds of claims need to stay separate — one holds up, one doesn't.

### These are correctly reframed as non-goals, not gaps

Hibernate/EclipseLink's dirty-checking, first/second-level caching, and automatic cascade/lazy-loading all exist to manage a **mutable object graph with hidden identity** — the entity instance you hold has to silently track whether it changed, silently know how to reach related entities, silently decide when to fire a query. That machinery is only necessary because the ORM's core unit is a mutable, identity-bearing Java object. Sharvit's DOP position — data as plain, immutable, transparent structures, decoupled from behavior — rejects that unit of design entirely, not just its caching layer. It's a genuine, structurally-grounded difference, not a rebranded weakness: it's consistent with the persistent, immutable data structures (HAMT map, bitmapped vector trie) at the core of the framework, and with the choice to validate incoming data against an explicit schema (`SchemaGuard`) rather than infer correctness from a class definition the way annotation-driven mapping does.

### This one is not resolved by DOP, and conflating the two would be dishonest

**No compile-time SQL/schema checking** is a different category of claim entirely. jOOQ's and Querydsl's codegen catches a renamed column or a type mismatch *before the query runs*, by comparing your query against the live schema at build time. That has nothing to do with mutability or object identity — it's a static-analysis property, and DOP doesn't argue against static analysis. Whatever validation `SchemaGuard` does for the shape of incoming data, that's a different check than confirming a `:java.payload::jsonb` template in `SqlParser` still points at a real column after a migration. So this gap is real, stands on its own, and isn't something "we don't need" — it's something Axiom currently trades away in exchange for having no codegen/schema-introspection step at all. A future reader deciding whether Axiom fits their project should know this is still an open risk, not a solved one.

The smaller ecosystem / single-maintainer-discovers-the-edge-cases point stands unchanged too — DOP alignment doesn't affect how many people have already hit a given bug in production.

---

## 6. What Axiom actually is, stated plainly

**Ensemblu · Project Axiom — immutable data-oriented infrastructure for modern Java. Zero reflection magic, zero dependency bloat, zero framework control flow.**

Against what this document actually traced through `axiom-spec`, `axiom-warp-jdbc`, and `axiom-warp-reactive`, that's a structural description, not a slogan:

- **Zero reflection magic** — confirmed end to end: `AxiomProtocol`'s enum-dispatched binding, `SqlParser`'s character-level template parsing, and `AxiomGhostCodec`'s deliberate neutering of Vert.x's own Jackson-backed codec in `axiom-warp-reactive`. No layer in the read or write path inspects a class at runtime.
- **Immutable data** — persistent data structures sit at the core of the framework rather than mutable POJOs, and validation of incoming data happens explicitly (`SchemaGuard`) rather than being inferred from a class definition.
- **Zero framework control flow** — the SQL you write is the SQL that runs, with one substitution pass (`SqlParser`) and, on the reactive engine, one further syntax translation (`Dialect`). Nothing decides to fire an extra query, cascade a write, or defer a load on your behalf.