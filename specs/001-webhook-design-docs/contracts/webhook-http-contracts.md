# Contract 2: Contratos HTTP e Catálogo de Erros do Sistema de Webhooks (Base para `docs/FDD.md`)

**Feature**: `001-webhook-design-docs`
**Date**: 2026-10-21

Este arquivo consolida os contratos de interface HTTP (endpoints de gestão da API OMS e disparo outbound do worker) e a matriz de códigos de erro `WEBHOOK_*` extraídos de `TRANSCRICAO.md` e alinhados às convenções de `src/app.ts`, `src/routes/index.ts`, `src/shared/http/response.ts` e `src/middlewares/error.middleware.ts`.

---

## 1. Endpoints da API REST (`/api/v1`)

> **Convenção de Rotas e Autenticação**: Todas as rotas da API são montadas sob `/api/v1` (`src/app.ts`) e exigem o middleware `authenticate` (`Authorization: Bearer <jwt>`, `src/middlewares/auth.middleware.ts`). Como o JWT pertence ao usuário autenticado (`ADMIN` ou `OPERATOR`), o `customerId` é informado explicitamente no corpo ou query (`[09:32] Bruno`, `[09:32] Marcos`, `[09:32] Larissa`).

### EP-01: Cadastrar Endpoint de Webhook
- **Método / Rota**: `POST /api/v1/webhooks`
- **Permissão**: Qualquer role autenticada (`OPERATOR` ou `ADMIN`, `[09:36] Marcos`, `[09:37] Sofia`)
- **Origem na Transcrição**: `[09:21] Bruno`, `[09:23] Sofia`, `[09:31] Marcos`, `[09:32] Larissa`, `[09:33] Marcos`
- **Request Body**:
  ```json
  {
    "customerId": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://api.atlascomercial.com.br/webhooks/oms",
    "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"],
    "active": true
  }
  ```
- **Response `201 Created`** (retorna a `secret` gerada pelo sistema na criação, `[09:31] Marcos`):
  ```json
  {
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "customerId": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://api.atlascomercial.com.br/webhooks/oms",
    "secret": "whsec_9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
    "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"],
    "active": true,
    "createdAt": "2026-10-21T12:00:00.000Z",
    "updatedAt": "2026-10-21T12:00:00.000Z"
  }
  ```
- **Status Codes**: `201 Created`, `400 Bad Request` (`VALIDATION_ERROR` / `WEBHOOK_INVALID_URL` se não for `https://`), `401 Unauthorized`, `404 Not Found` (`WEBHOOK_CUSTOMER_NOT_FOUND`).

### EP-02: Listar Endpoints de Webhook de um Cliente
- **Método / Rota**: `GET /api/v1/webhooks?customerId=<uuid>&page=1&pageSize=20`
- **Permissão**: Qualquer role autenticada (`OPERATOR` ou `ADMIN`)
- **Origem na Transcrição**: `[09:33] Bruno` e padrão `paginated` em `src/shared/http/response.ts`
- **Response `200 OK`** (sem expor a `secret` em texto plano após a criação):
  ```json
  {
    "data": [
      {
        "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "customerId": "550e8400-e29b-41d4-a716-446655440000",
        "url": "https://api.atlascomercial.com.br/webhooks/oms",
        "events": ["SHIPPED", "DELIVERED"],
        "active": true,
        "createdAt": "2026-10-21T12:00:00.000Z",
        "updatedAt": "2026-10-21T12:00:00.000Z"
      }
    ],
    "pagination": {
      "page": 1,
      "pageSize": 20,
      "total": 1,
      "totalPages": 1
    }
  }
  ```

### EP-03: Atualizar Configuração de Webhook
- **Método / Rota**: `PATCH /api/v1/webhooks/:id`
- **Permissão**: Qualquer role autenticada (`OPERATOR` ou `ADMIN`)
- **Origem na Transcrição**: `[09:33] Bruno`, `[09:33] Marcos`
- **Request Body**:
  ```json
  {
    "url": "https://hooks.atlascomercial.com.br/v2/orders",
    "events": ["SHIPPED", "DELIVERED"],
    "active": true
  }
  ```
