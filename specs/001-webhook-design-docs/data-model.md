# Modelo de Dados — Planejamento e Referência Canônica

**Feature**: `001-webhook-design-docs` · **Status**: proposta revisada documentalmente.

O schema de implementação proposto está integralmente na [seção 4.1 do FDD](../../docs/FDD.md#41-especificação-do-modelo-de-dados-prismaschemaprisma). `prisma/schema.prisma` permanece imutável nesta entrega. Campos e índices novos são propostas derivadas, fundamentadas em FDD-DET-01 a FDD-DET-08.

## Entidades Documentais

| Entidade | Regra |
| --- | --- |
| DocumentArtifact | PRD, RFC, FDD, ADRs e Tracker têm alturas distintas; o enunciado preservado define as seções obrigatórias. |
| TrackerEntry | ID único, documento existente, tipo, resumo, fonte TRANSCRICAO ou CODIGO e localização verificável. |
| Cobertura | Comparar o inventário de IDs explícitos com as linhas do Tracker; a revisão semântica verifica se cada origem sustenta a afirmação e se um detalhe é proposto. Existência de timestamp não comprova conteúdo. |

## Entidades de Domínio Propostas

| Entidade | Identificação e relações | Campos e invariantes relevantes |
| --- | --- | --- |
| WebhookEndpoint | UUID; Customer; outbox, deliveries e deadLetters | url HTTPS, secret atual, previousSecret e previousSecretExpiresAt, events, active. Rotação atômica recusa sobrescrever chave anterior ainda válida (FDD-DET-05). |
| WebhookOutbox | UUID público = event_id; webhookId; orderId | sequence BigInt única autoincremental desempata createdAt; cycle começa em 0; attempts conta resultados persistidos no ciclo, de 0 a 6. Snapshot imutável; nextAttemptAt controla retry. |
| WebhookDelivery | UUID; FK outboxId e webhookId | cycle, attemptNumber 1..6, requestPayload, responseStatus, responseBody, durationMs, success e errorMessage. Unicidade (outboxId, cycle, attemptNumber); resultado por upsert e estado condicional da outbox na mesma transação. |
| WebhookDeadLetter | UUID; FK outboxId e webhookId; relação opcional com User para replay | Unicidade composta (outboxId, cycle), sem unicidade isolada de outboxId. Snapshot, attempts = 6, failureReason, failedAt, replayedAt e replayedById. Nova falha após replay cria outro ciclo e preserva a linha antiga. |

Os relacionamentos reversos em Customer, Order e User fazem parte da proposta do FDD. As cascatas propostas encerram e removem registros ao excluir endpoint/pedido; não há promessa de retenção para remoções explícitas. A revisão de migração deve verificar esse efeito.

## Máquina de Estados Proposta

- PENDING → PROCESSING: reserva condicional de item elegível, sem predecessor não entregue do mesmo (webhookId, orderId).
- PROCESSING → DELIVERED: HTTP 2xx; persiste histórico e attempts = attemptNumber na mesma transação.
- PROCESSING → PENDING: falha com attemptNumber <= 5; persiste resultado, atualiza attempts e agenda a espera correspondente (1m/5m/30m/2h/12h).
- PROCESSING → FAILED: falha com attemptNumber = 6; persiste resultado e upsert da DLQ por (outboxId, cycle).
- PROCESSING → PENDING no bootstrap exclusivo: recuperação de interrupção sem alterar UUID, payload, attempts ou cycle; resultado não persistido pode gerar reenvio.
- FAILED → PENDING por replay ADMIN: reserva condicional do ciclo, marca autoria na DLQ, incrementa cycle e zera attempts; preserva sequence e event_id.

PENDING em retry e FAILED em DLQ bloqueiam sucessores do mesmo destino/pedido. Outros pedidos continuam elegíveis. A exclusividade do worker deve ser assegurada antes de recuperar PROCESSING; este desenho não suporta workers simultâneos.
