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
- Sibling prompts: `prompts/00-brd-to-hld.md`, `10-index-and-template.md`, `11-fsd-author.md`,
  `20-fsd-to-spec.md`. Read each at the stage it applies.

## Stages

1. **BRD → HLD** — if `<hld_dir>` has no HLD, follow `prompts/00-brd-to-hld.md` to read the BRD and
   write the HLD. If an HLD already exists, skip (idempotent re-runs).
2. **HLD → FSD index + numbering + tracker** — follow `prompts/10-index-and-template.md`: write
   `_TEMPLATE.md`, `docs/fsd/README.md`, the **`spec_plan`** (build-order numbering) and seed
   `docs/specs/TRACKER.md`.
3. **FSD authoring (parallel)** — for each batch in the stage-1 plan, **spawn a sub-agent** (Task tool,
   up to `fsd.authors_parallel` concurrent) following `prompts/11-fsd-author.md`. Each writes its FSDs.
4. **FSD → numbered specs (parallel)** — for each FSD, **spawn a sub-agent** following
   `prompts/20-fsd-to-spec.md`, naming files `SPEC-<number>-<slug>.md` from `spec_plan`. Collect each
   sub-agent's tracker metadata.
5. **Finalize the tracker** — fill `docs/specs/TRACKER.md` rows from the collected metadata (FSD,
   depends_on, sub-skills) in build order; set every Status to ☐ Not started; update the progress
   roll-up. Flag any cross-spec overlaps (e.g. a module's `forRoot` claimed by two specs).

## Rules
- Honor every decision in `<config>` (decomposition, template, numbering, tracker, authors_parallel).
- Stay rule-compliant per `<rules_dir>` throughout — the derived specs must pass `/review-spec`.
- Respect dependency/build order. Do not implement code — this driver stops at specs + tracker.
- Idempotent: re-running on the same inputs should converge, not duplicate.

## Output contract
Return JSON only:
```json
{ "hld": "<path or 'existing'>", "fsd_count": <int>, "spec_count": <int>,
  "tracker": "docs/specs/TRACKER.md", "overlaps": [...], "open_items": [...] }
```
Do not paste file contents.
