# OMS Webhook Design Docs Constitution

## Core Principles

### I. Rastreabilidade Estrita e Anti-Alucinação (NON-NEGOTIABLE)
- Toda informação, requisito funcional ou não funcional, decisão arquitetural, contrato ou
  restrição registrada nos documentos MUST ter origem identificável em `TRANSCRICAO.md`
  (no formato `[hh:mm] Nome`) ou no código-fonte existente do repositório (`src/`, `prisma/`).
- É estritamente proibido (MUST NOT) inventar requisitos, endpoints, payloads, decisões ou
  restrições sem lastro verificável nas fontes primárias.
- O arquivo `docs/TRACKER.md` MUST mapear no mínimo 80% dos itens identificáveis produzidos
  nos documentos, contendo pelo menos 70% das linhas com `Fonte = TRANSCRICAO` (com timestamp
  e nome do falante válidos) e pelo menos 5 linhas com `Fonte = CODIGO` (com caminho de arquivo
  real).

**Rationale**: Garante a integridade factual do pacote de documentação contra alucinações de
modelos de IA e assegura alinhamento absoluto com o que foi decidido na reunião técnica e com a
implementação real do Order Management System (OMS).

### II. Imutabilidade do Código e da Transcrição (NON-NEGOTIABLE)
- A entrega deste projeto é estritamente documental; os arquivos de código da aplicação
  (`src/`, `prisma/`, `tests/`, arquivos de configuração e manifesto) e a transcrição original
  (`TRANSCRICAO.md`) MUST NOT ser alterados, criados ou removidos.
- O código-fonte existente MUST ser utilizado exclusivamente como insumo de leitura e referência
  para fundamentar a especificação de integração da nova feature.

**Rationale**: Preserva a linha de base (baseline) técnica do repositório para auditoria e
avaliação, mantendo o foco exclusivo na produção de design docs acionáveis.

### III. Separação de Alturas e Fronteiras Documentais
- Cada artefato do pacote documental MUST operar em sua altura específica de abstração, sendo
  vedada (MUST NOT) a duplicação de detalhamento entre documentos:
  - **PRD (`docs/PRD.md`) — Altura Produto/Negócio**: Responde *"Por que e o quê?"* (problema,
    público-alvo, objetivos com metas quantitativas, escopo, requisitos e riscos).
  - **RFC (`docs/RFC.md`) — Altura Arquitetura**: Responde *"Como pretendemos resolver e o que
    ainda está em aberto?"* em formato conciso (2 a 4 páginas), cobrindo proposta geral,
    alternativas descartadas e questões em aberto, sem repetir o detalhe de implementação do FDD.
  - **ADRs (`docs/adrs/ADR-NNN-*.md`) — Altura Decisão Pontual**: Responde *"Por que decidimos
    exatamente assim?"* para cada decisão arquitetural isolada, com contexto, alternativas e
    consequências/trade-offs.
  - **FDD (`docs/FDD.md`) — Altura Implementação**: Responde *"Como construir, em detalhe?"*
    com fluxos, contratos públicos HTTP, matriz de erros `WEBHOOK_*`, resiliência,
    observabilidade e integração com o código existente.
  - **Tracker (`docs/TRACKER.md`) — Altura Transversal**: Responde *"De onde veio cada coisa?"*
    por meio de matriz de rastreabilidade cruzada.

**Rationale**: Evita redundância e inconsistências na manutenção documental, garantindo que cada
audiência encontre a informação na profundidade exata de sua responsabilidade.

### IV. Filtragem Rigorosa de Escopo e Exclusões Explícitas
- Ideias, abordagens ou funcionalidades mencionadas na reunião que foram descartadas ou adiadas
  para fases futuras MUST NOT ser registradas como requisitos ativos da solução.
- Pontos descartados ou adiados MUST ser classificados nos locais apropriados:
  - Na seção **"Fora de escopo"** do PRD (mínimo de 2 itens descartados ou adiados).
  - Na seção **"Alternativas consideradas"** do RFC (mínimo de 2 alternativas descartadas com o
    respectivo trade-off) e dos ADRs pertinentes.
  - Na seção **"Questões em aberto"** do RFC (mínimo de 2 pontos levantados e não decididos ou
    adiados na reunião).

