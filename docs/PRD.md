# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

- **ID do Documento**: `PRD-001`
- **Data**: 2026-10-21
- **Product Manager**: Marcos (`[09:00] Marcos`)
- **Tech Lead & Engenharia**: Larissa (Tech Lead), Bruno (Engenheiro Pleno — Pedidos), Diego (Engenheiro Sênior — Plataforma), Sofia (Engenheira de Segurança)
- **Prazo Alvo**: 3 Sprints — Entrega até o final de novembro (`[09:45] Marcos`, `[09:46] Larissa`, `[09:47] Larissa`)

---

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) gerencia o ciclo de vida transacional de pedidos B2B (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`), mas opera exclusivamente sob modelo passivo (requisição/resposta REST). Esta feature introduz um **Sistema de Outbound Webhooks de Notificação de Pedidos**, permitindo que clientes B2B cadastrem endpoints HTTPS próprios para receber notificações automáticas em tempo quase real (abaixo de 10 segundos) sempre que os status de seus pedidos mudarem (`[09:00] Marcos`, `[09:02] Marcos`, `[09:03] Sofia`).

A solução oferece API completa para gerenciamento de endpoints de webhook, seleção granular de quais transições de status escutar, assinatura criptográfica `HMAC-SHA256` com rotação de chave sem indisponibilidade, consulta ao histórico dos últimos 100 disparos e mecanismo de retentativa automática com fila de falha permanente (*Dead-Letter Queue* — DLQ) e reprocessamento administrativo (`[09:48] Larissa`).

---

## 2. Problema e Motivação

Na semana anterior à reunião técnica, três clientes B2B estratégicos — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — formalizaram a necessidade de receber notificações ativas de mudança de status de pedidos (`[09:00] Marcos`):

1. **Ineficiência e Custo de Integração por Polling**: Atualmente, os sistemas desses clientes precisam consultar repetidamente o endpoint `GET /api/v1/orders` de tempos em tempos para descobrir se houve atualização de status, tornando a integração lenta, ineficiente e custosa para os clientes e gerando carga desnecessária na API do OMS (`[09:00] Marcos`, `[09:02] Marcos`).
2. **Risco Iminente de Churn (Retenção de Receita B2B)**: A **Atlas Comercial** sinalizou explicitamente que, caso a funcionalidade não seja entregue até o final do trimestre (novembro), poderá migrar sua operação para um concorrente (`[09:00] Marcos`, `[09:45] Marcos`).
3. **Necessidade de Confiabilidade sem Intervenção Manual**: Os clientes precisam que as atualizações não fiquem pendentes de atualização manual e sejam entregues mesmo diante de oscilações ou janelas de manutenção planejada em seus servidores (`[09:02] Marcos`, `[09:16] Diego`).

---

## 3. Público-Alvo e Cenários de Uso

### Perfis de Usuário / Personas

1. **Integradores e Sistemas B2B dos Clientes (Atlas Comercial, MaxDistribuição, Nova Cargo)**:
   - Consomem os eventos enviados pelo OMS via HTTPS (`POST`), validam a assinatura `HMAC-SHA256` (`X-Signature`), deduplicam eventos usando `X-Event-Id` e, quando necessário, consultam detalhes completos dos itens do pedido em `GET /api/v1/orders/:id` (`[09:20] Sofia`, `[09:25] Diego`, `[09:43] Diego`).
2. **Usuários Autenticados Representantes do Cliente / Operadores (`OPERATOR` / `ADMIN`)**:
   - Utilizam a API do OMS (`Authorization: Bearer <jwt>`) para cadastrar, listar, editar e remover endpoints de webhook de um `customerId`, rotacionar `secret` em caso de suspeita de vazamento ou política de segurança e auditar os últimos 100 disparos em `GET /api/v1/webhooks/:id/deliveries` (`[09:21] Sofia`, `[09:31] Marcos`, `[09:32] Marcos`, `[09:34] Marcos`).
3. **Administradores da Plataforma OMS (`ADMIN`)**:
   - Monitoram falhas permanentes e acionam o reprocessamento manual de eventos na Dead-Letter Queue (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`) após um cliente restabelecer seu servidor (`[09:18] Diego`, `[09:35] Diego`, `[09:36] Sofia`).

