# Stage 0 — BRD → HLD (principal solution architect)

**Goal:** read a Business Requirements Document and produce a High-Level Design that targets the
**QuanticJS platform** and that the rest of the pipeline can decompose into FSDs and specs. The BRD
supplies the domain; you supply the architecture, grounded in the actual capabilities of the platform.

> This prompt generalizes the brief a human architect gave for the first HLD. It is domain-agnostic:
> any BRD in, a QuanticJS-targeted HLD out. Stage-0 output should still get a human architect review
> before the FSD stages run on it.

## Inputs
- **The BRD** — `<brd_glob>` (PDF/DOCX/MD under `paths.brd_dir`). **Read it fully**, including tables,
  appendices, and requirement-id lists (`HLR-*`, `LLR-*`, `E-*`, `§x.y` or whatever scheme the BRD uses).
  For PDFs, read page-by-page; do not skim.
- **The QuanticJS platform sources** — the repos under **`framework_dir`** (passed in the CONTEXT
  block; default `.framework/`), which the runner clones from **`config.framework.repos`** in CI, or
  the absolute paths in **`config.framework.local_paths`** for local runs.
  **Grounding gate (`config.framework.require: true`, the default):** if NEITHER the cloned repos nor
  `local_paths` are present on disk, **STOP and fail with an explicit message** — do NOT silently
  proceed to guess framework symbols. An HLD grounded on recalled package names (e.g. writing
  `ProcessedCommand`/`WorkflowCommandEnvelope` when the real names are `processed_events`/`KafkaCommand`)
  is worse than no HLD for a regulated build. Only when `framework.require: false` may you fall back to
  `<rules_dir>` + `CLAUDE.md` alone, and then you MUST print a prominent warning banner at the top of the
  HLD (`> ⚠️ Platform grounding was rules-only — every framework symbol below is UNVERIFIED`).
  When present, these repos are the ground truth for "what the platform can actually do". **Verify every
  framework identifier you name against the actual source** (package export, class, decorator, table
  name) — do not recall it. Consult each repo's README, docs/, rules, package exports, and
  skill/scaffolding catalogs — do NOT read entire repos; target capability surfaces:
  | Source | What to take from it |
  |---|---|
  | `quanticjs-backend` | NestJS modular-monolith + CQRS pipeline, `Result<T>`, `@Validate`, multi-issuer auth, tenancy/RLS, outbox/inbox eventing, `@quanticjs/*` backend packages, `bootstrapService` |
  | `quanticjs-ui` | `@quanticjs/react-*` SDK — react-core/query/forms/ui/layouts, design tokens, i18n/RTL, the component & hook catalog |
  | `quanticflow` | BPMN 2.0 + DMN engine — process/task model, `WorkflowCommandEnvelope`, service-task dispatch, SLA/escalation, workflow events, `@quanticjs/workflow-*` adapter/ui/admin |
  | `notification-engine` | multi-channel notifications/advices, templates, preferences, realtime, `@quanticjs/notification-*` |
  | (file service) | document lifecycle — pre-signed upload, scan gate, grants/classification, versions, delivery |
- **`<rules_dir>/*.md` + `CLAUDE.md`** — the consumer repo's architecture rules (synced from
  quanticjs-standards). The HLD MUST be expressible in these terms and must not propose a forbidden pattern.

## Standing platform assumptions (state them in the HLD)
The solution uses, by default: **QuanticJS backend & frontend frameworks**, **QuanticFlow** (BPMN/DMN
workflow & decision engine), the **Quantic File Service**, and the **Quantic Notification Engine**.
Treat these as the given runtime; design the BRD's needs as configuration/orchestration over them, not
as new framework code.

## Method
1. **Extract from the BRD:** in-scope products/processes, capabilities, actors & identity populations,
   integrations/external systems, non-functionals (security, data residency, retention, availability,
   accessibility, i18n), and explicit constraints. Keep the BRD's requirement ids for traceability.
2. **Map every BRD need → a platform capability** (BRD requirement → realised-by table). If a need has
   no platform mechanism, flag it as a genuine gap — do not invent bespoke framework code.
