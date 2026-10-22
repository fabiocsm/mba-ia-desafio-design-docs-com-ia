# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

## Método e Limites da Cobertura

Este inventário cobre os **161 IDs explícitos** de ADRs, RFC, FDD e PRD. Os itens de contexto, escopo e dependências são consolidados nos requisitos/decisões correspondentes; objetivos, critérios de aceite, modelos e riscos técnicos também receberam IDs para evitar contar somente os itens da primeira versão. A cobertura automática compara conjuntos de IDs, não a veracidade das frases.

- **134 linhas TRANSCRICAO (83,2%)** e **27 linhas CODIGO (16,8%)**; limites do enunciado: pelo menos 70% e cinco linhas de código.
- Todos os marcadores `[hh:mm] Nome` e caminhos devem existir; isso é uma condição necessária, não prova semântica.
- **Decisão/requisito** registra o que foi discutido. **Proposta derivada**, **contrato proposto** e **erro proposto** identificam elaboração técnica baseada na fonte, sem alegar que o participante definiu campos, códigos ou mecanismos que não mencionou. A coluna Localização indica o fundamento; as justificativas dos detalhes estão em FDD-DET-01 a FDD-DET-08 e sua revisão em RFC-REVIEW-01.
- A semântica foi revisada contra a transcrição e o código; escolhas derivadas permanecem propostas submetidas à equipe. Não se declara “zero alucinação” apenas pela existência de timestamps, nem cobertura de toda frase do texto.

