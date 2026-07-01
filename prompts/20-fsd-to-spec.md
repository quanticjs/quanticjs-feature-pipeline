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
- the FSD §6.6 Domain Inputs references — carry them into the spec (see "Concrete decision content" below),
- the relevant Edge Cases and Test Plan lines,
- the §11 row's declared `/implement-spec` sub-skills.

## Concrete decision content — the "never guess" contract (MANDATORY)
`/implement-spec` is forbidden to guess; it builds only what the spec states. So a spec whose decision
content is a bare "typed inputs / configured as a DMN table / mapped by the adapter" is **unimplementable**.
For any spec that carries a DMN decision, a SWIFT/message field map, an adapter endpoint/field mapping, or
a product `details` schema, you MUST do ONE of:
1. **Inline the actual values** (the Zod field list with types, the DMN rows, the MT tag map) into the
   spec, OR
2. **Reference the `<domain_dir>/*` file by path** that holds them (`see docs/domain/dmn/routing.dmn.yml`),
   so the implementer opens a concrete data file, OR
3. **Emit `TODO(domain-input): <exactly what value is missing> (owner: <role>)`** when neither the FSD nor
   the domain dictionary supplies it — a visible, assignable gap.

A decision spec with none of the three is a rejected spec. Never paraphrase a missing value into a
plausible-looking one (no invented GL codes, thresholds, tag numbers, or endpoints).

## Size budget (keep a spec to one `/implement-spec` pass)
Honor `config.spec.size_budget` (`max_acs`, `max_subskills`, `max_depends_on`). If the assigned row exceeds
any of them, do not silently emit an epic: emit the spec for the smallest coherent slice and, in the
output contract, propose the split (the remaining slices as sibling spec numbers) so the tracker/next run
picks them up. A product-issuance row that pulls 10 prerequisites + a full BPMN + DMN + page is the
canonical thing to split (BPMN-wiring / `details`-schema / product-DMN / page).

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
  "fsd": "FSD-NN", "depends_on": ["001", ...], "subskills": ["add-handler", ...], "acs": <int>,
  "domain_refs": ["docs/domain/dmn/routing.dmn.yml", ...], "todos": ["<domain-input missing>", ...],
  "split_proposed": ["<slug-of-slice-to-emit-as-a-sibling-spec>", ...] }
```
`depends_on` lists the prerequisite spec NUMBERS you named in `### Affected Modules` (build order).
`domain_refs`/`todos` record the concrete-content contract; `split_proposed` is non-empty only when the
row exceeded the size budget.