**Rationale**: Previne *scope creep* decorrente de interpretação acrítica da transcrição e
preserva a memória técnica sobre o que foi deliberadamente rejeitado ou postergado.

### V. Aderência Arquitetural e Acionabilidade no Código Base
- A especificação técnica MUST respeitar e reutilizar a arquitetura e as convenções do OMS
  existente (Node.js, TypeScript, Express, MySQL via Prisma, validação com Zod, máquina de
  estados de pedidos, transação em `changeStatus`, classes de erro, middleware `requireRole`,
  error middleware centralizado e logger Pino).
- O FDD MUST incluir a seção obrigatória **"Integração com o sistema existente"**, nomeando e
  detalhando a integração com pelo menos 4 caminhos de arquivo reais do código base.
- Pelo menos 1 ADR MUST referenciar explicitamente arquivos, módulos ou classes reais do código
  existente.
- 100% dos caminhos apresentados como código existente MUST existir no repositório. Arquivos
  futuros MUST ser identificados como propostas e MUST NOT ser usados como Fonte CODIGO.
- Detalhes de implementação derivados MUST indicar fundamento e justificativa; MUST NOT ser
  atribuídos aos participantes como decisões adicionais aprovadas sem evidência na reunião.

**Rationale**: Assegura que o pacote documental seja diretamente acionável pelo time de
engenharia sem introduzir dissonância arquitetural ou referências inexistentes.

## Requisitos Estruturais e Critérios de Aceite dos Entregáveis

Todos os documentos entregues MUST estar em formato Markdown e cumprir integralmente os
critérios obrigatórios abaixo:

- **PRD (`docs/PRD.md`)**:
  - MUST conter as seções: Resumo e contexto da feature, Problema e motivação, Público-alvo e
    cenários de uso, Objetivos e métricas de sucesso (com pelo menos 1 meta quantitativa),
    Escopo (incluso e fora de escopo com no mínimo 2 itens descartados/adiados), Requisitos
    funcionais (mínimo de 8 requisitos discutidos na reunião), Requisitos não funcionais,
    Decisões e trade-offs principais, Dependências, Riscos e mitigação (mínimo de 2 riscos com
    probabilidade, impacto e mitigação), Critérios de aceitação, e Estratégia de testes e
    validação.
- **RFC (`docs/RFC.md`)**:
  - MUST conter as seções: Metadados (autor, status, data e participantes da reunião como
    revisores), Resumo executivo (TL;DR), Contexto e problema, Proposta técnica, Alternativas
    consideradas (mínimo de 2 alternativas descartadas com trade-off), Questões em aberto
    (mínimo de 2 pontos adiados/não decididos), Impacto e riscos, e Decisões relacionadas
    (links para no mínimo 2 ADRs do pacote).
- **FDD (`docs/FDD.md`)**:
  - MUST conter as seções: Contexto e motivação técnica, Objetivos técnicos, Escopo e exclusões,
    Fluxos detalhados (criação do evento na outbox, processamento pelo worker, retry, DLQ),
    Contratos públicos (mínimo de 4 endpoints HTTP com payloads de exemplo de request/response,
    headers, status codes e semântica), Matriz de erros previstos (com códigos no padrão
    `WEBHOOK_*`), Estratégias de resiliência (timeouts, retries, backoff, fallback),
    Observabilidade (citando métricas, logs e tracing), Dependências e compatibilidade,
    Integração com o sistema existente (mínimo de 4 caminhos de arquivo reais), Critérios de
    aceite técnicos, e Riscos e mitigação.
- **ADRs (`docs/adrs/ADR-NNN-titulo-em-kebab-case.md`)**:
  - A pasta `docs/adrs/` MUST conter entre 5 e 8 arquivos seguindo o padrão MADR com as seções:
    Status, Contexto, Decisão, Alternativas Consideradas (mínimo de 1 alternativa real ou
    plausível) e Consequências (positivas e negativas, com trade-off explícito).
  - O conjunto de ADRs MUST cobrir no mínimo 5 das 6 decisões principais da reunião:
    1. Padrão Outbox no MySQL;
    2. Política de retry com backoff e DLQ;
    3. Autenticação HMAC-SHA256 com secret por endpoint;
    4. Garantia at-least-once com `X-Event-Id`;
    5. Worker em processo separado em polling;
    6. Reuso dos padrões existentes do projeto.
