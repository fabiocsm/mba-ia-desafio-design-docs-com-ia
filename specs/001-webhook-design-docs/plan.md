# Implementation Plan: Pacote de Design Docs — Sistema de Webhooks de Notificação de Pedidos

**Branch**: `001-webhook-design-docs` | **Date**: 2026-10-21 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-webhook-design-docs/spec.md`

## Summary

Produzir o pacote completo, consistente e rastreável de documentação técnica (`ADRs`, `RFC`, `FDD`, `PRD`, `Tracker` e `README` do processo) para o **Sistema de Webhooks de Notificação de Pedidos** do Order Management System (OMS), baseando-se estritamente nos fatos decididos na transcrição da reunião técnica (`TRANSCRICAO.md`) e na arquitetura existente do código-fonte (`src/` e `prisma/`). O plano segue a ordem de execução estrutural recomendada no enunciado (`ADRs → RFC → FDD → PRD → Tracker → README`) com ciclos de revisão crítica para impedir alucinações, sobreposição entre documentos ou violação da imutabilidade do código.

## Technical Context

**Language/Version**: Markdown (GitHub Flavored Markdown, PT-BR) documentando uma aplicação existente em Node.js $\ge 20$ + TypeScript `5.6.3` (ES Modules)

**Primary Dependencies**: Stack de referência do OMS analisada em modo somente-leitura: Express `4.21.1`, Prisma `5.22.0` (`@prisma/client`), Zod `3.23.8`, Pino `9.5.0` (`pino-http`), JSON Web Token (`jsonwebtoken` `9.0.2`), UUID `11.0.3`

**Storage**: Artefatos documentais persistidos em arquivos `.md` sob `docs/` e `README.md`; arquitetura documentada utiliza MySQL existente via Prisma (`orders`, `order_status_history`, `customers` + novas tabelas propostas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`)

**Testing**: Validação documental e auditoria automatizada via checklists e comandos de verificação (veja [quickstart.md](./quickstart.md)); suíte existente do OMS usa Vitest `2.1.4` + Supertest `7.0.0` (`tests/`)

**Target Platform**: Repositório Git/GitHub (entrega puramente documental; código da aplicação em `src/`, `prisma/` e `tests/` é estritamente imutável)

**Project Type**: Technical Design Documentation Package (`PRD`, `RFC`, `FDD`, `ADRs`, `Traceability Tracker`, `Process README`) para um Web Service REST + Background Polling Worker

**Performance Goals**:
- Meta da solução especificada: latência de notificação percebida pelo cliente $< 10\text{s}$ via worker em polling de $2\text{s}$ (`[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa`), timeout HTTP de $10\text{s}$ por disparo (`[09:42] Diego`) e janela de retry de ~15h em 5 retentativas após o envio inicial (`1m, 5m, 30m, 2h, 12h`, `[09:17] Diego`).
- Meta de qualidade documental: $\ge 80\%$ dos itens rastreados em `docs/TRACKER.md` ($\ge 70\%$ `TRANSCRICAO`, $\ge 5$ `CODIGO`) e $100\%$ de caminhos de arquivo válidos.

**Constraints**:
- Proibição absoluta de alterar `src/`, `prisma/`, `tests/`, configurações ou `TRANSCRICAO.md`.
- Proibição absoluta de inventar requisitos, endpoints ou decisões sem origem identificável em `TRANSCRICAO.md` ou no código base.
- Fronteira estrita de altura entre documentos (RFC conciso de 2–4 páginas sem repetir o detalhamento de implementação do FDD).
- Filtragem obrigatória de ideias descartadas ou adiadas na reunião (não podem entrar como requisitos ativos).

