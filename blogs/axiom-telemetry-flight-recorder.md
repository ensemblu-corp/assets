# A Black Box for Every Result: `axiom-telemetry`

![Axiom](https://img.shields.io/badge/axiom--telemetry-3.0.0-blue)
![Java](https://img.shields.io/badge/Java-26-orange)
![Dependencies](https://img.shields.io/badge/depends%20on-axiom-informational)

*How a lock-free, time-windowed buffer turns "it failed" into "here is what happened in the 30 seconds before it failed". Every snippet is written against the 3.0.0 sources.*

---

## 1. The problem with failure messages

A failure message describes **the moment of failure**:

```
Axiom Native Shot Error: PostgreSQL execution failed with status: PGRES_FATAL_ERROR
  — Details: duplicate key value violates unique constraint "accounts_pkey"
```

That's accurate, but often not enough. The questions you actually ask during an incident are about **what came before**: what was the last batch that succeeded, how fast were writes arriving, did this follow a burst?

Most stacks answer those questions with an external system (a metrics pipeline, a tracing backend, a log aggregator), which you then have to correlate by timestamp. Axiom 2.x had an in-engine answer, but only inside the reactive database engine (`warp.withHistory(..)`). In 3.0 that idea becomes a standalone, 138-line module that works with **any** operation returning a `Result`.

---

## 2. Two classes, no dependencies beyond `axiom`

| Class | Role |
|---|---|
| `TemporalStreamBuffer<T>` | Time-bounded rolling buffer of successful events |
| `SovereignExecution` | Runs an action, records successes, decorates failures with a snapshot |

```xml
<dependency>
    <groupId>com.ensemblu</groupId>
    <artifactId>axiom-telemetry</artifactId>
    <version>3.0.0</version>
</dependency>
```

---

## 3. The buffer

```java
TemporalStreamBuffer<Order> recorder = TemporalStreamBuffer.ofWindowDuration(Duration.ofMinutes(5));
```

Internally it's an `AtomicReference<PersistentList<TimestampedEvent<T>>>`. Every write is a single `updateAndGet` that does three things:

1. appends the new `(Instant, event)` pair to a **transient** copy of the list,
2. **evicts** events from the head that are older than `now − window`,
3. **freezes** the result into a shared `Arena` and publishes it atomically.

Readers never lock. They read one immutable snapshot, and a concurrent writer can't change it under them. That's the property you want in the code path that runs while something is going wrong.

| Method | Behaviour |
|---|---|
| `logEvent(T)` | Record an event now |
| `instrument(Result<T>)` | Record the value only if the result succeeded; returns the same result |
| `streamHistory(Duration ago)` | `Result<PersistentList<T>>` of events newer than `now − ago` |
| `clear()` | Reset (useful between test waves) |
| `getWindowDuration()` | The configured window |

---

## 4. The execution wrapper

```java
Result<Order> placed = SovereignExecution.wieldWithTelemetry(
        recorder,
        Duration.ofSeconds(30),        // how far back to include in a failure snapshot
        () -> orders.place(command));
```

- **On success**, the value is logged into the buffer and the result is returned unchanged.
- **On failure**, the wrapper pulls `streamHistory(lookback)`, renders it with `Dop.toString`, and rewrites the failure message:

```
💥 AHE BREACH | PRE-FAILURE SNAPSHOT: [Order{id=1041,…}, Order{id=1042,…}, …]
---------------------------
RAW BREACH EVIDENCE: Transaction failed, rolled back: Axiom Native Dynamic Query Error: …
```

The original evidence is always kept. The snapshot is **added to** the message and never replaces it. If the buffer is `null` or unavailable, the snapshot reads `Buffer Unavailable` and the original failure still comes through.

---

## 5. Wiring it to a native driver

`axiom-telemetry` doesn't depend on any driver. It only needs a `Supplier<Result<T>>`. That makes the database integration a one-liner around `wield`:

```java
TemporalStreamBuffer<PersistentList<PersistentMap<String, Object>>> ledgerRecorder =
        TemporalStreamBuffer.ofWindowDuration(Duration.ofMinutes(5));

Result<PersistentList<PersistentMap<String, Object>>> rows = SovereignExecution.wieldWithTelemetry(
        ledgerRecorder,
        Duration.ofSeconds(30),
        () -> pg.wield(conn -> pg
                .dynamicQuery("SELECT * FROM ledger_transactions WHERE account_id = :java.id")
                .to(accountId).bind(PostgresType.UUID)
                .execute(conn)));
```

The same wrapper works around `SchemaGuard` validation, an HTTP call, or a pure domain computation, as long as it returns a `Result`.

---

## 6. Sizing and placement

| Concern | Guidance |
|---|---|
| **Window** | The upper bound on retained history. Eviction runs on every write, so memory is roughly *events per window × event size*. |
| **Lookback** | Should be ≤ the window. 10–60 s is usually enough to explain an incident. |
| **What to store** | Prefer **small summaries** (ids, counts, statuses) over full payloads for high-throughput streams. Store full rows only where the volume is low. |
| **How many buffers** | One per stream you want to reason about: per endpoint, per tenant, or per job. Under very high contention on a single buffer, shard. |
| **Readability** | Snapshots are rendered with `Dop.toString`, so events should be Axiom data types or have a meaningful `toString()`. |
| **Sensitive data** | The snapshot ends up in failure messages, which often end up in logs. Don't buffer secrets or personal data you wouldn't log. |

---

## 7. Testing with it

Because the buffer is a plain object, tests can assert on history directly:

```java
recorder.clear();

service.process(batchA);
service.process(batchB);

var lastSecond = recorder.streamHistory(Duration.ofSeconds(1)).getOrThrow();
assert lastSecond.size() == 2;
```

`clear()` between test waves gives each scenario a clean flight recorder.

---

## 8. What it isn't

- **Not a metrics system.** There are no counters, histograms or exporters. It answers "what happened just before this failure?" and nothing else.
- **Not persistent.** History lives in memory and dies with the process, by design. Ship failure messages to wherever your logs go.
- **Not a tracer.** There's no cross-process context propagation. Each buffer describes one process's view of one stream.

What it does is give every failure the context of what led up to it, with zero dependencies and no extra infrastructure to run.

---

## Related

- [Axiom 2.0.0 → 3.0.0: Ultimate Release Changelog](changelog-2.0.0-to-3.0.0.md)
- [Why Axiom 3 Talks to Databases Natively](axiom-3-native-drivers.md)
- [Project Axiom homepage](https://ensemblu-corp.github.io/assets/index.html)
