---
name: component-refactor
description: 'Use when: a page or screen in this repo is a static, frozen transcription of a design (a one-shot mockup-to-JSX dump, hard-coded copy/data, one giant file per route, markup duplicated across pages) and needs to become a reactive, composed UI. Breaks a page down into named, reusable components with explicit props/state/data boundaries, WITHOUT changing rendered output or user-visible behavior, verified against the existing Playwright e2e specs. Human-in-the-loop: proposes the component decomposition and gets explicit approval before touching code, raises every ambiguity as a tracked Open Question, uses the version-control skill for branch/commits/PR, and appends a pass entry to docs/refactor/UI_REFACTOR_LOG.md.'
---

# Component Refactor Skill

Turns a page that was written *as a picture* into a page that is *built out of parts*.

Much of this repo's UI began life in `frontend-designs/*.png` and was transcribed once, straight through, into a route file. That transcription is rigid: layout, copy, data shape and interaction are all fused into one component, so every subsequent change means editing a frozen representation of a design rather than composing behavior. This skill unfreezes a page — decomposing it into components with clear props, state, and data boundaries — while guaranteeing the rendered result and behavior are unchanged.

## When to Use

- A route file (e.g. `src/app/**/page.tsx`) has grown into one long block of markup that mixes layout, formatting, fetching, and interaction
- The same visual block (card, table row, badge, empty state, modal, form field) is copy-pasted across two or more pages
- Copy, option lists, or sample data are hard-coded inline where they should be props or data
- Conditional rendering has become nested ternaries or duplicated branches of near-identical markup
- Before building a new screen that would otherwise copy-paste an existing frozen page

## When NOT to Use

- The page needs a *behavior* or *requirement* change — that is `senior-developer` work against an approved story, not a refactor. Never smuggle new behavior into a refactor pass.
- The design itself is being reworked (new layout/visuals) — that goes back through `requirement-analysis`.
- If a refactor turns out to require a behavior change, stop, raise it as an Open Question, and let the human operator decide whether to split it into a separate story.

## Communication Style Is Independent of This Procedure

This skill defines WHAT to do and WHAT goes in the documents it writes (`UI_REFACTOR_LOG.md` entries, decomposition tables, checklists). It does not define HOW to talk to the human operator. If a compressed communication-style skill (e.g. **caveman**) is active for this session, it governs chat prose for every step — inventory findings, decomposition proposals, Open Question call-outs, approval requests — all the way through. The only carve-outs are: (1) content written into the documents/templates this skill produces, and (2) commit/PR/merge text per **version-control**'s Message Language rule.

## The Two Hard Rules

### 1. Behavior and rendered output do not change

A refactor pass is only correct if a user cannot tell it happened. That means:
- Same DOM structure and same classes where they are load-bearing for layout; same visible text, same order, same states (loading/empty/error/success)
- Same network calls, with the same payloads, in the same order
- Same routing, same form submission targets, same accessibility attributes (roles, labels, `aria-*`)
- **Existing e2e specs in `e2e/` are not edited.** They are the contract. If a spec fails after the refactor, the refactor is wrong — fix the code, not the spec. If a spec appears to be asserting on an implementation detail that genuinely must move, that is an Open Question for the human operator, never a unilateral edit.
- New tests may be *added* (e.g. unit tests for an extracted component); existing ones may not be weakened, retargeted, or deleted.

### 2. Never assume

If the decomposition requires a decision the code does not already answer — what a component should be named, whether two near-identical blocks are truly the same component or coincidentally similar, whether a hard-coded string is real copy or placeholder, where shared components belong — do not guess the "reasonable" option. Raise it as an Open Question (`UQ-` prefix, step 6) and continue with the parts of the refactor that do not depend on the answer.

## Procedure

### 1. Scope and branch

Pick **one page or one cohesive screen family** per pass (e.g. the admin interviews list, or the interview question flow). Do not refactor the whole app in one pass — small passes keep the diff reviewable and the e2e signal meaningful.

Record the baseline first:
- `git status` must be clean before starting
- Run the relevant e2e specs and unit tests and note that they pass **before** any change. A red baseline means you cannot prove you preserved behavior — surface it and stop.

Then use the **version-control** skill to draft and get approval for a `refactor/<page-slug>-components` branch.

