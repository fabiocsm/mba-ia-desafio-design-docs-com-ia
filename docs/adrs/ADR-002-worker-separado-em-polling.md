# ADR-002: Worker em Processo Separado via Polling em Loop (2 Segundos)

- **ID de Rastreabilidade**: `ADR-002`
- **Data**: 2026-10-21
- **Decisores**: Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Marcos (PM)
- **Referências de Código**: `src/server.ts`, `src/config/database.ts`, `prisma/schema.prisma`

---

## Status

Aceito (Decidido na reunião técnica — `[09:10] Larissa`, `[09:11] Diego`, `[09:13] Larissa`, `[09:30] Bruno`).

---

## Contexto

Com a adoção do padrão Transactional Outbox no MySQL (`ADR-001`), os eventos de mudança de status de pedidos ficam registrados na tabela `webhook_outbox` com status `PENDING`. É necessário definir a arquitetura de execução do consumidor (*worker*) responsável por ler esses registros pendentes e efetuar as chamadas HTTP `POST` para os endpoints dos clientes B2B.

Três restrições técnicas e de negócio guiaram esta decisão:
1. **SLA de Latência Percebida**: Os clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) consideram "tempo real" qualquer notificação recebida em menos de 10 segundos após a mudança de status (`[09:02] Marcos`).
2. **Isolamento de Ciclo de Vida e Falhas**: A API HTTP atual é inicializada em `src/server.ts` com graceful shutdown (`SIGINT`/`SIGTERM`). Executar o loop de disparos HTTP dentro do mesmo processo Node.js da API acoplaria o ciclo de vida de ambos (`[09:11] Diego`).
3. **Ordenação de Eventos por Pedido**: Se um pedido transitar rapidamente por múltiplos status (ex.: `PAID` $\rightarrow$ `PROCESSING` $\rightarrow$ `SHIPPED`), os eventos de um mesmo `order_id` devem ser processados na ordem em que ocorreram (`[09:12] Larissa`, `[09:12] Diego`).

---

## Decisão

Implementaremos o processador de webhooks como um **processo Node.js independente** operando via **polling em loop a cada 2 segundos** em instância única (*single-worker*) na v1 (`ADR-002`):

1. **Entrypoint Dedicado (`[09:11] Larissa`, `[09:28] Bruno`)**: Será criado um novo ponto de entrada `src/worker.ts` (acionado via script `"worker"` no `package.json`, seguindo a estrutura de `src/server.ts`), delegando a lógica de processamento para `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`).
2. **Instância Própria de `PrismaClient` (`[09:11] Bruno`, `[09:29] Diego`, `[09:30] Bruno`)**: Por executar em um processo sistema operacional separado da API HTTP, o worker instanciará seu próprio `PrismaClient` (reutilizando a factory `createPrismaClient()` de `src/config/database.ts`) conectado ao mesmo banco MySQL (`DATABASE_URL`).
3. **Polling a Cada 2 Segundos em Pequenos Batches (`[09:08] Diego`, `[09:09] Diego`, `[09:10] Larissa`)**: O worker executará um loop com intervalo de 2 segundos consultando na tabela `webhook_outbox` os eventos mais antigos elegíveis (`status = 'PENDING'` e `next_attempt_at <= NOW()`), apoiado por índice composto em `(status, next_attempt_at, created_at)`.
4. **Single-Worker com Ordenação por `order_id` (`[09:12] Diego`, `[09:13] Larissa`)**: Na primeira versão, rodaremos uma única instância do worker (*single-worker*). A reunião descreveu a ordem por `created_at ASC`. A revisão identificou que isso não basta durante retries nem em timestamps empatados: o FDD propõe sequência de desempate e bloqueio por predecessor em `(webhookId, orderId)`, inclusive em DLQ (`FDD-DET-04`). Essa regra operacional é proposta derivada da garantia de ordem, não uma decisão adicional tomada na call.

---

## Alternativas Consideradas

