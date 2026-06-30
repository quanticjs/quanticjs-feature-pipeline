# Stage 4 — Spec Review (one bounded run)

You are stage 4 (final content stage) of the feature pipeline. The numbered specs and `TRACKER.md`
already exist (stage 3). Review **every** spec against `<rules_dir>` and the `/implement-spec`
contract, correct each one **in place**, and write one consolidated review report. Do not modify the
HLD or FSDs. Do not invoke slash commands — this prompt is self-contained (CI has no plugins).

## Inputs (CONTEXT block appended below)
- `config` — merged pipeline config (`spec.validate`, `spec.numbering`, paths).
- `<spec_dir>`, `<rules_dir>` from `config.paths` (default `docs/specs`, `.claude/rules`).

## Steps
1. Read **all** rule files under `<rules_dir>` (in parallel) — `backend-patterns.md`, `frontend-patterns.md`,
   `auth-patterns.md`, `database-patterns.md`, `api-patterns.md`, `testing-patterns.md`, `testing-e2e-ui.md`,
   `testing-integration.md`, `docker-patterns.md`, `observability-backend.md`, `observability-frontend.md`,
   `resilience-ops.md`, plus any others present. These are the source of truth for every finding.
2. List every `<spec_dir>/SPEC-*.md` (skip `*-v2.md`, `README.md`, `TRACKER.md`, `REVIEW.md`).
3. For **each spec**, spawn a sub-agent (Task tool, parallel) that runs the four review passes below and
   returns its findings + the corrected spec body.

### Four review passes (per spec)
- **Pass 1 — Structural completeness** vs the `/implement-spec` contract. Required sections:
  `## Problem`, `## Solution`, `## Acceptance Criteria`, `## Technical Design` (with `### Affected Modules`,
  `### Data Model Changes`, `### API Changes`, `### Events`), `## Edge Cases`, `## Test Plan`.
  Flag missing/empty sections and vague content (`### Affected Modules` must name real modules;
  `### Data Model Changes` must list entity names/fields/types; `### API Changes` must give method/path/
  shape/auth; `## Test Plan` must separate unit/integration/E2E).
- **Pass 2 — Rule violations.** Check the design against every NEVER/MANDATORY in `<rules_dir>`. Each
  finding MUST cite the rule file and the exact NEVER/MANDATORY clause, what the spec says, and the
  rule-compliant replacement.
- **Pass 3 — Missing edge cases.** auth (unauth/expired/insufficient perms), API (400/404/409/500),
  frontend (loading/empty/error — mandatory), concurrency (`@DistributedLock`), transaction boundaries,
  partial failures, data boundaries (empty/null/max-length/pagination), inter-module `Result.failure`.
- **Pass 4 — Acceptance-criteria quality.** Each criterion must be specific, testable, complete (happy +
  ≥1 failure mode), and measurable. Flag vague/duplicate/manual-only/missing-state criteria with rewrites.

## Corrections (apply in place — pipeline mode)
For each spec with findings, **overwrite the original `SPEC-*.md`** with the corrected content (the git
diff on the work branch is the audit trail — do NOT leave `-v2.md` files). The corrected spec:
- Fills missing sections with actionable content **derived from the spec + rules** (never invent business
  requirements; use `TODO:` for anything needing domain input and list it in the report).
- Replaces every rule violation with the compliant pattern; adds missing edge cases; rewrites weak criteria.
- Preserves the original intent, formatting, voice, provenance header, and spec number/filename.
- Adds as its first line after the provenance header: `<!-- spec-reviewed: corrected against .claude/rules -->`.
- If a spec is already fully compliant, leave it unchanged and record it as clean.

## Report
Write `<spec_dir>/REVIEW.md` — a table (one row per spec: Structural X/10, Violations, Edge Cases,
Criteria, Changed yes/no) plus a short per-spec findings list and any `TODO:` items surfaced. If `TRACKER.md`
needs status/dependency updates from corrections, update it too.

## Rules
- NEVER invent features or requirements; only fix rule violations or structural gaps.
- NEVER remove original content unless it directly contradicts a rule.
- Every rule-violation finding MUST cite the specific rule file + NEVER/MANDATORY clause.
- Keep each corrected spec small enough for one `/implement-spec` pass.

## Output contract
Return JSON only:
`{ "reviewed": <int>, "specs_changed": <int>, "violations_fixed": <int>, "report": "<spec_dir>/REVIEW.md" }`.
Do not paste file contents.