### 2. Inventory the frozen page

Read the target file(s) end to end and produce an inventory — this is analysis only, no edits yet:

| Block | Lines | What it does | Repeated? | Frozen bits |
|---|---|---|---|---|
| `<visual block>` | `<file:line-range>` | `<layout / data fetch / interaction>` | `<yes: where else>` | `<hard-coded copy, magic numbers, inline data>` |

Also note, per page: what state exists and where it lives, what data is fetched and by whom, and which parts are server components vs. client components (`"use client"` boundaries).

### 3. Propose the decomposition — and get approval before editing

Present the human operator with the proposed component breakdown **before writing any code**:

| Proposed component | Responsibility | Props (in) | Events/state (out) | Replaces | Client or server |
|---|---|---|---|---|---|
| `<Name>` | `<single responsibility, one sentence>` | `<typed props>` | `<callbacks / local state>` | `<file:line-range(s)>` | `<client/server>` |

Apply these decomposition principles, and say which one motivated each split:
- **One reason to change.** A component that renders a row and also fetches the list is two components.
- **Props down, events up.** Data flows in as typed props; interactions flow out as callbacks. No component reaches around its parent for data it was not given.
- **Push state down, lift only what is shared.** State lives in the lowest component that needs it; it is lifted only when a sibling genuinely needs it too.
- **Keep the client boundary thin.** Leave data fetching in server components; make only the interactive leaf a client component rather than marking a whole page `"use client"`.
- **Extract on the third use, not the first.** Two similar blocks may be a coincidence; three is a component. Do not build abstractions for uses that do not exist yet.
- **Name for the domain, not the shape.** `InterviewSummaryCard`, not `WhiteRoundedBox`.
- **Data and copy become inputs.** Hard-coded strings, option lists and sample values become props, constants, or fetched data — but their *values* do not change in this pass.

Wait for explicit approval of the decomposition. If the operator changes it, re-present the revised table; do not part-execute the old one.

### 4. Extract, one component at a time

