# UI Refactor Log

Append one entry per refactor pass (one page or screen family per pass), **most recent first**. Never edit or delete a previous pass's entry — this file is a traceable history, not a living document.

---

## Pass: `<date>` — `<page or screen family>`

**Branch:** `<branch-name>`
**Target file(s):** `<src/app/.../page.tsx, ...>`
**Related design:** `frontend-designs/<image>.png`, `docs/design/DESIGN.md` § US-`<n>` (if applicable)
**Implementation Log entry:** `docs/implementation/IMPLEMENTATION_LOG.md` § Pass `<n>` (headline only — this section is the detail record)
**Approval Log entry:** `docs/APPROVAL_LOG.md` § Active Entry (Stage: UI Refactor)
**PR:** `<link/number>`
**Other artifacts from this pass:** `<decomposition sketch / screenshot set / note file — or none>`

### Baseline (before any change)

| Check | Command | Result |
|---|---|---|
| Working tree clean | `git status` | `<clean>` |
| Unit tests | `<command>` | `<pass count>` |
| E2E specs for this page | `<command>` | `<pass count>` |
| Lint / typecheck | `<command>` | `<clean>` |

### Inventory of the Frozen Page

| Block | Lines (before) | What it did | Repeated elsewhere | Frozen bits |
|---|---|---|---|---|
| `<visual block>` | `<file:line-range>` | `<layout / fetch / interaction>` | `<yes: where / no>` | `<hard-coded copy, inline data, magic values>` |

### Approved Decomposition

Approved by `<human operator>` on `<date>` before any code was edited.

| Component | Responsibility | Props (in) | Events/state (out) | Replaces | Client/server | Principle |
|---|---|---|---|---|---|---|
| `<Name>` | `<one sentence>` | `<typed props>` | `<callbacks / local state>` | `<file:line-range>` | `<client/server>` | `<one reason to change / third use / push state down / ...>` |

### Behavior Preservation Evidence

| Check | Before | After |
|---|---|---|
| `<e2e spec file>` | ✅ Pass | ✅ Pass |
| `<unit test file>` | ✅ Pass | ✅ Pass |
| Lint / typecheck | ✅ Clean | ✅ Clean |

**States walked manually:** `<loading / empty / error / populated / each interactive path>`
**Existing tests modified:** None. (Any deviation here must be an approved, logged exception.)
**Diff is pure movement:** `<confirmed / list any intentional non-movement change and its approval>`

### Open Questions Raised This Pass

| ID | Question | Status | Resolution |
|---|---|---|---|
| UQ-`<n>` | `<the exact ambiguity>` | Open | — |

### Deliberately Left Alone

<!-- <block/pattern> — <why it was not extracted this pass: only two uses so far, needs a behavior decision, out of scope, ...> -->

### Version Control Actions

- Branch: `<branch-name>`
- Commits: `<one per extraction — list>`
- PR: `<link/number>`
