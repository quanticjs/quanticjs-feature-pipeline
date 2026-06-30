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
- **The QuanticJS platform sources** — `<framework_sources>` (paths or checked-out repos). These are the
  ground truth for "what the platform can actually do". Consult each one's README, docs/, rules, package
  exports, and skill/scaffolding catalogs — do NOT read entire repos; target capability surfaces:
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
5. **Defer detail to FSD/LLD:** field-level data models, exact integration field mappings, per-product
   checklists, SLA/escalation matrices — list as out-of-scope-of-HLD parameters, not as content.

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

## Rules
- Ground EVERY design choice in the platform sources + `<rules_dir>`; never propose a pattern the rules forbid.
- HLD altitude: architecture and building blocks, not field lists or code.
- Keep it product-agnostic by construction; product specifics are typed inputs, documented per product in the FSD/LLD.

## Output contract
Return JSON only:
```json
{ "hld": "<hld_dir>/<slug>-hld.md", "products": [...], "capabilities": [...],
  "shared_subprocesses": [...], "open_items": [...], "gaps": [...] }
```
Do not paste the HLD into the response.