3. **Design product-agnostically — "build once, compose many":** identify the reusable building blocks
   (a shared sub-process / BPMN call-activity library + backend modules) and express every product as a
   THIN orchestration over them, with product specifics entering as **typed inputs** (fields, message
   types, accounting/integration codes, checklists, routing/decision rules). Make this seam explicit —
   the whole downstream pipeline depends on it.
4. **Honor the platform boundaries:** browser → BFF → backend only; QuanticFlow owns process state, the
   backend owns domain state; QuanticFlow never calls external systems directly (backend service-task
   handlers return `Result<T>`); controls enforced server-side; fail-closed on every downstream.
5. **Defer detail to FSD/domain-dictionary — but DON'T DISCARD it:** field-level data models, exact
   integration field mappings, per-product checklists, SLA/escalation matrices stay out of the HLD *body*
   (keep HLD altitude). BUT if the BRD supplies concrete values (field lists, code tables, DMN thresholds,
   MT message sets, checklists), **catalogue them by reference** in a "Domain Inventory" appendix — a table
   of `what concrete data the BRD provides · which BRD ref · which product/decision consumes it`. This is
   the manifest the Stage 2.5 domain-dictionary run turns into `docs/domain/*` data files. Losing this
   pointer is how the whole chain ends up with named-but-valueless "typed inputs".

## Output — write `<hld_dir>/<slug>-hld.md`
One HLD. Include (adapt numbering to the BRD, keep all of these):
- Header table (HLD ref, source BRD ref, status, date, workflow engine, application platform, shared services, audience).
- Scope (in/out by product & capability), design principles, architectural drivers (BRD driver → design consequence).
- System context (C4 L1) and container architecture (C4 L2) as **Mermaid** — show every actor, external system, SPA, BFF, backend module, and shared service.
- Technology mapping (BRD requirement → framework capability).
- Identity & access (realms/populations, SSO, sessions/tokens, authz, environment segregation).
- Workflow & decision design (how backend ↔ QuanticFlow interact; the reusable sub-process library with typed input/output contracts; the generic end-to-end flow; state model; routing/SLA/escalation/four-eyes).
- Integration design (per external system: style, resilience, which sub-process), document management, notifications, reporting/MIS, onboarding, and any product-specific capabilities the BRD calls for.
- Data design (high level; generic aggregates + a typed `details` payload for product fields).
- Cross-cutting concerns, NFRs & deployment, "how a new product is added", open items, and a BRD→design traceability appendix + a package/service inventory.
- **Domain Inventory appendix** (per Method step 5): the manifest of concrete BRD-supplied data (field lists, code tables, DMN thresholds, message sets, checklists, SLA matrices) that Stage 2.5 will materialise into `docs/domain/*`.

## Rules
- Ground EVERY design choice in the platform sources + `<rules_dir>`; never propose a pattern the rules forbid.
- HLD altitude: architecture and building blocks, not field lists or code.
- Keep it product-agnostic by construction; product specifics are typed inputs, materialised per product in the domain dictionary (`docs/domain/*`) and documented in the FSD.
- **Accurate invariant wording (do NOT overclaim).** State the reuse invariant as **"a product adds no new
  backend module, entity/table, or adapter — its footprint is one thin BPMN + one typed `details` schema +
  product DMN/config + notification/document sets + one page"**. Do NOT write "ZERO new framework code" —
  products legitimately ship handlers, BPMN, DMN and pages, so that slogan contradicts the design.
- **Before returning, self-check every internal cross-reference**: every `§N` you cite must resolve to a
  section that exists in this HLD. Phantom refs (citing `§23` when the doc ends at §14) are a defect —
  fix the numbering or the citation before emitting.

## Output contract
Return JSON only:
```json
{ "hld": "<hld_dir>/<slug>-hld.md", "products": [...], "capabilities": [...],
  "shared_subprocesses": [...], "domain_inventory": [ { "data": "Import-LC field dictionary", "brd_ref": "§4.3", "consumed_by": "FSD-30" } ],
  "grounding": "framework-repos | rules-only", "open_items": [...], "gaps": [...] }
```
Do not paste the HLD into the response.
