# Phase 0 Research: Contextualização da Transcrição e do Código Base

**Feature**: `001-webhook-design-docs`
**Date**: 2026-10-21
**Sources Analyzed**:
- `TRANSCRICAO.md` (324 linhas, reunião técnica de ~55 min com Larissa, Marcos, Bruno, Diego e Sofia)
- Código-fonte do Order Management System (`package.json`, `prisma/schema.prisma`, `src/**`, `tests/**`)
- `README.md` (regras e critérios de aceite do desafio)

---

## 1. Decisões Arquiteturais Principais (Base para ADRs, RFC e FDD)

### Decision 1: Padrão Transactional Outbox no MySQL (`ADR-001`)
- **Decision**: Persistir eventos de mudança de status de pedidos em uma tabela `webhook_outbox` no próprio banco MySQL, dentro da mesma transação Prisma (`this.prisma.$transaction`) executada em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), salvando o snapshot do payload JSON já renderizado no momento da inserção e utilizando UUID como chave primária.
- **Rationale**: Garante atomicidade entre a mudança de status do pedido (`orders`, `order_status_history`, ajuste de estoque em `products`) e o registro do evento de notificação (`[09:06] Diego`, `[09:40] Bruno`). Se a transação principal commitar, o evento estará garantidamente gravado; se sofrer rollback, nenhum webhook fantasma será emitido (`[09:06] Diego`). O uso de snapshot JSON na inserção evita inconsistências caso o pedido mude novamente antes do envio (`[09:51] Bruno`, `[09:52] Larissa`, `[09:52] Diego`), e o ID em UUID mantém o padrão de chaves primárias de `prisma/schema.prisma` (`[09:51] Larissa`). Além disso, a filtragem de eventos ocorre na inserção: se nenhum webhook ativo do cliente assinar o novo status, nada é inserido na outbox (`[09:34] Bruno`, `[09:34] Diego`).
- **Alternatives considered**:
  1. **Disparo HTTP síncrono dentro de `OrderService.changeStatus`** (`[09:03] Larissa`, `[09:04] Bruno`, `[09:06] Diego`): Descartado porque a transação de mudança de status já é pesada e uma chamada HTTP externa lenta travaria outras transações ou exigiria rollback indevido da mudança de status caso o servidor do cliente estivesse indisponível.
  2. **Fila externa com Redis Streams / Redis Cluster** (`[09:07] Larissa`, `[09:07] Diego`): Descartado por introduzir complexidade operacional (*overengineering*) e nova infraestrutura para um time pequeno, sendo que o MySQL existente com índices em `status` e `created_at` atende com folga.
  3. **Renderização tardia do payload no momento do envio pelo worker** (`[09:51] Bruno`, `[09:52] Larissa`): Descartado porque mutações subsequentes no pedido alterariam o conteúdo histórico do evento correspondente à transição original.

### Decision 2: Worker em Processo Separado via Polling de 2 Segundos (`ADR-002`)
- **Decision**: Executar o processador de webhooks como um processo Node.js independente (`src/worker.ts`, acionado via script `npm run worker`), instanciando seu próprio `PrismaClient` conectado ao mesmo MySQL e realizando *polling* em loop a cada 2 segundos sobre os registros pendentes mais antigos em pequenos lotes (*batches*), operando inicialmente em instância única (*single-worker*).
- **Rationale**: Isola o ciclo de vida do worker em relação à API HTTP (`src/server.ts`), evitando perda de processamento em reinicializações da API (`[09:11] Diego`, `[09:11] Larissa`) e respeitando o isolamento de `PrismaClient` por processo (`[09:30] Bruno`). O intervalo de 2 segundos cumpre com folga o SLA de negócio exigido pelos clientes B2B ("abaixo de 10 segundos", `[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa`) e garante ordenação implícita por `order_id` (`created_at`) enquanto operar em *single-worker* (`[09:12] Diego`, `[09:13] Larissa`).
- **Alternatives considered**:
  1. **Worker embutido no mesmo processo da API Express** (`[09:11] Diego`): Descartado porque qualquer restart ou falha da API interromperia o processamento de entregas em andamento.
  2. **Triggers no MySQL para notificação reativa** (`[09:09] Bruno`, `[09:09] Diego`): Descartado porque o MySQL não possui mecanismo nativo equivalente ao `LISTEN/NOTIFY` do PostgreSQL; acionar processos externos via trigger exigiria gambiarras (arquivos ou chamadas HTTP do banco).
  3. **Múltiplos workers em paralelo com particionamento/lock pessimista já na v1** (`[09:12] Diego`, `[09:13] Diego`): Adiado para o futuro por não haver necessidade de escala imediata e porque os clientes não exigiram garantia de ordenação global (`[09:14] Marcos`).

