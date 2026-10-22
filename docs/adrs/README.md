# Architecture Decision Records (ADRs) — Sistema de Webhooks

Este diretório armazena os ADRs (Architecture Decision Records) do Sistema de Webhooks de Notificação de Pedidos do OMS, estruturados no formato MADR.

## Índice de Decisões

1. [ADR-001 — Padrão Transactional Outbox no MySQL com Snapshot JSON na Inserção](./ADR-001-outbox-no-mysql.md)
2. [ADR-002 — Worker em Processo Separado via Polling em Loop (2 Segundos)](./ADR-002-worker-separado-em-polling.md)
3. [ADR-003 — Política de Retry com Backoff Exponencial (5 Retentativas após o Envio Inicial) e Dead-Letter Queue Dedicada](./ADR-003-politica-de-retry-backoff-e-dlq.md)
4. [ADR-004 — Autenticação HMAC-SHA256 com Secret por Endpoint e Rotação com Grace Period de 24h](./ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md)
5. [ADR-005 — Semântica de Entrega At-Least-Once com Idempotência via `X-Event-Id`](./ADR-005-entrega-at-least-once-com-x-event-id.md)
6. [ADR-006 — Reuso Integral dos Padrões Arquiteturais e Infraestrutura Compartilhada do OMS](./ADR-006-reuso-dos-padroes-existentes-do-oms.md)
