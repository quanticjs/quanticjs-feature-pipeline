# Reference run — HLD → FSD, 2026-06-30

The actual run that produced `docs/fsd/` (25 FSDs → 92 specs) from
`docs/hld/trade-finance-platform-hld.md`. This is the faithful record the generalized
`prompts/10-*` and `prompts/11-*` are abstracted from. Use it to re-derive or audit those prompts.

## Decisions (the three AskUserQuestion gates)
1. **Decomposition:** module + sub-process hybrid.
2. **FSD template:** spec-aligned + traceability.
3. **Delivery:** generate everything now.

## Orchestration
1. Wrote `docs/fsd/README.md` (index, conventions §5, build order §7) and `docs/fsd/_TEMPLATE.md` first
   — locks the template so all authors are consistent.
2. Created `docs/fsd/{foundation,subprocess,capability,product}/`.
3. Fanned out **6 parallel general-purpose agents**, each owning a batch (assignments below).
4. Verified: all files present; every FSD has the `/implement-spec` sections; per-FSD spec counts match
   the index; updated the README grand total to the actual 92.

## The catalog produced (banded numbering)
- **foundation/** FSD-00 platform-foundation, 01 identity-auth-sso, 02 multi-tenancy-rls,
  03 workflow-bridge, 04 documents-file-service, 05 notification-bridge, 06 integration-adapters,
  07 iam-audit.
- **subprocess/** FSD-10 sp-auth, 11 sp-applval, 12 sp-partybic, 13 sp-limit, 14 sp-screen, 15 sp-ops,
  16 sp-fx, 17 sp-corepost, 18 sp-msg.
- **capability/** FSD-20 onboarding, 21 templates-clauses, 22 duplicate-detection-repair-queue,
  23 reporting-mis, 24 messaging-swift.
- **product/** FSD-30 import-lc, 31 letters-of-guarantee, 32 trade-loans.

## The 6 batch assignments (verbatim scope per FSD)

> Each agent was given: read the HLD + `docs/fsd/README.md` + `docs/fsd/_TEMPLATE.md` + `CLAUDE.md` +
> the relevant `.claude/rules/*.md`; follow the template exactly; content only from the HLD; rule-compliant
> §6; ACs specific+testable with a failure mode; §11 spec count as stated; return a compact summary only.
> The per-FSD scope notes below are exactly what each author was told to cover.

### Batch 1 — foundation-a (FSD-00..03)
- **FSD-00 Platform Foundation & Service Bootstrap** — HLD §5.2/§22/§23/App A; module: app shell; deps none; 3 specs.
  `bootstrapService()` modular-monolith bootstrap, fixed CQRS pipeline, `QuanticCoreModule.forRoot` once, global
  guards/interceptors, health live/ready/startup, graceful SIGTERM shutdown, observability baseline (Pino deny-by-default,
  OTel, Prometheus, correlationId), Redis/Kafka/Postgres wiring, two SPA shells (Trade Portal customers / Back-Office
  Workbench internal) provider stacks, design tokens, EN/AR+RTL.
- **FSD-01 Identity, Multi-Realm Auth & SSO** — HLD §7.1–7.3/§7.5/§3; module: auth/BFFs+IAM; deps 00; 5 specs.
  Two Keycloak realms (customers CIAM/MFA, internal staff+service accounts), two BFFs, multi-issuer via
  `KEYCLOAK_ISSUERS` tagging population from verified `iss`, `@Population('staff')`, SSO over EDB360/SmartConnect,
  httpOnly `__Host-sid` + server-side tokens, RFC 8693 same-realm token exchange, `runAsService()`, cross-realm exchange
  BANNED, per-env realms via AD groups, `resource:action` client roles, no admin bypass.
- **FSD-02 Multi-Tenancy & RLS** — HLD §3/§7/§20; module: shared kernel; deps 00,01; 3 specs.
  organizationId from verified JWT only, `TenantBaseEntity`, PG RLS, `TenantSubscriber` stamp/throw,
  `AuthContextInterceptor` binds tenant store, cross-tenant via `runAsService` w/ target org, staff cross-tenant read,
  tenant-prefixed cache keys, tenant-scoped locks at application-reference granularity.
- **FSD-03 Workflow-Bridge & QuanticFlow Integration** — HLD §8.1/§8.4/§8.5/§5.2; module: workflow-bridge; deps 00–02; 5 specs.
  Only module talking to QuanticFlow; HTTP start `POST /workflow/instances`; Kafka `quantic.commands`
  `WorkflowCommandEnvelope` (deduped by `commandId`/`ProcessedCommand`); QF→backend service-task dispatch
  (`WorkflowServiceTaskHandler` → `Result<T>`, failure→retry/backoff→`ProcessIncident`); workflow events
  `quantic.events.<Type>s.<defId>`; denormalised `WorkflowLink` status mirror; generic state model §8.4; realtime to portal;
  candidatePermissions/SLA dueAt/escalation on BullMQ+Redis; `@quanticjs/workflow-quanticflow` + `workflow-ui`.

### Batch 2 — foundation-b (FSD-04..07)
- **FSD-04 Documents & File Service (SP-DOC/SP-DELIVER)** — HLD §11/§20/§21; module: documents; deps 00–02; 4 specs.
  Pre-signed PUT (`POST /api/files/upload-url` 900s)+multipart, uploads staging `FileStatus.Scanning`, virus-scan gate →
  `clean/`, `file.ready`, classification internal vs customer-shared via `FileAccessGrantEntity`, staff see all customer
  docs in Workbench (per-hop exchange + service context org), `FileVersionEntity`, pre-signed GET 300s attachment audited,
  DMS/Omnidocs mirror + stage-and-retry, ≤15MB pdf/xls/doc/jpg/gif, optional RAG; `DocumentIndex` entity.
- **FSD-05 Notification-Bridge & Advices (SP-NOTIFY)** — HLD §16/§21; module: notification-bridge; deps 00–02; 3 specs.
  Publish request per milestone (Kafka `quantic.notifications.requests`/REST), 6 channels (in-app Socket.IO /realtime,
  email, SMS, push, webhook, chat; WhatsApp via SMS/chat), milestone set, Handlebars EN/AR templates + fallback,
  in-portal inbox `@quanticjs/notification-ui` appId-scoped, prefs/quiet-hours/caps, delivery logs, retry 3×→alert TFO,
  password-protected email advices.
- **FSD-06 Integration Adapters (T24/SWIFT/BIC/Compliance)** — HLD §21/§9/§3; module: integration; deps 00–02; 5 specs.
  Stable adapter interfaces; T24 (Conv+Islamic, idempotency keys, 3× back-off, breaker, hold-on-exhaustion, final limit
  re-check), SWIFT (gen+transmit, ACK/NAK, inbound receipt, validate-before-send, NAK→resolution, unmatched queue), BIC
  (validate, 3× retry then block+alert, reject inactive), Compliance (manual→API behind stable iface). No retry on 4xx;
  env routing from `details`. `/add-integration` pattern.
- **FSD-07 IAM Admin & Immutable Audit** — HLD §7.4/§10/§22/§5.2; module: iam-audit; deps 00–02; 3 specs.
  Admin/audit over Keycloak; `UserDirectoryService` (resolve id→name, findUsersByPermissions/getUsersByRealmRole,
  `IamModule.forRoot` once @Global); immutable append-only audit (identity+timestamp, core/message refs, 7yr UAE);
  IAM audit UI pages `@quanticjs/iam-ui` staff-gated; self-service `ProfilePage`.

### Batch 3 — subprocess-a (FSD-10..13)
- **FSD-10 SP-AUTH** §8.2/§10; deps 03; 2 specs — client maker-checker, `excludeCompletedBy` + backend guard, field freeze
  + previous-value highlight (LLR-7.7), single-user org holds at Pending Customer Checker (E-7.1), maker≠checker hard block.
- **FSD-11 SP-APPLVAL** §8.2/§21; deps 03,06; 2 specs — service task→T24, CIF active, filter active non-restricted accounts,
  eligibility rules (typed input), fail-closed hold (HLR-2).
- **FSD-12 SP-PARTYBIC** §8.2/§21/§20; deps 03,06; 3 specs — capture counterparties (Party roles typed input), BIC via
  Banker's Almanac (reject inactive, 3× retry→block+alert), DMN restricted-country, `PartyMaster` (LLR-3.7), prior-beneficiary
  auto-select (LLR-1.7); Party + PartyMaster entities.
- **FSD-13 SP-LIMIT** §8.2/§9; deps 03,06; 2 specs — service task→T24, advisory-not-blocking, typed input funded/non-funded
  + margin rule, final limit re-check at TFO Checker (LLR-4.5/E-4.3).

### Batch 4 — subprocess-b (FSD-14..18)
- **FSD-14 SP-SCREEN** §12/§21; deps 03,06; 3 specs — screen all named entities (LLR-8.2)→Clear/Potential/Positive, screening
  user task, DMN hit-classification, Positive blocks TFO Checker until resolution (LLR-8.3/8.4/8.5), re-screen on resubmit
  (E-9.3), audited (LLR-8.6), manual→API stable iface (E-9.1).
- **FSD-15 SP-OPS** §8.5/§10/§8.4; deps 03; 4 specs — BPMN user tasks + DMN routing (Compliance/Trade Sales/RM/Credit/Legal/
  CAC/Treasury/Business), candidatePermissions/candidateUserIds/priority/delegation/UserAbsence, SLA dueAt + badge (LLR-9.21),
  escalation ladder remind→reassign→escalateNotify→expire + `TaskEscalationLog` on BullMQ+Redis, TFO maker≠checker, Dept-Head
  per DoA, source tag. (Deepest sub-process.)
- **FSD-16 SP-FX** §13/§8.2; deps 03,06; 3 specs — DMN decision: special rate from T24 → else ≤ threshold card rate → else
  manual treasury rate (LLR-FX1.1–1.3); threshold maintained under maker-checker (LLR-FX1.4/E-FX1.2); rate+source audited,
  TFO override→Checker (LLR-FX1.5/1.6); forward FX (HLR-FX2).
- **FSD-17 SP-COREPOST** §9/§8.3; deps 03,06; 3 specs — idempotency keys per app-ref+step (E-11.3), domain write + outbox
  enqueue same tx, no advance to messaging until contract+postings+lien confirmed (LLR-11.4), partial success=failure→hold
  Pending Execution + `ProcessIncident` (E-11.2), final limit re-check (LLR-4.5/E-4.3). Rigorous on tx boundaries.
- **FSD-18 SP-MSG** §14/§8.2; deps 03,06; 3 specs — on confirmed posting generate SWIFT (MT760/767/765/799/999/103/202/
  pacs), validate-before-release, transmit, ACK→Issued / NAK→Pending NAK Resolution (HLR-12/C8), overflow flagged (E-12.2).
  Inbound/log lives in FSD-24.

### Batch 5 — capability (FSD-20..24)
- **FSD-20 Onboarding** §15/§20/§7.5; deps 01,03,06; 4 specs — pending provisioning (zero roles until approval), T24 master
  fetch (KYC Active), capture users+maker/checker/viewer+product assignment, docs via File Service, dual-control STAFF
  approval (population guard), activate w/ MFA + T24 channel flag; maintenance/offboarding; reuse SP-OPS.
- **FSD-21 Templates & Clauses** §19/§20; deps 03; 3 specs — application templates (field-values-only, process-scoped, unique
  name, max count, re-run validations on use, audited), clause library (read-only standard + customer; edit standard→non-standard
  flags Legal), copy/clone + prior-beneficiary (LLR-1.7).
- **FSD-22 Duplicate Detection & Repair Queue** §18/§17/§20; deps 03,15; 2 specs — DMN dup rule on BRD key set + n-day window,
  Repair Queue state + aging + alerts, confirm/reject, customer explicit-confirm gate (E-1.3).
- **FSD-23 Reporting, MIS & Dashboards** §17/§20/§22; deps 02,03; 5 specs — CQRS read projections off events; customer
  dashboards (trends, outstanding/turnover, limit utilisation, AED-equiv, exposure, exports XLS/CSV/PDF, VAT statement);
  back-office report set incl. Islamic Murabaha; EoD reconciliation Portal vs QF vs T24; 360° dashboard via
  `ProcessAnalyticsDaily`; tenancy RLS + subsidiary roll-up + staff cross-tenant masking; landing widgets. Frontend pages.
- **FSD-24 Messaging — Inward/Outward SWIFT & Message Log** §14/§20; deps 06,18; 4 specs — inbound auto-tag by ref →
  unmatched queue (HLR-MSG1/E-MSG.1), TFO compose responses maker-checker (HLR-MSG2), consolidated `MessageLog` per app
  (HLR-MSG3), customer-exposed/to-other-banks toggles audited (HLR-MSG4), customer extract MT7xx/MT4xx (§3.4.1).

### Batch 6 — product (FSD-30..32)
- **FSD-30 Import LC (Conventional)** §1.1/§8.3/§8.4; composes all SP-* + 21/22/24; 6 specs — Issuance/Amendment/Cancellation
  + docs-under-LC; typed inputs (fields/`details`, MT700/707/760-family, accounting code, doc checklist, screenable parties,
  funded/non-funded, FX threshold); schema-driven forms. Reference product.
- **FSD-31 Letters of Guarantee (+ Claim)** §1.1/§8/§14/App B; composes all SP-* + FSD-30 patterns; 6 specs —
  Issuance/Amendment/Re-issuance/Cancellation + CLAIM (distinct thin BPMN: inward MT765→claim-registered w/ Dispute/Extend/Pay
  →MT103/202/pacs settlement), non-standard-text→Legal DMN, notice/no-objection timers (RI1–3/X1–7).
- **FSD-32 Trade Loans** §1.1; composes SP-* (funded set) + FSD-30 patterns; 5 specs — financing under Import LC,
  open-account/WC/purchase-invoice financing, sales-invoice discounting, extension, pre-settlement; funded SP-COREPOST set +
  loan lifecycle states on §8.4.

Each product FSD: §6.2 = shared `Transaction` + `Party` + `WorkflowLink`, NO new product tables (fields in typed `details`);
§6.4 = thin BPMN + product DMN + call-activity sequence + notification milestones + output-document set; §11 splits specs by
process; §1/§2 assert ZERO new framework code (HLD §24).