### Decision 3: Política de Retry com Backoff Exponencial (5 Tentativas) e Dead-Letter Queue (`ADR-003`)
- **Decision**: Aplicar timeout HTTP de 10 segundos por tentativa de entrega (`[09:42] Diego`) e, em caso de falha, realizar até 5 retentativas com progressão de *backoff* de `1m → 5m → 30m → 2h → 12h` (cobrindo ~15 horas, `[09:15] Diego`, `[09:17] Diego`, `[09:17] Larissa`). Esgotadas as 5 tentativas, mover o evento para uma tabela dedicada `webhook_dead_letter` contendo payload, motivo da falha e timestamp (`[09:18] Diego`), disponibilizando reprocessamento manual via endpoint administrativo `POST /api/v1/admin/webhooks/dead-letter/:id/replay` protegido por `requireRole('ADMIN')` e auditado via log (`[09:18] Diego`, `[09:35] Diego`, `[09:36] Sofia`, `[09:36] Larissa`).
- **Rationale**: Cobre janelas reais de manutenção planejada dos clientes (histórico de indisponibilidades de ~2 horas, `[09:16] Diego`) sem manter eventos presos indefinidamente. Separar a DLQ em `webhook_dead_letter` mantém a leitura da tabela `webhook_outbox` leve e cria trilha de evidência para depuração e replay controlado por administradores (`[09:18] Diego`, `[09:36] Sofia`).
- **Alternatives considered**:
  1. **Retry indefinido com backoff** (`[09:15] Diego`): Descartado por acumular eventos eternamente caso o endpoint do cliente deixe de existir.
  2. **Limite curto de 3 tentativas (~30 minutos)** (`[09:16] Bruno`, `[09:16] Diego`): Descartado por ser agressivo demais e descartar eventos durante manutenções de rotina de clientes (ex.: 2 horas).
  3. **Manter eventos falhos permanentemente na própria tabela `webhook_outbox` apenas com status `failed`** (`[09:17] Larissa`, `[09:18] Diego`): Descartado para não degradar a leitura da tabela operacional de outbox e para isolar os dados de auditoria de falha permanente.

### Decision 4: Autenticação HMAC-SHA256 com Secret Única por Endpoint e Rotação com Grace Period (`ADR-004`)
- **Decision**: Assinar o corpo JSON de cada requisição de webhook com `HMAC-SHA256` utilizando uma `secret` exclusiva gerada pelo OMS para cada endpoint cadastrado, enviando a assinatura no cabeçalho `X-Signature` (junto a `X-Timestamp` e `X-Webhook-Id`, `[09:20] Sofia`, `[09:21] Sofia`, `[09:44] Diego`, `[09:44] Sofia`). Suportar rotação de `secret` via API mantendo a chave anterior válida em paralelo por um *grace period* de 24 horas (`[09:21] Sofia`, `[09:22] Sofia`), além de exigir URL `https://` validada via schema Zod (`[09:23] Sofia`) e limite máximo de 64KB por payload (`[09:24] Diego`, `[09:24] Larissa`).
- **Rationale**: Garante autenticidade e integridade do payload em trânsito contra adulteração e *replay attacks* usando padrão amplamente suportado no mercado (`[09:19] Sofia`, `[09:20] Sofia`). Isolar a `secret` por endpoint evita comprometimento global caso um cliente vaze sua chave em logs (`[09:21] Sofia`, `[09:22] Diego`), e o período de graça de 24h permite rotação sem downtime na integração do cliente (`[09:21] Sofia`).
- **Alternatives considered**:
  1. **Secret global única para toda a plataforma** (`[09:21] Sofia`): Descartado pelo alto risco de segurança (vazamento em um cliente comprometeria todos os demais).
  2. **Rotação imediata de secret sem período de transição (grace period)** (`[09:21] Sofia`): Descartado porque quebraria a validação no cliente enquanto ele atualiza suas credenciais em produção.
  3. **Permitir URLs HTTP sem TLS ou truncar payloads acima do limite** (`[09:23] Sofia`): Descartados; HTTP é recusado na validação Zod e payloads $> 64\text{KB}$ geram erro explícito em vez de truncamento silencioso (`[09:23] Sofia`, `[09:24] Larissa`).

