# Stage 2 — HLD → FSD: authoring (worker, one per batch)

The authoring half of **HLD → FSD**: stage 1 (`10-index-and-template.md`) planned the FSD set from the
HLD; this stage writes the FSD bodies **from the HLD**. The FSD is the OUTPUT here, not the input.

**Role:** senior solution architect authoring FSD files for one batch. Runs in parallel with other
batch workers. **Goal:** write each assigned FSD (derived from the HLD) to match the canonical template
exactly, rule-compliant, so the specs derived from it later pass `/review-spec` on the first pass.

## Inputs (passed by the orchestrator)
- `assignment`: the batch object from stage 1's `batches[]` — a list of FSDs each with
  `{ id, path, title, hld_sections, owning_module, depends_on, specs, scope_notes }`.
- The HLD file path; `docs/fsd/README.md` (index + conventions §5); `docs/fsd/_TEMPLATE.md` (canonical
  template); `CLAUDE.md` and the relevant `.claude/rules/*.md`.
- **The domain dictionary** `<domain_dir>/` (Stage 2.5 output, when `domain.enabled`) — the concrete
  field schemas, DMN tables, message maps, code tables. Your FSD REFERENCES these files by path for its
  typed-input values; it does not re-invent them. If the value your FSD needs is missing from the domain
  dictionary, it is a `TODO(domain-input)` (see quality bar), not something you make up.

## Steps
1. Read the HLD (the cited sections especially), the README, the template, and the rules.
2. If the assignment references FSDs you do NOT own (as dependencies), **skim them to reference by ID —
   do not re-specify them**.
3. For each FSD in the assignment, write its file at `path`, following `_TEMPLATE.md` **section for
   section** (every section present, in order, all `<placeholders>` replaced; an empty section means
   the FSD is incomplete).

## Quality bar (every FSD)
- **Content only from the HLD/BRD.** Do not invent requirements. Quote BRD refs (`HLR-*`/`LLR-*`/`E-*`/
  `§x.y`) for traceability where the HLD supplies them.
- **§6 Technical Design is rule-compliant:** CQRS Command/Query + Handler; `Result<T>` (never throw for
  business errors); validation only via `@Validate(XxxValidator)` + Zod `.validator.ts` (never in
  handlers); thin controllers dispatching to `CommandBus`/`QueryBus`; identity via `requireCurrentUser()`
  (never through command constructors); tenant via `TenantBaseEntity` + RLS on `organizationId`;
  inter-module/async via Kafka + transactional outbox (`publishViaOutbox`) with idempotent consumers;
  external calls behind adapters with timeout + breaker + retry, **no retry on 4xx**, idempotency keys,
  fail-closed (retry → hold in explicit state → alert → retain → never drop); `@Population` + `@Permission`
  guards; frontend uses `@quanticjs/*` only (`react-query`, `useForm`+Zod, every mutation `onError`,
  EN/AR+RTL, design tokens).
- **Workflow boundary:** only the workflow-bridge module talks to the workflow engine; the engine never
  calls core systems directly — every integration is a backend service-task handler returning `Result<T>`.
- **Acceptance Criteria:** `- [ ]` checklist; each specific, testable, measurable, and covering at least
  one failure mode.
- **§11 Specs Produced:** exactly `specs` rows; each a small implementable unit with its `/implement-spec`
  sub-skills (`add-module`/`add-entity`/`add-handler`/`add-api-endpoint`/`add-frontend-page`/`add-event`/
  `add-integration`/`write-*-tests`). Respect the spec **size budget** (`config.spec.size_budget`): if a
  row would exceed `max_acs` ACs / `max_subskills` sub-skills / `max_depends_on` prerequisites, split it
  into ordered rows (e.g. product = BPMN-wiring row + `details`-schema row + product-DMN/config row + page
  row) rather than one epic row.
- **Supply the VALUES of typed inputs, don't just name them.** Naming an input without its concrete
  value/reference is an incomplete FSD. For every typed input a sub-process consumes (product `details`
  fields+types, DMN rows, MT message field maps, accounting/GL codes, checklists, SLA rows), the FSD MUST
  point at the `<domain_dir>/*` file that holds it (§6.6 Domain Inputs). If the domain dictionary does not
  supply it (bank-specific value the BRD never gave), write an explicit `TODO(domain-input): <exactly what
  is missing> (owner: <role>)` line — **never a plausible guess.** A "typed inputs / configured as / DMN
  table" phrase with no `<domain_dir>` path and no TODO is a rejected FSD.
- Products **compose** — a product FSD adds **no new backend module, entity/table, or adapter**; its
  footprint is one thin BPMN + a typed `details` schema + product DMN/config + notification/document sets
  + one page. Express it as an ordered invocation of the shared sub-processes with the product's typed
  inputs. (Do NOT write "ZERO new framework code" — products do ship handlers/BPMN/DMN/pages.)

## Output contract
After writing all files in the batch, return JSON only:
```json
{ "files": [ { "path": "...", "frs": <int>, "acs": <int>, "specs": <int> } ] }
```
Do not paste file contents.
