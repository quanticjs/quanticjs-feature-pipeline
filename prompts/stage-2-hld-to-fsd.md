# Stage 2 — HLD → FSD (index + authoring, one bounded run)

You are stage 2 of the feature pipeline. The HLD already exists under `<hld_dir>` (produced by stage 1).
Do the WHOLE HLD → FSD transform in this run, then stop. Do not touch the BRD or write specs bodies.

## Inputs (CONTEXT block appended below)
- `config` — merged pipeline config (apply `fsd.*` / `spec.*` as fixed decisions; do not re-ask).
- `<hld_dir>`, `<fsd_dir>`, `<spec_dir>`, `<rules_dir>` from `config.paths`.
- Sibling prompts: `prompts/10-index-and-template.md`, `prompts/11-fsd-author.md`.

## Steps
1. **Index + template + numbering + tracker** — follow `prompts/10-index-and-template.md` exactly:
   write `<fsd_dir>/_TEMPLATE.md`, `<fsd_dir>/README.md`, assign the build-order `spec_plan`, and seed
   `<spec_dir>/TRACKER.md`. Keep its JSON output (batches + spec_plan) for the next step.
2. **Author every FSD** — for each batch in that plan, **spawn a sub-agent** (Task tool, up to
   `config.fsd.authors_parallel` concurrent) following `prompts/11-fsd-author.md`. Each writes its FSD
   files under `<fsd_dir>/<layer>/`. Wait for all to finish.
3. Verify: every FSD in the index exists on disk and matches `_TEMPLATE.md` (all sections present).

## Rules
- Rule-compliant per `<rules_dir>` throughout (so the specs derived later pass `/review-spec`).
- Respect dependency/build order. Do NOT write spec bodies here (that is stage 3) — only the FSDs,
  the index/template, the spec_plan, and the seeded tracker.

## Output contract
Return JSON only: `{ "fsd_count": <int>, "estimated_specs": <int>, "spec_plan": [ { "number": "001", "slug": "...", "fsd": "FSD-NN" }, ... ] }`.
Do not paste file contents.