- **Response `200 OK`**: Objeto atualizado do webhook.
- **Status Codes**: `200 OK`, `400 Bad Request` (`WEBHOOK_INVALID_URL` / `VALIDATION_ERROR`), `401 Unauthorized`, `404 Not Found` (`WEBHOOK_NOT_FOUND`).

### EP-04: Remover Endpoint de Webhook
- **Método / Rota**: `DELETE /api/v1/webhooks/:id`
- **Permissão**: Qualquer role autenticada (`OPERATOR` ou `ADMIN`)
- **Origem na Transcrição**: `[09:33] Bruno`
- **Response**: `204 No Content`
- **Status Codes**: `204 No Content`, `401 Unauthorized`, `404 Not Found` (`WEBHOOK_NOT_FOUND`).

### EP-05: Rotacionar Secret do Webhook (Grace Period de 24h)
- **Método / Rota**: `POST /api/v1/webhooks/:id/rotate-secret`
- **Permissão**: Qualquer role autenticada (`OPERATOR` ou `ADMIN`)
- **Origem na Transcrição**: `[09:21] Sofia`, `[09:22] Sofia`
- **Semântica**: Gera nova `secret`, move a `secret` atual para `previousSecret` e define `previousSecretExpiresAt = now + 24h`.
- **Response `200 OK`**:
  ```json
  {
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "secret": "whsec_new_4b227777d4dd1fc61c6f884f48641d02b4d121d3fd328cb08b5531fcacdabf8a",
    "previousSecretValidUntil": "2026-10-22T12:30:00.000Z",
    "rotatedAt": "2026-10-21T12:30:00.000Z"
  }
  ```

### EP-06: Consultar Histórico de Entregas do Webhook (Últimos 100 Disparos)
- **Método / Rota**: `GET /api/v1/webhooks/:id/deliveries?limit=100`
- **Permissão**: Qualquer role autenticada (`OPERATOR` ou `ADMIN`)
- **Origem na Transcrição**: `[09:34] Marcos`
- **Response `200 OK`**:
  ```json
  {
    "data": [
      {
        "id": "c4b02f10-8119-4a1c-9a0b-1029384756ab",
        "webhookId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "eventId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
        "attemptNumber": 1,
        "success": true,
        "requestPayload": {
          "event_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
          "event_type": "order.status_changed",
          "timestamp": "2026-10-21T12:10:00.000Z",
          "order_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
          "order_number": "ORD-000042",
          "customer_id": "550e8400-e29b-41d4-a716-446655440000",
          "from_status": "PROCESSING",
          "to_status": "SHIPPED",
          "subtotal_cents": 150000,
          "discount_cents": 5000,
          "total_cents": 145000
        },
        "responseStatus": 200,
        "responseBody": "{\"received\":true}",
        "durationMs": 184,
        "createdAt": "2026-10-21T12:10:01.200Z"
      }
    ]
  }
  ```

### EP-07: Reprocessar Evento da Dead-Letter Queue (Replay Manual por Admin)
- **Método / Rota**: `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
- **Permissão**: Exclusivo para role `ADMIN` via `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`), com registro obrigatório de log de auditoria contendo `userId` e `email` do administrador (`[09:18] Diego`, `[09:35] Diego`, `[09:36] Sofia`, `[09:36] Larissa`).
- **Semântica**: Recoloca o evento correspondente na `webhook_outbox` com `status = PENDING`, `attempts = 0` e `nextAttemptAt = now()`, registrando `replayedAt` e `replayedById` na `webhook_dead_letter`.
- **Response `200 OK`**:
  ```json
  {
    "deadLetterId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "outboxId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "status": "PENDING",
    "replayedBy": "admin-user-uuid",
    "replayedAt": "2026-10-21T14:00:00.000Z"
  }
  ```
- **Status Codes**: `200 OK`, `401 Unauthorized`, `403 Forbidden` (`FORBIDDEN` caso seja `OPERATOR`), `404 Not Found` (`WEBHOOK_DEAD_LETTER_NOT_FOUND`), `409 Conflict` (`WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`).

---

## 2. Contrato de Saída (Outbound Webhook HTTP POST enviado pelo Worker ao Cliente)

- **Método**: `POST <webhook.url>` (obrigatoriamente `https://`)
- **Timeout**: `10000 ms` (10 segundos, `[09:42] Diego`)
- **Tamanho Máximo do Payload**: `64 KB` (`65536` bytes, `[09:24] Diego`, `[09:24] Larissa`)
- **Cabeçalhos Obrigatórios** (`[09:20] Sofia`, `[09:25] Diego`, `[09:44] Diego`, `[09:44] Sofia`):
  - `Content-Type: application/json`
  - `X-Event-Id: <uuid-do-evento-na-outbox>`
  - `X-Webhook-Id: <uuid-do-endpoint-webhook>`
  - `X-Timestamp: <iso-8601-timestamp-do-envio>`
  - `X-Signature: sha256=<hex-hmac-sha256(secret, rawBody)>` (durante o *grace period* de 24h após rotação, pode incluir a assinatura secundária ou header adicional `X-Signature-Previous` para validação sem downtime, `[09:21] Sofia`)