**Scale/Scope**: 11 arquivos Markdown finais a produzir/atualizar (`6` ADRs em `docs/adrs/`, `docs/RFC.md`, `docs/FDD.md`, `docs/PRD.md`, `docs/TRACKER.md` e `README.md`), cobrindo 6 decisões arquiteturais, 7 contratos de endpoints HTTP + 1 contrato de payload outbound, 11 códigos de erro `WEBHOOK_*`, 12 requisitos funcionais, 8 requisitos não funcionais e 266 unidades identificáveis rastreadas no Tracker.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Princípio da Constituição | Status Pré-Phase 0 | Status Pós-Phase 1 | Evidência de Conformidade no Plano |
| :--- | :--- | :--- | :--- |
| **I. Rastreabilidade Estrita e Anti-Alucinação (NON-NEGOTIABLE)** | PASS | PASS | Todos os requisitos, decisões, alternativas e contratos em [research.md](./research.md), [data-model.md](./data-model.md) e [contracts/](./contracts/) estão vinculados a timestamps `[hh:mm] Nome` de `TRANSCRICAO.md` ou a arquivos reais de `src/` e `prisma/`. |
| **II. Imutabilidade do Código e da Transcrição (NON-NEGOTIABLE)** | PASS | PASS | O plano restringe todas as operações de escrita aos arquivos `docs/**`, `README.md` e `specs/001-webhook-design-docs/**`. Incluída auditoria por comparação com a base conhecida, incluindo alterações commitadas e arquivos novos em [quickstart.md](./quickstart.md). |
| **III. Separação de Alturas e Fronteiras Documentais** | PASS | PASS | Definido em [deliverables-contract.md](./contracts/deliverables-contract.md) o recorte exato de cada documento: ADRs (decisão isolada), RFC (proposta arquitetural concisa de 2–4 páginas), FDD (implementação profunda), PRD (produto/negócio) e Tracker (rastreabilidade). |
| **IV. Filtragem Rigorosa de Escopo e Exclusões Explícitas** | PASS | PASS | Mapeadas em [research.md](./research.md) todas as exclusões (inbound webhooks, e-mail em falhas, dashboard visual, arquivamento >30 dias), alternativas descartadas (HTTP síncrono, Redis Streams, triggers MySQL, 3 retries/indefinido, secret global, *exactly-once*) e questões em aberto (rate limiting, multi-worker, endurecimento RBAC). |
| **V. Aderência Arquitetural e Acionabilidade no Código Base** | PASS | PASS | Mapeados 12 arquivos reais do código base em [research.md](./research.md) (`prisma/schema.prisma`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`). |

## Project Structure

### Documentation (this feature)

```text
specs/001-webhook-design-docs/
├── plan.md                                  # Este plano de execução ($speckit-plan)
├── spec.md                                  # Especificação da feature ($speckit-specify)
├── research.md                              # Phase 0: Mapeamento completo da transcrição e do código
├── data-model.md                            # Phase 1: Entidades documentais e modelo de dados de webhooks
├── quickstart.md                            # Phase 1: Guia e comandos de validação dos critérios de aceite
├── checklists/
│   └── requirements.md                      # Checklist de qualidade da especificação
├── contracts/
│   ├── deliverables-contract.md             # Contrato estrutural dos 6 grupos de entregáveis
│   └── webhook-http-contracts.md            # Contratos HTTP (7 endpoints + outbound POST) e erros WEBHOOK_*
└── tasks.md                                 # Phase 2 output ($speckit-tasks - criado no próximo passo)
```

### Deliverables & Source Code Reference (repository root)

```text
.
├── README.md                                # [ENTREGÁVEL P6] Documentação final do processo com IA
├── TRANSCRICAO.md                           # [IMUTÁVEL - FONTE PRIMÁRIA] Gravação textual da reunião
├── docs/                                    # [DIRETÓRIO DE ENTREGA DOCUMENTAL]
│   ├── PRD.md                               # [ENTREGÁVEL P4] Product Requirements Document
│   ├── RFC.md                               # [ENTREGÁVEL P2] Request for Comments (arquitetura)
│   ├── FDD.md                               # [ENTREGÁVEL P3] Feature Design Document (implementação)
│   ├── TRACKER.md                           # [ENTREGÁVEL P5] Matriz de rastreabilidade cruzada
│   └── adrs/                                # [ENTREGÁVEL P1] Architecture Decision Records (6 ADRs)
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-separado-em-polling.md
│       ├── ADR-003-politica-de-retry-backoff-e-dlq.md
│       ├── ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md
│       ├── ADR-005-entrega-at-least-once-com-x-event-id.md
│       └── ADR-006-reuso-dos-padroes-existentes-do-oms.md
├── prisma/                                  # [IMUTÁVEL - REFERÊNCIA DE CÓDIGO]
│   └── schema.prisma                        # Modelos existentes: User, Customer, Product, Order, OrderItem, OrderStatusHistory
├── src/                                     # [IMUTÁVEL - REFERÊNCIA DE CÓDIGO]
│   ├── app.ts                               # Composição de controllers e middlewares
│   ├── server.ts                            # Entrypoint HTTP e graceful shutdown (base para entrypoint dedicado do worker proposto)
│   ├── config/
│   │   ├── database.ts                      # Factory createPrismaClient()
│   │   └── env.ts                           # Validação Zod de variáveis de ambiente
│   ├── middlewares/
│   │   ├── auth.middleware.ts               # authenticate e requireRole('ADMIN')
│   │   ├── error.middleware.ts              # Tratamento centralizado de AppError, ZodError e Prisma
│   │   ├── request-logger.middleware.ts     # Logging HTTP próprio via logger Pino
│   │   └── validate.middleware.ts           # Validação de body, query e params com Zod
│   ├── modules/
│   │   └── orders/
│   │       ├── order.controller.ts
│   │       ├── order.repository.ts
│   │       ├── order.routes.ts
│   │       ├── order.schemas.ts
│   │       ├── order.service.ts             # Ponto crítico de extensão transacional em changeStatus()
│   │       └── order.status.ts              # Máquina de estados de OrderStatus
│   ├── routes/
│   │   └── index.ts                         # Montagem de rotas sob /api/v1
│   └── shared/
│       ├── errors/
│       │   ├── app-error.ts                 # Classe base AppError
│       │   └── http-errors.ts               # Hierarquia de erros HTTP e de domínio
│       ├── http/
│       │   └── response.ts                  # Helper paginated()
│       └── logger/
│           └── index.ts                     # Logger estruturado Pino com redaction
└── tests/                                   # [IMUTÁVEL - REFERÊNCIA DE TESTES]
    ├── auth.test.ts
    └── orders.test.ts
```

**Structure Decision**: O projeto mantém intocada toda a árvore de código-fonte existente (`src/`, `prisma/`, `tests/`) como referência de leitura e concentra a produção nos diretórios `docs/` (`docs/adrs/ADR-001..006-*.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/PRD.md`, `docs/TRACKER.md`) e `README.md`, seguindo a ordem sequencial de 6 fases definida nas User Stories P1 a P6 da especificação.

## Complexity Tracking

A revisão posterior corrigiu inconsistências não detectadas pelos gates iniciais. As verificações e seus limites estão em [review.md](./review.md); detalhes derivados estão explicitamente submetidos à revisão em RFC-REVIEW-01. Os rótulos PASS acima registram a avaliação do plano e não substituem a auditoria final.

**Sincronização após revisão**: o FDD é a fonte canônica dos fluxos, schema e contratos. O Tracker possui 266 unidades identificáveis (224 TRANSCRICAO e 42 CODIGO); a existência da referência é validada automaticamente e seu conteúdo passa por revisão semântica.
