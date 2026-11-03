---
name: logic-simplification
description: 'Use when: backend logic (API routes, services, lib modules, data access) has accumulated complexity from compounding decisions by multiple people over time — duplicated implementations of the same rule, dead or unreachable code, unused exports/params/config/dependencies, speculative generality built for cases that never arrived, needless indirection, and inefficient patterns (N+1 queries, repeated work, over-fetching). Applies KISS and YAGNI to remove that weight WITHOUT changing behavior and WITHOUT touching integration/e2e tests, which are the contract. Human-in-the-loop: proposes a ranked simplification plan and gets explicit approval before editing, raises every ambiguity as a tracked Open Question, uses the version-control skill for branch/commits/PR, appends a pass entry to docs/refactor/SIMPLIFICATION_LOG.md, and imports the normalization-analysis skill whenever the database schema is in scope.'
---

# Logic Simplification Skill

A large codebase does not become complicated because anyone made a bad decision. It becomes complicated because many people each made a locally reasonable decision, at different times, without seeing each other's. The result is a system that is harder than the problem it solves: two functions that compute the same rule slightly differently, a branch that can no longer be reached, an abstraction built for a second implementation that never arrived, a loop that queries per row.

This skill removes that accumulated weight — and nothing else. It is a subtractive discipline: the system should do exactly what it did before, with less code, fewer paths, and less work per request.

## When to Use

- A module, route family, or service layer has grown hard to reason about
- The same rule (validation, formatting, authorization, scoring) appears to be implemented in more than one place
- Suspected dead code, unused exports/params, or config that nothing reads
- A visible inefficiency: queries inside loops, refetching the same data, work done and thrown away
- An abstraction (interface, factory, generic helper, layer) that only ever has one caller or one implementation
- Before extending a module, to avoid extending the wrong shape
- Schema-level concerns (redundant columns, update anomalies, duplicated facts) → import **normalization-analysis** (see below)

## When NOT to Use

- Behavior is wrong and needs fixing → that is a bug fix, and belongs to `senior-developer` against an approved story
- New capability is wanted → `requirement-analysis` → `solution-architect` → `senior-developer`
- Frontend page structure/componentization → **component-refactor**
- If simplification turns out to require a behavior change, **stop**, raise it as an Open Question, and let the human operator decide whether to split a separate story. Never smuggle a behavior change into a simplification pass.

## Communication Style Is Independent of This Procedure

This skill defines WHAT to do and WHAT goes in the documents it writes (`SIMPLIFICATION_LOG.md` entries, findings tables, checklists). It does not define HOW to talk to the human operator. If a compressed communication-style skill (e.g. **caveman**) is active, it governs chat prose for every step — findings, plan proposals, Open Question call-outs, approval requests. The carve-outs are: (1) content written into the documents/templates this skill produces, and (2) commit/PR/merge text per **version-control**'s Message Language rule.

## The Two Hard Rules

### 1. Functionality does not change, and the test contract is untouchable

- **No integration or e2e test in `e2e/` may be edited, retargeted, weakened, skipped, or deleted.** They encode the behavior that was signed off. If one fails after a simplification, the simplification is wrong — revert it. If a spec genuinely appears to assert an implementation detail that must move, that is an Open Question for the human operator, never a unilateral edit.
- Unit tests in `tests/` may only be edited when their *subject* was deliberately removed or merged as part of the approved plan, and every such edit must be listed individually in the pass entry with its justification. A unit test may never be relaxed to make a simplification pass.
- Externally observable behavior is preserved exactly: API request/response shapes and status codes, error messages and error types, ordering, pagination, timing-sensitive semantics, side effects (writes, emails, LLM/OCR calls), and logging that anything depends on.
- New tests may be *added* — in particular, characterization tests that pin down behavior before you touch it (step 3).

### 2. Never assume

Do not guess whether something is truly unused, whether two near-identical functions are meant to be identical, whether a branch is genuinely unreachable, or whether an unused export is a public surface someone outside this repo depends on. Verify with evidence (step 2), and where evidence is not conclusive, raise an Open Question (`SQ-` prefix, step 7) and leave that item alone in this pass.

## The Simplification Lenses