- **Corpo do Payload (Snapshot JSON enxuto, sem `items`)** (`[09:43] Diego`, `[09:52] Larissa`):
  ```json
  {
    "event_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "event_type": "order.status_changed",
    "timestamp": "2026-10-21T12:10:00.000Z",
    "order_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "order_number": "ORD-000042",
    "customer_id": "550e8400-e29b-41d4-a716-446655440000",
    "from_status": "PROCESSING",
    "to_status": "SHIPPED",
    "subtotal_cents": 150000,
    "discount_cents": 5000,
    "total_cents": 145000
  }
  ```

---

## 3. Matriz de Erros Padronizada (`WEBHOOK_*`)

Todos os erros do domínio herdam de `AppError` (`src/shared/errors/app-error.ts`) e seguem o formato `{ error: { code, message, details } }` de `src/middlewares/error.middleware.ts` (`[09:28] Bruno`, `[09:29] Larissa`):

| Código (`errorCode`) | HTTP Status | Classe Base | Cenário de Disparo | Origem |
| :--- | :--- | :--- | :--- | :--- |
| `WEBHOOK_NOT_FOUND` | `404` | `NotFoundError` / `AppError` | Endpoint de webhook não encontrado para o `:id` informado. | `[09:28] Bruno` |
| `WEBHOOK_INVALID_URL` | `400` | `ValidationError` / `BadRequestError` | URL cadastrada/editada não utiliza protocolo `https://` ou é malformada. | `[09:23] Sofia`, `[09:28] Bruno` |
| `WEBHOOK_SECRET_REQUIRED` | `400` / `422` | `UnprocessableEntityError` | Tentativa de assinar/processar webhook sem `secret` válida configurada. | `[09:28] Bruno` |
| `WEBHOOK_INVALID_EVENTS` | `400` | `ValidationError` | Lista `events` vazia ou contendo valores fora do enum `OrderStatus`. | `[09:31] Marcos`, `[09:33] Marcos` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | `404` | `NotFoundError` / `AppError` | `customerId` informado no cadastro de webhook não existe na tabela `customers`. | `[09:32] Larissa`, `src/modules/orders/order.service.ts` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | `422` | `UnprocessableEntityError` | Snapshot JSON serializado do evento excede o teto de 64KB (65.536 bytes). | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | `404` | `NotFoundError` / `AppError` | Registro `:id` na tabela `webhook_dead_letter` não encontrado para replay. | `[09:18] Diego`, `[09:35] Diego` |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | `409` | `ConflictError` | Evento da DLQ já foi reprocessado anteriormente e já possui `replayedAt` preenchido. | `[09:18] Diego`, `[09:35] Diego` |
| `WEBHOOK_DELIVERY_TIMEOUT` | N/A (Worker) | Código interno de falha na entrega | Endpoint do cliente não respondeu dentro do limite de 10 segundos. | `[09:42] Sofia`, `[09:42] Diego` |
| `WEBHOOK_DELIVERY_FAILED` | N/A (Worker) | Código interno de falha na entrega | Endpoint do cliente retornou HTTP $\ge 400$ ou erro de conexão TCP/TLS. | `[09:15] Diego`, `[09:18] Diego` |
