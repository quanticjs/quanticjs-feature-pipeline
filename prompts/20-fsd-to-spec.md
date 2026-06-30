# Stage 3 — FSD → SPEC emitter (one per §11 row)

**Role:** implementation lead. **Goal:** turn one row of an FSD's §11 "Specs Produced" table into a
single `docs/specs/SPEC-<slug>.md` that `/implement-spec` can consume directly.

## Inputs
- The FSD file path and the target `SPEC-<slug>` (one §11 row).
- `docs/fsd/README.md` (conventions) and `.claude/rules/*.md`.

## What to extract from the FSD for THIS spec
Pull only the slice of the FSD relevant to this row:
- the FRs and ACs that map to this spec's scope,
- the matching part of §6 Technical Design (Affected Modules / Data Model / API / Events),
- the relevant Edge Cases and Test Plan lines,
- the §11 row's declared `/implement-spec` sub-skills.

## Output — write `docs/specs/SPEC-<slug>.md` in EXACTLY this shape
`/implement-spec` extracts these sections by name — do not rename or drop any:

```markdown
# Feature: <name>

## Problem
<from the FSD §1 + the BRD refs for this slice>

## Solution
<what to build for this spec; how it composes existing modules/sub-processes>

## Acceptance Criteria
- [ ] <specific, testable, with a failure mode>

## Technical Design
### Affected Modules
### Data Model Changes
### API Changes
### Events

## Edge Cases

## Test Plan
- Unit tests:
- Integration tests:
- E2E tests:
```

## Rules
- Keep the spec small enough to implement in one `/implement-spec` pass. If a §11 row is too big, note
  it and propose a split — do not silently merge scope.
- Stay rule-compliant (same constraints as the FSD §6). The spec must not introduce a pattern the FSD
  didn't already sanction.
- Respect dependency order: a spec must name its prerequisite specs/modules in `### Affected Modules`.
- After writing, recommend running `/review-spec SPEC-<slug>.md` and applying the `-v2` corrections.

## Output contract
Return JSON only: `{ "spec": "docs/specs/SPEC-<slug>.md", "subskills": ["add-handler", ...] }`.
