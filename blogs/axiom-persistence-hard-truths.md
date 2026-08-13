# Hard Truths About Java Persistence

*Eight statements every team should confront before the next ORM, repository layer, or “zero-SQL” abstraction ships to production.*

> Ensemblu / Project Axiom · Architecture notes  
> Not a product pitch. A set of positions.

---

## 1. “It is still Hibernate entity-per-table — but now repository-per-controller, not per-entity.”

The industry did not escape the entity model. It renamed the folders.

You still have one Java type per table (or per aggregate that *behaves* like a table). What changed is the packaging: repositories stopped living next to entities and started living next to controllers. The result is the same mapping burden, only more scattered — and harder to see when a single screen’s data needs six joins that no single “entity repository” was designed to express.

**Axiom’s stance:** stop pretending the table is a class. Rows are maps. Queries are SQL. The controller (or handler) owns the query it needs, not a fictional domain object that hopes the ORM will figure it out.

---

## 2. “A program with one screen should just write a SQL query and return a record.”

Most screens are not domain models. They are projections.

A list of accounts with balance, currency, and status is not an `Account` entity graph. It is a query result. Forcing that through an entity, a repository, a DTO, and a mapper is ceremony that pays for itself only when the same shape is reused everywhere — which, for most screens, it is not.

**Axiom’s stance:** one screen → one explicit query → one immutable result structure. If that is a `PersistentList<PersistentMap<String, Object>>`, good. You can still validate it, emit JSON from it, and test it without a session factory.

---

## 3. “Type safety is the most powerful tool in the toolbox — still, don’t overreact and become jOOQ or Hibernate.”

Compile-time checks are real power. Generated DSLs and annotated entities buy some of that power by putting a large, schema-derived (or annotation-derived) type graph between you and the database.

The cost is not “learning the API.” The cost is **authority drift**: the Java type system becomes the place where database truth is supposed to live. Migrations, casts, and edge types (`jsonb`, enums, arrays) then require a parallel registry of converters, bindings, or `UserType`s — so the type safety you bought becomes a maintenance surface.

**Axiom’s stance:** keep type safety where it is cheap and honest — closed carriers (`AxiomProtocol`), explicit contracts, `Result` at boundaries. Do not build a second database inside the JVM so the compiler can pretend to understand Postgres.

---

## 4. “Dynamic query is where the Java persistence topic starts and ends.”

Static CRUD is the brochure. Production is filters, optional joins, sort orders, and “this screen needs three extra columns for this role.”

Every stack that hides SQL eventually reintroduces it as Criteria, Specifications, Querydsl predicates, or string-built JPQL. The teams that never learn to write the query they need spend their time fighting the abstraction that was supposed to protect them from SQL.

**Axiom’s stance:** start from the dynamic query. Named parameters (`:java.status`), explicit contracts, parallel strikes when you need them. If you can express the hard case cleanly, the easy case is trivial.

---

## 5. “Now you can do stuff without understanding what you’re doing — which will be painful when it goes to production.”

Convenience that removes the need to understand *what* runs against the database is not a feature. It is deferred debt with interest compounded at p99 latency and incident time.

Lazy loads, dirty checking, cascade rules, and magic flush order all work — until they don’t, and the person on call has never seen the SQL that actually executed.

**Axiom’s stance:** the SQL you write is the SQL that runs (one substitution pass for placeholders, dialect translation on the reactive path). No second query for a proxy. No cascade you did not write. Understanding is not optional; it is the product.

---

## 6. “Hibernate and other ORMs became popular because it seemed like you didn’t have to worry about SQL and databases.”

That was the sales pitch. It was never true for systems under load, with non-trivial schemas, or with Postgres features that are not “a Java field with an annotation.”

Popularity is not evidence that the model matched the problem. It is evidence that the model matched the desire to *postpone* learning the database.

**Axiom’s stance:** you should worry about SQL and databases. Tools should make that worry precise and local — not invisible until production.

---

## 7. “Don’t tell me about NoSQL if you don’t emphasize most of your mind on an SQL-centric philosophy of accessing a database.”

Document stores, key-value layers, and graph databases have real uses. Using them as an escape hatch from relational thinking usually means the team never learned how to model access paths, constraints, and transactions in SQL — then rediscovers the same problems under different names (eventual consistency, ad-hoc secondary indexes, application-level joins).

**Axiom’s stance:** SQL-centric access is the default intellectual posture. Other stores are tools for specific shapes of data and scale, not a substitute for understanding relations, sets, and queries.

---

## 8. “Hibernate is designed to do the thing you would do naturally if you were an expert at writing SQL to efficiently fetch data — please tell me you aim to be an expert at writing SQL.”

This is the fairest defense of an ORM: a good mapping layer encodes patterns an expert would use. The unspoken premise is that *someone* on the team is that expert, and the mapping stays aligned with reality.

If the team’s goal is to *avoid* becoming expert at SQL, the ORM is not a teacher. It is a wall. When the wall cracks, there is no skill underneath.

**Axiom’s stance:** aim to be expert at writing SQL. Use a thin, explicit contract (protocols, parsers, binders) so that expertise shows up in the templates and the plans — not in framework configuration. If Hibernate is doing what an expert would do, an expert should still be able to read and own that SQL.

---

## What this is not

- Not “never use Hibernate.” Some codebases are committed; migration cost is real.  
- Not “raw JDBC everywhere with no discipline.” Unstructured string SQL and silent type coercion are how you get the same production pain without the ORM.  
- Not “type safety is bad.” Closed, minimal contracts beat open, reflection-driven registries for the problems Axiom is built to solve.

## What this is

A refusal to treat the database as an implementation detail of a Java object graph.

Rows as data. Queries as text you own. Types as a small set of carriers plus explicit casts where Postgres needs them. Validation at the perimeter. No reflection in the path that binds parameters or materializes results.

If those constraints feel extreme, the statements above are why they exist.

---

## Related

- [Axiom Type Bridging Guide](axiom-type-bridging-guide.md) — when `::cast` is required  
- [Axiom vs the Java Persistence Landscape](axiom-vs-java-persistence-landscape.md) — tool-by-tool matrix  
- [Axiom 1.0.0 → 2.0.0 Changelog](changelog-1.0.0-to-2.0.0.md) — byte-native parsers, removed `Dop.toJson`, dual-engine contract  
- Live demo: [axiom-strike-jdbc](https://github.com/ensemblu-corp/axiom-strike-jdbc)
