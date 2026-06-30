<!--
Canonical FSD template (domain-agnostic) — QuanticJS feature pipeline.
Copy this file to the right layer folder and rename FSD-NN-<kebab-title>.md.
Do NOT rename/drop sections — §6.1–6.4 map 1:1 onto the /implement-spec contract.
Replace every <angle-bracket> placeholder. An empty section means the FSD is incomplete.
-->

# FSD-NN: <Title>

| | |
|---|---|
| **FSD Ref** | TF-QF-2026-FSD-NN |
| **HLD Ref** | TF-QF-2026-HLD-002 §<sections> |
| **Source BRD** | Smart_Trade_BRD_v0.5 Final — <BRD areas / HLR/LLR refs> |
| **Layer** | Foundation \| Sub-process \| Capability \| Product |
| **Owning module(s)** | `<module>` |
| **Status** | Draft |
| **Depends on** | <FSD-.. / —> |

## 1. Problem
<Why this unit exists, in business terms. The pain/requirement from the BRD/HLD it resolves.>

## 2. Solution
<What we are building, at a functional level. How it composes existing foundation/sub-process units.
Name the QuanticFlow sub-process(es), backend module(s), and shared services involved.>

## 3. Actors & Permissions
<Who interacts. For each: population (customer/staff), Keycloak role(s) — customer maker/checker/viewer
or staff `resource:action` client roles — and what they may do. Note @Population guards.>

## 4. Functional Requirements
<Numbered, atomic, testable. Trace each to a BRD ref where possible.>

- **FR-1** — <requirement> *(BRD: <ref>)*
- **FR-2** — <requirement> *(BRD: <ref>)*

## 5. Acceptance Criteria
<Each specific, testable, measurable; covers happy path AND at least one failure mode. Maps to FRs.>

- [ ] **AC-1** — <criterion>
- [ ] **AC-2** — <criterion>

## 6. Technical Design

### 6.1 Affected Modules
<Name specific existing modules and/or new bounded contexts. State module boundaries (CommandBus/
QueryBus or Kafka only — no cross-module injection) and schema ownership. Flag if /add-module is needed.>

### 6.2 Data Model Changes
<Entity names, fields + types, tenancy (BaseEntity vs TenantBaseEntity), indexes, RLS. Product-specific
fields go in a typed `details` payload, not new tables. camelCase columns. Note migrations.>

### 6.3 API Changes
<Each endpoint: HTTP method, path, request DTO, response DTO, auth (@Population/@Permission), the
Command/Query it dispatches. Thin controller. RFC 9457 error mapping. Note Swagger decorators.>

### 6.4 Events & Workflow
<Domain events (topic `quantic.events.<category>s`, outbox), consumers (idempotent). Workflow
interaction: BPMN sub-process/call-activity, service-task handlers, `WorkflowCommandEnvelope` commands
(`signalprocess`/`executeaction`/…), workflow events consumed, DMN tables, SLA/escalation.>

### 6.5 Integrations
<External systems touched (core systems / messaging / third-party APIs / File Service / Notification Engine / Keycloak), the
adapter, resilience (timeout, breaker, retry, no-retry-on-4xx), idempotency keys, fail-closed state.>

## 7. Edge Cases & Failure Handling
<Concrete failure modes and the handling: unauthenticated/expired/insufficient-permission; invalid input
(400)/not-found (404)/conflict (409)/server (500); concurrency (`@DistributedLock` at app-reference
granularity); partial failure / transaction boundaries; downstream outage (retry→hold→alert→retain);
duplicate delivery (idempotency); empty/null/max-length/pagination boundaries; cross-module Result.failure.>

## 8. Non-Functional Requirements
<Performance/latency targets, availability, security (MFA/masking/AES-256), data residency (UAE),
retention (≥7yr), accessibility (WCAG 2.2 AA), i18n (EN/AR + RTL) where applicable.>

## 9. Test Plan
- **Unit tests:** <handlers, validators, controllers — mocked repos, Result<T> assertions, DMN/decision logic>
- **Integration tests:** <real PostgreSQL/Redis/Keycloak — DB constraints, RLS/tenant isolation, permissions, outbox/idempotency, workflow contracts>
- **E2E tests:** <Playwright 4-state coverage (loading/empty/error/success), auth-mocked + real-stack journeys, a11y>

## 10. HLD & BRD Traceability
| Requirement (BRD / HLD) | Covered by |
|---|---|
| <BRD ref / HLD §> | <FR-/AC- / design element> |

## 11. Specs Produced (FSD → SPEC mapping)
<Each row becomes one docs/specs/SPEC-<slug>.md. Keep specs small enough to implement in one pass.>

| Spec file | Scope | implement-spec sub-skills |
|---|---|---|
| `SPEC-<slug>` | <what this spec delivers> | <add-module/add-entity/add-handler/add-api-endpoint/add-frontend-page/add-event/add-integration/write-*-tests> |