---

## 4. Objetivos e Métricas de Sucesso

| ID | Objetivo de Produto / Negócio | Métrica Indicadora | Meta Quantitativa | Origem |
| :--- | :--- | :--- | :--- | :--- |
| `PRD-OBJ-01` | **Notificação em Tempo Quase Real** | Latência entre o commit da mudança de status em `OrderService.changeStatus` e o disparo HTTP pelo worker em condições normais | **$< 10\text{ segundos}$** (operando com polling de **$2\text{ segundos}$** no worker) | `[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa` |
| `PRD-OBJ-02` | **Retenção dos Clientes B2B e Cumprimento de Prazo Comercial** | Entrega em produção homologada com revisão de segurança para atender Atlas Comercial, MaxDistribuição e Nova Cargo | **Até 3 Sprints** (entrega até o **fim de novembro**, incluindo **2 dias úteis** de revisão de segurança) | `[09:00] Marcos`, `[09:45] Marcos`, `[09:46] Larissa`, `[09:46] Sofia` |
| `PRD-OBJ-03` | **Alta Resiliência a Manutenções de Clientes** | Janela temporal de recuperação automática de falhas antes de mover para a DLQ | **5 tentativas** cobrindo **~15 horas** (`1m, 5m, 30m, 2h, 12h`), superando janelas históricas de manutenção de 2h | `[09:16] Diego`, `[09:17] Diego`, `[09:17] Larissa` |

---

## 5. Escopo (Incluso e Fora de Escopo)

