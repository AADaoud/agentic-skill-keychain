---
name: normalization-analysis
description: 'Use when: a relational database schema needs to be checked for structural safety — whatever defines it (raw SQL DDL, migration files, or an ORM/schema layer such as Prisma, Drizzle, TypeORM, Sequelize, SQLAlchemy, Django, Rails/ActiveRecord, Ecto, Entity Framework, Hibernate/JPA) — or when a finding sounds like "this fact is stored in two places", "these columns can disagree", "this column holds a list", or a migration/model is being reviewed. Performs a formal 1NF → 2NF → 3NF → BCNF analysis per table: candidate keys, functional dependencies, the highest normal form satisfied, and the insert/update/delete anomalies any violation actually enables. Distinguishes real defects from deliberate, justified denormalization. ANALYSIS ONLY by default — it never writes a migration or edits the schema; it produces docs/architecture/NORMALIZATION_ANALYSIS.md plus ranked proposals for a human operator to route through solution-architect. Imported by the logic-simplification skill; also usable standalone.'
---

# Normalization Analysis Skill

Answers one question about the data model: **can this schema be put into a state that contradicts itself?**

Application code can be simplified with confidence only when the structures underneath it are sound. A table that stores the same fact twice will eventually store it two different ways — and no amount of careful service-layer logic prevents that permanently, because the constraint lives in the code's discipline rather than in the schema. This skill finds those structures, states precisely which anomaly each one enables, and separates genuine defects from denormalization that was chosen on purpose.

The analysis is about the **relational model**, not the tool that declares it. The same procedure applies whether the schema lives in `.sql` files, a migration framework, or ORM model definitions.

## When to Use

- Invoked by **logic-simplification** when a pass touches schema definitions or migrations, or a finding is about redundantly-stored data
- Reviewing a new model, table, or migration before it lands
- A bug is suspected to be a data-consistency problem ("these two numbers disagree")
- Periodically, as the schema grows past what any one person designed

## The One Hard Rule: Analysis Only

**This skill does not change anything.** It never edits a schema definition (DDL, ORM model, schema file), never writes or runs a migration, never touches data.

A normalization finding is a *proposal*. Schema changes are architecture decisions: they go to the human operator, who decides whether one becomes a story routed through **solution-architect** (which owns entity/invariant/boundary/concurrency modelling) and then **senior-developer**. A schema change is never bundled into a refactor or simplification pass.

Corollary: never assume a redundancy is a mistake. Denormalization is often deliberate — a cached count, a snapshot of a value at a point in time, a copy that must *not* follow later edits. Ask what the column is for before calling it a violation; if the intent is not documented and not evident, that is an Open Question, not a finding.

## Communication Style Is Independent of This Procedure

This skill defines WHAT to analyze and WHAT goes in `NORMALIZATION_ANALYSIS.md`. It does not define HOW to talk to the human operator. If a compressed communication-style skill (e.g. **caveman**) is active, it governs chat prose throughout — findings, anomaly explanations, proposals, Open Question call-outs. Document content and git artifact text stay in their own register.

## Procedure

### 1. Locate the schema's sources of truth

Before analyzing, identify where the schema is actually defined in this project. Common locations:

| Stack | Model / schema definition | Migrations |
|---|---|---|
| Raw SQL | `schema.sql`, `db/*.sql`, DDL dumps | Flyway (`V*__*.sql`), Liquibase changelogs, sqitch, dbmate, golang-migrate, plain numbered `.sql` files |
| Prisma | `prisma/schema.prisma` | `prisma/migrations/*/migration.sql` |
| Drizzle | `schema.ts` / `db/schema/*.ts` | `drizzle/*.sql` |
| TypeORM / MikroORM | `@Entity` classes | `migrations/*.ts` |
| Sequelize / Knex | model definitions | `migrations/*.js` |
| SQLAlchemy | declarative models | Alembic `versions/*.py` |
| Django | `models.py` | `*/migrations/*.py` |
| Rails / ActiveRecord | `db/schema.rb` or `db/structure.sql`, `app/models/*.rb` | `db/migrate/*.rb` |
| Ecto (Elixir) | `schema` modules | `priv/repo/migrations/*.exs` |
| Entity Framework | entity classes + `DbContext` / fluent config | `Migrations/*.cs` |
| Hibernate / JPA | `@Entity` classes, `orm.xml` | Flyway / Liquibase |

