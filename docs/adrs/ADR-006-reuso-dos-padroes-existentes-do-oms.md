# ADR-006: Reuso Integral dos Padrões Arquiteturais e Infraestrutura Compartilhada do OMS

- **ID de Rastreabilidade**: `ADR-006`
- **Data**: 2026-10-21
- **Decisores**: Bruno (Engenheiro Pleno — Pedidos), Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Sofia (Engenheira de Segurança)
- **Referências de Código**: `src/app.ts`, `src/routes/index.ts`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`, `src/config/database.ts`

---

## Status

Aceito (Decidido na reunião técnica — `[09:27] Bruno`, `[09:28] Bruno`, `[09:29] Larissa`, `[09:30] Larissa`, `[09:36] Larissa`, `[09:41] Diego`).

---

## Contexto

O repositório do Order Management System já possui uma arquitetura modular madura e bem padronizada em TypeScript:
- **`ADR-006-CTX-01`** — **Estrutura de Módulos (`src/modules/*`)**: Os domínios `auth`, `users`, `customers`, `products` e `orders` seguem organização modular com controllers, services, routes e schemas; os domínios de dados possuem repositories, enquanto a autenticação reutiliza o repositório de usuários. Os controllers são compostos em `src/app.ts` (`buildControllers`) e montados sob `/api/v1` em `src/routes/index.ts` (`buildApiRouter`).
- **`ADR-006-CTX-02`** — **Hierarquia e Tratamento Centralizado de Erros**: `src/shared/errors/app-error.ts` define `AppError(message, statusCode, errorCode, details)` e `src/shared/errors/http-errors.ts` define subclasses HTTP e de domínio (`NotFoundError`, `ConflictError`, `ValidationError`, `UnprocessableEntityError`, `InvalidStatusTransitionError`, `InsufficientStockError`). O middleware `src/middlewares/error.middleware.ts` intercepta automaticamente `AppError`, `ZodError` e erros do Prisma (`P2002`, `P2025`), formatando `{ error: { code, message, details } }`.
- **`ADR-006-CTX-03`** — **Autenticação, Autorização e Validação**: `src/middlewares/auth.middleware.ts` fornece `authenticate` (JWT Bearer) e `requireRole(...roles)` para RBAC (`ADMIN` | `OPERATOR`); `src/middlewares/validate.middleware.ts` aplica validação de `body`, `query` e `params` com schemas Zod; `src/shared/http/response.ts` padroniza paginação via `paginated()`.
- **`ADR-006-CTX-04`** — **Logging e Banco de Dados**: `src/shared/logger/index.ts` provê logger estruturado em JSON via `pino` com *redaction* de campos sensíveis, e `src/config/database.ts` encapsula a factory `createPrismaClient()`.

Ao introduzir o módulo de webhooks e o worker de entregas, precisamos garantir total coesão com esses componentes e definir como `OrderService` (`src/modules/orders/order.service.ts`) se comunicará com o novo módulo sem criar acoplamento excessivo (`[09:27] Bruno`, `[09:41] Bruno`).

---

## Decisão

Determinamos o **reuso integral dos padrões, middlewares e utilitários existentes no OMS**, estruturando a feature conforme as seguintes diretrizes (`ADR-006`):

1. **`ADR-006-DEC-01`** — **Módulo de Webhooks Padronizado (`[09:27] Bruno`, `[09:28] Bruno`, `[09:30] Larissa`)**: O novo domínio seguirá a anatomia dos demais módulos: controller, service, repository, routes e schemas, com um processador de entregas. Será registrado em `buildControllers` (`src/app.ts`) e montado em `buildApiRouter` (`src/routes/index.ts`).
2. **`ADR-006-DEC-02`** — **Desacoplamento Transacional via Função Pura `publishWebhookEvent` (`[09:40] Bruno`, `[09:41] Bruno`, `[09:41] Diego`)**: Em vez de injetar `WebhookRepository` ou `WebhookService` no construtor de `OrderService`, o módulo de webhooks exportará uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `Prisma.TransactionClient` (`tx`) ativo em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`).
3. **`ADR-006-DEC-03`** — **Erros de Domínio Herdando de `AppError` com Prefixo `WEBHOOK_` (`[09:28] Bruno`, `[09:29] Larissa`, `[09:29] Bruno`)**: Todos os erros de domínio do módulo herdarão de `AppError` / subclasses de `src/shared/errors/http-errors.ts` utilizando códigos padronizados com prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`), sendo capturados sem qualquer alteração por `src/middlewares/error.middleware.ts`.
4. **`ADR-006-DEC-04`** — **Reuso de `validate`, `authenticate`, `requireRole('ADMIN')`, `paginated` e `logger` (`[09:23] Sofia`, `[09:29] Bruno`, `[09:30] Larissa`, `[09:36] Larissa`)**:
   - Schemas Zod validados pelo adaptador local descrito no FDD, reutilizando `src/middlewares/validate.middleware.ts` (importando `OrderStatus` de `@prisma/client`, definido em `prisma/schema.prisma`; `src/modules/orders/order.status.ts` contém as funções de transição e não exporta o enum);
   - Proteção de rotas CRUD com `authenticate` e proteção de `POST /api/v1/admin/webhooks/dead-letter/:id/replay` com `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`);
   - Logging estruturado na API e no entrypoint dedicado do worker usando `src/shared/logger/index.ts` (Pino).

---

## Alternativas Consideradas

### 1. Injetar `WebhookRepository` Completo no Construtor de `OrderService` (`ADR-006-ALT-01`)
- **Descrição**: Alterar a assinatura de `new OrderService(orderRepository, prisma, webhookRepository)` em `src/modules/orders/order.service.ts` e `src/app.ts` para injetar o repositório completo de webhooks (`[09:41] Bruno`).
- **Motivo do Descarte (`[09:41] Bruno`, `[09:41] Diego`)**: Aumentaria o acoplamento entre os módulos `orders` e `webhooks`, expondo métodos de CRUD de webhooks ao serviço de pedidos e quebrando a assinatura atual de instanciação nos testes existentes (`tests/orders.test.ts`). A função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` mantém o contrato mínimo necessário.

### 2. Criar Tratamento de Erros, Logger ou Validação Próprios para o Módulo de Webhooks (`ADR-006-ALT-02`)
- **Descrição**: Utilizar bibliotecas ou formatos de resposta distintos para erros de webhook e logs do worker (`[09:28] Bruno`, `[09:29] Bruno`).
- **Motivo do Descarte (`[09:29] Bruno`, `[09:30] Larissa`)**: Criaria inconsistência nos contratos JSON da API (`{ error: { code, message, details } }`), duplicaria código já testado em `src/middlewares/error.middleware.ts` e fragmentaria a observabilidade que já utiliza Pino (`src/shared/logger/index.ts`).

---

## Consequências

### Positivas
- **`ADR-006-CONS-01`** — **Zero Mudança Estrutural nos Middlewares Existentes**: `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts` e `src/middlewares/validate.middleware.ts` funcionam imediatamente para todas as rotas de webhooks (`[09:29] Bruno`, `[09:36] Larissa`).
- **`ADR-006-CONS-02`** — **Baixo Acoplamento em `OrderService`**: A integração em `src/modules/orders/order.service.ts` resume-se a uma única chamada `await publishWebhookEvent(tx, refreshed, from, to)` dentro da transação existente (`[09:41] Bruno`, `[09:41] Diego`).
- **`ADR-006-CONS-03`** — **Previsibilidade de Prazo**: O reuso máximo permite entregar toda a solução (outbox, worker, CRUD, deliveries, DLQ e revisão de segurança) dentro da estimativa de 3 sprints (`[09:46] Larissa`).

### Negativas
- **`ADR-006-CONS-04`** — **Compartilhamento do Banco de Dados e Schema Prisma**: As novas tabelas (`webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`) passam a coexistir no mesmo `prisma/schema.prisma` do core transacional do OMS (`[09:11] Bruno`, `[09:30] Bruno`).

### Trade-off Explícito (`ADR-006-TRD-01`)
Trocamos a **autonomia de um microsserviço independente de notificações** pela **coesão arquitetural e velocidade de entrega de um módulo interno (módulo de webhooks) integrado via função pura transacional (`publishWebhookEvent`)**, aceitando compartilhar o mesmo schema Prisma e codebase em troca de zero duplicação de infraestrutura, autenticação, validação e tratamento de erros (`[09:27] Bruno`, `[09:30] Larissa`, `[09:41] Diego`).

**Compatibilidade verificada no código**: `ValidationError` fixa `VALIDATION_ERROR` e `NotFoundError` fixa `NOT_FOUND`. A proposta `FDD-DET-06` usa `AppError`/construtores parametrizáveis para códigos de domínio `WEBHOOK_*` e um adaptador local de `validate`; erros estruturais, de autenticação e de autorização mantêm os códigos genéricos existentes. Nenhum middleware compartilhado precisa mudar.