### Decision 5: Garantia de Entrega At-Least-Once com Idempotência via `X-Event-Id` (`ADR-005`)
- **Decision**: Adotar semântica de entrega *at-least-once* (pelo menos uma vez), gerando um UUID único por evento no momento da entrada na `webhook_outbox` e transmitindo-o tanto no cabeçalho `X-Event-Id` quanto no campo `event_id` do payload JSON, delegando ao cliente receptor a deduplicação por `event_id` (`[09:24] Diego`, `[09:25] Diego`, `[09:26] Larissa`, `[09:43] Diego`).
- **Rationale**: Em sistemas distribuídos com retentativas sob falhas de rede ou timeouts (>10s), um cliente pode processar o evento mas falhar ao responder a tempo, ocasionando reenvio. A entrega *at-least-once* combinada com identificador único `X-Event-Id` segue o padrão da indústria (Stripe, GitHub) sem o custo proibitivo de coordenação transacional distribuída (`[09:25] Diego`).
- **Alternatives considered**:
  1. **Garantia *exactly-once* coordenada entre OMS e cliente** (`[09:25] Diego`): Descartado pela alta complexidade de protocolo bilateral com sistemas externos heterogêneos.
  2. **Entrega *best-effort* (*at-most-once*) sem retry/deduplicação** (`[09:02] Marcos`, `[09:15] Diego`): Descartado porque perder notificações obrigaria os clientes a manterem o polling manual em `GET /api/v1/orders`.