For each approved component, in dependency order (leaves first):
1. Create the component file with an explicit props type — no `any`, no implicit props.
2. Replace the original block with the component, passing through exactly what was there.
3. Run lint, typecheck, unit tests, and the page's e2e spec. Green before moving to the next extraction.
4. Commit that extraction as its own logical unit (per **version-control**'s commit cadence) — one component per commit keeps a bisectable trail if a visual regression appears later.

Never batch several extractions into one unverified edit. If an extraction turns red, revert it rather than layering fixes on top.

### 5. Verify the page is genuinely unchanged

After the last extraction:
- Full unit suite green, full relevant e2e suite green, lint and typecheck clean
- Walk the page manually (or via the **run** skill) through every state the inventory listed: loading, empty, error, populated, and each interactive path
- Diff review: confirm the change is pure movement — flag to the operator any place where the diff shows a changed string, changed class, changed condition, or changed call, and justify or revert it

### 6. Log Open Questions

Any ambiguity encountered becomes a row in the pass entry:

| ID | Question | Status | Resolution |
|---|---|---|---|
| UQ-`<n>` | `<the exact ambiguity>` | Open | — |

Mark `Answered` the moment the operator responds (copy their exact answer into **Resolution**), and `Resolved` once the code reflects it. Never delete an Open Question — it is part of the traceable history.

### 7. Write the UI_REFACTOR_LOG.md pass entry

Target file: `docs/refactor/UI_REFACTOR_LOG.md`, created from [templates/UI_REFACTOR_LOG.md](./templates/UI_REFACTOR_LOG.md) if missing. Append one entry per pass (never edit a previous pass's entry) containing: the page refactored, the before/after structure, the approved decomposition table with what each component replaced, the behavior-preservation evidence (which specs ran, before and after), Open Questions, anything deliberately left alone, and the version-control actions taken.

This file is the **detail record**. It is where the full decomposition table and evidence live, and every other document points at it rather than repeating it.

### 8. Append a pointer pass to the Implementation Log

`docs/implementation/IMPLEMENTATION_LOG.md` is the central narrative record of everything delivered to this codebase, and a refactor pass belongs in it like any other pass — otherwise the history shows the code changing with no account of why. Append a numbered entry continuing the existing numbering and matching the surrounding style, kept deliberately **short** and pointing outward at the detail record:

```
## Pass <n>: <date> — UI refactor: <page or screen family>

**Type:** Refactor pass — no new user story, no behavior change.
**Branch:** `<branch-name>`
**Detail:** `docs/refactor/UI_REFACTOR_LOG.md` § Pass `<date>` — inventory, approved decomposition
table, per-component responsibilities, and behavior-preservation evidence.
**Traces to:** `docs/design/DESIGN.md` § US-`<n>` (the story this page implements),
`frontend-designs/<image>.png`, `<src/app/.../page.tsx>`
**Headline:** `<n>` components extracted from `<n>` lines in `<n>` file(s); `<n>` duplicated blocks
collapsed. Behavior unchanged — `<n>` e2e specs and `<n>` unit tests green before and after,
no test modified.
**Open Questions:** UQ-`<n>` `<one line each, with status>`
**Approval:** decomposition approved by `<operator>` on `<date>`; pass pending approval in
`docs/APPROVAL_LOG.md`.
```

Cross-referencing rules — apply them in both directions, so any document can be entered from any other:
- **One canonical location per fact.** The decomposition table, the evidence and the Open Questions live in `UI_REFACTOR_LOG.md`. The Implementation Log carries the headline and the pointer, never a copy.
- Every reference names the **file and the section** (`docs/refactor/UI_REFACTOR_LOG.md § Pass 2026-08-05`), not just the file — a reader landing in a long append-only log needs the anchor.
- The `UI_REFACTOR_LOG.md` entry carries the **reverse pointer**: its Implementation Log pass number, the branch, the PR, and the `DESIGN.md` story the page belongs to.
- If the pass produced any other artifact — a decomposition sketch, a screenshot set, a note file — link it from the `UI_REFACTOR_LOG.md` entry, and let the Implementation Log reach it through that one hop.
- Once the PR number exists, fill it into both entries so the git history and the document trail resolve to each other.

### 9. Maintain the shared Approval Log entry

In `docs/APPROVAL_LOG.md`, per the pipeline convention:
- **Stage:** `UI Refactor`, listing the page/pass covered
- **Related Doc(s)** must list both `docs/refactor/UI_REFACTOR_LOG.md` (with the pass date) and `docs/implementation/IMPLEMENTATION_LOG.md` (with the pass number), plus the target source files
- Exactly one active entry at a time across the whole pipeline — update in place
- **Approved By**/**Approval Date** stay blank until a human operator explicitly approves the pass (typically alongside PR review) — never fill these in yourself

### 10. Open the PR via version-control

Use the **version-control** skill for the PR, with the description's **Testing** section naming the specs that were green before and after, and stating explicitly that no e2e spec was modified.

## Completion Checklist

- [ ] Baseline tests were observed **passing before** any change, from a clean working tree
- [ ] Exactly one page/screen family was refactored in this pass
- [ ] The component decomposition was presented and **explicitly approved** before any code was edited
- [ ] Every extracted component has one responsibility, an explicit props type, and a domain-meaningful name
- [ ] Client/server boundaries were kept as thin as before or thinner, and no whole page was newly marked `"use client"` without approval
- [ ] No abstraction was introduced for a use that does not yet exist (YAGNI)
- [ ] Visible text, DOM/layout, network calls, routing, and accessibility attributes are unchanged
- [ ] **No existing test in `e2e/` or `tests/` was edited, retargeted, weakened, or deleted**
- [ ] Full relevant e2e + unit suites, lint, and typecheck are green after the refactor
- [ ] Each extraction was committed as its own logical unit via **version-control**, with approval before each state-changing git action
- [ ] `docs/refactor/UI_REFACTOR_LOG.md` has a new pass entry with the decomposition table and preservation evidence
- [ ] `docs/implementation/IMPLEMENTATION_LOG.md` has a short numbered pass entry pointing at that detail record by file **and section**, with no facts duplicated between them
- [ ] Cross-references resolve in both directions — the refactor log names its Implementation Log pass, branch, PR, and the `DESIGN.md` story; any other artifact produced this pass is linked from the refactor log
- [ ] No decomposition decision was assumed — every ambiguity became a tracked `UQ-` Open Question
- [ ] `docs/APPROVAL_LOG.md` has an active entry (Stage: UI Refactor) pending human approval for this pass
