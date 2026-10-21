# ADR-004: Autenticação HMAC-SHA256 com Secret por Endpoint e Rotação com Grace Period de 24h

- **ID de Rastreabilidade**: `ADR-004`
- **Data**: 2026-10-21
- **Decisores**: Sofia (Engenheira de Segurança), Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Marcos (PM)
- **Referências de Código**: `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `prisma/schema.prisma`

---

## Status

Aceito (Decidido na reunião técnica — `[09:22] Sofia`, `[09:23] Sofia`, `[09:44] Diego`, `[09:44] Sofia`).

---

## Contexto

Ao disparar webhooks para URLs externas na internet pública, o OMS envia dados comerciais de pedidos (`order_id`, `order_number`, `from_status`, `to_status`, `total_cents`, `customer_id`). A engenharia de segurança (Sofia) apontou três vetores de risco críticos (`[09:19] Sofia` a `[09:23] Sofia`):
1. **Falsificação de Requisições (*Spoofing*) e Adulteração em Trânsito**: O sistema receptor do cliente precisa validar criptograficamente que a chamada HTTP `POST` originou-se de fato do OMS e que o payload JSON não foi modificado no caminho (`[09:19] Sofia`), além de poder mitigar *replay attacks* (`[09:44] Diego`).
2. **Vazamento de Credenciais no Lado do Cliente**: Em experiências anteriores da empresa, um cliente já vazou sua chave secreta nos logs da própria aplicação (`[09:22] Diego`). Se a chave for global ou não puder ser trocada sem interromper o tráfego, a resposta a incidentes torna-se crítica.
3. **Transporte Inseguro e Payloads Anômalos**: Cadastro de URLs sem criptografia (`http://`) ou geração acidental de payloads excessivamente grandes (ex.: 500KB) expõem dados ou degradam a rede (`[09:23] Sofia`).

---

## Decisão

Adotaremos o padrão de assinatura criptográfica **HMAC-SHA256 por endpoint** com suporte a **rotação de secret com período de graça (*grace period*) de 24 horas** (`ADR-004`):

1. **Assinatura `HMAC-SHA256` sobre o Corpo da Requisição (`[09:20] Sofia`, `[09:22] Sofia`, `[09:44] Diego`, `[09:44] Sofia`)**: Cada disparo HTTP POST calculará o digest `HMAC-SHA256` sobre o corpo JSON serializado usando a `secret` do endpoint e o enviará no cabeçalho `X-Signature` (acompanhado dos cabeçalhos `X-Timestamp` em ISO 8601 para detecção de *replay attack*, `X-Event-Id` e `X-Webhook-Id`).
2. **Secret Única Gerada por Endpoint (`[09:21] Sofia`, `[09:31] Marcos`)**: Cada registro em `webhook_endpoints` possuirá sua própria `secret` criptográfica gerada pelo OMS no momento do cadastro (`POST /api/v1/webhooks`) e retornada na resposta de criação. Nenhuma secret será compartilhada entre endpoints ou clientes distintos.
3. **Rotação de Secret com Grace Period de 24 Horas (`[09:21] Sofia`, `[09:22] Sofia`)**: O cliente poderá solicitar uma nova secret via endpoint `POST /api/v1/webhooks/:id/rotate-secret`. Ao rotacionar, a secret anterior (`previous_secret`) permanece válida em paralelo por **24 horas** (`previous_secret_expires_at = NOW() + 24h`) para que o cliente atualize seus sistemas sem perda de webhooks; após 24 horas, a chave antiga expira definitivamente.
4. **TLS Obrigatório (`https://`) e Teto de 64KB (`[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa`)**: O schema Zod aplicado via `src/middlewares/validate.middleware.ts` recusará qualquer URL que não utilize o protocolo `https://` (`WEBHOOK_INVALID_URL`). Payloads que ultrapassem o limite de **64KB** resultarão em erro explícito (`WEBHOOK_PAYLOAD_TOO_LARGE`) em vez de truncamento silencioso.

---

## Alternativas Consideradas

### 1. Secret Global Única da Plataforma OMS (`ADR-004-ALT-01`)
- **Descrição**: Utilizar uma única chave secreta compartilhada no `.env` da plataforma para assinar os webhooks de todos os clientes (`[09:21] Sofia`).
- **Motivo do Descarte (`[09:21] Sofia`, `[09:22] Diego`)**: Risco inaceitável de segurança (*blast radius* total). Se um único cliente vazar a chave em seus logs, todos os demais clientes ficariam vulneráveis a falsificação de eventos e a troca da chave quebraria todas as integrações simultaneamente.

### 2. Rotação Imediata de Secret sem Período de Transição (`ADR-004-ALT-02`)
- **Descrição**: Invalidar instantaneamente a secret anterior no exato segundo em que o endpoint de rotação for chamado (`[09:21] Sofia`).
- **Motivo do Descarte (`[09:21] Sofia`)**: Provocaria falha de validação de assinatura em todos os webhooks disparados entre o momento da chamada na API do OMS e o deploy/atualização da nova secret no servidor do cliente.

### 3. Truncar Payloads que Excedam o Limite de Tamanho (`ADR-004-ALT-03`)
- **Descrição**: Cortar/truncar o JSON do evento caso ultrapasse o limite de tamanho em vez de lançar erro (`[09:23] Sofia`).
- **Motivo do Descarte (`[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa`)**: Truncar um payload produziria JSON inválido ou dados incompletos para o cliente e mascararia anomalias graves de serialização. Falhar explicitamente com teto de 64KB é mais seguro e previsível.

---

## Consequências

### Positivas
- **Isolamento de Segurança (*Zero Blast Radius* entre Clientes)**: O comprometimento ou rotação da `secret` de um endpoint afeta exclusivamente aquele endpoint (`[09:21] Sofia`).
- **Rotação Zero-Downtime**: O *grace period* de 24 horas permite que equipes de clientes B2B efetuem a troca de credenciais em produção sem rejeitar notificações em trânsito (`[09:21] Sofia`).
- **Aderência a Padrões de Mercado**: `HMAC-SHA256` em `X-Signature` com `X-Timestamp` é amplamente suportado por bibliotecas padrão em qualquer linguagem (`[09:20] Sofia`, `[09:44] Diego`).

### Negativas
- **Complexidade Adicional no Modelo e na Assinatura**: A tabela `webhook_endpoints` precisa manter `previous_secret` e `previous_secret_expires_at`, e o worker precisa gerenciar a vigência temporal da chave anterior durante a janela de 24 horas (`[09:21] Sofia`).
- **Cuidado com Exposição em Logs**: Exige garantir que `secret` e `previousSecret` nunca vazem nos logs do Pino (`src/shared/logger/index.ts`), adicionando caminhos de *redaction* adequados e revisão dedicada de 2 dias úteis pela segurança antes do deploy (`[09:46] Sofia`).

### Trade-off Explícito (`ADR-004-TRD-01`)
Trocamos a **simplicidade de uma chave estática ou invalidação imediata** pelo **armazenamento dual temporário (`secret` + `previous_secret` com expiração de 24h) por endpoint**, aceitando gerenciar o ciclo de expiração de duas chaves em troca de isolamento criptográfico por cliente e rotação de credenciais sem downtime (`[09:21] Sofia`, `[09:22] Sofia`).