### Decision 6: Reuso Integral dos Padrões Arquiteturais e Módulos do OMS (`ADR-006`)
- **Decision**: Estruturar o novo domínio em `src/modules/webhooks/` seguindo a convenção modular existente (`controller`, `service`, `repository`, `routes`, `schemas`), acoplar a publicação de eventos em `src/modules/orders/order.service.ts` através de uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `Prisma.TransactionClient` (`[09:41] Bruno`, `[09:41] Diego`), reutilizar a hierarquia `AppError` (`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`) com prefixo `WEBHOOK_*` tratado automaticamente por `src/middlewares/error.middleware.ts`, validar entradas com `src/middlewares/validate.middleware.ts` (Zod), controlar acesso com `authenticate` e `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`) e registrar logs estruturados com `src/shared/logger/index.ts` (Pino) (`[09:27] Bruno` a `[09:30] Larissa`).
- **Rationale**: Mantém coesão arquitetural com os módulos existentes (`auth`, `users`, `customers`, `products`, `orders`), reduz esforço de implementação (estimado em 3 sprints, `[09:46] Larissa`) e evita acoplamento excessivo no `OrderService` ao usar função pura com `tx` em vez de injetar todo o repositório de webhooks (`[09:41] Bruno`, `[09:41] Diego`).
- **Alternatives considered**:
  1. **Injetar `WebhookRepository` completo no construtor do `OrderService`** (`[09:41] Bruno`, `[09:41] Diego`): Descartado em favor da função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` para manter baixo acoplamento entre módulos.
  2. **Introduzir novo framework de logging, validação ou tratamento de erros separado para webhooks/worker** (`[09:29] Bruno`, `[09:30] Larissa`): Descartado porque `Pino`, `Zod` e `AppError`/`errorMiddleware` já atendem integralmente aos requisitos.

---

## 2. Mapeamento de Exclusões de Escopo e Questões em Aberto

| Item | Classificação na Reunião | Localização na Transcrição | Onde Documentar |
| :--- | :--- | :--- | :--- |
| **Inbound Webhooks** (receber webhooks dos clientes) | Fora de escopo (apenas *outbound*) | `[09:02] Sofia`, `[09:02] Marcos`, `[09:03] Sofia` | PRD (Fora de escopo) |
| **Arquivamento/expurgo de eventos entregues (>30 dias)** | Fora de escopo desta feature | `[09:08] Diego` | PRD (Fora de escopo) / RFC (Questões em aberto) |
| **Notificação por e-mail ao cliente após falhas seguidas** | Fora de escopo (adiado para próxima fase) | `[09:37] Marcos`, `[09:37] Larissa`, `[09:38] Marcos` | PRD (Fora de escopo) / RFC (Questões em aberto) |
| **Dashboard/painel visual para gestão de webhooks** | Fora de escopo (projeto separado de frontend) | `[09:39] Marcos`, `[09:40] Larissa` | PRD (Fora de escopo) |
| **Rate limiting de envio de webhooks para o cliente** | Questão em aberto ("observar e decidir depois") | `[09:38] Diego`, `[09:39] Diego`, `[09:39] Larissa` | RFC (Questões em aberto) / PRD (Fora de escopo) |
| **Escalonamento horizontal multi-worker com particionamento por `order_id` ou lock pessimista** | Questão em aberto / evolução futura | `[09:12] Diego`, `[09:13] Bruno`, `[09:13] Diego`, `[09:13] Larissa` | RFC (Questões em aberto) / ADR-002 (Limitação conhecida) |
| **Restrição granular de permissões (RBAC) no CRUD de webhooks** | Questão em aberto ("por enquanto qualquer role autenticada, endurecer mais pra frente") | `[09:36] Marcos`, `[09:37] Sofia` | RFC (Questões em aberto) |

---

## 3. Mapeamento de Arquivos Reais do Código Base para Integração (`FDD` e `ADRs`)

| Caminho Real no Repositório | Papel Atual no Código | Integração Especificada para Webhooks |
| :--- | :--- | :--- |
| `prisma/schema.prisma` | Define enums (`UserRole`, `OrderStatus`) e modelos (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`) com IDs `@id @default(uuid()) @db.Char(36)`. | Adição dos modelos `WebhookEndpoint` (`webhook_endpoints`), `WebhookOutbox` (`webhook_outbox`), `WebhookDelivery` (`webhook_deliveries`) e `WebhookDeadLetter` (`webhook_dead_letter`), relacionando `Customer`, `Order` e `User`. |
| `src/modules/orders/order.service.ts` | Contém `OrderService.changeStatus` (linhas 126–179) que executa `this.prisma.$transaction` validando transição (`canTransition`), debitando/repondo estoque, atualizando `tx.order.update` e criando `tx.orderStatusHistory.create`. | Invocação de `publishWebhookEvent(tx, refreshedOrder, from, to)` dentro do mesmo bloco `this.prisma.$transaction` logo após registrar o histórico de status, garantindo atomicidade total (`[09:40] Bruno`, `[09:41] Bruno`). |
| `src/modules/orders/order.status.ts` | Define o grafo de transições válidas do enum `OrderStatus` (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) e regras de estoque. | Reutilização do enum `OrderStatus` para validar a lista de eventos assinados (`events`) nos schemas do webhook e filtrar na inserção da outbox (`[09:33] Marcos`, `[09:34] Bruno`). |
| `src/shared/errors/app-error.ts` & `src/shared/errors/http-errors.ts` | Definem a classe base `AppError(message, statusCode, errorCode, details)` e subclasses (`BadRequestError`, `ValidationError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, etc.). | Criação/uso de erros de domínio com prefixo `WEBHOOK_*` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`) herdando de `AppError` (`[09:28] Bruno`, `[09:29] Larissa`). |
| `src/middlewares/error.middleware.ts` | Intercepta `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` (`P2002`, `P2025`), formatando resposta JSON padronizada `{ error: { code, message, details } }` e logando erros 500 via Pino. | Processa automaticamente todas as exceções `WEBHOOK_*` e erros de validação Zod das novas rotas sem necessidade de alterar o middleware (`[09:29] Bruno`). |
| `src/middlewares/auth.middleware.ts` | Exporta `authenticate` (validação Bearer JWT populando `req.user` com `id`, `email`, `role`) e `requireRole(...roles)`. | Protege todas as rotas de `/api/v1/webhooks` com `authenticate` e restringe `POST /api/v1/admin/webhooks/dead-letter/:id/replay` com `requireRole('ADMIN')` (`[09:32] Larissa`, `[09:36] Larissa`). |
| `src/middlewares/validate.middleware.ts` | Middleware `validate({ body, query, params })` usando schemas Zod e convertendo falhas em `ValidationError`. | Valida payloads de criação/edição de webhooks (incluindo obrigatoriedade de `https://` na URL e filtro de `OrderStatus`), parâmetros `:id` UUID e paginação (`[09:23] Sofia`, `[09:30] Larissa`). |
| `src/shared/logger/index.ts` | Configura o logger `pino` com redaction de dados sensíveis e timestamp ISO. | Reutilizado no módulo `webhooks` (incluindo log de auditoria de quem executou replay de DLQ em `[09:36] Sofia` e adição de redaction para `*.secret`) e no processo separado `src/worker.ts` (`[09:29] Bruno`). |
| `src/routes/index.ts` & `src/app.ts` | Composição de `Controllers` (`buildControllers`) e montagem de sub-rotas em `/api/v1` (`buildApiRouter`). | Registro do `WebhookController` e montagem das rotas `/webhooks` e `/admin/webhooks` sob `/api/v1`. |
| `src/server.ts` & `src/config/database.ts` | Entrypoint da API HTTP com graceful shutdown (`SIGINT`/`SIGTERM`) e factory `createPrismaClient()`. | Referência direta para o novo entrypoint `src/worker.ts` (`[09:11] Larissa`), que instancia seu próprio `PrismaClient` via `createPrismaClient()` (`[09:30] Bruno`) e gerencia graceful shutdown do loop de polling. |
| `src/shared/http/response.ts` | Utilitário `paginated(data, page, pageSize, total)` para respostas paginadas. | Reutilizado na listagem de webhooks (`GET /api/v1/webhooks`) e histórico de entregas (`GET /api/v1/webhooks/:id/deliveries`). |

---

## 4. Resolução de Incertezas

Todas as questões técnicas e de negócio foram integralmente esclarecidas pelo cruzamento entre `TRANSCRICAO.md`, `README.md` e o código-fonte. Nenhum marcador `NEEDS CLARIFICATION` permanece pendente.