Rules for reading them:

- **Read both the model layer and the migrations.** ORMs often cannot express every constraint (check constraints, partial or expression unique indexes, exclusion constraints, triggers, generated columns), so those frequently exist only as raw SQL inside a migration. A constraint that closes an anomaly may be invisible from the model file alone.
- **Note declared-vs-actual drift.** If a schema dump (`structure.sql`, `pg_dump --schema-only`, introspection output) is available, compare it with the model layer. If they disagree, the database is the truth for what anomalies are reachable today; record the drift as an Open Question.
- **Note the engine and version** (PostgreSQL, MySQL/MariaDB, SQLite, SQL Server, Oracle…). It determines which constraint-based fixes are actually enforceable — e.g. older MySQL versions parse but ignore `CHECK`, SQLite cannot add most constraints to an existing table without a rebuild.
- If the project mixes stores (e.g. a relational DB plus a document store), scope this analysis to the relational schema. For document stores, embedded copies are normally deliberate denormalization; analyze them only if asked, using the same "can it contradict itself" question.

### 2. Scope

Analyze either the whole schema or a named subset of tables. State which, list the source files read (from step 1), and read the relevant approved sections of `docs/architecture/ARCHITECTURE*.md` — the architecture docs say what each entity is *supposed* to mean, and meaning is what determines a functional dependency. Migration history shows how the table arrived at its current shape, which often explains a redundancy.

Analyze **tables as they exist in the database**, not ORM classes. Where the mapping is not 1:1 — single-table inheritance, polymorphic associations, implicit many-to-many join tables generated by the ORM, embedded/owned value objects flattened into a parent table — describe the physical table and note how the ORM maps onto it.

### 3. Per table: keys and functional dependencies

For each table in scope:

```
### Table: <name as it exists in the database>

**Defined in:** <file(s): model + migration(s) that shape it>
**Attributes:** <col: type, ...>
**Candidate keys:** <all minimal attribute sets that uniquely determine a row>
**Primary key:** <as declared>
**Unique constraints / indexes:** <as declared, including partial/expression indexes found only in migrations>
**Other constraints:** <foreign keys, CHECKs, NOT NULLs, triggers relevant to dependencies>

| Functional dependency | Justification |
|---|---|
| `<X → Y>` | `<why this holds in the domain, citing ARCHITECTURE.md or the business rule>` |
```

Every dependency must be justified from the domain, not from the current data. "These two columns happen to correlate today" is not a functional dependency. A surrogate primary key (`id`) does not remove the need to identify natural candidate keys — the natural key is usually where missing unique constraints hide.

### 4. Test each normal form, in order

Evaluate strictly in sequence and stop reporting higher forms once one fails (a table that is not in 2NF is not in 3NF).

| Form | Test | Typical violation |
|---|---|---|
| **1NF** | Every attribute is atomic and single-valued; no repeating groups; each row unique | A JSON/JSONB, array, hstore, or XML column holding a list that is queried or filtered by its elements; `tag1, tag2, tag3` columns; a comma-joined string; a table with no enforced uniqueness on its natural key |
| **2NF** | In 1NF, and every non-key attribute depends on the **whole** candidate key | Only possible with composite keys: a join table also storing a fact about just one of its two sides |
| **3NF** | In 2NF, and no non-key attribute depends on another non-key attribute (no transitive dependency) | A denormalized copy of a related row's field, so it can go stale when the source changes |
| **BCNF** | Every determinant is a candidate key | An overlapping-candidate-key table where a non-key attribute determines part of a key — rarer, but exactly where subtle anomalies hide |