## Matriz de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Adoção do padrão Transactional Outbox no MySQL existente dentro da mesma transação de mudança de status do pedido | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Descarte de disparo HTTP síncrono dentro de OrderService.changeStatus por travar transações e impedir rollback coerente | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Descarte de fila externa com Redis Streams / Redis Cluster por configurar overengineering para o time atual | TRANSCRICAO | [09:07] Diego |
| ADR-001-ALT-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Descarte de renderização tardia do payload no envio em favor de snapshot JSON gravado na inserção da outbox | TRANSCRICAO | [09:52] Larissa |
| ADR-001-TRD-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Troca de broker dedicado pela garantia ACID transacional no MySQL existente com futuro expurgo após 30 dias | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Execução do worker em processo Node.js separado (src/worker.ts) realizando polling em loop a cada 2 segundos | TRANSCRICAO | [09:09] Diego; [09:11] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Descarte de rodar o worker no mesmo processo da API para não perder o worker em reinicializações da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Descarte de uso de triggers do MySQL por ausência de mecanismo nativo LISTEN/NOTIFY para processos externos | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Adiamento de múltiplos workers em paralelo com particionamento por order_id ou lock pessimista para evolução futura | TRANSCRICAO | [09:13] Diego |
| ADR-002-TRD-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off / proposta derivada | Single-worker com ordem por pedido; proposta de bloqueio por predecessor e sequência detalhada no FDD | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| ADR-003 | docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md | Decisão / interpretação proposta | Retry com cinco intervalos e DLQ; interpretação explícita: envio inicial + cinco retentativas | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md | Alternativa Descartada | Descarte de retry indefinido com backoff para evitar eventos pendurados para sempre se o cliente sumir | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md | Alternativa Descartada | Descarte de limite curto de 3 tentativas (~30m) por não cobrir manutenções planejadas de 2 horas de clientes | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md | Alternativa Descartada | Descarte de manter falhas definitivas na própria outbox em favor da tabela webhook_dead_letter separada | TRANSCRICAO | [09:18] Diego |
| ADR-003-TRD-01 | docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md | Trade-off | Janela de quase 15h de backoff e recuperação manual ADMIN após esgotamento | TRANSCRICAO | [09:17] Diego; [09:18] Diego; [09:36] Sofia |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md | Decisão | Assinatura HMAC-SHA256 no header X-Signature com secret única por endpoint e rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md | Alternativa Descartada | Descarte de secret global única da plataforma para evitar comprometimento total em caso de vazamento | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md | Alternativa Descartada | Descarte de invalidação imediata da secret na rotação em favor de janela de transição paralela de 24 horas | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-03 | docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md | Alternativa Descartada | Descarte de truncamento de payloads grandes em favor de erro explícito acima do limite de 64KB | TRANSCRICAO | [09:23] Sofia |
| ADR-004-TRD-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md | Trade-off | Manutenção temporária de duas secrets válidas por 24 horas para permitir migração sem downtime no cliente | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Semântica de entrega at-least-once com UUID único por evento no header X-Event-Id para deduplicação no cliente | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa Descartada | Descarte de garantia exactly-once por exigir coordenação bilateral complexa entre OMS e sistemas externos | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-02 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa Descartada | Descarte de disparo sem retentativa para não perder notificações em oscilações de rede do cliente | TRANSCRICAO | [09:24] Diego |
| ADR-005-TRD-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Transferência da responsabilidade de deduplicação por event_id ao cliente receptor em troca de simplicidade | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-oms.md | Decisão | Reuso integral do padrão modular (src/modules/webhooks/), AppError (WEBHOOK_*), errorMiddleware, Zod e Pino | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-oms.md | Alternativa Descartada | Descarte de injeção do WebhookRepository inteiro no OrderService em favor da função pura publishWebhookEvent(tx, ...) | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-oms.md | Alternativa Descartada | Descarte de introduzir novo logger ou tratamento de erros separado para o módulo de webhooks e worker | TRANSCRICAO | [09:29] Bruno |
| ADR-006-TRD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-oms.md | Trade-off derivado | Coesão do módulo interno e integração transacional em troca de autonomia de serviço separado | TRANSCRICAO | [09:27] Bruno; [09:30] Larissa; [09:41] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Alternativa descartada de disparo HTTP síncrono na transação de mudança de status do pedido | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Alternativa descartada de adoção de Redis Streams / Redis Cluster como fila externa | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Alternativa descartada de triggers no MySQL ou execução do worker no mesmo processo da API | TRANSCRICAO | [09:09] Diego; [09:11] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Alternativa descartada de 3 tentativas (~30m) ou retry indefinido sem teto de tentativas | TRANSCRICAO | [09:15] Diego; [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Secret global descartada; consequência de rotação sem transição é análise derivada do grace period | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Alternativa descartada de protocolo exactly-once coordenado entre OMS e servidores dos clientes | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio de webhooks para clientes em cenários de rajada (observar e decidir depois) | TRANSCRICAO | [09:38] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Estratégia futura de escalonamento multi-worker com particionamento por order_id ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Envio de notificação por e-mail ao cliente após falhas seguidas no endpoint (adiado para próxima fase) | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-04 | docs/RFC.md | Questão em Aberto | Automação de arquivamento/expurgo de linhas entregues na webhook_outbox após 30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-05 | docs/RFC.md | Questão em Aberto | Endurecimento futuro de permissões (RBAC) nos endpoints CRUD de configuração de webhooks | TRANSCRICAO | [09:37] Sofia |
| RFC-PROP-01 | docs/RFC.md | Decisão | Proposta arquitetural de Outbox Transacional no MySQL com snapshot JSON pré-renderizado e ID UUID | TRANSCRICAO | [09:06] Diego; [09:51] Larissa; [09:52] Larissa |
| RFC-PROP-02 | docs/RFC.md | Decisão | Proposta de worker em processo separado (src/worker.ts) executando polling a cada 2s em instância única | TRANSCRICAO | [09:09] Diego; [09:11] Larissa; [09:13] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão | Proposta de resiliência com timeout de 10s, 5 retries (1m/5m/30m/2h/12h), tabela DLQ e replay por ADMIN | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-04 | docs/RFC.md | Decisão | Proposta de segurança com TLS obrigatório (https), limite de 64KB e HMAC-SHA256 com rotação de 24h | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia; [09:23] Sofia; [09:24] Larissa |
| RFC-PROP-05 | docs/RFC.md | Decisão | Proposta de entrega at-least-once com identificador imutável X-Event-Id gerado na entrada da outbox | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-06 | docs/RFC.md | Decisão | Proposta de organização modular em src/modules/webhooks/ reutilizando AppError, Pino e middlewares existentes | TRANSCRICAO | [09:27] Bruno |
| RFC-REVIEW-01 | docs/RFC.md | Revisão proposta | Submete os detalhes derivados à revisão técnica, sem atribuí-los como decisões da reunião | TRANSCRICAO | [09:50] Larissa |
| RFC-RISK-01 | docs/RFC.md | Risco | Risco de impacto na transação de OrderService.changeStatus mitigado por filtro na inserção e função pura | TRANSCRICAO | [09:40] Bruno |
| RFC-RISK-02 | docs/RFC.md | Risco | Risco de crescimento da tabela webhook_outbox mitigado por índices em status/created_at e payload enxuto | TRANSCRICAO | [09:08] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Risco de falha na implementação criptográfica mitigado por revisão de 2 dias úteis da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| RFC-RISK-04 | docs/RFC.md | Risco | Risco de ordem e capacidade: instância única e proposta de predecessor; retry/DLQ podem bloquear sucessores | TRANSCRICAO | [09:13] Larissa |
| FDD-CA-01 | docs/FDD.md | Critério / proposta derivada | Aceite de atomicidade e filtro de status | TRANSCRICAO | [09:34] Bruno; [09:40] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério / proposta derivada | Aceite de cadastro e validação HTTPS | TRANSCRICAO | [09:23] Sofia; [09:31] Marcos |
| FDD-CA-03 | docs/FDD.md | Critério / proposta derivada | Aceite de rotação e validade das chaves; sobreposição é proposta derivada | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-04 | docs/FDD.md | Critério / proposta derivada | Aceite de worker, timeout e headers | TRANSCRICAO | [09:09] Diego; [09:42] Diego; [09:44] Diego; [09:44] Sofia |
| FDD-CA-05 | docs/FDD.md | Critério / proposta derivada | Aceite de todos os intervalos e DLQ por ciclo proposto | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| FDD-CA-06 | docs/FDD.md | Critério / proposta derivada | Aceite de histórico e replay ADMIN com auditoria | TRANSCRICAO | [09:34] Marcos; [09:36] Sofia |
| FDD-CA-07 | docs/FDD.md | Critério / proposta derivada | Aceite derivado de recuperação de crash e ordem sob retry/DLQ | TRANSCRICAO | [09:24] Diego; [09:12] Diego |
| FDD-DET-01 | docs/FDD.md | Detalhamento proposto | Interpretação de contagem: inicial + cinco retries para executar os cinco intervalos | TRANSCRICAO | [09:17] Diego |
| FDD-DET-02 | docs/FDD.md | Detalhamento proposto | Proposta de recuperar PROCESSING no bootstrap exclusivo mantendo UUID e snapshot | TRANSCRICAO | [09:24] Diego; [09:11] Diego |
| FDD-DET-03 | docs/FDD.md | Detalhamento proposto | Proposta de ciclos de DLQ e replay condicional preservando auditoria | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-DET-04 | docs/FDD.md | Detalhamento proposto | Proposta de sequência e bloqueio por predecessor para cumprir ordem por pedido | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| FDD-DET-05 | docs/FDD.md | Detalhamento proposto | Proposta de múltiplos digests em X-Signature e recusa de rotações sobrepostas para preservar 24h | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| FDD-DET-06 | docs/FDD.md | Detalhamento proposto | Proposta de adaptar localmente códigos genéricos de validação e erro aos códigos WEBHOOK_* | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-DET-07 | docs/FDD.md | Detalhamento proposto | Proposta de separar duração HTTP, fila e latência de notificação | TRANSCRICAO | [09:02] Marcos; [09:34] Marcos |
| FDD-DET-08 | docs/FDD.md | Detalhamento proposto | Schema e contratos propostos seguem tipos, IDs e relações existentes; detalhes completos no FDD | CODIGO | prisma/schema.prisma |
| FDD-EP-01 | docs/FDD.md | Contrato proposto | Contrato POST /api/v1/webhooks para cadastro de endpoint com geração e retorno da secret na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-EP-02 | docs/FDD.md | Contrato proposto | Listagem por customerId; paginação proposta conforme helper existente | TRANSCRICAO | [09:33] Bruno |
| FDD-EP-03 | docs/FDD.md | Contrato proposto | Contrato PATCH /api/v1/webhooks/:id para edição de URL, lista de eventos assinados e estado ativo | TRANSCRICAO | [09:33] Bruno |
| FDD-EP-04 | docs/FDD.md | Contrato proposto | Contrato DELETE /api/v1/webhooks/:id para remoção de endpoint de webhook cadastrado | TRANSCRICAO | [09:33] Bruno |
| FDD-EP-05 | docs/FDD.md | Contrato proposto | Rotação com 24h; formato de assinatura e rejeição de sobreposição propostos em FDD-DET-05 | TRANSCRICAO | [09:21] Sofia |
| FDD-EP-06 | docs/FDD.md | Contrato proposto | Contrato GET /api/v1/webhooks/:id/deliveries retornando os últimos 100 disparos com status, payload e durationMs | TRANSCRICAO | [09:34] Marcos |
| FDD-EP-07 | docs/FDD.md | Contrato proposto | Replay manual ADMIN auditado; controle concorrente e de ciclo propostos em FDD-DET-03 | TRANSCRICAO | [09:18] Diego; [09:36] Sofia; [09:36] Larissa |
| FDD-EP-08 | docs/FDD.md | Contrato proposto | Contrato outbound HTTP POST com headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e JSON sem items | TRANSCRICAO | [09:20] Sofia; [09:25] Diego; [09:43] Diego; [09:44] Diego; [09:44] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro proposto / padrão existente | Código de erro WEBHOOK_NOT_FOUND (404) quando o endpoint de webhook não é encontrado | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-02 | docs/FDD.md | Erro proposto / padrão existente | Código de erro WEBHOOK_INVALID_URL (400) quando a URL cadastrada não utiliza protocolo https | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-03 | docs/FDD.md | Erro proposto / padrão existente | Código de erro WEBHOOK_SECRET_REQUIRED (422) quando a secret obrigatória está ausente ou inválida | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-04 | docs/FDD.md | Erro proposto / padrão existente | Código de erro WEBHOOK_INVALID_EVENTS (400) quando o filtro de eventos contém status inválidos | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-05 | docs/FDD.md | Erro proposto / padrão existente | Código de erro WEBHOOK_CUSTOMER_NOT_FOUND (404) quando o customerId informado no body não existe | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-06 | docs/FDD.md | Erro proposto / padrão existente | Código de erro WEBHOOK_PAYLOAD_TOO_LARGE (422) quando o snapshot do evento ultrapassa o teto de 64KB | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-07 | docs/FDD.md | Erro proposto / padrão existente | Código de erro WEBHOOK_DEAD_LETTER_NOT_FOUND (404) quando o item da DLQ não existe para replay | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-08 | docs/FDD.md | Erro proposto / padrão existente | Proposta de erro 409 com ConflictError para replay duplicado ou ciclo incompatível; não é decisão literal da call | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-09 | docs/FDD.md | Erro proposto / timeout definido | Código interno WEBHOOK_DELIVERY_TIMEOUT quando o cliente não responde ao POST em até 10 segundos | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-10 | docs/FDD.md | Erro proposto / fundamento de retry | Código interno WEBHOOK_DELIVERY_FAILED quando o disparo HTTP falha por erro de rede ou status não-2xx | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-11 | docs/FDD.md | Erro proposto / padrão existente | Proposta de 409 para impedir sobrescrever secret anterior ainda válida | CODIGO | src/shared/errors/http-errors.ts |
| FDD-FLOW-01 | docs/FDD.md | Resiliência/Observabilidade | Fluxo detalhado de criação atômica na outbox com filtro de status assinado e snapshot JSON pré-renderizado | TRANSCRICAO | [09:41] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo / proposta derivada | Polling separado, recuperação de interrupção e seleção com predecessor; detalhes propostos fundamentados em entrega e ordem | TRANSCRICAO | [09:09] Diego; [09:12] Diego; [09:24] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo / interpretação proposta | Envio inicial + cinco retries, seis resultados persistidos por ciclo e cinco esperas somando 14h36m | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| FDD-FLOW-04 | docs/FDD.md | Fluxo / proposta derivada | DLQ por ciclo e replay condicional ADMIN com auditoria; permite novo esgotamento após replay | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração com Código | Extensão do schema Prisma com novos modelos de webhooks seguindo o padrão de IDs UUID Char(36) | CODIGO | prisma/schema.prisma |
| FDD-INT-02 | docs/FDD.md | Integração com Código | Gancho transacional em OrderService.changeStatus dentro de this.prisma.$transaction chamando publishWebhookEvent | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-03 | docs/FDD.md | Integração com Código | Reuso das funções de transição; OrderStatus deve ser importado de @prisma/client, definido no schema | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-04 | docs/FDD.md | Integração com Código | Herança da classe base AppError(message, statusCode, errorCode, details) para erros do módulo de webhooks | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-05 | docs/FDD.md | Integração com Código | AppError e construtores parametrizáveis para WEBHOOK_*; ValidationError e NotFoundError fixam códigos genéricos | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-06 | docs/FDD.md | Integração com Código | Captura e formatação automática dos erros WEBHOOK_* e ZodError pelo errorMiddleware centralizado | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração com Código | Uso de authenticate nas rotas de webhooks e requireRole('ADMIN') no endpoint de replay da DLQ | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração / proposta derivada | Adaptador local reutiliza validate e converte erros url/events para códigos de domínio | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração com Código | Reutilização do logger estruturado Pino na API e no worker com redaction de credenciais sensíveis | CODIGO | src/shared/logger/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração com Código | Uso do helper paginated(data, page, pageSize, total) para respostas paginadas na listagem de webhooks | CODIGO | src/shared/http/response.ts |
| FDD-INT-11 | docs/FDD.md | Integração com Código | Registro do módulo de webhooks em buildControllers e montagem das rotas sob /api/v1 em buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-INT-12 | docs/FDD.md | Integração com Código | Estrutura de bootstrap e graceful shutdown (SIGINT/SIGTERM) usada como base para o entrypoint src/worker.ts | CODIGO | src/server.ts |
| FDD-MODEL-01 | docs/FDD.md | Modelo proposto | WebhookEndpoint: extensão proposta dos padrões do schema; ver FDD-DET-08 | CODIGO | prisma/schema.prisma |
| FDD-MODEL-02 | docs/FDD.md | Modelo proposto | WebhookOutbox: extensão proposta dos padrões do schema; ver FDD-DET-08 | CODIGO | prisma/schema.prisma |
| FDD-MODEL-03 | docs/FDD.md | Modelo proposto | WebhookDelivery: extensão proposta dos padrões do schema; ver FDD-DET-08 | CODIGO | prisma/schema.prisma |
| FDD-MODEL-04 | docs/FDD.md | Modelo proposto | WebhookDeadLetter: extensão proposta dos padrões do schema; ver FDD-DET-08 | CODIGO | prisma/schema.prisma |
| FDD-OBJ-01 | docs/FDD.md | Objetivo | Objetivo de atomicidade, UUID e snapshot | TRANSCRICAO | [09:06] Diego; [09:51] Larissa; [09:52] Larissa |
| FDD-OBJ-02 | docs/FDD.md | Objetivo | Objetivo de polling separado e baixa latência | TRANSCRICAO | [09:02] Marcos; [09:09] Diego; [09:11] Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo | Objetivo de timeout, retry e DLQ | TRANSCRICAO | [09:17] Diego; [09:18] Diego; [09:42] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo | Objetivo de HMAC, HTTPS, payload limitado e rotação | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia; [09:23] Sofia; [09:24] Larissa |
| FDD-OBJ-05 | docs/FDD.md | Objetivo | Objetivo de reuso dos padrões | TRANSCRICAO | [09:30] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade / proposta derivada | Tempo HTTP separado da latência total de notificação e da fila; métricas adicionais propostas | TRANSCRICAO | [09:02] Marcos; [09:34] Marcos |
| FDD-OBS-02 | docs/FDD.md | Observabilidade / proposta derivada | Logs estruturados via Pino incluindo auditoria obrigatória de quem executou replay de DLQ e redaction de secret | TRANSCRICAO | [09:29] Bruno; [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade / proposta derivada | Tracing e correlação fim a fim de eventos usando os cabeçalhos X-Event-Id e X-Webhook-Id | TRANSCRICAO | [09:25] Diego; [09:44] Diego; [09:44] Sofia |
| FDD-RES-01 | docs/FDD.md | Resiliência/Observabilidade | Timeout estrito de 10 segundos por chamada HTTP do worker tratando lentidão como falha para retry | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência/Observabilidade | Estratégia de 5 retentativas após o envio inicial com progressão de backoff 1m, 5m, 30m, 2h e 12h cobrindo quase 15 horas | TRANSCRICAO | [09:17] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência/Observabilidade | Fallback em tabela webhook_dead_letter separada com reprocessamento manual via endpoint admin | TRANSCRICAO | [09:18] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência/Observabilidade | Validade paralela da secret antiga por 24 horas (grace period) após rotação pela API | TRANSCRICAO | [09:21] Sofia |
| FDD-RES-05 | docs/FDD.md | Resiliência/Observabilidade | Limite máximo de 64KB por payload com erro explícito (sem truncar) e omissão de items do pedido | TRANSCRICAO | [09:24] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco / mitigação proposta | Lentidão na transação de `OrderService.changeStatus` durante inserção na outbox | TRANSCRICAO | [09:40] Bruno; [09:34] Bruno |
| FDD-RISK-02 | docs/FDD.md | Risco / mitigação proposta | Degradação do polling em `webhook_outbox` com acúmulo de registros | TRANSCRICAO | [09:07] Bruno; [09:08] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco / mitigação proposta | Exposição acidental de `secret` em logs ou erro na validação HMAC | TRANSCRICAO | [09:22] Diego; [09:46] Sofia |
| FDD-RISK-04 | docs/FDD.md | Risco / mitigação proposta | Concorrência indevida caso mais de um processo `worker` seja iniciado | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| PRD-CA-01 | docs/PRD.md | Critério / proposta derivada | Aceite de CRUD autenticado e rejeição de HTTP | TRANSCRICAO | [09:23] Sofia; [09:31] Marcos; [09:33] Bruno |
| PRD-CA-02 | docs/PRD.md | Critério / proposta derivada | Aceite de inserção atômica e filtro | TRANSCRICAO | [09:34] Bruno; [09:40] Bruno |
| PRD-CA-03 | docs/PRD.md | Critério / proposta derivada | Aceite da meta de latência de notificação e dos headers | TRANSCRICAO | [09:02] Marcos; [09:44] Diego; [09:44] Sofia |
| PRD-CA-04 | docs/PRD.md | Critério / proposta derivada | Aceite de rotação com 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-05 | docs/PRD.md | Critério / proposta derivada | Aceite de histórico dos últimos disparos | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-06 | docs/PRD.md | Critério / proposta derivada | Aceite de retry com cinco intervalos e DLQ | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| PRD-CA-07 | docs/PRD.md | Critério / proposta derivada | Aceite de replay ADMIN auditado | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de endpoint via POST /api/v1/webhooks com URL, lista de eventos e secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Endpoints PATCH para editar, DELETE para remover e GET para listar os webhooks de um customerId | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Filtragem de eventos por status assinado realizada no momento da inserção na webhook_outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Inserção atômica na webhook_outbox dentro da mesma transação de OrderService.changeStatus com snapshot JSON | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Formato de payload JSON enxuto com metadados do evento e dados básicos do pedido (sem lista de items) | TRANSCRICAO | [09:43] Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Envio dos cabeçalhos X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e Content-Type application/json | TRANSCRICAO | [09:20] Sofia; [09:25] Diego; [09:44] Diego; [09:44] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Endpoint para rotação de secret mantendo a secret antiga válida em paralelo por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Endpoint GET /api/v1/webhooks/:id/deliveries listando os últimos 100 disparos com status, payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Política de retry automático em até 5 retentativas após o envio inicial com backoff exponencial de 1m, 5m, 30m, 2h e 12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Persistência em DLQ após esgotamento do envio inicial e cinco retentativas | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Endpoint POST /api/v1/admin/webhooks/dead-letter/:id/replay exclusivo para ADMIN com log de auditoria | TRANSCRICAO | [09:36] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Acesso às rotas CRUD de webhooks permitido a qualquer role autenticada, passando customerId no body ou query | TRANSCRICAO | [09:32] Larissa; [09:37] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação inferior a 10 segundos com worker realizando polling a cada 2 segundos | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos para a chamada HTTP do worker ao endpoint do cliente | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Limite máximo de 64KB por payload de evento, retornando erro explícito caso ultrapasse | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Obrigatoriedade de protocolo TLS (URL https://) validada no schema Zod na criação e edição | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Assinatura criptográfica HMAC-SHA256 sobre o corpo da requisição com secret exclusiva por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once com UUID único em X-Event-Id para deduplicação pelo cliente | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito / proposta derivada | Ordenação por pedido e destino em single-worker; proposta com predecessor bloqueia sucessores sob retry/DLQ | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Execução do worker em processo Node.js separado com instância própria de PrismaClient e reuso de padrões do OMS | TRANSCRICAO | [09:30] Bruno |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Meta quantitativa de latência de notificação abaixo de 10 segundos atendida via polling de 2 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Meta de entrega em 3 sprints (até fim de novembro) para reter Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Meta de resiliência cobrindo quase 15 horas em 5 retentativas após o envio inicial de retry (1m, 5m, 30m, 2h, 12h) | TRANSCRICAO | [09:17] Diego |
| PRD-OOS-01 | docs/PRD.md | Fora de Escopo | Exclusão de inbound webhooks (sistema apenas envia notificações outbound para os clientes) | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-02 | docs/PRD.md | Fora de Escopo | Exclusão de envio de e-mail de alerta ao cliente após falhas consecutivas (adiado para próxima fase) | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de Escopo | Exclusão de dashboard/painel visual para clientes gerenciarem webhooks (projeto separado de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de Escopo | Exclusão da rotina de arquivamento/expurgo de eventos entregues na outbox após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de Escopo | Exclusão de rate limiting de envio para clientes na primeira versão (observar e decidir depois) | TRANSCRICAO | [09:39] Larissa |
| PRD-RISK-01 | docs/PRD.md | Risco | Risco comercial de churn da Atlas Comercial caso a feature não seja entregue até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Risco de vazamento de secret em logs do cliente mitigado por secret única por endpoint e rotação de 24h | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Risco de indisponibilidade prolongada do cliente em manutenção planejada (2h+) mitigado por 5 retries (~15h) e DLQ | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Risco de recebimento duplicado em entrega at-least-once mitigado por X-Event-Id e documentação no portal | TRANSCRICAO | [09:26] Marcos |
| PRD-RISK-05 | docs/PRD.md | Risco | Risco de rajadas de chamadas em clientes com muitos pedidos mudando de status no mesmo minuto | TRANSCRICAO | [09:38] Diego |
