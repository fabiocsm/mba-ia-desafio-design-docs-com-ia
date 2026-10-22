# Contract 1: Estrutura e Critérios de Aceite dos Entregáveis Documentais

**Feature**: `001-webhook-design-docs`
**Date**: 2026-10-21

Este contrato define a interface documental exata que os 6 grupos de artefatos devem satisfazer para aprovação nos critérios de aceite do `README.md` e da `constitution.md`.

---

## 1. ADRs (`docs/adrs/ADR-001-*.md` a `ADR-006-*.md`)

- **Quantidade**: 6 arquivos (dentro da faixa obrigatória de 5 a 8 arquivos).
- **Lista de Arquivos Planejados**:
  1. `docs/adrs/ADR-001-outbox-no-mysql.md` — Padrão Transactional Outbox no MySQL com snapshot JSON na inserção e ID UUID.
  2. `docs/adrs/ADR-002-worker-separado-em-polling.md` — Worker em processo Node.js separado (entrypoint dedicado do worker) com polling a cada 2 segundos e ordenação por pedido com as condições de predecessor do FDD.
  3. `docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md` — Política de retry com backoff exponencial (5 retentativas após o envio inicial: 1m, 5m, 30m, 2h, 12h), tabela `webhook_dead_letter` separada e replay manual por `ADMIN`.
  4. `docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md` — Assinatura HMAC-SHA256 sobre o payload com secret única por endpoint, rotação com grace period de 24h e exigência de HTTPS.
  5. `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` — Semântica de entrega *at-least-once* com deduplicação no cliente via cabeçalho `X-Event-Id` (UUID).
  6. `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-oms.md` — Reuso integral da arquitetura modular (módulo de webhooks), função pura `publishWebhookEvent(tx, ...)` em `src/modules/orders/order.service.ts`, `AppError` (`WEBHOOK_*`), `errorMiddleware`, `requireRole` e logger `Pino`.
- **Seções Obrigatórias em Cada ADR**:
  1. `## Status`
  2. `## Contexto`
  3. `## Decisão`
  4. `## Alternativas Consideradas` (mínimo de 1 alternativa real discutida ou plausível com motivo de descarte)
  5. `## Consequências` (Positivas, Negativas e Trade-off explícito)
- **Esquema de IDs para o Tracker**: `ADR-001` a `ADR-006` (e subitens `ADR-001-ALT-01`, etc., se aplicável).

---

## 2. RFC (`docs/RFC.md`)

- **Extensão Alvo**: Conciso (2 a 4 páginas), focado em decisão arquitetural sem duplicar o nível de implementação do FDD.
- **Seções Obrigatórias**:
  1. `## Metadados` (Autor, Status, Data, Revisores: Larissa, Marcos, Bruno, Diego, Sofia)
  2. `## Resumo Executivo (TL;DR)`
  3. `## Contexto e Problema`
  4. `## Proposta Técnica` (visão geral arquitetural: fluxo outbox, worker 2s, retry/DLQ, segurança HMAC/TLS, semântica at-least-once)
  5. `## Alternativas Consideradas` (mínimo de 2 alternativas reais descartadas na reunião com trade-off: Disparo HTTP síncrono na transação; Redis Streams / Redis Cluster; Triggers no MySQL; Retry de 3 tentativas ou indefinido; Secret global única; Entrega *exactly-once*)
  6. `## Questões em Aberto` (mínimo de 2 pontos adiados/em aberto na reunião: Rate limiting de envio para clientes; Escalonamento multi-worker com particionamento/lock por `order_id`; Notificação por e-mail em falhas consecutivas; Política de expurgo/arquivamento de eventos entregues após 30 dias; Endurecimento futuro de RBAC no CRUD de webhooks)
  7. `## Impacto e Riscos`
  8. `## Decisões Relacionadas` (links Markdown diretos para os 6 ADRs em `./adrs/ADR-001-*.md` .. `./adrs/ADR-006-*.md`)
- **Esquema de IDs para o Tracker**: `RFC-PROP-01..N`, `RFC-ALT-01..N`, `RFC-OPEN-01..N`, `RFC-RISK-01..N`.

---

## 3. FDD (`docs/FDD.md`)

