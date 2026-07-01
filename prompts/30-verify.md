# Stage 5 — Verify (deterministic gate over the generated artifacts)

**Role:** release gate. After the HLD, FSDs, domain dictionary, and specs are written, run a set of
**mechanical, grep-driven checks** and write `<spec_dir>/VERIFY.md`. This is NOT an LLM opinion review
(that is `/review-spec`); it is a checklist of things that must be *literally true on disk*. Prefer real
`bash`/`grep` over judgement — quote the failing lines. Skip if `verify.enabled: false`.

## Inputs
- `<hld_dir>`, `<fsd_dir>`, `<spec_dir>`, `<domain_dir>`, `<rules_dir>` from `config.paths`.
- `config.spec.size_budget` and `config.verify.*`.

## Checks (each is BLOCKER unless marked WARN). Run them; record pass/fail with the offending path+line.

1. **Cross-reference integrity (BLOCKER).** Every `§<n>` / `HLD §` reference in the HLD, FSDs, and specs
   resolves to a heading that exists in the cited document. Grep each `§NN` and confirm the target section
   number exists. (This catches phantom refs like a body citing `§23` when the doc ends at §14.)
2. **Provenance headers (BLOCKER).** Every `SPEC-*.md` has, as its first blockquote, the provenance header
   with `Source FSD:` (a link that resolves to a real file on disk) and `Depends on:`.
3. **Dependency graph (BLOCKER).** Every number in a spec's `Depends on:` exists as a `SPEC-<number>-*.md`
   file; no self-reference; every `Depends on` number is **lower** than the spec's own number (build-order
   monotonicity — no forward refs, no cycles). Cross-check against `TRACKER.md`.
4. **Single module owner (BLOCKER).** For every module a spec declares "new" (`/add-module`), exactly one
   spec in the set carries `add-module` for it. Flag both "new module, no `add-module` owner" and "two
   specs `add-module` the same module". (Catches the messaging/duplicate-queue registration seams.)
5. **Shared-flow consistency (BLOCKER).** Where the generic sub-process sequence (e.g. `SP-AUTH → … →
   SP-DELIVER`) is quoted in more than one doc (HLD generic flow + each product FSD/spec), the ordering
   must be identical. Extract each quoted chain and diff them; report any product that reorders the shared
   flow versus the HLD.
6. **No terminal deferral of decision content (BLOCKER).** No DMN / adapter / message / product-`details`
   spec may leave its concrete content as a bare "typed inputs / configured as / DMN table" with neither
   (a) the values inline, nor (b) a `docs/domain/*` reference path, nor (c) an explicit `TODO(domain-input)`.
   Grep the decision specs for a `docs/domain/` reference or a `TODO(domain-input)` marker; a decision spec
   with none of the three fails.
7. **Open domain inputs surfaced, not hidden (WARN).** Reconcile `TODO(domain-input)` markers across
   specs/FSDs with `<domain_dir>/README.md`'s register — every code-level TODO should appear in the
   register with an owner. Count them; list any unregistered.
8. **Spec size budget (WARN).** Per `config.spec.size_budget`: flag any spec exceeding `max_acs` acceptance
   criteria, `max_subskills` sub-skills, or `max_depends_on` prerequisites — these are the "epic disguised
   as a thin spec" candidates that should have been split.
9. **Invariant wording (WARN).** Grep for the forbidden slogan `ZERO new framework code` (and variants) —
   the accurate invariant is "no new module/entity/table/adapter". Report occurrences.
10. **Structural completeness (BLOCKER).** Every spec contains all `/implement-spec` sections
    (`## Problem`, `## Solution`, `## Acceptance Criteria`, `## Technical Design` with
    `### Affected Modules` / `### Data Model Changes` / `### API Changes` / `### Events`, `## Edge Cases`,
    `## Test Plan`). No empty section.

## Output — write `<spec_dir>/VERIFY.md`
A table of the 10 checks (status ✅/⚠️/❌, count of findings) followed by a numbered list of each finding
with the exact `path:line` and the offending text. End with a one-line **GATE: PASS** (no BLOCKER failures)
or **GATE: FAIL** (≥1 BLOCKER).

## Output contract
Return JSON only:
```json
{ "verify": "docs/specs/VERIFY.md", "gate": "PASS|FAIL",
  "blockers": <int>, "warnings": <int>,
  "findings": [ { "check": 5, "severity": "blocker", "path": "docs/specs/SPEC-075-...md", "line": 27, "note": "SP-SCREEN before SP-DOC contradicts HLD §7.3" } ] }
```
Do not paste file contents. On `GATE: FAIL`, the driver reports it and stops before declaring success —
the run is not "done" with an open BLOCKER.