- **Tracker (`docs/TRACKER.md`)**:
  - MUST seguir exatamente a tabela `| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |`,
    atendendo aos limiares mínimos de cobertura (80% geral, 70% `TRANSCRICAO`, >= 5 linhas
    `CODIGO`).
- **README (`README.md`)**:
  - Ao final da produção dos documentos em `docs/`, o `README.md` na raiz MUST ser atualizado
    para documentar a jornada de engenharia assistida por IA, contendo obrigatoriamente: Sobre o
    desafio (1 a 2 parágrafos), Ferramentas de IA utilizadas (mínimo de 1 ferramenta e seu
    papel), Workflow adotado, Prompts customizados (mínimo de 2 prompts em blocos de código),
    Iterações e ajustes (mínimo de 2 correções/ajustes concretos e total de iterações), e Como
    navegar a entrega.

## Fluxo de Trabalho e Orquestração com IA

- **Orquestração Human-in-the-Loop (Papel de Maestro)**: A IA MUST ser empregada com prompts
  dirigidos e contextualizados no código e na transcrição. Todo artefato gerado MUST passar por
  ciclos iterativos de revisão crítica humana (tipicamente 3 a 5 iterações) para corrigir
  superficialidades, remover trechos vagos, eliminar alucinações e assegurar consistência entre
  documentos.
- **Sequência de Produção**: A elaboração dos artefatos SHOULD seguir a ordem de dependência
  estrutural para maximizar consistência e evitar retrabalho:
  1. Exploração e mapeamento cruzado de `TRANSCRICAO.md` e do código-fonte (`src/`, `prisma/`);
  2. **ADRs (`docs/adrs/`)**: Formalização das decisões arquiteturais fechadas;
  3. **RFC (`docs/RFC.md`)**: Consolidação da visão arquitetural, alternativas e pendências,
     vinculando os ADRs;
  4. **FDD (`docs/FDD.md`)**: Especificação técnica profunda de implementação e integração;
  5. **PRD (`docs/PRD.md`)**: Consolidação de produto, escopo, requisitos e métricas;
  6. **Tracker (`docs/TRACKER.md`)**: Verificação exaustiva de rastreabilidade item a item;
  7. **README (`README.md`)**: Relato transparente do workflow, prompts e iterações;
  8. **Auditoria de Aceite**: Validação integral da checklist obrigatória antes da entrega.

## Governance

- **Supremacia e Conformidade**: Esta Constituição governa toda a produção documental do
  repositório. Qualquer especificação, plano, tarefa ou documento gerado MUST ser verificado
  contra estes princípios e critérios de aceite. Contradições com `TRANSCRICAO.md` ou com o
  código-fonte em `src/` e `prisma/` invalidam o artefato até sua correção.
- **Procedimento de Emenda (Amendment Procedure)**: Alterações nesta Constituição MUST ser
  realizadas via atualização estruturada de `.specify/memory/constitution.md`, acompanhadas de
  justificativa documentada, validação de aderência às regras do desafio e atualização do
  Relatório de Impacto de Sincronização (Sync Impact Report).
- **Política de Versionamento (Versioning Policy)**: A versão da Constituição MUST seguir o
  padrão Semantic Versioning (`MAJOR.MINOR.PATCH`):
  - `MAJOR`: Remoção ou redefinição incompatível de princípios inegociáveis ou restrições do
    desafio.
  - `MINOR`: Inclusão de novos princípios, seções de governança ou expansão material de
    critérios de qualidade.
  - `PATCH`: Refinamentos de redação, correções ortográficas ou esclarecimentos sem alteração de
    regras.
- **Revisão de Qualidade (Compliance Review)**: Antes de considerar qualquer entrega finalizada,
  cada item da seção de Critérios de Aceite e a existência real de todos os arquivos de código
  mencionados MUST ser auditados e validados.

**Version**: 1.0.1 | **Ratified**: 2026-10-21 | **Last Amended**: 2026-10-21

**Emenda 1.0.1 — Clarificação de rastreabilidade**: distingue arquivos existentes de propostas futuras e detalhes derivados de decisões da reunião. Não relaxa a imutabilidade nem altera os limiares do desafio. Impacto sincronizado em spec.md, FDD, RFC, Tracker e quickstart.md.
