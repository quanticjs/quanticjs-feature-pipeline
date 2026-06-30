# Stage 1 — HLD → FSD index & template (orchestrator)

**Role:** senior solution architect. **Goal:** read an HLD and produce (a) the canonical FSD template
and (b) the FSD index that drives stage-2 authoring. You DO NOT write individual FSD bodies here.

## Inputs
- The HLD markdown file (path passed in).
- `docs/pipeline/pipeline.config.yml` — apply `fsd.*` as fixed decisions (do not re-ask).
- `.claude/rules/*.md` and `CLAUDE.md` — the architecture rules the FSD technical designs must obey.

## Decisions (from config — do not deviate)
- **Decomposition** = `fsd.decomposition`. For `module-subprocess-hybrid`: one FSD per backend
  module/foundation capability, one per shared sub-process / reusable call-activity, one per product
  family. Products are THIN compositions over the shared library — no product branching in shared units.
- **Template** = `fsd.template` (`spec-aligned-traceability`): the template MUST contain the exact
  sections `/implement-spec` consumes — `Problem`, `Solution`, `Acceptance Criteria`, `Technical
  Design` → `Affected Modules` / `Data Model Changes` / `API Changes` / `Events`, `Edge Cases`,
  `Test Plan` — plus a traceability table and a §11 "Specs Produced" table.
- **Numbering** = `fsd.numbering` (`banded`): foundation 00–09, subprocess 10–19, capability 20–29,
  product 30–39. Leave gaps as headroom; never couple the number to insertion order.

## Tasks

1. **Write `docs/fsd/_TEMPLATE.md`** — the canonical template. Every section present and ordered; each
   with a one-line instruction comment; `§6.1–6.4` are the four `/implement-spec` subsections and are
   mandatory. (If a `_TEMPLATE.md` already exists and matches the config, reuse it.)

2. **Derive the FSD catalog from the HLD.** Walk the HLD's container/module diagram, its shared
   sub-process library, its capability sections, and its product list. Produce one catalog row per FSD
   with: id (banded), title, owning module(s), HLD section refs, depends-on, and an estimated
   "specs produced" count.

3. **Write `docs/fsd/README.md`** — the index. It MUST contain:
   - The HLD→FSD→SPEC layering explanation and the architectural invariant quoted from the HLD.
   - The directory layout (`foundation/ subprocess/ capability/ product/`).
   - The FSD index tables (one per layer), **in dependency/build order**.
   - A pointer to `_TEMPLATE.md` as canonical.
   - Authoring conventions binding on every FSD (CQRS + `Result<T>`, `@Validate`+Zod, thin controllers,
     `requireCurrentUser`, `TenantBaseEntity`/RLS, outbox + idempotent consumers, multi-issuer
     `@Population` auth, adapters with breaker/no-retry-on-4xx, frontend `@quanticjs/*` only,
     observability + ≥7yr retention) — pulled from `.claude/rules/`.
   - Identifier conventions (`FSD-NN`, `FR-N`, `AC-N` checkboxes, `SPEC-<slug>`).
   - The FSD→SPEC conversion workflow (run `/specify` or hand-author per §11 row → `/review-spec` →
     `/implement-spec`, respecting dependency order).
   - A critical-path / build-sequence diagram.

4. **Emit the stage-2 batch plan** as the final output (machine-readable): group the catalog into
   `fsd.authors_parallel` batches along layer boundaries, and for each FSD include the per-FSD scope
   notes (the concrete HLD content each author must cover). This plan is the input to stage 2.

## Output contract
After writing the two files, return JSON:
```json
{
  "template": "docs/fsd/_TEMPLATE.md",
  "index": "docs/fsd/README.md",
  "fsd_count": <int>,
  "estimated_specs": <int>,
  "batches": [
    { "name": "foundation-a", "fsds": [
        { "id": "FSD-00", "path": "docs/fsd/foundation/FSD-00-...md", "title": "...",
          "hld_sections": "...", "owning_module": "...", "depends_on": [], "specs": 3,
          "scope_notes": "concrete HLD content this FSD must cover" }
    ]},
    ...
  ]
}
```
Do not paste file contents into the response.
