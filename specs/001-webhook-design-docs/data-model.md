# Phase 1 Data Model: Entidades Documentais e Modelo de Dados da Feature

**Feature**: `001-webhook-design-docs`
**Date**: 2026-10-21

---

## Parte A — Modelo de Entidades do Pacote Documental (Rastreabilidade e Governança)

### 1. `DocumentArtifact` (Artefato do Pacote de Design Docs)
Representa cada arquivo Markdown entregue no repositório como parte da solução documental.

| Atributo | Tipo | Regra de Validação |
| :--- | :--- | :--- |
| `path` | `string` | Caminho relativo obrigatório (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, `docs/adrs/ADR-001..006-*.md`, `README.md`). |
| `layer` | `enum` | `PRODUCT` (`PRD`), `ARCHITECTURE` (`RFC`), `DECISION` (`ADR`), `IMPLEMENTATION` (`FDD`), `TRACEABILITY` (`TRACKER`), `PROCESS` (`README`). |
| `requiredSections` | `string[]` | Lista de seções obrigatórias definidas no `README.md` e na `constitution.md` (100% obrigatórias). |
| `traceableItems` | `TrackerEntry[]` | Conjunto de requisitos, decisões, alternativas, contratos e restrições contidos no documento ($\ge 80\%$ mapeados no `TRACKER.md`). |

### 2. `TrackerEntry` (Linha em `docs/TRACKER.md`)
Representa o vínculo auditável entre uma afirmação nos documentos e sua fonte primária.

| Coluna na Tabela | Formato / Domínio | Regra de Validação |
| :--- | :--- | :--- |
| `ID` | `^(PRD|RFC|FDD|ADR)-[A-Z0-9-]+$` | Identificador único (ex.: `PRD-FR-01`, `PRD-NFR-01`, `PRD-OOS-01`, `PRD-RISK-01`, `RFC-ALT-01`, `RFC-OPEN-01`, `FDD-EP-01`, `FDD-ERR-01`, `FDD-INT-01`, `ADR-001`). |
| `Documento` | Caminho relativo | Deve apontar para um arquivo existente em `docs/` (ex.: `docs/PRD.md`, `docs/adrs/ADR-001-outbox-no-mysql.md`). |
| `Tipo` | `string` | `Requisito Funcional`, `Requisito Não Funcional`, `Fora de Escopo`, `Objetivo/Métrica`, `Risco`, `Decisão`, `Alternativa Descartada`, `Questão em Aberto`, `Trade-off`, `Contrato HTTP`, `Erro`, `Integração com Código`, `Resiliência/Observabilidade`. |
| `Conteúdo (resumo)` | `string` | Descrição concisa de 1 linha sem contradizer a fonte primária. |
| `Fonte` | `TRANSCRICAO` \| `CODIGO` | Pelo menos 70% das linhas totais devem ser `TRANSCRICAO`; pelo menos 5 linhas devem ser `CODIGO`. |
| `Localização` | Timestamp ou Filepath | Se `TRANSCRICAO`: formato `[hh:mm] Nome` existente em `TRANSCRICAO.md`. Se `CODIGO`: caminho relativo existente no repositório (`src/...` ou `prisma/...`). |

---

## Parte B — Modelo de Dados de Domínio do Sistema de Webhooks (Especificado para `docs/FDD.md`)

> **Nota de Imutabilidade**: O arquivo `prisma/schema.prisma` real **não deve ser modificado**. O modelo abaixo descreve a especificação técnica que será documentada no `docs/FDD.md` (e referenciada nos ADRs/RFC) estendendo o schema existente do OMS.

### 1. Enumerações de Domínio Novas

- **`WebhookOutboxStatus`**:
  - `PENDING`: Evento inserido na transação do pedido ou reenfileirado via replay de DLQ, aguardando coleta pelo worker (`[09:08] Diego`, `[09:18] Diego`).
  - `PROCESSING`: Evento reservado pelo worker no ciclo atual de polling (`[09:08] Diego`).
  - `DELIVERED`: Evento entregue com sucesso (HTTP 2xx em até 10s) ao endpoint do cliente (`[09:08] Diego`).
  - `FAILED`: Evento que falhou na tentativa atual ou esgotou as 5 tentativas e foi movido para `webhook_dead_letter` (`[09:08] Diego`, `[09:15] Diego`).

