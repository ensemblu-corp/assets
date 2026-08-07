# Axiom Type Bridging: When You Must Cast, and When You Must Not

*A working reference for the JSON → Java → `AxiomProtocol` → Postgres pipeline, covering `axiom-spec`, `axiom-warp-jdbc`, and `axiom-warp-reactive`.*

---

## 1. The core mechanism, in one paragraph

JSON has 5 types. Postgres has 40+. Axiom does not try to close that gap in Java. It defines **7 carriers** in `AxiomProtocol` (`STRING`, `INTEGER`, `LONG`, `DOUBLE`, `BOOLEAN`, `TIMESTAMP`, and the catch-all `OPAQUE`). `SqlParser.forge()` turns your `:java.name` placeholders into bare `?` (or `$1, $2...` via `Dialect` in the reactive engine), leaving every other character in your SQL template — including any `::cast` you wrote — completely untouched. The binder (`JdbcBinder` / `ReactiveBinder`) sends the Java value across the wire with a generic type tag. **The real type resolution happens on the Postgres side, at bind time, against the actual destination column or the explicit `::cast` you wrote.** Java never tries to model Postgres's type catalog — it delegates that job to Postgres itself.

---

## 2. Why this works at all: Postgres's coercion graph

Postgres doesn't require the parameter's declared type to exactly match the column's type. If the two types belong to the same **implicit/assignment-cast family**, Postgres coerces automatically and no cast is needed. If they don't, Postgres will reject the bind (or silently produce the wrong value) unless you force it with `::type`.

### Families that coerce for free (no `::cast` needed)

| Carrier | Covers these Postgres types "for free" | Why |
|---|---|---|
| `STRING` | `varchar(n)`, `text`, `char(n)`/`bpchar`, `name`, domains over any of these | All mutually interchangeable via built-in casts. Length/padding rules (`varchar(50)` overflow, `char(10)` padding) are enforced by Postgres at assignment — not by Java. |
| `INTEGER` / `LONG` / `DOUBLE` | `int2`, `int4`, `int8`, `numeric`, `real`, `double precision` | See §2a — it's a strict hierarchy, not a flat family. |
| `TIMESTAMP` | `date`, `timestamp`, `timestamptz` | Postgres's temporal coercion graph connects all three without a cast error — **but see §4, this one has a real caveat.** |
| `BOOLEAN` | `bool` | Family of one — Postgres has no sibling boolean-ish type, so there's nothing to "cover for free," it's just a direct match. |

**Rule of thumb:** if the DB column's type is a plain member of one of these families, **do not write a `::cast`.** It's unnecessary noise — Postgres already resolves it, and the `AxiomProtocol` carrier already routes correctly.

### 2a. The numeric family is a hierarchy, not a flat family — and it has one real precision caveat

Unlike the string family (fully interchangeable both directions, no information loss), the numeric family is a **strict linear chain**, confirmed against Postgres's own cast catalog design:

```
int2 → int4 → int8 → numeric → real → double precision
```