### 5.1 Escopo Incluso (v1)
- Emissão de **outbound webhooks** para transições de status de pedidos (`OrderStatus`).
- API REST autenticada para CRUD de endpoints de webhook (`POST`, `GET`, `PATCH`, `DELETE /api/v1/webhooks`) com filtro configurável de status por endpoint.
- Geração automática de `secret` exclusiva por endpoint e endpoint de rotação com período de graça de 24 horas (`POST /api/v1/webhooks/:id/rotate-secret`).
- Consulta ao histórico dos últimos 100 disparos por endpoint (`GET /api/v1/webhooks/:id/deliveries`).
- Worker assíncrono em processo separado com polling a cada 2s, timeout de 10s, 5 retentativas com backoff exponencial e tabela de Dead-Letter Queue (`webhook_dead_letter`).
- Endpoint administrativo (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`) exclusivo para `ADMIN` com log de auditoria.
- Documentação no portal do desenvolvedor orientando sobre validação `HMAC-SHA256` e deduplicação por `X-Event-Id` (`[09:26] Marcos`, `[09:40] Marcos`).

### 5.2 Fora de Escopo (Descartados ou Adiados na Reunião)

Os itens abaixo foram explicitamente discutidos na reunião e **excluídos desta entrega**:

1. **Inbound Webhooks (Recebimento de Webhooks dos Clientes) (`PRD-OOS-01`)**: O fluxo é estritamente unidirecional do OMS para os clientes (*outbound*); o OMS não receberá callbacks/webhooks enviados pelos clientes (`[09:02] Sofia`, `[09:02] Marcos`, `[09:03] Sofia`).
2. **Notificação por E-mail ao Cliente em Falhas Consecutivas (`PRD-OOS-02`)**: O envio de alertas por e-mail caso o endpoint de um cliente falhe repetidamente (ex.: 3 falhas seguidas) foi explicitamente descartado para esta fase e adiado para avaliação futura após medição de impacto (`[09:37] Marcos`, `[09:37] Larissa`, `[09:38] Marcos`).
3. **Dashboard / Painel Visual de Gestão de Webhooks (`PRD-OOS-03`)**: Não será construído nenhum painel gráfico (frontend) nesta entrega; a gestão e consulta ocorrem exclusivamente via endpoints de API, ficando um eventual painel como projeto separado do time de frontend (`[09:39] Marcos`, `[09:40] Larissa`).
4. **Arquivamento / Expurgo Automático de Eventos Entregues (>30 Dias) (`PRD-OOS-04`)**: A rotina de expurgo ou arquivamento de linhas antigas já entregues (`DELIVERED`) na `webhook_outbox` após 30 dias foi classificada como fora do escopo desta feature (`[09:08] Diego`).
5. **Rate Limiting de Saída nos Disparos para Clientes (`PRD-OOS-05`)**: Não haverá controle de *throttling*/rate limiting de chamadas enviadas ao cliente na v1; o comportamento será observado em produção para decisão futura (`[09:38] Diego`, `[09:39] Diego`, `[09:39] Larissa`).

---

## 6. Requisitos Funcionais

Todos os requisitos funcionais abaixo refletem diretamente as definições fechadas na reunião técnica:

| ID | Requisito Funcional | Descrição e Regras de Negócio | Origem na Transcrição |
| :--- | :--- | :--- | :--- |
| `PRD-FR-01` | **Cadastro de Endpoint de Webhook** | Permitir cadastrar um endpoint via `POST /api/v1/webhooks` informando `customerId`, `url` (`https://`), lista de status assinados (`events`) e estado `active`, gerando e retornando na criação uma `secret` exclusiva do endpoint. | `[09:21] Sofia`, `[09:31] Marcos`, `[09:32] Larissa` |
| `PRD-FR-02` | **Listagem, Edição e Remoção de Webhooks** | Permitir listar (`GET /api/v1/webhooks`), atualizar URL/eventos/status ativo (`PATCH /api/v1/webhooks/:id`) e remover (`DELETE /api/v1/webhooks/:id`) configurações de webhook de um cliente. | `[09:33] Bruno`, `[09:33] Marcos` |
| `PRD-FR-03` | **Filtro de Eventos por Status na Inserção** | Permitir que cada endpoint escolha quais status (`OrderStatus`) deseja ouvir; na mudança de status do pedido, filtrar no momento da inserção na `webhook_outbox` (não inserindo linha caso nenhum webhook ativo do cliente assine o novo status). | `[09:33] Marcos`, `[09:34] Bruno`, `[09:34] Diego` |
| `PRD-FR-04` | **Registro Transacional Atômico na Outbox com Snapshot JSON** | Gravar o evento na tabela `webhook_outbox` dentro da mesma transação SQL de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), salvando o snapshot JSON já renderizado no instante da transição (e sofrendo rollback caso a transação falhe). | `[09:06] Diego`, `[09:40] Bruno`, `[09:52] Larissa` |
| `PRD-FR-05` | **Payload Enxuto sem Lista de Itens** | Estruturar o payload JSON do evento com `event_id` (UUID), `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e valores básicos (`subtotal_cents`, `discount_cents`, `total_cents`), omitindo `items` para manter o payload enxuto. | `[09:43] Diego`, `[09:44] Bruno` |
| `PRD-FR-06` | **Disparo Assíncrono com Cabeçalhos de Segurança e Rastreio** | Enviar requisição `POST` ao endpoint do cliente contendo obrigatoriamente os cabeçalhos `X-Event-Id`, `X-Signature` (`HMAC-SHA256`), `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`. | `[09:20] Sofia`, `[09:25] Diego`, `[09:44] Diego`, `[09:44] Sofia` |
| `PRD-FR-07` | **Rotação de Secret com Grace Period de 24 Horas** | Disponibilizar endpoint (`POST /api/v1/webhooks/:id/rotate-secret`) para gerar nova `secret`, mantendo a `secret` anterior válida em paralelo por 24 horas antes de expirar. | `[09:21] Sofia`, `[09:22] Sofia` |
| `PRD-FR-08` | **Histórico dos Últimos 100 Disparos (`Deliveries`)** | Disponibilizar `GET /api/v1/webhooks/:id/deliveries` retornando os últimos 100 disparos realizados para o endpoint, incluindo indicador de sucesso/falha, payload enviado, resposta recebida e tempo de resposta. | `[09:34] Marcos`, `[09:35] Larissa` |
| `PRD-FR-09` | **Retentativa Automática com Backoff Exponencial (5 Tentativas)** | Reagendar automaticamente entregas que falharem (erro HTTP, erro de rede ou timeout $> 10\text{s}$) por até 5 tentativas seguindo a progressão `1m, 5m, 30m, 2h, 12h`. | `[09:15] Diego`, `[09:17] Diego`, `[09:17] Larissa` |
| `PRD-FR-10` | **Persistência de Falhas Permanentes em Dead-Letter Queue (DLQ)** | Mover eventos que falharem após a 5ª tentativa para uma tabela dedicada `webhook_dead_letter`, registrando payload, motivo da falha e timestamp. | `[09:15] Diego`, `[09:18] Diego` |
| `PRD-FR-11` | **Replay Manual da DLQ Exclusivo para `ADMIN` com Auditoria** | Disponibilizar `POST /api/v1/admin/webhooks/dead-letter/:id/replay` restrito à role `ADMIN` (via `requireRole('ADMIN')`), reenfileirando o evento na `webhook_outbox` como `PENDING` e registrando em log quem executou o replay. | `[09:18] Diego`, `[09:35] Diego`, `[09:36] Sofia`, `[09:36] Larissa` |
| `PRD-FR-12` | **Controle de Acesso nas Rotas de Configuração** | Permitir que qualquer usuário autenticado com JWT válido (`OPERATOR` ou `ADMIN`) acesse os endpoints CRUD de configuração e histórico de webhooks na v1, recebendo `customerId` explicitamente na requisição. | `[09:32] Marcos`, `[09:32] Larissa`, `[09:36] Marcos`, `[09:37] Sofia` |

---

## 7. Requisitos Não Funcionais

| ID | Requisito Não Funcional | Especificação e Limite Técnico | Origem na Transcrição |
| :--- | :--- | :--- | :--- |
| `PRD-NFR-01` | **Latência de Notificação e Intervalo de Polling** | Latência percebida pelo cliente inferior a **10 segundos**, sustentada por polling do worker a cada **2 segundos** (latência mínima de fila de 2s no pior caso). | `[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa` |
| `PRD-NFR-02` | **Timeout de Chamada HTTP do Worker** | Timeout máximo de **10 segundos** (`10.000 ms`) por tentativa de envio HTTP ao servidor do cliente; respostas mais lentas são abortadas e tratadas como falha para retry. | `[09:42] Sofia`, `[09:42] Diego` |
| `PRD-NFR-03` | **Limite Máximo de Tamanho de Payload** | Teto estrito de **64KB** por payload de evento; caso ultrapasse 64KB, o sistema deve falhar com erro explícito (`WEBHOOK_PAYLOAD_TOO_LARGE`) e nunca truncar o conteúdo. | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| `PRD-NFR-04` | **Obrigatoriedade de TLS (`HTTPS`)** | Rejeitar no schema Zod qualquer cadastro ou atualização de webhook cuja URL não utilize protocolo `https://`. | `[09:23] Sofia` |
| `PRD-NFR-05` | **Autenticidade Criptográfica e Isolamento de Secret** | Assinatura `HMAC-SHA256` sobre o corpo do request usando `secret` única por endpoint (nunca global da plataforma). | `[09:20] Sofia`, `[09:21] Sofia`, `[09:22] Sofia` |
| `PRD-NFR-06` | **Garantia de Entrega e Idempotência** | Semântica *at-least-once* com identificador `UUID` único por evento enviado em `X-Event-Id` para deduplicação no cliente receptor. | `[09:24] Diego`, `[09:25] Diego`, `[09:26] Larissa` |
| `PRD-NFR-07` | **Ordenação de Eventos por Pedido** | Ordenação implícita por `order_id` (via leitura `created_at ASC` na `webhook_outbox`) garantida enquanto o sistema operar com instância única de worker (*single-worker*). | `[09:12] Diego`, `[09:13] Larissa`, `[09:14] Marcos` |
| `PRD-NFR-08` | **Isolamento de Processo e Padronização Arquitetural** | O worker deve rodar em processo Node.js separado da API (`src/worker.ts`) com instância própria de `PrismaClient`, reutilizando `AppError` (prefixo `WEBHOOK_`), `errorMiddleware`, `validate` (Zod), `requireRole` e `logger` Pino. | `[09:11] Diego`, `[09:28] Bruno`, `[09:29] Larissa`, `[09:30] Bruno` |

---

## 8. Decisões e Trade-offs Principais

1. **Outbox Transacional no MySQL vs. Chamada Síncrona ou Redis Streams**: Priorizou-se a consistência ACID com a mudança de status do pedido e zero infraestrutura adicional em vez de introduzir um cluster Redis ou arriscar travar transações com chamadas HTTP síncronas (`[09:04] Bruno`, `[09:06] Diego`, `[09:07] Diego`).
2. **Polling de 2 Segundos em Single-Worker vs. Triggers ou Multi-Worker Complexo**: Escolheu-se um loop simples de 2s em processo separado que cumpre com folga a meta $< 10\text{s}$ e mantém a ordem dos eventos por pedido sem precisar de particionamento ou locks distribuídos na v1 (`[09:09] Diego`, `[09:12] Diego`).
3. **5 Tentativas (~15h) + DLQ Dedicada vs. 3 Tentativas (~30m) ou Retry Infinito**: Equilibrou-se a cobertura de manutenções planejadas reais dos clientes (2h+) com a proteção do banco contra eventos eternamente pendurados (`[09:15] Diego`, `[09:16] Diego`, `[09:17] Diego`).
4. **At-Least-Once com `X-Event-Id` vs. Exactly-Once**: Adotou-se o padrão de mercado (Stripe, GitHub), mantendo o protocolo leve e documentando para o cliente a responsabilidade de deduplicar por `X-Event-Id` (`[09:25] Diego`, `[09:26] Marcos`).

---

## 9. Dependências

- **Código e Infraestrutura Existentes do OMS**:
  - Banco de dados MySQL e schema Prisma (`prisma/schema.prisma`, `src/config/database.ts`).
  - Serviço e máquina de estados de pedidos (`src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`).
  - Middlewares de autenticação/RBAC (`src/middlewares/auth.middleware.ts`), validação Zod (`src/middlewares/validate.middleware.ts`), tratamento de erros (`src/middlewares/error.middleware.ts`) e logger Pino (`src/shared/logger/index.ts`).
- **Revisão de Segurança (Sofia)**: Alocação obrigatória de **2 dias úteis** na sprint 3 antes do deploy para revisão da implementação de `HMAC-SHA256` e geração/rotação de `secret` (`[09:46] Sofia`).
- **Documentação Externa no Portal do Desenvolvedor (Marcos)**: Publicação do guia de integração explicando os endpoints de API, validação de `X-Signature` e deduplicação por `X-Event-Id` (`[09:26] Marcos`, `[09:40] Marcos`).

---

## 10. Riscos e Mitigação

| ID | Descrição do Risco | Probabilidade | Impacto | Plano de Mitigação | Origem |
| :--- | :--- | :---: | :---: | :--- | :--- |
| `PRD-RISK-01` | **Atraso na entrega além do fim de novembro resultar em churn da Atlas Comercial** | Baixa | Alto | Escopo enxuto (sem painel visual, sem e-mail, sem Redis novo), reuso total dos padrões do OMS e cronograma estruturado em 3 sprints com confirmação imediata junto ao cliente. | `[09:00] Marcos`, `[09:45] Marcos`, `[09:46] Larissa` |
| `PRD-RISK-02` | **Vazamento de `secret` em logs do cliente ou tentativa de falsificação de eventos** | Média | Alto | Isolamento de `secret` única por endpoint, assinatura `HMAC-SHA256` com `X-Timestamp`, endpoint de rotação com grace period de 24h, redaction nos logs do OMS e revisão de 2 dias úteis pela Segurança. | `[09:20] Sofia`, `[09:21] Sofia`, `[09:22] Diego`, `[09:46] Sofia` |
| `PRD-RISK-03` | **Indisponibilidade prolongada no servidor do cliente B2B durante manutenções** | Média | Médio | Política de 5 retentativas com backoff exponencial (`1m, 5m, 30m, 2h, 12h`, cobrindo ~15h), persistência na tabela `webhook_dead_letter` e endpoint de replay manual por `ADMIN`. | `[09:16] Diego`, `[09:17] Diego`, `[09:18] Diego` |
| `PRD-RISK-04` | **Processamento duplicado de pedidos no ERP do cliente devido a reenvios (*at-least-once*)** | Média | Médio | Envio do UUID imutável no header `X-Event-Id` e no campo `event_id` do payload, aliado a destaque explícito no portal do desenvolvedor sobre deduplicação pelo cliente. | `[09:24] Diego`, `[09:25] Sofia`, `[09:26] Marcos` |
| `PRD-RISK-05` | **Sobrecarga de chamadas em rajada para um cliente com alto volume de pedidos no mesmo minuto** | Baixa | Médio | Monitorar o comportamento real de entregas na v1 (`webhook_deliveries`) e implementar *rate limiting* de saída em fase subsequente caso se observe gargalo nos clientes. | `[09:38] Diego`, `[09:39] Diego`, `[09:39] Larissa` |

---

## 11. Critérios de Aceitação

1. Um usuário autenticado consegue cadastrar (`POST /api/v1/webhooks`), listar (`GET`), editar (`PATCH`) e excluir (`DELETE`) webhooks para um `customerId`, recebendo a `secret` gerada na criação e erro `400` caso informe uma URL sem `https://`.
2. Quando um pedido muda de status em `PATCH /api/v1/orders/:id/status`, o evento é gravado atomicamente na `webhook_outbox` com snapshot JSON ($\le 64\text{KB}$, sem `items`) apenas para os webhooks ativos daquele cliente que assinam o novo status.
3. O worker em processo separado (`src/worker.ts`) envia o webhook em menos de 10 segundos (polling de 2s) com os headers `X-Event-Id`, `X-Signature` (`HMAC-SHA256`), `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.
4. Ao rotacionar a secret (`POST /api/v1/webhooks/:id/rotate-secret`), a chave antiga continua válida em paralelo por 24 horas.
5. O cliente consegue consultar os últimos 100 disparos em `GET /api/v1/webhooks/:id/deliveries` com status de sucesso/falha, payload, response e tempo de resposta.
6. Falhas de entrega (timeout $> 10\text{s}$ ou HTTP não-2xx) são retentadas 5 vezes (`1m, 5m, 30m, 2h, 12h`) e, após a 5ª falha, movidas para `webhook_dead_letter`.
7. Apenas usuários com role `ADMIN` conseguem acionar `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, reenfileirando o evento na outbox e gerando log de auditoria.

---

## 12. Estratégia de Testes e Validação

Planejamento de validação distribuído ao longo das 3 sprints (`[09:46] Larissa`, `[09:46] Sofia`):

1. **Testes Unitários (Vitest)**:
   - Geração de `secret`, cálculo de assinatura `HMAC-SHA256` e lógica de vigência do *grace period* de 24h.
   - Validação de schemas Zod (rejeição de `http://`, validação de `OrderStatus[]`, limite de 64KB).
   - Cálculo dos intervalos de backoff exponencial (`1m, 5m, 30m, 2h, 12h`).
2. **Testes de Integração e Ponta a Ponta (Vitest + Supertest + MySQL)**:
   - Verificação da atomicidade transacional em `OrderService.changeStatus` (garantindo que falha na outbox reverte a mudança de status e vice-versa, e que status não assinados não geram linhas na outbox).
   - Testes de contrato das 7 rotas HTTP (`/api/v1/webhooks/*` e `/api/v1/admin/webhooks/dead-letter/:id/replay`), validando bloqueio `403 FORBIDDEN` para `OPERATOR` no endpoint de replay e emissão do log de auditoria.
   - Simulação do ciclo do worker com servidor receptor mockado (sucesso `200`, timeout $> 10\text{s}$, 5 falhas consecutivas e transição para `webhook_dead_letter`).
3. **Revisão Dedicada de Segurança (Gate Pré-Deploy)**:
   - Reserva obrigatória de **2 dias úteis** ao final da sprint 3 para auditoria de código pela engenheira de segurança (**Sofia**), inspecionando a implementação de `HMAC-SHA256`, entropia e rotação de `secret` e *redaction* de logs antes da liberação em produção (`[09:46] Sofia`, `[09:47] Larissa`, `[09:49] Sofia`).