### 2. Entidade `WebhookEndpoint` (Tabela `webhook_endpoints`)
Representa a assinatura de webhook configurada para um cliente B2B (`Customer`).

| Campo | Tipo Prisma / MySQL | Restrições e Regras (Rastreabilidade) |
| :--- | :--- | :--- |
| `id` | `String @id @default(uuid()) @db.Char(36)` | UUID conforme padrão do projeto (`[09:51] Larissa`, `prisma/schema.prisma`). |
| `customerId` | `String @db.Char(36)` | FK para `Customer.id` (`customers`); passado no body/query pois o JWT é do operador/usuário (`[09:21] Bruno`, `[09:32] Larissa`). |
| `url` | `String @db.VarChar(500)` | Deve iniciar obrigatoriamente com `https://` validado via Zod (`[09:23] Sofia`). |
| `secret` | `String @db.VarChar(128)` | Secret única por endpoint gerada pelo OMS para assinatura `HMAC-SHA256` (`[09:21] Sofia`, `[09:31] Marcos`). |
| `previousSecret` | `String? @db.VarChar(128)` | Secret anterior mantida durante rotação (`[09:21] Sofia`). |
| `previousSecretExpiresAt` | `DateTime?` | Timestamp de expiração da secret anterior (`now + 24h` no momento da rotação, `[09:21] Sofia`). |
| `events` | `Json` (`OrderStatus[]`) | Lista de status de `OrderStatus` que este endpoint assina (`[09:31] Marcos`, `[09:33] Marcos`). |
| `active` | `Boolean @default(true)` | Flag de ativação do endpoint (`[09:21] Bruno`). |
| `createdAt` | `DateTime @default(now())` | Auditoria padrão (`prisma/schema.prisma`). |
| `updatedAt` | `DateTime @updatedAt` | Auditoria padrão (`prisma/schema.prisma`). |

- **Índices**: `@@index([customerId, active])` para busca rápida durante `OrderService.changeStatus` (`[09:34] Bruno`).

### 3. Entidade `WebhookOutbox` (Tabela `webhook_outbox`)
Armazena os eventos de mudança de status gerados atomicamente na transação do pedido.

| Campo | Tipo Prisma / MySQL | Restrições e Regras (Rastreabilidade) |
| :--- | :--- | :--- |
| `id` | `String @id @default(uuid()) @db.Char(36)` | UUID único do evento, enviado como `X-Event-Id` e `event_id` no payload (`[09:25] Diego`, `[09:51] Larissa`). |
| `webhookId` | `String @db.Char(36)` | FK para `WebhookEndpoint.id`, enviado no header `X-Webhook-Id` (`[09:44] Sofia`). |
| `orderId` | `String @db.Char(36)` | FK para `Order.id` (`[09:43] Diego`). |
| `eventType` | `String @db.VarChar(64)` | Tipo do evento, ex.: `"order.status_changed"` (`[09:43] Diego`). |
| `payload` | `Json` | Snapshot JSON renderizado no momento da inserção, máximo de 64KB (`[09:24] Diego`, `[09:43] Diego`, `[09:52] Larissa`). |
| `status` | `WebhookOutboxStatus @default(PENDING)` | Estado do evento na outbox (`[09:08] Diego`). |
| `attempts` | `Int @default(0)` | Contador de tentativas realizadas (máximo de 5 retentativas após falha inicial, `[09:15] Diego`, `[09:17] Larissa`). |
| `nextAttemptAt` | `DateTime @default(now())` | Agendamento da próxima execução segundo backoff `1m, 5m, 30m, 2h, 12h` (`[09:17] Diego`). |
| `lastError` | `String? @db.VarChar(500)` | Último erro de transporte/HTTP registrado. |
| `deliveredAt` | `DateTime?` | Timestamp de conclusão com sucesso (`[09:08] Diego`). |
| `createdAt` | `DateTime @default(now())` | Usado para ordenação FIFO por `created_at` no worker (`[09:08] Diego`, `[09:12] Diego`). |
| `updatedAt` | `DateTime @updatedAt` | Timestamp da última atualização. |

- **Índices**: `@@index([status, nextAttemptAt, createdAt])` e `@@index([orderId, createdAt])` (`[09:08] Diego`, `[09:12] Diego`).

