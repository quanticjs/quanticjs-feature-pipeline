# Stage 3 — FSD → SPEC emitter (one per §11 row)

**Role:** implementation lead. **Goal:** turn one row of an FSD's §11 "Specs Produced" table into a
single spec file under `docs/specs/` (named per the **Filename** section below) that `/implement-spec`
can consume directly.

## Inputs
- The FSD file path and the target spec for one §11 row: its `slug`, and (when `spec.numbering:
  build-order`) its assigned 3-digit `number` from the stage-1 `spec_plan`.
- `docs/fsd/README.md` (conventions) and `.claude/rules/*.md`.

## Filename
- `spec.numbering: build-order` → write `docs/specs/SPEC-<number>-<slug>.md` (e.g. `SPEC-001-backend-bootstrap.md`).
- `spec.numbering: none` → write `docs/specs/SPEC-<slug>.md`.
The slug is always kept so cross-references resolve.

## What to extract from the FSD for THIS spec
Pull only the slice of the FSD relevant to this row:
- the FRs and ACs that map to this spec's scope,
- the matching part of §6 Technical Design (Affected Modules / Data Model / API / Events),
- the relevant Edge Cases and Test Plan lines,
- the §11 row's declared `/implement-spec` sub-skills.

## Output — write the spec file in EXACTLY this shape
`/implement-spec` extracts these sections by name — do not rename or drop any. The first line after the
title is a **provenance header** (a blockquote) so the spec's source FSD is obvious at a glance:

```markdown
# Feature: <name>

> **↳ Source FSD:** [<FSD-NN> — <FSD title>](<relative path to the FSD, e.g. ../fsd/foundation/FSD-NN-...md>) · **Spec <number>** (build order) · **Depends on:** <prerequisite spec numbers, comma-listed, or "none">

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
- **Provenance header is mandatory** — emit it as the first blockquote after the title. The link is the
  relative path from `docs/specs/` to the source FSD file; `Depends on` is the prerequisite spec numbers
  (same as the `depends_on` in the output contract), or `none`. This keeps a flat `docs/specs/` folder
  traceable to its FSDs without nesting (which would fragment the global build order and break the
  flat `/implement-spec` discovery).
- Keep the spec small enough to implement in one `/implement-spec` pass. If a §11 row is too big, note
  it and propose a split — do not silently merge scope.
- Stay rule-compliant (same constraints as the FSD §6). The spec must not introduce a pattern the FSD
  didn't already sanction.
- Respect dependency order: a spec must name its prerequisite specs/modules in `### Affected Modules`.
- After writing, recommend running `/review-spec` on the file you wrote and applying the `-v2` corrections.

## Output contract
Return JSON only — this row's tracker metadata (the tracker job aggregates these):
```json
{ "spec": "docs/specs/SPEC-<number>-<slug>.md", "number": "001", "slug": "<slug>",
  "fsd": "FSD-NN", "depends_on": ["001", ...], "subskills": ["add-handler", ...], "acs": <int> }
```
`depends_on` lists the prerequisite spec NUMBERS you named in `### Affected Modules` (build order).