- **Papel**: Especificação profunda e acionável ("como construir, em detalhe").
- **Seções Obrigatórias**:
  1. `## 1. Contexto e Motivação Técnica`
  2. `## 2. Objetivos Técnicos`
  3. `## 3. Escopo e Exclusões Técnicas`
  4. `## 4. Arquitetura de Dados e Fluxos Detalhados` (criação do evento na outbox com filtro e snapshot JSON, processamento pelo worker em polling de 2s, curva de retry 1m/5m/30m/2h/12h, movimentação e replay de DLQ)
  5. `## 5. Contratos Públicos` (mínimo de 4 endpoints HTTP da API + contrato do POST de saída do webhook, com payloads de exemplo de request/response, headers, status codes e semântica)
  6. `## 6. Matriz de Erros Previstos` (códigos padronizados com prefixo `WEBHOOK_*` mapeados às classes de `src/shared/errors/http-errors.ts`)
  7. `## 7. Estratégias de Resiliência` (timeout de 10s, 5 retries com backoff exponencial, DLQ como fallback, grace period de 24h na rotação de secret, limite de 64KB por payload)
  8. `## 8. Observabilidade` (Métricas, Logs estruturados via Pino com redaction e auditoria de replay, e Tracing / correlação via `X-Event-Id`, `X-Webhook-Id` e `requestId`)
  9. `## 9. Dependências e Compatibilidade`
  10. `## 10. Integração com o Sistema Existente` (**Seção Obrigatória Específica**: referencia e detalha a integração com pelo menos 4 — planejados 12 — caminhos reais de arquivos do repositório: `prisma/schema.prisma`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`)
  11. `## 11. Critérios de Aceite Técnicos`
  12. `## 12. Riscos Técnicos e Mitigação`
- **Esquema de IDs para o Tracker**: `FDD-FLOW-01..N`, `FDD-EP-01..N`, `FDD-ERR-01..N`, `FDD-RES-01..N`, `FDD-OBS-01..N`, `FDD-INT-01..N`.

---

## 4. PRD (`docs/PRD.md`)

- **Papel**: Visão de produto e negócio ("por que e o quê").
- **Seções Obrigatórias**:
  1. `## 1. Resumo e Contexto da Feature`
  2. `## 2. Problema e Motivação` (clientes B2B Atlas Comercial, MaxDistribuição e Nova Cargo fazendo polling custoso em `GET /api/v1/orders`; risco de churn da Atlas até o fim do trimestre)
  3. `## 3. Público-Alvo e Cenários de Uso`
  4. `## 4. Objetivos e Métricas de Sucesso` (mínimo de 1 objetivo com métrica e meta quantitativa: latência de notificação $< 10\text{s}$ com polling de $2\text{s}$, retenção dos 3 clientes B2B / entrega em 3 sprints até fim de novembro, cobertura de entrega via retry em janelas de até ~15h)
  5. `## 5. Escopo (Incluso e Fora de Escopo)` (listando explicitamente em "Fora de escopo" pelo menos 4 itens descartados/adiados na reunião: Inbound webhooks, Notificação por e-mail em falhas seguidas, Dashboard/painel visual de webhooks, Arquivamento/expurgo automático após 30 dias e Rate limiting de saída na v1)
  6. `## 6. Requisitos Funcionais` (mínimo de 8 — planejados 12 — requisitos funcionais discutidos na reunião: `PRD-FR-01` a `PRD-FR-12`)
  7. `## 7. Requisitos Não Funcionais` (`PRD-NFR-01` a `PRD-NFR-08`: latência $< 10\text{s}$, timeout HTTP $10\text{s}$, limite payload $64\text{KB}$, TLS `https://` obrigatório, HMAC-SHA256 com secret por endpoint e grace period de 24h, semântica *at-least-once* com `X-Event-Id`, ordering por `order_id` em single-worker, processo separado do worker)
  8. `## 8. Decisões e Trade-offs Principais`
  9. `## 9. Dependências`
  10. `## 10. Riscos e Mitigação` (mínimo de 2 — planejados 5 — riscos completos com Probabilidade, Impacto e Mitigação)
  11. `## 11. Critérios de Aceitação`
  12. `## 12. Estratégia de Testes e Validação` (incluindo janela de 2 dias úteis de revisão de segurança pela Sofia antes do deploy)

---

## 5. Tracker (`docs/TRACKER.md`)

- **Formato Obrigatório**:
  `| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |`
- **Regras Quantitativas**:
  - Cobertura $\ge 80\%$ de todos os itens identificáveis em `PRD.md`, `RFC.md`, `FDD.md` e `adrs/ADR-*.md` (meta planejada: 100% dos itens numerados nos 4 grupos de documentos, $> 60$ linhas).
  - Proporção `Fonte = TRANSCRICAO` $\ge 70\%$ com timestamps reais `[hh:mm] Nome` verificados contra `TRANSCRICAO.md`.
  - Quantidade `Fonte = CODIGO` $\ge 5$ linhas (meta planejada: $\ge 10$ linhas) apontando para arquivos reais em `src/` e `prisma/`.

---

## 6. README (`README.md`)

- **Seções Obrigatórias**:
  1. `## Sobre o Desafio` (1 a 2 parágrafos em palavras próprias)
  2. `## Ferramentas de IA Utilizadas` (lista de ferramentas e papel de cada uma)
  3. `## Workflow Adotado` (ordem de produção dos documentos e organização da interação com IA via Spec Kit)
  4. `## Prompts Customizados` (mínimo de 2 prompts relevantes em blocos de código)
  5. `## Iterações e Ajustes` (mínimo de 2 momentos concretos de correção/refinamento e número de iterações realizadas)
  6. `## Como Navegar a Entrega` (árvore/caminhos dos arquivos entregues e ordem sugerida de leitura)

## Atualização da Revisão

O contrato de aceite continua sendo o enunciado preservado. A implementação proposta usa envio inicial + cinco retentativas (até seis resultados HTTP persistidos por ciclo), 11 códigos de erro de domínio e recuperação/ordenação descritas no FDD. O inventário atual tem 161 IDs explícitos; sua contagem não substitui a revisão semântica nem significa que cada detalhe técnico foi decidido na reunião. Ver [review.md](../review.md).