#### Máquina de Estados de `WebhookOutbox`:
- `PENDING` $\rightarrow$ `PROCESSING`: Coletado pelo worker no ciclo de polling de 2s (`[09:09] Diego`).
- `PROCESSING` $\rightarrow$ `DELIVERED`: Endpoint do cliente respondeu HTTP `2xx` dentro do timeout de 10s (`[09:08] Diego`, `[09:42] Diego`).
- `PROCESSING` $\rightarrow$ `PENDING`: Falha de entrega (timeout > 10s, erro de rede ou HTTP $\ge 400$) com `attempts < 5`; incrementa `attempts` e agenda `nextAttemptAt` conforme curva `1m / 5m / 30m / 2h / 12h` (`[09:15] Diego`, `[09:17] Diego`).
- `PROCESSING` $\rightarrow$ `FAILED`: Falha na 5ª tentativa; cria registro correspondente em `webhook_dead_letter` (`[09:15] Diego`, `[09:18] Diego`).

### 4. Entidade `WebhookDelivery` (Tabela `webhook_deliveries`)
Registra o histórico individual de cada tentativa de disparo HTTP para consulta pelo cliente (`GET /api/v1/webhooks/:id/deliveries`, últimos 100 disparos, `[09:34] Marcos`).

| Campo | Tipo Prisma / MySQL | Restrições e Regras (Rastreabilidade) |
| :--- | :--- | :--- |
| `id` | `String @id @default(uuid()) @db.Char(36)` | UUID da tentativa de entrega (`[09:51] Larissa`). |
| `webhookId` | `String @db.Char(36)` | FK para `WebhookEndpoint.id` (`[09:34] Marcos`). |
| `outboxId` | `String @db.Char(36)` | FK para `WebhookOutbox.id`. |
| `eventId` | `String @db.Char(36)` | UUID do evento (`X-Event-Id`). |
| `attemptNumber` | `Int` | Número da tentativa (`1..5`). |
| `success` | `Boolean` | Indica sucesso (`true`) ou falha (`false`) (`[09:34] Marcos`). |
| `requestPayload` | `Json` | Payload enviado na tentativa (`[09:34] Marcos`). |
| `responseStatus` | `Int?` | Status HTTP retornado pelo cliente (ou `null` em caso de timeout/erro de rede) (`[09:34] Marcos`). |
| `responseBody` | `String? @db.Text` | Corpo da resposta capturado para diagnóstico (`[09:34] Marcos`). |
| `durationMs` | `Int` | Tempo de resposta em milissegundos (`[09:34] Marcos`). |
| `errorMessage` | `String? @db.VarChar(500)` | Motivo da falha caso `success = false`. |
| `createdAt` | `DateTime @default(now())` | Timestamp do disparo (`@@index([webhookId, createdAt(sort: Desc)])`). |

### 5. Entidade `WebhookDeadLetter` (Tabela `webhook_dead_letter`)
Armazena eventos que esgotaram as 5 tentativas de entrega para diagnóstico e reprocessamento manual por `ADMIN` (`[09:18] Diego`, `[09:35] Diego`, `[09:36] Sofia`).

| Campo | Tipo Prisma / MySQL | Restrições e Regras (Rastreabilidade) |
| :--- | :--- | :--- |
| `id` | `String @id @default(uuid()) @db.Char(36)` | UUID do registro na DLQ (`[09:18] Diego`, `[09:51] Larissa`). |
| `outboxId` | `String @unique @db.Char(36)` | Referência ao evento original na `webhook_outbox`. |
| `webhookId` | `String @db.Char(36)` | FK para `WebhookEndpoint.id`. |
| `orderId` | `String @db.Char(36)` | FK para `Order.id`. |
| `payload` | `Json` | Snapshot original do evento que falhou (`[09:18] Diego`). |
| `failureReason` | `String @db.VarChar(500)` | Motivo detalhado da última falha (`[09:18] Diego`). |
| `attempts` | `Int` | Total de tentativas executadas até a falha permanente (`5`, `[09:17] Larissa`). |
| `failedAt` | `DateTime @default(now())` | Timestamp de entrada na DLQ (`[09:18] Diego`). |
| `replayedAt` | `DateTime?` | Timestamp do reprocessamento manual via endpoint admin (`[09:18] Diego`). |
| `replayedById` | `String? @db.Char(36)` | FK para `User.id` (`ADMIN` que executou o replay, para auditoria exigida em `[09:36] Sofia`). |
