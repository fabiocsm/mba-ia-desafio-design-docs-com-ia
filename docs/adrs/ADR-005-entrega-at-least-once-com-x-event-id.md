# ADR-005: Semântica de Entrega At-Least-Once com Idempotência via `X-Event-Id`

- **ID de Rastreabilidade**: `ADR-005`
- **Data**: 2026-10-21
- **Decisores**: Diego (Engenheiro Sênior — Plataforma), Larissa (Tech Lead), Sofia (Engenheira de Segurança), Bruno (Engenheiro Pleno — Pedidos), Marcos (PM)
- **Referências de Código**: `prisma/schema.prisma`, `src/modules/orders/order.service.ts`

---

## Status

Aceito (Decidido na reunião técnica — `[09:24] Diego`, `[09:25] Diego`, `[09:26] Larissa`, `[09:44] Diego`).

---

## Contexto

Em uma arquitetura de notificações assíncronas via HTTP com política de retry (`ADR-003`), falhas de rede ambíguas são inevitáveis. Por exemplo, o worker do OMS pode enviar um HTTP `POST` para o endpoint do cliente B2B, o servidor do cliente pode processar a mensagem com sucesso, mas a conexão TCP pode cair antes do retorno do HTTP `200 OK` ou a resposta pode ultrapassar o timeout de 10 segundos (`[09:42] Diego`). Nesse cenário, o worker tratará a tentativa como falha e reenviará o evento no ciclo de retry seguinte (`[09:24] Diego`).

A equipe precisava decidir qual garantia formal de entrega o OMS oferecerá aos clientes e como evitar que reenvios legítimos causem processamento duplicado nos sistemas receptores (`[09:24] Diego`, `[09:25] Bruno`).

---

## Decisão

Adotaremos a garantia de entrega **At-Least-Once** (pelo menos uma vez) combinada com chave de idempotência única **`X-Event-Id`** gerada na origem (`ADR-005`):

1. **`ADR-005-DEC-01`** — **Garantia *At-Least-Once* (`[09:24] Diego`, `[09:26] Larissa`)**: O OMS mantém a intenção de entrega de cada evento elegível e permite reenvios, com envio inicial, cinco retentativas e replay manual da DLQ; a entrega depende da recuperação do receptor e da intervenção administrativa após esgotamento. Enquanto os registros forem preservados, o evento permanece recuperável; podem ocorrer entregas duplicadas em cenários de timeout ou falha de confirmação de rede.
2. **`ADR-005-DEC-02`** — **UUID Único por Evento Gerado na Entrada da Outbox (`[09:25] Diego`, `[09:51] Larissa`)**: No momento em que o evento é inserido na tabela `webhook_outbox` dentro da transação de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), um identificador `UUID` único é atribuído ao evento (seguindo o padrão `@default(uuid()) @db.Char(36)` de `prisma/schema.prisma`).
3. **`ADR-005-DEC-03`** — **Propagação Invariante em Header (`X-Event-Id`) e Payload (`event_id`) (`[09:25] Diego`, `[09:43] Diego`, `[09:44] Diego`)**: Em todas as tentativas de entrega (incluindo retentativas automáticas e replays manuais a partir da DLQ), o worker enviará exatamente o mesmo identificador tanto no cabeçalho HTTP `X-Event-Id` quanto na propriedade `event_id` do corpo JSON.
4. **`ADR-005-DEC-04`** — **Deduplicação no Cliente Receptor (`[09:25] Diego`, `[09:25] Sofia`, `[09:26] Marcos`)**: O contrato de integração documentado no portal do desenvolvedor especificará explicitamente que os clientes receptores devem implementar idempotência/deduplicação armazenando os `event_id` já processados.

---

## Alternativas Consideradas

### 1. Garantia *Exactly-Once* Coordenada entre OMS e Cliente (`ADR-005-ALT-01`)
- **Descrição**: Implementar um protocolo transacional de duas fases (*two-phase commit* / handshake de confirmação bilateral) entre o OMS e os servidores dos clientes para assegurar entrega exatamente uma única vez (`[09:25] Diego`).
- **Motivo do Descarte (`[09:25] Diego`)**: Exigiria coordenação de estado distribuído entre o OMS e múltiplos sistemas externos heterogêneos sobre os quais não temos controle, elevando drasticamente a complexidade de integração para os próprios clientes B2B sem benefício prático sobre a deduplicação por ID.

### 2. Entrega *At-Most-Once* (Disparo Único sem Retentativa) (`ADR-005-ALT-02`)
- **Descrição**: Enviar o webhook apenas uma vez e, em caso de timeout ou erro HTTP, descartar o evento para nunca enviar duplicatas (`[09:14] Larissa`, `[09:15] Diego`).
- **Motivo do Descarte (`[09:00] Marcos`, `[09:15] Diego`, `[09:16] Diego`)**: Qualquer oscilação de rede ou manutenção no cliente causaria perda silenciosa de atualizações de pedidos, forçando os clientes a manterem o polling periódico em `GET /api/v1/orders` — o que invalidaria o objetivo central da feature.

---

## Consequências

### Positivas
- **`ADR-005-CONS-01`** — **Confiabilidade de Entrega**: Eventos persistidos podem ser reenviados após falhas de rede, mantendo a simplicidade do protocolo HTTP `POST` padrão de mercado (mesmo modelo utilizado por Stripe e GitHub — `[09:25] Diego`).
- **`ADR-005-CONS-02`** — **Rastreabilidade Fim a Fim**: O mesmo UUID (`X-Event-Id` / `event_id`) acompanha o evento desde a `webhook_outbox`, passando pelos logs do worker e registros em `webhook_deliveries` até o sistema do cliente (`[09:25] Diego`, `[09:34] Marcos`).

### Negativas
- **`ADR-005-CONS-03`** — **Responsabilidade de Idempotência no Cliente (`[09:25] Sofia`)**: Transfere para os sistemas consumidores dos clientes B2B a obrigação técnica de descartar eventos com `X-Event-Id` repetido.

### Trade-off Explícito (`ADR-005-TRD-01`)
Trocamos a **complexidade proibitiva de coordenação distribuída *exactly-once*** (ou o risco de perda de eventos de *at-most-once*) pela **entrega *at-least-once* com identificador imutável `X-Event-Id`**, aceitando que o cliente possa receber o mesmo evento mais de uma vez em falhas de confirmação de rede e delegando a deduplicação por `event_id` ao consumidor (`[09:24] Diego`, `[09:25] Sofia`, `[09:25] Diego`).

**Limites operacionais da proposta**: o FDD descreve recuperação de `PROCESSING` no reinício e replays preservando o mesmo UUID (`FDD-DET-02`/`FDD-DET-03`). Não se promete entrega a receptor permanentemente indisponível. A remoção explícita de endpoint ou pedido elimina seus registros conforme as relações propostas; duplicatas são possíveis e a deduplicação permanece responsabilidade do cliente.
