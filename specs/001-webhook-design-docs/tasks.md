---

description: "Task list for Pacote de Design Docs — Sistema de Webhooks de Notificação de Pedidos"
---

# Tasks: Pacote de Design Docs — Sistema de Webhooks de Notificação de Pedidos

**Input**: Design documents from `/specs/001-webhook-design-docs/`

**Prerequisites**: [plan.md](./plan.md) (required), [spec.md](./spec.md) (required for user stories), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: Automated validation commands and acceptance checklists are integrated into story checkpoints and the Final Phase per `quickstart.md` (application code in `src/`, `prisma/`, and `tests/` is strictly read-only).

**Organization**: Tasks are grouped by user story (`US1` to `US6`) following the execution order recommended in `README.md` (`ADRs → RFC → FDD → PRD → Tracker → README`) so each deliverable is produced and verified systematically.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (`US1` to `US6`)
- Exact file paths are included in every task description

## Path Conventions

- **Deliverables**: `docs/adrs/ADR-001..006-*.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/PRD.md`, `docs/TRACKER.md`, `README.md`
- **Read-Only Reference Sources**: `TRANSCRICAO.md`, `prisma/schema.prisma`, `src/**`, `tests/**`
- **Spec Kit Artifacts**: `specs/001-webhook-design-docs/**`

---

## Phase 1: Setup (Shared Infrastructure & Baseline Protection)

**Purpose**: Preserve the original challenge instructions for auditability and prepare the documentation directory structure while enforcing codebase immutability.

