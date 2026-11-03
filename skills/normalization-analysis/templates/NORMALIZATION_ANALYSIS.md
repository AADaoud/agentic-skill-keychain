# Normalization Analysis

Append one dated section per analysis run, **most recent first**. Never overwrite a previous section — how a schema drifted is itself evidence.

**This document is analysis only.** Nothing here has been applied. A proposal becomes real work only when a human operator routes it through `solution-architect` → `senior-developer`.

---

## Analysis: `<date>` — `<schema subset analyzed>`

**Scope:** `<whole schema / named tables>`
**Database engine:** `<e.g. PostgreSQL 16 / MySQL 8.0 / SQLite 3.45 / SQL Server 2022>`
**Schema definition layer:** `<raw SQL / Prisma / Drizzle / TypeORM / Sequelize / SQLAlchemy / Django / Rails / Ecto / EF Core / Hibernate / other>`
**Sources read:**
- Model / schema definitions: `<e.g. prisma/schema.prisma, app/models/*.rb, models.py, db/schema/*.ts, schema.sql>`
- Migrations: `<e.g. db/migrate/, alembic/versions/, prisma/migrations/, V*__*.sql — relevant ones only>`
- Schema dump / introspection: `<e.g. db/structure.sql, pg_dump --schema-only output, or "none available">`
- Architecture: `docs/architecture/<ARCHITECTURE file(s)>`

**Model-vs-database drift:** `<none found / description + NQ-<n> / not checked — no dump available>`
**Invoked by:** `<standalone / logic-simplification pass N — docs/refactor/SIMPLIFICATION_LOG.md § <pass>>`
**Implementation Log entry:** `docs/implementation/IMPLEMENTATION_LOG.md` § Pass `<n>`

### Summary

| Table | Highest normal form satisfied | Violating dependency | Classification |
|---|---|---|---|
| `<table name as in the database>` | `<1NF / 2NF / 3NF / BCNF>` | `<X → Y, or —>` | `<defect / deliberate-documented / deliberate-undocumented / unclear>` |

---

### Table: `<name as it exists in the database>`

**Defined in:** `<model file + migration(s) that shape this table>`
**ORM mapping notes:** `<1:1 with model <Name> / single-table inheritance for <A, B> / implicit join table generated for <A ↔ B> / embedded value object <X> flattened in / n/a — raw SQL>`
**Attributes:** `<col: type, ...>`
**Candidate keys:** `<minimal determining attribute sets — include natural keys, not just the surrogate id>`
**Primary key:** `<as declared>`
**Unique constraints / indexes:** `<as declared — include partial/expression indexes that exist only in migrations, and say where>`
**Other constraints:** `<foreign keys, CHECKs, NOT NULLs, triggers, generated columns relevant to the dependencies — note any defined only in raw SQL>`

| Functional dependency | Justification |
|---|---|
| `<X → Y>` | `<domain rule, citing ARCHITECTURE.md § or the business rule>` |

**Normal form results**

| Form | Result | Reason |
|---|---|---|
| 1NF | `<✅ / ❌>` | `<atomicity of each attribute; JSON/array/hstore/XML columns judged on whether their internal structure is queried, filtered, indexed, or constrained — cite the query or index if so>` |
| 2NF | `<✅ / ❌ / n/a — single-attribute key>` | `<partial dependency, if any>` |
| 3NF | `<✅ / ❌>` | `<transitive dependency, if any>` |
| BCNF | `<✅ / ❌>` | `<determinant that is not a candidate key, if any>` |

**Highest form satisfied:** `<form>`

**Anomalies enabled** (concrete sequences, not categories)

- *Insert:* `<what fact cannot be recorded without inventing unrelated data>`
- *Update:* `<the exact update, and which columns then disagree; name the code path that would do it>`
- *Delete:* `<what deleting a row silently destroys>`
- *Reachable today?* `<yes — via file:line / no — no current write path, but only until a second one is written>`
- *Write paths considered:* `<ORM calls, raw SQL, bulk/batch updates bypassing hooks, background jobs, admin tools, other services sharing the database>`
- *Current guard:* `<database constraint/trigger (schema-enforced) / ORM hook, callback, or signal (application discipline only) / none>`

**Classification:** `<defect / deliberate-documented / deliberate-undocumented / unclear>`
**Basis:** `<architecture doc reference, migration that introduced it, or "no record found">`

---

### Ranked Proposals (defects only — nothing applied)

Ranked by anomaly reachability × data-loss severity, not by distance from BCNF.

| # | Table | Violation | Anomaly enabled | Proposed structure | Cheaper constraint-only fix? | Enforceable here? | Migration risk | Blast radius |
|---|---|---|---|---|---|---|---|---|
| `<n>` | `<table>` | `<form + dependency>` | `<concrete sequence>` | `<split / extract / drop redundant column>` | `<unique index, check, FK, or trigger that closes it, or —>` | `<engine enforces it? declarable in the ORM, or needs a raw-SQL migration?>` | `<low/med/high — backfill? destructive? reversible? table rewrite or long lock on this engine?>` | `<code paths, queries, and ORM models affected>` |

### Open Questions

| ID | Question | Status | Resolution |
|---|---|---|---|
| NQ-`<n>` | `<usually: is this column intentional, and what is it for? — or: model and database disagree on X; which is correct?>` | Open | — |

### Handoff

- Presented to human operator on `<date>`, with explicit statement that **nothing was changed**
- Operator decision: `<which proposals, if any, become stories via solution-architect>`