Note explicitly for each table: **the highest normal form it satisfies**, and if it stops short, **which dependency causes it**.

A structured column (JSON, array, etc.) is not automatically a 1NF violation. It is a violation when its internal structure is queried, filtered, joined, indexed, or constrained as if it were columns; it is legitimate when the value is opaque payload written and read whole. Check queries and indexes (e.g. GIN indexes, `->>`/`JSON_EXTRACT` in queries, ORM JSON-path filters) before deciding.

### 5. State the anomaly, concretely

A normal-form violation only matters through the anomaly it enables. For each violation, write the failure as a concrete sequence, not as a category name:

- **Insert anomaly:** what fact cannot be recorded without inventing unrelated data
- **Update anomaly:** what update, applied in the obvious way, leaves two rows disagreeing — name the exact columns and the code path that would do it
- **Delete anomaly:** what deleting a row silently destroys that nothing else records

Then judge reachability: can application code actually reach this state? Consider every write path — ORM calls, raw SQL, bulk/batch updates that bypass ORM hooks or callbacks, background jobs, admin tools, other services sharing the database. A violation that is impossible to trigger given the current write paths is real but low-priority — and, crucially, only stays unreachable as long as nobody writes a second write path. Say that rather than dismissing it. Guards that live only in ORM hooks/callbacks/signals count as application discipline, not schema enforcement.

### 6. Classify: defect or deliberate denormalization

| Classification | Meaning | Action |
|---|---|---|
| **Defect** | Redundancy with no stated purpose, reachable anomaly | Propose normalization |
| **Deliberate — documented** | Denormalized on purpose (cache, counter cache, snapshot, historical copy, materialized view), stated in `ARCHITECTURE.md` | Record and move on; note whether a constraint, trigger, or recompute path guards it |
| **Deliberate — undocumented** | Looks intentional, but no record of the decision | Open Question; propose documenting the intent rather than changing the schema |
| **Unclear** | Cannot determine intent from schema, architecture docs, or migration history | Open Question. Do not propose a change on a guess |

### 7. Rank the proposals

Only for items classified **Defect**:

| # | Table | Violation | Anomaly enabled | Proposed structure | Migration risk | Blast radius |
|---|---|---|---|---|---|---|
| `<n>` | `<table>` | `<form + the dependency>` | `<concrete sequence>` | `<split / extract table / add constraint / drop redundant column>` | `<low/med/high — backfill needed? destructive? reversible? table rewrite or lock on this engine?>` | `<code paths, queries, and ORM models that would need changing>` |

Rank by anomaly reachability × data-loss severity, not by how far the table is from BCNF. Note where a **constraint or unique index** would close the anomaly without restructuring — that is often the far cheaper correct answer, and should be offered alongside the structural proposal rather than after it. State whether the target engine enforces that constraint, and whether the project's ORM can declare it or it would need a raw-SQL migration.

### 8. Log Open Questions

| ID | Question | Status | Resolution |
|---|---|---|---|
| NQ-`<n>` | `<the exact ambiguity — usually "is this column intentional and why", or "model and database disagree">` | Open | — |

Mark `Answered` when the human operator responds (copy their exact answer into **Resolution**), and `Resolved` once the analysis document reflects it. Never delete an Open Question.

### 9. Write the analysis document