- [X] T001 Preserve a reference copy of the original challenge specification from `README.md` into `specs/001-webhook-design-docs/enunciado-original.md`
- [X] T002 Verify target deliverable directories in `docs/` and `docs/adrs/` and confirm zero working-tree modifications in `src/`, `prisma/`, `tests/`, and `TRANSCRICAO.md`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Initialize the cross-artifact traceability structure and validate all primary source locations before writing any deliverable.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [X] T003 Initialize the Markdown table schema (`| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |`) and ID naming rules (`^(PRD|RFC|FDD|ADR)-[A-Z0-9-]+$`) in `docs/TRACKER.md`
- [X] T004 Verify existence of all 12 real codebase paths (`prisma/schema.prisma`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`) and transcript timestamps (`[hh:mm] Nome` in `TRANSCRICAO.md`) mapped in `specs/001-webhook-design-docs/research.md`

**Checkpoint**: Foundation ready — User Story 1 (`ADRs`) can now begin.

---

## Phase 3: User Story 1 - Registro de Decisões Arquiteturais Isoladas (ADRs) (Priority: P1) 🎯 MVP

**Goal**: Produce 6 Architecture Decision Records (`docs/adrs/ADR-001-*.md` to `ADR-006-*.md`) in MADR format covering all 6 core decisions from the technical meeting and explicitly referencing real codebase files.

**Independent Test**: Verify that `docs/adrs/` contains 6 files matching `ADR-NNN-titulo-em-kebab-case.md`, each containing `Status`, `Contexto`, `Decisão`, `Alternativas Consideradas` ($\ge 1$ alternative), and `Consequências` (positive, negative, and explicit trade-offs), with at least 1 ADR referencing real files in `src/` and `prisma/`.

### Implementation for User Story 1

- [X] T005 [P] [US1] Create `docs/adrs/ADR-001-outbox-no-mysql.md` with MADR sections (`Status`, `Contexto`, `Decisão`, `Alternativas Consideradas`, `Consequências`) documenting the Transactional Outbox in MySQL within `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), pre-rendered JSON payload snapshot at insertion (`[09:52] Larissa`), UUID primary key `String @id @default(uuid()) @db.Char(36)` (`[09:51] Larissa`), event filtering at insertion (`[09:34] Bruno`), and discarded alternatives (synchronous HTTP call, Redis Streams/Cluster, lazy rendering)
- [X] T006 [P] [US1] Create `docs/adrs/ADR-002-worker-separado-em-polling.md` with MADR sections documenting the standalone Node.js worker process (`src/worker.ts` modeled on `src/server.ts` with dedicated `PrismaClient` via `src/config/database.ts`), 2-second polling loop on `webhook_outbox` (`@@index([status, nextAttemptAt, createdAt])`), single-worker ordering per order, with the proposed predecessor gate and sequence tie-breaker in the FDD (`[09:12] Diego`), and discarded/deferred alternatives (in-process worker, MySQL triggers, multi-worker partitioning in v1)
- [X] T007 [P] [US1] Create `docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md` with MADR sections documenting the 10-second HTTP timeout (`[09:42] Diego`), one initial send plus 5 retries (`attempts` counts up to six persisted results per cycle) with exponential backoff progression `1m, 5m, 30m, 2h, 12h` (~15h window, `[09:17] Diego`), dedicated `webhook_dead_letter` table (`[09:18] Diego`), manual replay endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` protected by `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`) with audit logging (`[09:36] Sofia`), and discarded alternatives (infinite retries, 3 retries in 30m, keeping failed events only in `webhook_outbox`)
- [X] T008 [P] [US1] Create `docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md` with MADR sections documenting `HMAC-SHA256` payload signature in `X-Signature`, unique `secret String @db.VarChar(128)` per `WebhookEndpoint`, secret rotation with 24-hour grace period (`previousSecret String? @db.VarChar(128)` and `previousSecretExpiresAt DateTime?`, `[09:21] Sofia`), mandatory `https://` URL validation (`url String @db.VarChar(500)`), 64KB payload limit (`[09:24] Diego`), and discarded alternatives (global platform secret, immediate rotation without grace period, HTTP URLs or payload truncation)
- [X] T009 [P] [US1] Create `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` with MADR sections documenting *at-least-once* delivery guarantee (`[09:24] Diego`), unique UUID `X-Event-Id` header and `event_id` payload field generated at outbox insertion (`[09:25] Diego`), client-side deduplication contract, and discarded alternatives (distributed *exactly-once* coordination, *at-most-once* delivery)
- [X] T010 [P] [US1] Create `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-oms.md` with MADR sections explicitly referencing `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, and `src/shared/http/response.ts`, documenting the modular structure `src/modules/webhooks/`, pure function `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:41] Bruno`), and `WEBHOOK_*` error codes
- [X] T011 [US1] Perform critical review and refinement across `docs/adrs/ADR-001-outbox-no-mysql.md` through `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-oms.md` to verify MADR completeness, explicit trade-offs, and valid file paths

**Checkpoint**: User Story 1 (`ADRs`) is complete and independently verifiable.

---

## Phase 4: User Story 2 - Proposta Técnica para Revisão da Equipe (RFC) (Priority: P2)

**Goal**: Produce a concise (2–4 pages) architecture proposal in `docs/RFC.md` consolidating the technical approach, discarded alternatives, open questions, impacts/risks, and links to the ADRs without duplicating FDD implementation depth.

**Independent Test**: Inspect `docs/RFC.md` to confirm all 8 mandatory sections exist, meeting participants are listed as reviewers, at least 2 discarded alternatives include trade-offs, at least 2 open questions are documented, and relative links point to at least 2 ADRs in `docs/adrs/`.

### Implementation for User Story 2

- [X] T012 [US2] Write Metadata (Author, Status, Date, Reviewers: Larissa, Marcos, Bruno, Diego, Sofia), Executive Summary (TL;DR), Context & Problem, and concise Architecture Proposal in `docs/RFC.md`
- [X] T013 [US2] Write "Alternativas Consideradas" (detailing 6 discarded alternatives from the meeting with explicit trade-offs) and "Questões em Aberto" (detailing 5 open/deferred items: outbound rate limiting `[09:38] Diego`, multi-worker partitioning/locking `[09:13] Diego`, email notifications on consecutive failures `[09:37] Larissa`, >30-day outbox archiving `[09:08] Diego`, and RBAC hardening on webhook CRUD `[09:37] Sofia`) in `docs/RFC.md`
- [X] T014 [US2] Write "Impacto e Riscos" and "Decisões Relacionadas" in `docs/RFC.md` linking all 6 ADRs (`./adrs/ADR-001-outbox-no-mysql.md` to `./adrs/ADR-006-reuso-dos-padroes-existentes-do-oms.md`) and refine text to preserve strict RFC vs. FDD abstraction boundaries

**Checkpoint**: User Stories 1 AND 2 (`ADRs` + `RFC`) are complete and independently verifiable.

---

## Phase 5: User Story 3 - Especificação Detalhada de Construção (FDD) (Priority: P3)

**Goal**: Produce an actionable implementation specification in `docs/FDD.md` covering data models, operational flows, HTTP contracts, `WEBHOOK_*` error matrix, resilience, observability, and codebase integration with real file paths.

**Independent Test**: Validate in `docs/FDD.md` all 12 mandatory sections, $\ge 4$ HTTP contracts with request/response payloads and status codes, `WEBHOOK_*` error codes, observability (metrics, logs, tracing), and $\ge 4$ real repository file paths in "Integração com o sistema existente".

### Implementation for User Story 3

- [X] T015 [US3] Write sections 1 to 4 in `docs/FDD.md`: technical context/objectives, scope, the four proposed Prisma models and operational flows. Keep UUID public IDs; define internal sequence for ordering, cycle and attempts 0..6, unique delivery result `(outboxId, cycle, attemptNumber)`, unique DLQ entry `(outboxId, cycle)`, reverse relations, crash recovery and predecessor selection. Label new mechanisms as derived proposals with source and rationale (FDD-DET-01 to FDD-DET-08); keep the FDD schema canonical rather than duplicating field definitions here.
- [X] T016 [US3] Write sections 5 and 6 in `docs/FDD.md` detailing "Contratos Públicos" (7 REST endpoints: `POST /api/v1/webhooks`, `GET /api/v1/webhooks`, `PATCH /api/v1/webhooks/:id`, `DELETE /api/v1/webhooks/:id`, `POST /api/v1/webhooks/:id/rotate-secret`, `GET /api/v1/webhooks/:id/deliveries`, `POST /api/v1/admin/webhooks/dead-letter/:id/replay` plus Outbound Webhook HTTP POST contract with headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`, example request/response JSON payloads, status codes, and semantics) and "Matriz de Erros Previstos" (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_INVALID_EVENTS`, `WEBHOOK_CUSTOMER_NOT_FOUND`, `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`, `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`, `WEBHOOK_DELIVERY_TIMEOUT`, `WEBHOOK_DELIVERY_FAILED`, `WEBHOOK_SECRET_ROTATION_IN_PROGRESS`), including the local validation adapter and fixed constructor codes in the existing source
- [X] T017 [US3] Write sections 7, 8, and 9 in `docs/FDD.md` covering "Estratégias de Resiliência" (10s HTTP timeout, 5 retries with `1m, 5m, 30m, 2h, 12h` backoff, DLQ fallback, 24h secret rotation grace period, 64KB payload limit), "Observabilidade" (Metrics, structured Pino Logs with secret redaction and admin replay audit, and Tracing via `X-Event-Id`, `X-Webhook-Id`, and `requestId`), and "Dependências e Compatibilidade"
- [X] T018 [US3] Write mandatory section 10 "Integração com o sistema existente" and sections 11–12 (Technical Acceptance Criteria, Technical Risks & Mitigation) in `docs/FDD.md`, detailing integration with real repository files (`prisma/schema.prisma`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`)

