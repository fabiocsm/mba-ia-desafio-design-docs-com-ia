# ADR-003: Política de Retry com Backoff Exponencial (5 Retentativas após o Envio Inicial) e Dead-Letter Queue Dedicada

- **ID de Rastreabilidade**: `ADR-003`
- **Data**: 2026-10-21
- **Decisores**: Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Marcos (PM), Sofia (Segurança)
- **Referências de Código**: `src/middlewares/auth.middleware.ts`, `src/shared/logger/index.ts`, `prisma/schema.prisma`

---

## Status

Aceito (Decidido na reunião técnica — `[09:17] Larissa`, `[09:18] Diego`, `[09:36] Larissa`, `[09:42] Diego`).

---

## Contexto

Como os webhooks são enviados para servidores externos operados pelos clientes B2B (fora da infraestrutura do OMS), falhas transitórias de rede, lentidão ou janelas de manutenção nos sistemas receptores são inevitáveis (`[09:14] Larissa`, `[09:16] Diego`). Na prática operacional da empresa, clientes já apresentaram janelas de indisponibilidade de até 2 horas durante manutenções planejadas (`[09:16] Diego`).

A equipe precisava definir:
1. Quanto tempo aguardar por uma resposta HTTP antes de considerar a tentativa falha;
2. Quantas vezes retentar e com qual espaçamento temporal (*backoff*);
3. Onde armazenar eventos que esgotarem todas as tentativas (*Dead-Letter Queue* — DLQ) e quem pode autorizar seu reprocessamento manual.

---

## Decisão

Estabelecemos uma política determinística de resiliência baseada em **timeout de 10s, 5 retentativas com backoff exponencial (~15h) e tabela dedicada de Dead-Letter Queue com replay restrito a `ADMIN`** (`ADR-003`):

1. **Timeout HTTP de 10 Segundos (`[09:42] Sofia`, `[09:42] Diego`)**: Cada requisição HTTP POST disparada pelo worker terá timeout estrito de `10.000 ms` (10 segundos). Caso o endpoint do cliente não responda dentro de 10s, ou retorne status HTTP não-2xx / erro de rede, a tentativa é registrada no histórico (`webhook_deliveries`) como falha e agendada para retry.
2. **Limite de 5 Retentativas após o Envio Inicial com Backoff Exponencial (`[09:15] Diego`, `[09:16] Larissa`, `[09:17] Diego`, `[09:17] Larissa`)**: Após uma falha, o evento é reagendado por até **5 retentativas após o envio inicial** seguindo a progressão de intervalos:
   - Retentativa 1 (após falha inicial): **1 minuto**
   - Retentativa 2: **5 minutos**
   - Retentativa 3: **30 minutos**
   - Retentativa 4: **2 horas**
   - Retentativa 5: **12 horas**
   - **Janela Total Coberta**: ~14 horas e 36 minutos (quase 15 horas entre a primeira falha e a última tentativa — `[09:17] Diego`).

**Interpretação documentada para revisão (`FDD-DET-01`)**: a fala usa “5 tentativas”, mas fornece cinco esperas e quase 15h. Este pacote adota envio inicial + cinco retentativas (seis chamadas por ciclo), preservando todos os intervalos. A contagem explícita é proposta de interpretação, não frase literal da reunião.

3. **Tabela Dedicada `webhook_dead_letter` (`[09:17] Larissa`, `[09:18] Diego`)**: Esgotados o envio inicial e as 5 retentativas sem sucesso, o evento na `webhook_outbox` é marcado como `FAILED` e um registro é persistido na tabela separada `webhook_dead_letter` contendo o `payload` original, o motivo da falha (`failure_reason`), número de tentativas e timestamp (`failed_at`). O FDD propõe identificar cada ciclo por `(outboxId, cycle)`, permitindo nova entrada após replay sem perder o histórico anterior (`FDD-DET-03`).
4. **Replay Manual Exclusivo para `ADMIN` com Auditoria (`[09:18] Diego`, `[09:35] Diego`, `[09:36] Sofia`, `[09:36] Larissa`)**: O reprocessamento de eventos da DLQ será exposto no endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, protegido obrigatoriamente pelo middleware `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`), recolocando o evento na `webhook_outbox` como `PENDING` e emitindo log estruturado de auditoria via Pino (`src/shared/logger/index.ts`) identificando o administrador (`req.user.id` e `req.user.email`).

