# Agentic Development Skills

This repository contains a set of skills for use in software engineering projects using a closed-loop agentic development paradigm.

Each skill is a folder containing a `SKILL.md` (and, where needed, a `templates/` folder). An agent loads a skill when its description matches the task in front of it. The skills hand work to each other through documents in the target project's `docs/` folder, and a human operator approves each stage before the next one builds on it.

## The loop

```
requirement-analysis ──► solution-architect ──► senior-developer ──► (back to requirements for the next change)
        ▲                       ▲                      │
        │                       │                      ├── meaningful-tests
        │                       │                      └── version-control
        │                       │
        │               normalization-analysis ◄── logic-simplification
        │                                              component-refactor
        │
   human operator approves every stage (docs/APPROVAL_LOG.md)
```

The loop is closed in two ways:

- **Nothing advances without a human.** Every stage records a pending entry in `docs/APPROVAL_LOG.md`. The next stage proceeds only once a human operator has explicitly approved it.
- **Findings flow back upstream instead of being fixed in place.** Analysis and refactoring skills never make architecture decisions on their own. A schema problem found during simplification becomes a proposal that the operator can route back through `solution-architect`, not an edit bundled into the refactor.

## Skills

| Skill | Role |
|---|---|
| [`requirement-analysis`](./skills/requirement-analysis/) | Turns a feature idea or change request into user stories with testable acceptance criteria, and surfaces the ambiguities as open questions rather than resolving them by guesswork. |
| [`solution-architect`](./skills/solution-architect/) | Turns approved requirements into architecture: entities, invariants, boundaries, concurrency behaviour, and technology choices tied to specific acceptance criteria. Owns all schema and data-model decisions. |
| [`senior-developer`](./skills/senior-developer/) | Implements approved architecture one story at a time and records each pass in `docs/implementation/IMPLEMENTATION_LOG.md`. |
| [`meaningful-tests`](./skills/meaningful-tests/) | Makes tests catch real bugs rather than just turn green: scenario design before test code, characterization tests for existing behaviour, boundary and rejection-path coverage, correct mocking, and a falsification check for every test. |
| [`logic-simplification`](./skills/logic-simplification/) | Simplifies application logic without changing behaviour, logged in `docs/refactor/SIMPLIFICATION_LOG.md`. Invokes `normalization-analysis` when a pass touches the schema. |
| [`normalization-analysis`](./skills/normalization-analysis/) | Analysis-only check of a relational schema (raw SQL or any ORM) from 1NF through BCNF. Reports the anomalies each violation enables, separates defects from deliberate denormalization, and ranks proposals for the operator. Never changes the schema. |
| [`component-refactor`](./skills/component-refactor/) | Used to simplify specific components, |
| [`version-control`](./skills/version-control/) | Defines rules for GitHub version control by agents. |
| [`ponytail`](./skills/ponytail/) | Adds YAGNI and Laziness to senior-developer behaviour. |
| [`caveman`](./skills/caveman/) | Compressed communication style for chat with the human operator. When active, it governs chat prose only; documents and git artifacts keep their own register. |

## Shared conventions

- **Open questions are tracked, never assumed away.** When intent is unclear, a skill logs a numbered open question with a status (`Open` → `Answered` → `Resolved`) instead of drawing a conclusion. Open questions are never deleted.
- **One canonical location per fact.** Detail lives in the document owned by the skill that produced it; other logs link to it rather than copying it.
- **History is append-only.** Analysis documents and logs gain new dated sections; earlier sections are never overwritten.
- **Communication style is separate from procedure.** A skill defines what to do and what goes in its documents; a style skill such as `caveman` defines how to talk to the operator.

## Project documents the skills produce

| Path | Written by |
|---|---|
| `docs/APPROVAL_LOG.md` | Every stage — the human approval gate |
| `docs/architecture/ARCHITECTURE*.md` | `solution-architect` |
| `docs/architecture/NORMALIZATION_ANALYSIS.md` | `normalization-analysis` |
| `docs/implementation/IMPLEMENTATION_LOG.md` | `senior-developer`, plus short pointer entries from analysis passes |
| `docs/refactor/SIMPLIFICATION_LOG.md` | `logic-simplification` |

## Using the skills

Copy the skill folders into wherever your agent loads skills from (for example `.claude/skills/` in a project, or upload them as custom skills). Each skill's `description` frontmatter says when it should fire; most can also be invoked by name.
