# ADR-001: Padrão Transactional Outbox no MySQL com Snapshot JSON na Inserção

- **ID de Rastreabilidade**: `ADR-001`
- **Data**: 2026-10-21
- **Decisores**: Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Marcos (PM), Sofia (Segurança)
- **Referências de Código**: `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `prisma/schema.prisma`

---

## Status

Aceito (Decidido na reunião técnica — `[09:08] Larissa`, `[09:48] Larissa`, `[09:52] Bruno`).

---

## Contexto

O Order Management System (OMS) gerencia o ciclo de vida de pedidos B2B (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED` definidos em `prisma/schema.prisma` e controlados por `src/modules/orders/order.status.ts`). Três clientes estratégicos (Atlas Comercial, MaxDistribuição e Nova Cargo) demandam notificações ativas via webhook sempre que o status de seus pedidos sofrer transição (`[09:00] Marcos`), com latência percebida inferior a 10 segundos (`[09:02] Marcos`).

Atualmente, a transição de estado ocorre em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126–179) dentro de uma transação interativa do Prisma (`this.prisma.$transaction`), que:
1. **`ADR-001-CTX-01`** — Busca o pedido e seus itens (`tx.order.findUnique`);
2. **`ADR-001-CTX-02`** — Valida a transição na máquina de estados (`canTransition(from, to)`);
3. **`ADR-001-CTX-03`** — Debita ou repõe o estoque de produtos (`this.debitStock` / `this.replenishStock` atualizando `tx.product.update`);
4. **`ADR-001-CTX-04`** — Atualiza o status na tabela `orders` (`tx.order.update`);
5. **`ADR-001-CTX-05`** — Insere o registro de auditoria em `order_status_history` (`tx.orderStatusHistory.create`).

A equipe precisava decidir como acoplar a emissão de eventos de webhook a essa operação crítica sem degradar a latência da transação SQL, sem correr o risco de perder notificações quando a mudança de status é efetivada e sem emitir notificações "fantasmas" caso a transação sofra rollback (`[09:03] Larissa`, `[09:04] Bruno`, `[09:06] Diego`).

---

## Decisão

Adotaremos o padrão **Transactional Outbox** persistido no próprio banco de dados **MySQL** já utilizado pela aplicação (`prisma/schema.prisma`), regido pelas seguintes regras arquiteturais (`ADR-001`):

1. **`ADR-001-DEC-01`** — **Atomicidade Transacional (`[09:06] Diego`, `[09:40] Bruno`)**: Durante a execução de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro do mesmo bloco `this.prisma.$transaction(async (tx) => { ... })` que atualiza `orders` e `order_status_history`, será invocada uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:41] Bruno`, `[09:41] Diego`) para inserir os eventos correspondentes na tabela `webhook_outbox`. Se a inserção na outbox falhar, toda a transação de mudança de status sofre rollback (`[09:40] Bruno`).
2. **`ADR-001-DEC-02`** — **Filtragem na Inserção (`[09:33] Marcos`, `[09:34] Bruno`, `[09:34] Diego`)**: Antes de gravar na `webhook_outbox`, a função verifica quais endpoints ativos (`active = true`) do `customerId` daquele pedido assinam o status de destino (`toStatus`). Se nenhum webhook ativo do cliente escutar aquele status, nenhuma linha é inserida na `webhook_outbox`, economizando armazenamento e I/O.
3. **`ADR-001-DEC-03`** — **Snapshot JSON Pré-Renderizado na Inserção (`[09:51] Bruno`, `[09:52] Larissa`, `[09:52] Diego`)**: O campo `payload` da tabela `webhook_outbox` armazenará o snapshot JSON já renderizado no instante exato da transição (contendo `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `customer_id`, `from_status`, `to_status`, `subtotal_cents`, `discount_cents`, `total_cents`, sem a lista de `items` — `[09:43] Diego`). Isso garante que mutações posteriores no pedido não adulterem o estado histórico do evento.
4. **`ADR-001-DEC-04`** — **Chave Primária UUID e Indexação (`[09:08] Diego`, `[09:51] Diego`, `[09:51] Larissa`)**: A tabela `webhook_outbox` utilizará `UUID` (`String @id @default(uuid()) @db.Char(36)`) como chave primária, seguindo o padrão de todas as entidades de negócio em `prisma/schema.prisma`, e contará com índices compostos sobre `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`) e `created_at` para manter a leitura do worker leve.

---

