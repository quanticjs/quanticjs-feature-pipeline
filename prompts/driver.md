# Driver — end-to-end orchestrator (single Claude Code invocation)

**Use this to TEST the pipeline on a BRD upload.** It runs the whole chain in ONE Claude Code session,
fanning out to sub-agents internally (the Task tool) for the parallel stages — the same way the pipeline
was first built by hand. This avoids the multi-job matrix + cross-stage JSON plumbing, so it's the
simplest thing to wire to a single `claude -p` call. With an `ANTHROPIC_API_KEY`, the parallel
sub-agents run without subscription rate limits.

You are the orchestrator. Execute the stages in order, applying `<config>` decisions. The per-stage
*instructions* live in sibling prompt files — read and follow them; do not re-derive them here.

## Inputs
- `<config>` — merged pipeline config (defaults + consumer overrides).
- `<brd_glob>`, `<hld_dir>`, `<fsd_dir>`, `<spec_dir>`, `<rules_dir>` — from `config.paths`.
- `framework_dir` (from the CONTEXT block; default `.framework/`) — the QuanticJS framework repos the
  runner cloned from `config.framework.repos` for stage-0 grounding; or `config.framework.local_paths`
  locally. Absent → stage 0 grounds on `<rules_dir>` + `CLAUDE.md` only.
- Sibling prompts: `prompts/00-brd-to-hld.md`, `10-index-and-template.md`, `15-domain-dictionary.md`,
  `11-fsd-author.md`, `20-fsd-to-spec.md`, `30-verify.md`. Read each at the stage it applies.

## Stages

1. **BRD → HLD** — if `<hld_dir>` has no HLD, follow `prompts/00-brd-to-hld.md` to read the BRD and
   write the HLD. If an HLD already exists, skip (idempotent re-runs). **Honor the grounding gate**: if
   `framework.require` and no framework sources are on disk, STOP — do not produce a guessed HLD.
2. **HLD → FSD index + numbering + tracker** — follow `prompts/10-index-and-template.md`: write
   `_TEMPLATE.md` (incl. the §6.6 Domain Inputs subsection), `docs/fsd/README.md`, the **`spec_plan`**
   (build-order numbering, size-budget-split) and seed `docs/specs/TRACKER.md`.
3. **BRD → Domain Dictionary** — when `domain.enabled`, follow `prompts/15-domain-dictionary.md`: mine the
   BRD for concrete decision content (field schemas, DMN rows, message maps, code tables, checklists, SLA)
   and write `<domain_dir>/*` + its `TODO(domain-input)` register. Runs before authoring so FSD/spec
   authors reference real values instead of naming valueless "typed inputs".
4. **FSD authoring (parallel)** — for each batch in the stage-2 plan, **spawn a sub-agent** (Task tool,
   up to `fsd.authors_parallel` concurrent) following `prompts/11-fsd-author.md`. Each writes its FSDs,
   referencing `<domain_dir>/*` for typed-input values (§6.6) or emitting `TODO(domain-input)`.
5. **FSD → numbered specs (parallel)** — for each FSD, **spawn a sub-agent** following
   `prompts/20-fsd-to-spec.md`, naming files `SPEC-<number>-<slug>.md` from `spec_plan`. Each spec
   carries the mandatory **provenance header** (linked source FSD + spec number + deps) as its first
   blockquote, and honors the **concrete-content contract** (inline values / `<domain_dir>` ref /
   `TODO(domain-input)`) and the **size budget**. Collect each sub-agent's tracker metadata.
6. **Finalize the tracker** — fill `docs/specs/TRACKER.md` rows from the collected metadata (FSD,
   depends_on, sub-skills, domain_refs, todos) in build order; set every Status to ☐ Not started; update
   the progress roll-up. Flag any cross-spec overlaps (e.g. a module's `forRoot` claimed by two specs).
7. **Spec review** — when `spec.validate: review-spec`, follow `prompts/stage-4-spec-review.md`: review
   every spec against `<rules_dir>` + the `/implement-spec` contract, correct in place, write `docs/specs/REVIEW.md`.
8. **Verify (deterministic gate)** — when `verify.enabled`, follow `prompts/30-verify.md`: run the
   mechanical checks (cross-ref integrity, dependency monotonicity, single module owner, shared-flow
   consistency, no terminal deferral, size budget) and write `docs/specs/VERIFY.md`. If the gate FAILs,
   report the blockers and DO NOT declare the run successful.

## Rules
- **WRITE EVERY ARTIFACT TO DISK before returning the JSON contract.** The HLD, FSDs, domain dict, specs,
  tracker, and VERIFY.md must actually exist on disk (Write tool) — the CI gate checks `git status` and
  fails on zero changes under the expected path. Emitting the JSON without having written the files is a
  FAILED run. When you fan out to sub-agents, WAIT for them and confirm their files landed before reporting.
- Honor every decision in `<config>` (decomposition, template, numbering, tracker, authors_parallel,
  domain, size_budget, verify, framework.require).
- Stay rule-compliant per `<rules_dir>` throughout — the derived specs must pass `/review-spec`.
- **Never invent domain values** (GL codes, thresholds, MT tags, endpoints) — extract, cite a standard, or
  emit `TODO(domain-input)`. A visible TODO is correct output; a plausible fake is a defect.
- Respect dependency/build order. Do not implement code — this driver stops at specs + tracker + verify.
- Idempotent: re-running on the same inputs should converge, not duplicate.

## Output contract
Return JSON only:
```json
{ "hld": "<path or 'existing'>", "grounding": "framework-repos | rules-only", "fsd_count": <int>,
  "domain_files": <int>, "domain_todos": <int>, "spec_count": <int>,
  "tracker": "docs/specs/TRACKER.md", "verify_gate": "PASS|FAIL", "blockers": <int>,
  "overlaps": [...], "open_items": [...] }
```
Do not paste file contents.
