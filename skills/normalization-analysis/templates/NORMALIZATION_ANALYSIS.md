# Normalization Analysis

Append one dated section per analysis run, **most recent first**. Never overwrite a previous section — how a schema drifted is itself evidence.

**This document is analysis only.** Nothing here has been applied. A proposal becomes real work only when a human operator routes it through `solution-architect` → `senior-developer`.

---

## Analysis: `<date>` — `<schema subset analyzed>`

**Scope:** `<tables>`
**Sources read:** `prisma/schema.prisma`, `docs/architecture/<ARCHITECTURE file(s)>`, `prisma/migrations/<relevant>`
**Invoked by:** `<standalone / logic-simplification pass N — docs/refactor/SIMPLIFICATION_LOG.md § <pass>>`
**Implementation Log entry:** `docs/implementation/IMPLEMENTATION_LOG.md` § Pass `<n>`

### Summary

| Table | Highest normal form satisfied | Violating dependency | Classification |
|---|---|---|---|
| `<Table>` | `<1NF / 2NF / 3NF / BCNF>` | `<X → Y, or —>` | `<defect / deliberate-documented / deliberate-undocumented / unclear>` |

---

### Table: `<Name>`

**Attributes:** `<col: type, ...>`
**Candidate keys:** `<minimal determining attribute sets>`
**Primary key:** `<as declared>`
**Unique constraints / indexes:** `<as declared>`

| Functional dependency | Justification |
|---|---|
| `<X → Y>` | `<domain rule, citing ARCHITECTURE.md § or the business rule>` |

**Normal form results**

| Form | Result | Reason |
|---|---|---|
| 1NF | `<✅ / ❌>` | `<atomicity of each attribute; JSON columns judged on whether their structure is queried>` |
| 2NF | `<✅ / ❌ / n/a — single-attribute key>` | `<partial dependency, if any>` |
| 3NF | `<✅ / ❌>` | `<transitive dependency, if any>` |
| BCNF | `<✅ / ❌>` | `<determinant that is not a candidate key, if any>` |

**Highest form satisfied:** `<form>`

**Anomalies enabled** (concrete sequences, not categories)

- *Insert:* `<what fact cannot be recorded without inventing unrelated data>`
- *Update:* `<the exact update, and which columns then disagree; name the code path that would do it>`
- *Delete:* `<what deleting a row silently destroys>`
- *Reachable today?* `<yes — via file:line / no — no current write path, but only until a second one is written>`

**Classification:** `<defect / deliberate-documented / deliberate-undocumented / unclear>`
**Basis:** `<architecture doc reference, migration that introduced it, or "no record found">`

---

### Ranked Proposals (defects only — nothing applied)

| # | Table | Violation | Anomaly enabled | Proposed structure | Cheaper constraint-only fix? | Migration risk | Blast radius |
|---|---|---|---|---|---|---|---|
| `<n>` | `<table>` | `<form + dependency>` | `<concrete sequence>` | `<split / extract / drop redundant column>` | `<unique index or check that closes it, or —>` | `<low/med/high — backfill? destructive? reversible?>` | `<code paths/queries affected>` |

### Open Questions

| ID | Question | Status | Resolution |
|---|---|---|---|
| NQ-`<n>` | `<usually: is this column intentional, and what is it for?>` | Open | — |

### Handoff

- Presented to human operator on `<date>`, with explicit statement that **nothing was changed**
- Operator decision: `<which proposals, if any, become stories via solution-architect>`