---

## Alternativas Consideradas

### 1. Retry Indefinido com Backoff (`ADR-003-ALT-01`)
- **Descrição**: Retentar o envio infinitamente enquanto o endpoint do cliente não retornar HTTP 2xx (`[09:15] Diego`).
- **Motivo do Descarte (`[09:15] Diego`)**: Caso um cliente desative um endpoint sem remover o cadastro no OMS, eventos ficariam pendurados perpetuamente na fila, consumindo recursos de banco e ciclos de processamento do worker.

### 2. Limite Agressivo de 3 Tentativas (~30 Minutos) (`ADR-003-ALT-02`)
- **Descrição**: Realizar apenas 3 retentativas rápidas cobrindo cerca de 30 minutos antes de desistir (`[09:16] Bruno`, `[09:16] Diego`).
- **Motivo do Descarte (`[09:16] Diego`)**: Insuficiente para a realidade operacional dos clientes B2B do OMS, que já tiveram manutenções planejadas de 2 horas. Com apenas 30 minutos de janela, eventos legítimos cairiam na DLQ prematuramente durante qualquer manutenção matinal do cliente.

### 3. Manter Eventos Falhos Apenas na Tabela `webhook_outbox` com Status `FAILED` (`ADR-003-ALT-03`)
- **Descrição**: Não criar tabela `webhook_dead_letter` e apenas atualizar a coluna `status` para `FAILED` na própria `webhook_outbox` (`[09:17] Larissa`).
- **Motivo do Descarte (`[09:18] Diego`)**: Separar em `webhook_dead_letter` mantém a leitura da tabela quente (`webhook_outbox`) mais limpa e enxuta, além de isolar as evidências de falha permanente (payload, motivo e metadados de replay administrativo) para depuração e auditoria.

---

## Consequências

### Positivas
- **Resiliência a Manutenções Reais de Clientes**: A curva `1m / 5m / 30m / 2h / 12h` recupera automaticamente falhas de segundos (rede instável), minutos (deploy do cliente) e horas (manutenção planejada de até ~15h) sem intervenção humana (`[09:16] Diego`, `[09:17] Diego`).
- **Governança e Segurança Operacional**: O uso de `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`) aliado ao log de auditoria impede que operadores comuns manipulem filas de falha e garante rastreabilidade completa exigida pela equipe de Segurança (`[09:36] Sofia`).
- **Isolamento da Tabela Operacional**: A tabela `webhook_dead_letter` preserva a performance do índice de polling da `webhook_outbox` (`[09:18] Diego`).

### Negativas
- **Necessidade de Ação Manual Após 15 Horas**: Se o servidor do cliente ficar indisponível por mais de 15 horas, os eventos não serão reenviados automaticamente quando o cliente voltar, exigindo acionamento manual do endpoint de replay por um `ADMIN`; a duração real inclui tempos HTTP, polling e eventual espera por predecessores (`[09:17] Marcos`, `[09:18] Diego`).
- **Ausência de Alerta Ativo por E-mail na v1**: Como notificações por e-mail em caso de falhas consecutivas ficaram fora do escopo desta fase (`[09:37] Larissa`), o cliente precisará consultar `GET /api/v1/webhooks/:id/deliveries` ou acionar o suporte para identificar endpoints em falha.

### Trade-off Explícito (`ADR-003-TRD-01`)
Trocamos a **recuperação automática infinita** (que acumularia lixo operacional de endpoints abandonados) e o **descarte rápido em 30 minutos** (que perderia eventos em manutenções de rotina) por uma **janela finita de 5 retentativas após o envio inicial em ~15 horas com isolamento em tabela `webhook_dead_letter`**, aceitando que indisponibilidades superiores a 15h demandem replay manual auditado por um usuário `ADMIN` (`[09:15] Diego`, `[09:17] Diego`, `[09:18] Diego`, `[09:36] Sofia`).
