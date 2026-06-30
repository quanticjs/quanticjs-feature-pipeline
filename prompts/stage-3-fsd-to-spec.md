# Stage 3 — FSD → Specs (one bounded run)

You are stage 3 of the feature pipeline. The FSD set + index + `TRACKER.md` already exist (stage 2).
Turn every FSD into implementable specs, then stop. Do not modify the HLD or FSDs.

## Inputs (CONTEXT block appended below)
- `config` — merged pipeline config (`spec.*` decisions: numbering, granularity, tracker).
- `<fsd_dir>`, `<spec_dir>`, `<rules_dir>` from `config.paths`.
- Sibling prompt: `prompts/20-fsd-to-spec.md`.

## Steps
1. Read `<fsd_dir>/README.md` (the index, conventions, and the build-order numbering).
2. For **each FSD** and **each row of its §11 "Specs Produced" table**, **spawn a sub-agent** (Task tool,
   parallel) following `prompts/20-fsd-to-spec.md` to write `<spec_dir>/SPEC-<number>-<slug>.md` — each
   with the mandatory **provenance header** (linked source FSD + spec number + deps) as its first
   blockquote, and the exact `/implement-spec` section structure.
3. **Finalize `<spec_dir>/TRACKER.md`** — fill every row from the specs produced (FSD, depends_on,
   sub-skills), set Status ☐ Not started, update the progress roll-up. Flag cross-spec overlaps.

## Rules
- Numbering + filenames per `config.spec.numbering` (`build-order` → `SPEC-<number>-<slug>.md`).
- Rule-compliant per `<rules_dir>`; keep each spec small enough for one `/implement-spec` pass.
- Respect dependency order — name prerequisite specs in each spec's `### Affected Modules`.

## Output contract
Return JSON only: `{ "spec_count": <int>, "tracker": "<spec_dir>/TRACKER.md", "overlaps": [...] }`.
Do not paste file contents.