Target file: `docs/architecture/NORMALIZATION_ANALYSIS.md`, created from [templates/NORMALIZATION_ANALYSIS.md](./templates/NORMALIZATION_ANALYSIS.md) if missing. Append one dated analysis section per run (never overwrite a previous one — a schema's normalization history is evidence of how it drifted). Include the schema sources and engine identified, the per-table analysis, the anomaly statements, the classification table, the ranked proposals, and the Open Questions.

### 10. Cross-reference and hand off

- If invoked by **logic-simplification**, put a one-paragraph summary plus a link to this analysis section into that pass's `SIMPLIFICATION_LOG.md` **Schema Findings** section — the detail lives here, not there.
- Append a short entry to `docs/implementation/IMPLEMENTATION_LOG.md` (see below) recording that the analysis happened, its headline result, and where the detail lives.
- Present the ranked proposals to the human operator with an explicit statement that nothing has been changed, and ask whether any should become a story via **solution-architect**.

### 11. Implementation Log entry

`docs/implementation/IMPLEMENTATION_LOG.md` is the central narrative record of everything delivered. Append a numbered pass entry (continuing the existing numbering, most recent last, matching the surrounding style) that is deliberately **short** and points outward:

```
## Pass <n>: <date> — Normalization analysis: <schema subset>

**Type:** Analysis pass — no code, schema, or data was changed.
**Schema sources:** `<files read, e.g. db/schema.rb + db/migrate/>`; engine `<e.g. PostgreSQL 16>`
**Detail:** `docs/architecture/NORMALIZATION_ANALYSIS.md` § `<dated section>` — per-table keys,
functional dependencies, normal-form results, anomaly statements, and ranked proposals.
**Invoked by:** `<standalone, or logic-simplification pass N — docs/refactor/SIMPLIFICATION_LOG.md § <pass>>`
**Headline:** `<n>` tables analyzed; highest form satisfied per table `<summary>`; `<n>` defects,
`<n>` deliberate denormalizations, `<n>` unclear.
**Open Questions:** NQ-`<n>` `<one line each>`
**Proposed next:** `<which proposals, if any, the operator was asked to route through solution-architect>`
```

Never duplicate the full analysis into the implementation log — one canonical location per fact, with the log pointing to it.

### 12. Approval Log

If run standalone, add/update the single active entry in `docs/APPROVAL_LOG.md` with **Stage:** `Normalization Analysis`, Related Doc(s) pointing at `NORMALIZATION_ANALYSIS.md` and the Implementation Log pass. If invoked by another skill, that skill's entry covers this work — list this analysis in its Related Doc(s) rather than creating a second pending entry. **Approved By**/**Approval Date** stay blank until a human operator explicitly approves.

## Completion Checklist

- [ ] The schema's sources of truth (model layer, migrations, and any schema dump) and the database engine were identified
- [ ] Constraints that exist only in raw-SQL migrations were found, and any model-vs-database drift was recorded
- [ ] Scope was stated, and the relevant `ARCHITECTURE*.md` sections were read before analyzing
- [ ] Tables were analyzed as they physically exist, with ORM mapping quirks (inheritance, implicit join tables, embedded types) noted
- [ ] Every table in scope has candidate keys and domain-justified functional dependencies listed
- [ ] Normal forms were tested in order (1NF → 2NF → 3NF → BCNF), with the highest satisfied form stated per table
- [ ] Every violation names the exact dependency that causes it
- [ ] Every violation states a concrete insert/update/delete anomaly and whether current write paths (including ones that bypass the ORM) can reach it
- [ ] Every JSON/array/structured column was judged on whether its structure is actually queried, not assumed to violate 1NF
- [ ] Each finding is classified defect / deliberate-documented / deliberate-undocumented / unclear
- [ ] Cheaper constraint-based fixes were offered alongside structural proposals, with engine enforcement and ORM expressibility stated
- [ ] **Nothing was changed** — no schema edit, no migration written or run, no data touched
- [ ] Proposals are ranked by anomaly reachability and severity, with migration risk and blast radius stated
- [ ] Every intent-related uncertainty became a tracked `NQ-` Open Question instead of an assumed conclusion
- [ ] `docs/architecture/NORMALIZATION_ANALYSIS.md` has a new dated section; no previous section was overwritten
- [ ] `docs/implementation/IMPLEMENTATION_LOG.md` has a short pass entry cross-referencing the analysis document (and the invoking skill's log, if any)
- [ ] The human operator was told explicitly that nothing changed, and asked whether any proposal should become a story
