# Contratos HTTP — Índice da Especificação Canônica

**Feature**: `001-webhook-design-docs` · **Status**: proposta revisada documentalmente.

A especificação canônica de payloads, headers, respostas e erros está na [seção 5 do FDD](../../../docs/FDD.md#5-contratos-públicos) e na [matriz de erros](../../../docs/FDD.md#6-matriz-de-erros-previstos). Este artefato de planejamento aponta para esses contratos, evitando manter uma segunda cópia divergente.

| ID | Operação proposta | Referência canônica |
| --- | --- | --- |
| EP-01 | POST /api/v1/webhooks | FDD-EP-01: criação com secret gerada; 201; validação e cliente inexistente. |
| EP-02 | GET /api/v1/webhooks | FDD-EP-02: customerId explícito e envelope paginated; 200. |
| EP-03 | PATCH /api/v1/webhooks/:id | FDD-EP-03: edição de URL, eventos e active; 200. |
| EP-04 | DELETE /api/v1/webhooks/:id | FDD-EP-04: remoção; 204 sem corpo; cascatas conforme FDD seção 9. |
| EP-05 | POST /api/v1/webhooks/:id/rotate-secret | FDD-EP-05: grace period de 24h; 200; rotação sobreposta recebe 409. |
| EP-06 | GET /api/v1/webhooks/:id/deliveries | FDD-EP-06: até 100 entregas; cycle e attemptNumber; 200. |
| EP-07 | POST /api/v1/admin/webhooks/dead-letter/:id/replay | FDD-EP-07: ADMIN; reserva condicional do ciclo; 200/409; auditoria. |
| EP-08 | POST outbound para URL HTTPS cadastrada | FDD-EP-08: snapshot sem items, máximo 65536 bytes, HMAC sobre bytes enviados e timeout de 10s. |

Todas as rotas de gestão usam authenticate; replay usa também requireRole('ADMIN'), conforme `src/middlewares/auth.middleware.ts`. Erros estruturais são `400 VALIDATION_ERROR`; os erros de domínio da matriz usam `WEBHOOK_*`; autenticação/autorização preservam `UNAUTHORIZED` e `FORBIDDEN`.

O adaptador local reutiliza `src/middlewares/validate.middleware.ts`, convertendo falhas de url/events para `BadRequestError` com código explícito. `ValidationError` e `NotFoundError` não aceitam código customizado no construtor; erros de domínio 404 usam `AppError`. Base: `src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts`.

**Convenções derivadas para revisão da equipe**: envio inicial + cinco retentativas (até seis resultados HTTP persistidos por ciclo), ciclos de DLQ preservando event_id, múltiplos digests `sha256=` no mesmo X-Signature e recusa de rotações sobrepostas. Essas escolhas estão fundamentadas em FDD-DET-01 a FDD-DET-08 e não são apresentadas como falas literais dos participantes. X-Timestamp sozinho não autentica a idade do request.