## Alternativas Consideradas

### 1. Disparo HTTP Síncrono dentro de `OrderService.changeStatus` (`ADR-001-ALT-01`)
- **Descrição**: Realizar a chamada HTTP `POST` para a URL do cliente diretamente dentro do método `changeStatus` em `src/modules/orders/order.service.ts` (`[09:03] Larissa`).
- **Motivo do Descarte (`[09:04] Bruno`, `[09:06] Diego`)**: A transação de mudança de status já executa múltiplas leituras e escritas (incluindo lock de linhas de produtos para ajuste de `stockQuantity`). Embutir uma chamada HTTP externa sujeita a até 10 segundos de timeout reteria conexões e locks no MySQL, degradando o throughput de toda a API, além de criar um dilema insolúvel caso o endpoint do cliente estivesse fora do ar (dar rollback em uma mudança de status legítima ou perder a notificação).

### 2. Mensageria Externa com Redis Streams / Redis Cluster (`ADR-001-ALT-02`)
- **Descrição**: Publicar os eventos de mudança de status em um broker dedicado como Redis Streams (`[09:07] Larissa`).
- **Motivo do Descarte (`[09:07] Diego`)**: Rejeitado por configurar *overengineering* para o porte atual da equipe e da operação. Introduziria nova infraestrutura para provisionar, monitorar e manter em alta disponibilidade, além de não oferecer transação ACID nativa compartilhada com o MySQL (exigindo um mecanismo de outbox ou *dual-write* de qualquer forma).

### 3. Armazenar Apenas `order_id` na Outbox e Renderizar o Payload no Envio (`ADR-001-ALT-03`)
- **Descrição**: Salvar apenas a referência `order_id` na `webhook_outbox` e montar o JSON consultando a tabela `orders` apenas no momento em que o worker fosse disparar o HTTP POST (`[09:51] Bruno`).
- **Motivo do Descarte (`[09:52] Larissa`, `[09:52] Diego`)**: Caso o pedido sofresse duas transições rápidas (ex.: `PAID` $\rightarrow$ `PROCESSING` $\rightarrow$ `SHIPPED`) ou demorasse a ser entregue por motivo de retry, a leitura tardia refletiria o estado futuro do pedido em vez do estado exato do momento em que a transição ocorreu.

---

## Consequências

### Positivas
- **`ADR-001-CONS-01`** — **Consistência Transacional Absoluta**: Quando existe endpoint ativo assinando o status, a mudança em `orders` e o evento correspondente na `webhook_outbox` são atômicos, nem de um evento ser emitido caso a transação do pedido falhe (`[09:06] Diego`, `[09:40] Bruno`).
- **`ADR-001-CONS-02`** — **Zero Infraestrutura Nova**: Aproveita integralmente a instância MySQL e o `PrismaClient` já configurados no projeto (`prisma/schema.prisma`, `src/config/database.ts`), acelerando a entrega dentro do prazo de 3 sprints (`[09:07] Diego`, `[09:46] Larissa`).
- **`ADR-001-CONS-03`** — **Imutabilidade Histórica do Evento**: O snapshot JSON gravado na inserção preserva fielmente os valores e status do instante da transição, facilitando auditoria e reprocessamento (`[09:52] Larissa`).

### Negativas
- **`ADR-001-CONS-04`** — **Aumento de Escrita e Armazenamento no MySQL**: Cada mudança de status assinada por um cliente adiciona `INSERT`s dentro da transação de `OrderService.changeStatus` e armazena documentos JSON de até 64KB na tabela `webhook_outbox` (`[09:07] Bruno`, `[09:24] Diego`).
- **`ADR-001-CONS-05`** — **Necessidade Futura de Expurgo**: Linhas com status `DELIVERED` acumularão ao longo do tempo e exigirão uma rotina futura de arquivamento/expurgo (ex.: após 30 dias), deixada fora do escopo desta v1 (`[09:08] Diego`).

### Trade-off Explícito (`ADR-001-TRD-01`)
Trocamos a **separação física de armazenamento entre banco transacional e fila de mensagens** (que um broker dedicado como Redis Streams proporcionaria) pela **garantia ACID transacional imediata e simplicidade operacional no MySQL existente**, aceitando um leve custo adicional de escrita na transação de `OrderService.changeStatus` e a responsabilidade futura de expurgar registros entregues após 30 dias (`[09:06] Diego`, `[09:07] Diego`, `[09:08] Diego`).
