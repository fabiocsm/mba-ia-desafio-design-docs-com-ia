# Revisão Documental do PR #1

**Base do desafio**: `e7f6311b96ee3b21bf8cb7077f53a6e4045143eb`.
**Versão revisada inicialmente**: `53568271893a2efa72003246717da0fcf668eb48`.
**Escopo**: documentação; código da aplicação e transcrição preservados.

## Correções da Primeira Revisão

| Achado | Correção documental |
| --- | --- |
| Espera final de 12h inalcançável | Inicial + cinco retries; seis resultados persistidos por ciclo; tabela e contador alinhados. A interpretação da fala ambígua está explícita para revisão. |
| PROCESSING abandonado após crash | Recuperação no bootstrap, condicionada à exclusividade do worker; UUID e snapshot preservados. |
| Unicidade da DLQ impede nova falha após replay | Unicidade (outboxId, cycle); replay condicional; histórico preservado em novos ciclos. |
| Retry permite ultrapassagem de eventos | Seleção com predecessor por endpoint/pedido; retry e DLQ bloqueiam sucessores. |
| Construtores de erro incompatíveis com WEBHOOK_* | AppError/construtores parametrizáveis e adaptador local de validate; erros genéricos mantêm códigos existentes. |
| Origem incorreta de OrderStatus | Enum importado de @prisma/client, definido em prisma/schema.prisma. |
| Dois formatos alternativos de assinatura | Formato único de múltiplos digests sha256= em X-Signature; verificação descrita no FDD. |
| Referências existentes não sustentam todos os detalhes | Inventário ampliado e rótulos de proposta derivada; fontes indicam fundamento, não aprovação literal de cada código/regra. |
| git status ignora alterações já commitadas | Auditoria compara a entrega com a base conhecida e verifica arquivos novos nos caminhos protegidos. |

## Segunda Revisão e Ajustes Adicionais

A leitura cruzada após as correções identificou riscos de inconsistência que também foram tratados:

- Eventos com createdAt igual: sequência interna autoincremental desempata sem substituir o UUID público.
- Replay simultâneo: atualização condicional de estado/ciclo evita dois novos ciclos para a mesma linha de DLQ.
- Rotação repetida dentro de 24h: proposta de 409 preserva a chave anterior até sua expiração; atualização serializada na linha do endpoint.
- Tempo HTTP confundido com latência total: duração da chamada, fila e confirmação da entrega são métricas distintas. A aproximação usa o timestamp do evento anterior ao commit, não promete medir o commit exato nem o processamento interno do receptor.
- X-Timestamp tratado como proteção criptográfica contra replay: explicitado que o header não está coberto pelo HMAC sobre o corpo e não basta sozinho; deduplicação continua necessária.
- Garantias absolutas de entrega, retrocompatibilidade e SLA: condicionadas à recuperação do receptor, preservação dos registros, capacidade do worker e efeitos das cascatas; backlog e predecessor bloqueado aparecem como limites.
- Persistência após resultado de commit incerto: unicidade (outboxId, cycle, attemptNumber), upsert do resultado e atualização condicional da outbox evitam duplicação de histórico ao repetir a persistência.
- Planejamento com contratos duplicados: data-model.md e webhook-http-contracts.md apontam para o FDD canônico e refletem os mesmos estados e contadores.

## Validação da Primeira Correção (Snapshot Histórico)

| Verificação executada | Resultado |
| --- | --- |
| Imutabilidade contra a base, incluindo commits e arquivos novos nos caminhos protegidos | Passou; código/configuração da aplicação e transcrição preservados. |
| Cópia do enunciado versus README da base | Idêntica. |
| Inventário do Tracker | 161/161 IDs explícitos, sem duplicatas; 134 TRANSCRICAO (83,2%) e 27 CODIGO. |
| Existência de timestamps, documentos e caminhos; links locais dos entregáveis | Passou. A evidência semântica foi revisada separadamente. |
| Schema proposto + schema existente e relações reversas, em arquivo temporário | `prisma validate` com Prisma 5.22.0: válido; sem migração nem conexão a banco. |
| Simulação de retry | Seis chamadas; intervalos 1m/5m/30m/2h/12h; 14h36m de espera acumulada. |
| Simulação de seleção/recuperação | PROCESSING recuperado; timestamps iguais desempataram por sequência; retry/DLQ bloquearam sucessores sem bloquear outros destinos/pedidos. |
| Modelo SQL de invariantes de replay e persistência (SQLite em memória) | Replay repetido não criou outro ciclo; novo esgotamento preservou a DLQ anterior; resultado repetido não duplicou histórico. Não é teste de concorrência MySQL. |
| Modelo criptográfico | Chaves atual/anterior verificaram o mesmo header na janela; chave antiga expirou; alteração do corpo foi rejeitada. Condição de rotação sobreposta conferida. |
| Exemplos JSON do FDD | Nove blocos válidos. |
| Última varredura de todos os Markdown alterados | Links locais, tabelas e referências concretas à transcrição válidos; diff sem erros de whitespace. |

Após os ajustes adicionais acima, a última leitura cruzada e as verificações não identificaram novas inconsistências no escopo revisado. As limitações operacionais foram documentadas em vez de apresentadas como garantias universais.

As propostas FDD-DET-01 a FDD-DET-08 estão submetidas à revisão técnica em RFC-REVIEW-01. Esta revisão documental não inventa aprovação dos participantes, não implementa a feature e não comprova seu comportamento em produção.

## Auditoria Posterior dos Critérios de Aceite

A solicitação de conferência item a item ampliou a auditoria: o inventário anterior de 161 IDs foi substituído por 266 unidades identificáveis, levantadas também nas seções sem ID. Todas as 266 unidades têm linha no Tracker; 224 apontam para a transcrição (84,2%) e 42 para código existente.

A [matriz de aceite](./checklists/acceptance.md) registra 31/31 critérios contemplados, as evidências e as correções: cobertura mensurável, referências estritas a arquivos existentes e quatro pares explícitos de payloads JSON de entrada/saída no FDD. A conferência semântica também condicionou a atomicidade do ADR-001 à existência de assinaturas e removeu a promessa residual de latência garantida pelo polling no PRD. As fontes dos novos itens foram conferidas por conteúdo, incluindo o timeout/retry/DLQ e os componentes reais da arquitetura.

O [quickstart](./quickstart.md) valida agora todas as referências concretas nos Markdown da entrega, sem exceções por prefixo, e a presença de cada ID no documento indicado pelo Tracker. Esta etapa não alterou os modelos Prisma propostos, apenas a documentação e os exemplos de request; os números e limites da tabela anterior são históricos, não os resultados atuais.

**Verificação final desta auditoria**: 31 critérios registrados com evidência, 266/266 unidades rastreadas, 224 fontes TRANSCRICAO e 42 CODIGO, 15 caminhos concretos distintos existentes, 14 caminhos na integração e 11 exemplos JSON válidos no FDD. Tabelas, links locais, referências de transcrição e imutabilidade passaram; diff sem erros de whitespace.