Sweep the scoped code through each lens. Every finding must cite `file:line` evidence and name the lens.

| Lens | Looking for | Typical evidence |
|---|---|---|
| **Duplication** | The same rule/computation/query implemented more than once | Two functions with the same semantics; copy-pasted validation; parallel API routes doing one thing |
| **Dead code** | Code that cannot execute | Unreachable branches, conditions that are constant, `catch` blocks for impossible errors, feature flags permanently one value |
| **Unused** | Code nothing references | Unexported-and-uncalled functions, unused exports, unused params, unused deps in `package.json`, env vars nothing reads, orphaned files |
| **Speculative generality (YAGNI)** | Flexibility built for a case that never came | Interfaces/abstract types with one implementation, options/params only ever passed one value, factories/registries with one entry, hooks nothing subscribes to, generic helpers with one caller |
| **Needless indirection (KISS)** | Layers that only forward | Wrappers that just delegate, single-use helpers that obscure rather than name, config indirection for a constant, deep chains where a direct call would read better |
| **Complexity** | Logic harder than the rule it encodes | Deep nesting, long boolean chains, repeated ad-hoc parsing, functions doing three jobs, error handling scattered across layers |
| **Inefficiency** | Work that need not happen | Queries inside loops (N+1), refetching the same data per request, over-fetching columns/relations, work computed then discarded, sequential awaits that could be one round trip, missing indexes on filtered/sorted columns |
| **Drift** | Compounding decisions that disagree | Two error-handling conventions, two ways to read the session, two date/format helpers, a pattern half-migrated |
| **Schema** | Structural redundancy in the data model | Duplicated facts across tables, multi-valued columns, partial/transitive dependencies → hand off to **normalization-analysis** |

## Imported Subskill: normalization-analysis

Whenever the pass touches `prisma/schema.prisma` or the reasoning behind a finding is "this data is stored redundantly / can go out of sync", **invoke the [normalization-analysis](../normalization-analysis/SKILL.md) skill** for that part of the work. It performs the 1NF → 2NF → 3NF → BCNF check, classifies which anomalies are real risks versus deliberate denormalization, and produces the schema section of this pass's log entry.

Its findings are **analysis only by default.** A schema or migration change is never bundled into a simplification pass — it is proposed to the human operator, who decides whether it becomes a separate story through `solution-architect`. This pass may only act on schema findings if the operator explicitly approves that in the plan.

## Procedure

### 1. Scope, baseline, branch

Pick **one cohesive area** per pass (one route family, one service, one lib module). Whole-codebase passes produce unreviewable diffs and destroy the test signal.

Baseline first:
- `git status` must be clean
- Run the full unit suite, the relevant e2e specs, lint and typecheck, and **record the counts**. A red baseline means you cannot prove preservation — surface it and stop.
- Note current behavior costs where relevant (query counts, obvious hot paths) so efficiency claims can be substantiated rather than asserted.

Then use **version-control** to draft and get approval for a `refactor/<area>-simplification` branch.

### 2. Sweep and gather evidence

Run every lens over the scoped code. For each candidate, gather evidence before proposing anything:

- **Unused/dead:** repo-wide search for every reference (including dynamic access, string-keyed lookups, route conventions, config files, seeds, scripts, and tests). Next.js file-system routing and Prisma-generated code mean "no import found" is not sufficient on its own — check the framework's conventions too.
- **Duplication:** confirm the two implementations are semantically identical, including edge and error cases. Near-identical is not identical; differences may be intentional.
- **Inefficiency:** confirm the cost is real (a query in a loop is only worth changing if the loop can be non-trivial), and note how you will verify the optimized version returns identical results.

Anything you cannot conclusively verify does not become a plan item — it becomes an Open Question.

### 3. Characterize before you touch

For any logic in scope whose behavior is not already pinned by an existing test, add characterization tests **first**, on the current code, and see them pass. Apply the **meaningful-tests** discipline when writing them (no vacuous or structural-only assertions, real boundary and negative cases) — a characterization test that cannot fail proves nothing. These tests are the safety net that makes the rest of the pass verifiable.

### 4. Propose the ranked plan — and get approval before editing

Present the human operator with the findings and the proposed plan **before writing any non-test code**:

| # | Finding | Lens | Evidence | Proposed action | Risk | Behavior change? |
|---|---|---|---|---|---|---|
| `<n>` | `<what is wrong>` | `<lens>` | `<file:line, references found>` | `<delete / merge / inline / flatten / batch query>` | `<low/med/high>` | None |

Rank by value-to-risk: high-confidence deletions first, structural merges last. State explicitly which findings you are **not** acting on and why (insufficient evidence, needs a decision, out of scope). Every row must say **Behavior change? None** — if any row cannot, it does not belong in this pass.

Wait for explicit approval. If the operator trims or reorders the plan, re-present the revised table; do not part-execute the old one.

### 5. Apply, one finding at a time

For each approved item, in the approved order:
1. Make the single change.
2. Run unit tests, the relevant e2e specs, lint, typecheck. Green before continuing.
3. Commit that item as its own logical unit (per **version-control**'s commit cadence) — one finding per commit keeps the trail bisectable if something surfaces later.

If an item turns red, revert it rather than layering fixes on top, and record it in the log as attempted-and-reverted with the reason.

Prefer deletion to abstraction. The best outcome of a simplification pass is less code, not differently-arranged code.

### 6. Verify preservation

After the last item:
- Full unit suite, full e2e suite, lint, typecheck — all green, with counts matching or exceeding the baseline
- Confirm **zero** modifications to `e2e/`; list any `tests/` edits individually with justification
- For each efficiency change, state the measured or counted before/after (e.g. "N+1 → 1 query for a 20-row list"), not a claim of "faster"
- Diff review: confirm every hunk maps to an approved plan item; anything else is scope creep and must be reverted or explicitly approved

### 7. Log Open Questions

| ID | Question | Status | Resolution |
|---|---|---|---|
| SQ-`<n>` | `<the exact ambiguity>` | Open | — |

Mark `Answered` when the operator responds (copy their exact answer into **Resolution**), and `Resolved` once the code reflects it. Never delete an Open Question.

### 8. Write the SIMPLIFICATION_LOG.md pass entry

Target file: `docs/refactor/SIMPLIFICATION_LOG.md`, created from [templates/SIMPLIFICATION_LOG.md](./templates/SIMPLIFICATION_LOG.md) if missing. Append one entry per pass (never edit a previous pass's entry) containing: scope, baseline and after test evidence, the full findings table including rejected/deferred items, what was deleted with line counts, efficiency before/after, any schema findings from **normalization-analysis**, Open Questions, and version-control actions.

This file is the **detail record** for the pass. Every other document points at it rather than repeating it.

### 9. Append a pointer pass to the Implementation Log

`docs/implementation/IMPLEMENTATION_LOG.md` is the central narrative record of everything delivered to this codebase. A simplification pass deletes and restructures real code, so it must appear there — otherwise the history shows code vanishing with no account of why, which is exactly the condition that produced the complexity this skill removes. Append a numbered entry continuing the existing numbering and matching the surrounding style, kept deliberately **short** and pointing outward:

```
## Pass <n>: <date> — Logic simplification: <area>

**Type:** Simplification pass — KISS/YAGNI cleanup, no behavior change, no new story.
**Branch:** `<branch-name>`
**Detail:** `docs/refactor/SIMPLIFICATION_LOG.md` § Pass `<date>` — full findings table (including
deferred and reverted items), evidence per finding, net effect, and preservation evidence.
**Traces to:** `<the modules simplified>`, `docs/architecture/<ARCHITECTURE file>` § `<decisions the
code implements>`, `docs/design/DESIGN.md` § US-`<n>` (stories whose behavior had to be preserved)
**Headline:** `<n>` findings applied of `<n>` found — `<n>` lines removed, `<n>` duplicated
implementations collapsed, `<n>` dead/unused symbols deleted, `<efficiency change with before/after
counts>`. Behavior unchanged — `<n>` unit and `<n>` e2e tests green before and after; no `e2e/` spec
modified; `<n>` characterization tests added.
**Schema:** `<none in scope, or: analyzed via normalization-analysis —
docs/architecture/NORMALIZATION_ANALYSIS.md § <dated section>; nothing applied>`
**Open Questions:** SQ-`<n>` `<one line each, with status>`
**Approval:** plan approved by `<operator>` on `<date>`; pass pending approval in `docs/APPROVAL_LOG.md`.
```

Cross-referencing rules — apply them in both directions, so any document can be entered from any other:
- **One canonical location per fact.** Findings, evidence, and Open Questions live in `SIMPLIFICATION_LOG.md`; schema analysis lives in `NORMALIZATION_ANALYSIS.md`. The Implementation Log carries the headline and the pointers, never a copy.
- Every reference names the **file and the section** (`docs/refactor/SIMPLIFICATION_LOG.md § Pass 2026-08-05`), not just the file.
- The `SIMPLIFICATION_LOG.md` entry carries the **reverse pointers**: its Implementation Log pass number, the branch, the PR, and any `NORMALIZATION_ANALYSIS.md` section produced by the imported subskill.
- When **normalization-analysis** runs inside this pass, it appears once as its own short Implementation Log entry *or* is folded into this pass's Schema line — not both. State which, so a reader never has to guess whether two entries describe one piece of work.
- If the pass produced any other artifact — a query-count measurement note, a benchmark script, a reference-search transcript backing a deletion — link it from the `SIMPLIFICATION_LOG.md` entry and let the Implementation Log reach it through that one hop.
- Once the PR number exists, fill it into both entries so the git history and the document trail resolve to each other.

### 10. Maintain the shared Approval Log entry

In `docs/APPROVAL_LOG.md`, per the pipeline convention:
- **Stage:** `Logic Simplification`, listing the area/pass covered
- **Related Doc(s)** must list `docs/refactor/SIMPLIFICATION_LOG.md` (with the pass date), `docs/implementation/IMPLEMENTATION_LOG.md` (with the pass number), any `docs/architecture/NORMALIZATION_ANALYSIS.md` section produced, and the source files touched
- Exactly one active entry at a time across the whole pipeline — update in place
- **Approved By**/**Approval Date** stay blank until a human operator explicitly approves the pass — never fill these in yourself

### 11. Open the PR via version-control

The PR description's **Testing** section must name the suites green before and after and state explicitly that no `e2e/` spec was modified, listing any `tests/` edits with their justification.

## Completion Checklist

- [ ] Baseline suites were observed **passing before** any change, from a clean working tree, with counts recorded
- [ ] Exactly one cohesive area was simplified in this pass
- [ ] Every lens was swept, and every finding cites `file:line` evidence
- [ ] "Unused"/"dead" claims were verified against framework conventions and dynamic references, not just import search
- [ ] Characterization tests were added and observed passing **before** touching unpinned logic
- [ ] The ranked plan was presented and **explicitly approved** before any non-test code was edited
- [ ] Every applied item was behavior-preserving; nothing new was added and no abstraction was introduced for a use that does not exist
- [ ] **No test in `e2e/` was edited, retargeted, weakened, skipped, or deleted**
- [ ] Any `tests/` edit is listed individually with justification tied to a deliberately removed subject
- [ ] API shapes, status codes, error messages, ordering, and side effects are unchanged
- [ ] Efficiency claims are backed by before/after counts or measurements, not assertions
- [ ] **normalization-analysis** was invoked if the schema was in scope, and its findings were left as proposals unless separately approved
- [ ] Full unit + e2e suites, lint, and typecheck are green after the pass
- [ ] Each finding was committed as its own logical unit via **version-control**, with approval before each state-changing git action
- [ ] `docs/refactor/SIMPLIFICATION_LOG.md` has a new pass entry, including deferred and reverted items
- [ ] `docs/implementation/IMPLEMENTATION_LOG.md` has a short numbered pass entry pointing at that detail record (and at any `NORMALIZATION_ANALYSIS.md` section) by file **and section**, with no facts duplicated between them
- [ ] Cross-references resolve in both directions — the simplification log names its Implementation Log pass, branch, PR, and any schema analysis section; any other artifact produced this pass is linked from it
- [ ] Nothing was assumed — every unverifiable item became a tracked `SQ-` Open Question instead of a plan item
- [ ] `docs/APPROVAL_LOG.md` has an active entry (Stage: Logic Simplification) pending human approval for this pass