- **Widening (moving right)** is an **implicit** cast — safe everywhere, including bare expressions (e.g. `WHERE` comparisons).
- **Narrowing (moving left)** is an **assignment-only** cast — it applies automatically when binding into an `INSERT`/`UPDATE` column target (which is what Axiom's binders do), but would need an explicit cast in a bare expression. Practically: binding a `LONG` into an `int4` column, or a `DOUBLE` into a `numeric` column, works with no `::cast` — but it's an assignment-context free ride, not a symmetric equivalence.
- Range errors on narrowing (`smallint out of range`, etc.) are thrown by Postgres at bind time, not caught by Java — same as before.

**Real caveat — `real`/`double precision` vs `numeric`:** `real`/`double precision` are approximate binary floating-point; `numeric` is exact decimal. The cast between them is legal and throws no SQL error, but it crosses a genuine representation boundary. A `double` carrying accumulated binary floating-point rounding error gets written into an exact-decimal `numeric` column artifacts and all — Postgres's cast catalog has no way to catch that, because nothing about it is actually illegal. If a column needs exact decimal semantics (money, precise quantities), be deliberate about using `BigDecimal` on the Java side and verify the value going in, rather than trusting the `DOUBLE` carrier by default.

---

## 3. Where you MUST cast: everything routed through `OPAQUE`

`OPAQUE` binds as a plain string (`bindString` → `setString`/`addString`). Postgres does **not** offer free implicit/assignment coercion from a bare string into these targets in the general case. If you skip the cast, you'll get errors like `column "x" is of type jsonb but expression is of type character varying`, or worse — no error, wrong result.

**Always write an explicit `::type` next to the placeholder for:**

- `jsonb` / `json` → `:java.payload::jsonb`
- `uuid` → `:java.id::uuid`
- enums (any custom enum type) → `:java.status::order_status`
- arrays → `:java.tags::text[]`, `:java.ids::int[]`
- `interval` — **not** covered by the `TIMESTAMP` carrier's coercion family, despite being temporal-ish → `:java.duration::interval`
- `numeric(p,s)` with a specific precision/scale you want enforced deliberately at the boundary, if you want the error surfaced clearly rather than relying on implicit narrowing
- PostGIS / custom domain types / any extension type
- Anything else that isn't a plain member of the five families in §2

**Template pattern:**
```sql
INSERT INTO events(id, payload, tags, status)
VALUES (:java.id::uuid, :java.payload::jsonb, :java.tags::text[], :java.status::order_status)
```

### Null values through `OPAQUE`
- **JDBC path:** `JdbcBinder.bindNull()` already handles this correctly — `OPAQUE` nulls bind as `Types.OTHER`, not `VARCHAR`, so Postgres can infer the type instead of rejecting a mismatched null. You don't need to do anything extra here.
- **Reactive path:** `ReactiveBinder.bindNull()` sends every null as untyped (`tuple.addValue(null)`) regardless of protocol — also fine, just via a different mechanism (Vert.x sends an untyped null on the wire by default).

---

## 4. The one carrier that isn't as clean as it looks: `TIMESTAMP`

`date` / `timestamp` / `timestamptz` don't throw cast errors against each other — but "no error" isn't the same as "correct":

- **`timestamptz` — safe, no caveat.** It stores an absolute instant. Both `JdbcBinder` (`java.sql.Timestamp`, built from epoch millis) and `ReactiveBinder` (`Instant`) represent the same unambiguous point in time. No timezone has to be *guessed* on write. Treat `TIMESTAMP` as fully safe here.
- **`timestamp` (naive, no time zone) — not actually safe.** Writing an absolute instant into a naive column requires picking a timezone to render the wall-clock digits. `JdbcBinder`'s `setTimestamp()` (no explicit `Calendar`) falls back to the **JVM's default timezone**, silently. This is a real, currently-existing divergence between the JDBC and reactive engines and a genuine footgun.

**Guidance:**
- If your schema only uses `timestamptz` (the standard Postgres recommendation), you're fully covered — no cast, no special handling needed.
- If any column is a naive `timestamp`, **do not trust the default binder behavior.** Either avoid naive `timestamp` columns entirely, or explicitly control the timezone used at the JDBC layer (e.g. `setTimestamp(i, ts, Calendar.getInstance(TimeZone.getTimeZone("UTC")))`) rather than relying on `AxiomBinder`'s current implementation as-is.

---

## 5. Quick reference

| Situation | Cast needed? |
|---|---|
| Target column is `text`/`varchar`/`char`/`name` | ❌ No |
| Target column is `int2`/`int4`/`int8`/`numeric`/`real`/`double precision` | ❌ No cast needed (assignment-cast covers it) — but see §2a if the value is crossing the `real`/`double precision` ↔ `numeric` boundary and needs exact decimal precision |
| Target column is `bool` | ❌ No |
| Target column is `timestamptz` | ❌ No |
| Target column is naive `timestamp` | ⚠️ No cast needed syntactically, but verify timezone handling manually — don't trust it blindly |
| Target column is `jsonb` / `json` | ✅ Yes — `::jsonb` / `::json` |
| Target column is `uuid` | ✅ Yes — `::uuid` |
| Target column is a custom enum | ✅ Yes — `::your_enum_type` |
| Target column is an array type | ✅ Yes — `::type[]` |
| Target column is `interval` | ✅ Yes — `::interval` |
| Target column is any other extension/custom/domain type | ✅ Yes |

---

## 6. Portability note

This entire design is **Postgres-specific**, not general SQL:

1. `::type` cast syntax doesn't exist in MySQL, SQL Server, or Oracle — they need `CAST(value AS type)` or vendor-specific equivalents.
2. The "free coercion families" in §2 are Postgres's own cast catalog. Other engines have different (and sometimes more dangerous — e.g. MySQL's non-strict-mode silent truncation) coercion rules.
3. `Dialect` in `axiom-warp-reactive` currently only has a real implementation for `POSTGRES`; `GENERIC` is a no-op pass-through (`sql -> sql`) with no evidence the coercion assumptions above have been re-validated for any other database.

**If Axiom ever targets a database other than Postgres, every rule in this document needs to be re-derived from that database's own cast catalog — none of it transfers automatically.**
