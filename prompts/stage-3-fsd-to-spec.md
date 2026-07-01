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
   sub-skills, domain_refs, todos), set Status ☐ Not started, update the progress roll-up. Flag cross-spec
   overlaps.
4. **Verify** — when `config.verify.enabled`, follow `prompts/30-verify.md`: run the deterministic checks
   and write `<spec_dir>/VERIFY.md`. Report the gate result; a BLOCKER means the run is not done.

## Rules
- **WRITE FILES TO DISK before returning JSON.** You MUST create every `SPEC-<number>-<slug>.md` (and
  finalize `TRACKER.md`, `VERIFY.md`) with the Write tool and confirm they exist BEFORE emitting the output
  contract. The CI gate fails on zero `docs/specs/SPEC-*.md` — returning JSON without the files is a FAILED
  run. Do the work, then report it.
- Numbering + filenames per `config.spec.numbering` (`build-order` → `SPEC-<number>-<slug>.md`).
- Rule-compliant per `<rules_dir>`; keep each spec within `config.spec.size_budget` (one `/implement-spec` pass).
- **Concrete-content contract:** every DMN/adapter/message/`details` spec inlines its values, references a
  `<domain_dir>/*` file, or emits `TODO(domain-input)` — never a bare "typed inputs". Never invent a value.
- Respect dependency order — name prerequisite specs in each spec's `### Affected Modules`.

## Output contract
Return JSON only: `{ "spec_count": <int>, "tracker": "<spec_dir>/TRACKER.md", "verify_gate": "PASS|FAIL", "overlaps": [...] }`.
Do not paste file contents.