### 1. Executar o Worker Dentro do Mesmo Processo da API Express (`ADR-002-ALT-01`)
- **Descrição**: Iniciar um `setInterval` dentro do próprio bootstrap da API HTTP em `src/server.ts` (`[09:11] Diego`).
- **Motivo do Descarte (`[09:11] Diego`, `[09:11] Larissa`)**: Se o processo da API reiniciar (por deploy, crash ou escalonamento), o processamento em andamento do worker é interrompido abruptamente. Além disso, picos de processamento de webhooks competiriam pelo mesmo *event loop* e pool de conexões Prisma das requisições HTTP síncronas da API.

### 2. Acionamento Reativo via Triggers do MySQL (`ADR-002-ALT-02`)
- **Descrição**: Utilizar triggers de banco de dados no MySQL para acionar imediatamente o worker quando uma linha fosse inserida em `webhook_outbox` (`[09:09] Bruno`).
- **Motivo do Descarte (`[09:09] Diego`)**: Diferente do PostgreSQL (que possui `LISTEN/NOTIFY` nativo), o MySQL não dispõe de mecanismo nativo para notificar processos externos a partir de triggers. Qualquer solução via trigger exigiria improvisos frágeis (como escrever em arquivos locais ou realizar chamadas HTTP de dentro do SGBD).

### 3. Múltiplos Workers em Paralelo com Particionamento ou Lock Pessimista na v1 (`ADR-002-ALT-03`)
- **Descrição**: Subir múltiplas instâncias concorrentes do worker desde o primeiro lançamento, utilizando particionamento por `order_id` ou `SELECT ... FOR UPDATE SKIP LOCKED` para preservar a ordem (`[09:12] Diego`, `[09:13] Bruno`, `[09:13] Diego`).
- **Motivo do Descarte / Adiamento (`[09:13] Diego`, `[09:13] Larissa`, `[09:14] Marcos`)**: Os clientes não exigiram garantia de ordenação global entre pedidos distintos, e o volume atual é perfeitamente atendido por um único worker lendo em batches a cada 2 segundos. A complexidade de coordenação multi-worker foi registrada como evolução futura caso a escala exija.

---

## Consequências

### Positivas
- **Meta de Latência de Negócio**: Sem backlog e com capacidade livre, o polling acrescenta até aproximadamente 2s de espera para detecção. O tempo de rede e a fila também compõem a meta de menos de 10s; o intervalo de polling isolado não garante seu cumprimento (`[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa`).
- **Isolamento Operacional**: Reinicializações ou picos de tráfego na API (`src/server.ts`) não afetam o worker (`src/worker.ts`) e vice-versa (`[09:11] Diego`).
- **Ordenação Simples e Determinística**: Em modo *single-worker*, a proposta de seleção com predecessor e sequência no FDD preserva a ordem de processamento para um mesmo destino/pedido (`PAID` $\rightarrow$ `PROCESSING` $\rightarrow$ `SHIPPED`) (`[09:12] Diego`).

### Negativas
- **Limitação Conhecida de Escala Horizontal (`[09:12] Diego`, `[09:13] Larissa`)**: Enquanto não houver particionamento por `order_id` ou lock pessimista coordenado, não é possível escalar horizontalmente para múltiplos workers em paralelo sem perder a garantia de ordem para transições rápidas de um mesmo pedido.
- **Bloqueio por Predecessor (proposta derivada)**: Um evento em retry ou DLQ impede sucessores do mesmo destino/pedido até a recuperação; esses sucessores podem exceder a meta de 10s. A recuperação de `PROCESSING` após reinício também é proposta no FDD para preservar entregas.
- **Consumo Adicional de Conexões no MySQL**: O processo separado do worker abre uma segunda instância de `PrismaClient` (`[09:30] Bruno`), consumindo seu próprio pool de conexões no banco de dados.

### Trade-off Explícito (`ADR-002-TRD-01`)
Trocamos a **notificação instantânea sub-milissegundo e o paralelismo multi-worker imediato** pela **simplicidade, previsibilidade operacional e ordem por pedido, com as condições operacionais detalhadas no FDD de um worker único em polling de 2 segundos**, aceitando uma latência base de até 2 segundos e postergando o particionamento multi-worker para quando o volume justificar (`[09:09] Diego`, `[09:10] Larissa`, `[09:13] Diego`).
