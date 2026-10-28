# Ensemblu Engineering Notes

Long-form articles on the design, migration paths and opinions behind **Project Axiom**.

> Current release: **Axiom 3.0.0** · Java 26 · [Homepage](https://ensemblu-corp.github.io/assets/index.html)

---

## Axiom 3.0

| Article | Type | What you'll get |
|---|---|---|
| [Axiom 2.0.0 → 3.0.0: Ultimate Release Changelog](changelog-2.0.0-to-3.0.0.md) | Release | Every breaking change, the new 10-module graph, a 10-step migration and a paste-ready checklist |
| [Why Axiom 3 Talks to Databases Natively](axiom-3-native-drivers.md) | Deep dive | Project Panama, five vendor C clients, one `Executioner` shape, contract-verified bulk ingest, and the honest trade-offs of no-pool connections |
| [A Black Box for Every Result: `axiom-telemetry`](axiom-telemetry-flight-recorder.md) | Deep dive | A lock-free, time-windowed flight recorder that attaches pre-failure context to any failed `Result` |

## Philosophy (version-independent)

| Article | Type | What you'll get |
|---|---|---|
| [Hard Truths About Java Persistence](axiom-persistence-hard-truths.md) | Opinion | Eight statements about ORMs, dynamic queries and SQL fluency |

## Axiom 2.x (archive)

| Article | Type | Notes |
|---|---|---|
| [Axiom vs. the Java Persistence Landscape](axiom-vs-java-persistence-landscape.md) | Comparison | Hibernate, EclipseLink, Spring Data, jOOQ, Querydsl, MyBatis, Spring JDBC, JDBI. The 2.x mechanics are superseded by native drivers in 3.0 |
| [Axiom Type Bridging Guide](axiom-type-bridging-guide.md) | Guide | The `AxiomProtocol` + `::cast` model. In 3.0, binds carry explicit engine types |
| [Axiom 1.0.0 → 2.0.0 Changelog](changelog-1.0.0-to-2.0.0.md) | Release | `String` → `byte[]` parsers, `Dop` as a final utility, `JsonEmitter` |

---

© Ensemblu Corp · [contact@ensemblu.com](mailto:contact@ensemblu.com)
