# Checklist da Revisão Documental

**Matriz completa dos 31 critérios**: [acceptance.md](./acceptance.md).

**Referência**: [enunciado original](../enunciado-original.md). **Resultados e limites**: [review.md](../review.md).

## Estrutura e Escopo

- [x] PRD com 12 seções, pelo menos oito requisitos funcionais, metas quantitativas, exclusões e riscos com probabilidade/impacto/mitigação.
- [x] RFC com metadados, revisores da reunião, proposta arquitetural, alternativas, pontos em aberto e links para os ADRs; sem duplicar contratos do FDD.
- [x] FDD com fluxos, contratos e exemplos HTTP, erros de domínio WEBHOOK_*, resiliência, métricas/logs/tracing e integração com arquivos existentes.
- [x] Seis ADRs com Status, Contexto, Decisão, Alternativas Consideradas e Consequências, cobrindo as seis decisões centrais.
- [x] README com ferramentas, workflow, prompts, iterações e guia de leitura.
- [x] Nenhuma alteração em código/configuração existente da aplicação ou transcrição, comparando a entrega com a base do PR.

## Rastreabilidade e Coerência

- [x] Todas as 266 unidades identificáveis têm linha única no Tracker; 224 fontes TRANSCRICAO e 42 CODIGO.
- [x] Marcadores de transcrição e caminhos de código do Tracker existem nas fontes; fontes adicionais relevantes foram incluídas.
- [x] Propostas derivadas distinguem fundamento e interpretação de decisão literal da reunião; revisão técnica encaminhada no RFC.
- [x] Retry distingue envio inicial e cinco retentativas; quinta espera de 12h alcançável.
- [x] Recuperação de PROCESSING, bloqueio por predecessor, desempate e ciclos de DLQ descritos de modo consistente.
- [x] Formato HMAC único, expiração da chave anterior e rotação sobreposta tratados como propostas explícitas.
- [x] Importação de OrderStatus e códigos dos construtores existentes conferidos no código.
- [x] Schema proposto validado com Prisma 5.22.0 em arquivo temporário, incluindo relações reversas.
- [x] Exemplos JSON válidos e simulações dos invariantes documentados executadas.

## Limites da Verificação

A checklist registra verificações documentais e simulações, não testes de uma feature implementada. Aprovação das propostas pela equipe, concorrência em MySQL e comportamento sob carga não foram executados nem alegados. Não se usa checklist genérica de software como prova de que a especificação é livre de detalhes técnicos: o desafio exige justamente documentação técnica em suas alturas apropriadas.