**Checkpoint**: User Stories 1, 2, AND 3 (`ADRs`, `RFC`, `FDD`) are complete and independently verifiable.

---

## Phase 6: User Story 4 - Consolidação de Requisitos de Produto e Negócio (PRD) (Priority: P4)

**Goal**: Produce `docs/PRD.md` with all 12 mandatory product sections, $\ge 8$ functional requirements, $\ge 1$ quantitative goal, $\ge 2$ explicit out-of-scope items discarded/deferred in the meeting, and $\ge 2$ risks with probability, impact, and mitigation.

**Independent Test**: Audit `docs/PRD.md` for all 12 mandatory sections, $\ge 8$ functional requirements (`PRD-FR-01..12`), $\ge 1$ quantitative success metric, $\ge 2$ out-of-scope items, and $\ge 2$ complete risk entries.

### Implementation for User Story 4

- [X] T019 [US4] Write sections 1 to 5 in `docs/PRD.md` covering Feature Summary & Context, Problem & Motivation (B2B clients Atlas Comercial, MaxDistribuição, and Nova Cargo polling `GET /api/v1/orders` and churn risk), Target Audience & Use Cases, Objectives & Success Metrics (quantitative targets: notification latency $< 10\text{s}$ via $2\text{s}$ polling, delivery in 3 sprints by end of November, retry window of ~15h), and Scope (In-Scope and Out-of-Scope listing 5 items discarded/deferred in the meeting: inbound webhooks, email notifications on consecutive failures, visual webhook dashboard, >30-day automatic archiving, and outbound rate limiting in v1)
- [X] T020 [US4] Write sections 6 and 7 in `docs/PRD.md` specifying 12 Functional Requirements (`PRD-FR-01` to `PRD-FR-12`) and 8 Non-Functional Requirements (`PRD-NFR-01` to `PRD-NFR-08`) strictly grounded in `TRANSCRICAO.md`
- [X] T021 [US4] Write sections 8 to 12 in `docs/PRD.md` covering Main Decisions & Trade-offs, Dependencies, Risks & Mitigation (5 risks with explicit Probability, Impact, and Mitigation), Acceptance Criteria, and Testing & Validation Strategy (including Sofia's 2-business-day security review window `[09:46] Sofia`)

**Checkpoint**: User Stories 1 through 4 (`ADRs`, `RFC`, `FDD`, `PRD`) are complete and aligned.

---

## Phase 7: User Story 5 - Rastreabilidade Cruzada Anti-Alucinação (Tracker) (Priority: P5)

**Goal**: Populate `docs/TRACKER.md` mapping $\ge 80\%$ of all identifiable items across `PRD.md`, `RFC.md`, `FDD.md`, and `adrs/ADR-*.md` to their exact origin in `TRANSCRICAO.md` ($\ge 70\%$ with `[hh:mm] Nome`) or `CODIGO` ($\ge 5$ rows with real file paths).

**Independent Test**: Run the Tracker validation check in `quickstart.md` on `docs/TRACKER.md` to confirm table format compliance, $\ge 80\%$ item coverage, $\ge 70\%$ `TRANSCRICAO` rows with valid `[hh:mm] Nome` timestamps, and $\ge 5$ `CODIGO` rows with existing file paths.

### Implementation for User Story 5

- [X] T022 [US5] Populate `docs/TRACKER.md` with traceability rows for all ADRs (`docs/adrs/ADR-001-*.md` to `ADR-006-*.md`) and `docs/RFC.md` items (`RFC-PROP-*`, `RFC-ALT-*`, `RFC-OPEN-*`, `RFC-RISK-*`) referencing exact `[hh:mm] Nome` timestamps in `TRANSCRICAO.md` and real file paths in `src/` and `prisma/`
- [X] T023 [US5] Populate `docs/TRACKER.md` with traceability rows for all `docs/FDD.md` items (`FDD-FLOW-*`, `FDD-EP-*`, `FDD-ERR-*`, `FDD-RES-*`, `FDD-OBS-*`, `FDD-INT-*`) and `docs/PRD.md` items (`PRD-OBJ-*`, `PRD-OOS-*`, `PRD-FR-01..12`, `PRD-NFR-01..08`, `PRD-RISK-*`) Include the added objectives, acceptance criteria, model, technical-risk and derived-detail IDs from the revision.
- [X] T024 [US5] Audit and refine `docs/TRACKER.md` against `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, and `docs/adrs/` to verify $\ge 80\%$ coverage, $\ge 70\%$ `TRANSCRICAO` rows, $\ge 5$ `CODIGO` rows, and zero unverified claims

**Checkpoint**: All `docs/` deliverables (`ADRs`, `RFC`, `FDD`, `PRD`, `TRACKER`) are complete and cross-verified.

---

## Phase 8: User Story 6 - Documentação do Processo de Produção com IA (README) (Priority: P6)

**Goal**: Update root `README.md` to document the AI-assisted engineering process, tools used, workflow order, custom prompts, concrete iterations/refinements, and deliverable navigation guide.

**Independent Test**: Inspect `README.md` to verify all 6 mandatory sections exist, at least 1 AI tool is listed with its role, at least 2 custom prompts are shown in code blocks, at least 2 concrete iterations/adjustments are described, and the reading guide links to all delivered files.

### Implementation for User Story 6

- [X] T025 [US6] Update `README.md` at the repository root with the 6 mandatory sections ("Sobre o desafio", "Ferramentas de IA utilizadas", "Workflow adotado", "Prompts customizados" with $\ge 2$ prompts in code blocks, "Iterações e ajustes" with $\ge 2$ concrete refinements and iteration count, and "Como navegar a entrega" linking all deliverables and `specs/001-webhook-design-docs/enunciado-original.md`)

**Checkpoint**: All 6 user stories (`US1` to `US6`) are complete.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Final end-to-end quality gate verification across all deliverables and codebase immutability check.

- [X] T026 Run the end-to-end validation commands from `specs/001-webhook-design-docs/quickstart.md` across `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-001..006-*.md`, `docs/TRACKER.md`, and `README.md` to verify 100% of acceptance criteria
- [X] T027 Verify application/transcript immutability against the known PR base with the tracked and untracked checks in quickstart.md; git status alone does not audit committed changes

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — starts immediately.
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all user stories.
- **User Story 1 — ADRs (Phase 3, P1)**: Depends on Phase 2 — Establishes the architectural decisions skeleton.
- **User Story 2 — RFC (Phase 4, P2)**: Depends on US1 (`ADRs`) so `docs/RFC.md` can consolidate decisions and link directly to `./adrs/ADR-001..006-*.md`.
- **User Story 3 — FDD (Phase 5, P3)**: Depends on US1 (`ADRs`) and US2 (`RFC`) to detail implementation without duplicating the RFC's level of abstraction.
- **User Story 4 — PRD (Phase 6, P4)**: Depends on US1–US3 to consolidate product scope, requirements, exclusions, and risks with full technical alignment.
- **User Story 5 — Tracker (Phase 7, P5)**: Depends on US1–US4 to cross-reference all items produced in `ADRs`, `RFC`, `FDD`, and `PRD`.
- **User Story 6 — Process README (Phase 8, P6)**: Depends on US1–US5 so the real workflow, prompts, and review iterations across the completed package are accurately documented.
- **Polish (Phase 9)**: Depends on all 6 user stories being complete.

### Parallel Opportunities

- Within **Phase 3 (User Story 1 — ADRs)**, tasks `T005`, `T006`, `T007`, `T008`, `T009`, and `T010` write to 6 distinct files in `docs/adrs/` and can be executed in parallel (`[P]`).

---

## Parallel Example: User Story 1 (ADRs)

```bash
# All 6 ADRs operate on independent files under docs/adrs/ and can be generated in parallel:
Task T005: "Create docs/adrs/ADR-001-outbox-no-mysql.md"
Task T006: "Create docs/adrs/ADR-002-worker-separado-em-polling.md"
Task T007: "Create docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md"
Task T008: "Create docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md"
Task T009: "Create docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md"
Task T010: "Create docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-oms.md"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (`T001`–`T002`)
2. Complete Phase 2: Foundational (`T003`–`T004`)
3. Complete Phase 3: User Story 1 — ADRs (`T005`–`T011`)
4. **STOP and VALIDATE**: Verify all 6 ADRs in `docs/adrs/` against MADR and codebase reference rules.

### Incremental Delivery (`ADRs → RFC → FDD → PRD → Tracker → README`)

1. Complete Setup + Foundational → Baseline ready
2. Complete US1 (`docs/adrs/ADR-001..006-*.md`) → Validate ADR criteria
3. Complete US2 (`docs/RFC.md`) → Validate RFC criteria and ADR links
4. Complete US3 (`docs/FDD.md`) → Validate FDD contracts, `WEBHOOK_*` errors, and $\ge 4$ codebase paths
5. Complete US4 (`docs/PRD.md`) → Validate PRD 12 sections, $\ge 8$ FRs, quantitative metrics, exclusions, and risks
6. Complete US5 (`docs/TRACKER.md`) → Validate $\ge 80\%$ coverage, $\ge 70\%$ `TRANSCRICAO`, and $\ge 5$ `CODIGO` rows
7. Complete US6 (`README.md`) + Polish (`T026`–`T027`) → Run full `quickstart.md` audit

## Revisão Posterior do PR

- Correções e nova revisão registradas em [review.md](./review.md); o procedimento final de auditoria está em [quickstart.md](./quickstart.md). A conclusão das tarefas originais não substitui a revisão semântica nem significa aprovação das propostas derivadas pela equipe.
