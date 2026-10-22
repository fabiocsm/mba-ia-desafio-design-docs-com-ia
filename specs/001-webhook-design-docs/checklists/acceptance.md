# Matriz de Critérios de Aceite

**Resultado após correções: 31/31 critérios contemplados documentalmente.** A referência é o [enunciado preservado](../enunciado-original.md) e a checklist fornecida pelo usuário. A auditoria compara o conteúdo com a transcrição e o código, além das verificações mecânicas. Não se limita a marcar títulos existentes.

## Resultado por Critério

| ID | Grupo e critério obrigatório | Resultado | Evidência |
| --- | --- | --- | --- |
| ACE-01 | PRD: Arquivo existe e está em Markdown | Atendido | [PRD](../../../docs/PRD.md). |
| ACE-02 | PRD: Contém todas as seções obrigatórias do requisito 1 | Atendido | [PRD](../../../docs/PRD.md), seções 1 a 12: contexto, problema, público, objetivos, escopo, RF/RNF, decisões, dependências, riscos, aceitação e testes. |
| ACE-03 | PRD: Identifica no mínimo 8 requisitos funcionais discutidos na reunião | Atendido | [PRD §6](../../../docs/PRD.md#6-requisitos-funcionais): 12 RF; cadastro/CRUD, filtro na inserção, outbox, payload, headers, rotação, histórico, retry, DLQ, replay e acesso, confrontados com as falas citadas. |
| ACE-04 | PRD: Inclui objetivo com métrica e meta quantitativa | Atendido | [PRD §4](../../../docs/PRD.md#4-objetivos-e-métricas-de-sucesso): três objetivos; latência inferior a 10s, prazo de três sprints e janela de retry de quase 15h. |
| ACE-05 | PRD: Fora de escopo lista pelo menos 2 itens descartados/adiados | Atendido | [PRD §5.2](../../../docs/PRD.md#52-fora-de-escopo-descartados-ou-adiados-na-reunião): cinco itens com origem, incluindo inbound, e-mail e dashboard. |
| ACE-06 | PRD: Riscos incluem pelo menos 2 com probabilidade, impacto e mitigação | Atendido | [PRD §10](../../../docs/PRD.md#10-riscos-e-mitigação): cinco linhas completas; probabilidades explicitadas como estimativas da elaboração. |
| ACE-07 | RFC: Arquivo existe e está em Markdown | Atendido | [RFC](../../../docs/RFC.md). |
| ACE-08 | RFC: Contém todas as seções obrigatórias do requisito 2 | Atendido | [RFC](../../../docs/RFC.md): oito seções, autor/status/data e os cinco participantes como revisores. |
| ACE-09 | RFC: Alternativas consideradas inclui pelo menos 2 descartadas, com trade-off | Atendido | [RFC](../../../docs/RFC.md#alternativas-consideradas): seis alternativas. HTTP síncrono e Redis são rejeições explícitas; efeitos técnicos derivados são identificados como análise. |
| ACE-10 | RFC: Questões em aberto inclui pelo menos 2 pontos adiados/não decididos | Atendido | [RFC](../../../docs/RFC.md#questões-em-aberto): cinco assuntos futuros levantados na reunião; propostas derivadas da implementação têm subseção separada. |
| ACE-11 | RFC: Referencia pelo menos 2 ADRs com link | Atendido | [RFC](../../../docs/RFC.md#decisões-relacionadas): seis links locais existentes. |
| ACE-12 | FDD: Arquivo existe e está em Markdown | Atendido | [FDD](../../../docs/FDD.md). |
| ACE-13 | FDD: Contém todas as seções obrigatórias do requisito 3 | Atendido | [FDD](../../../docs/FDD.md), seções 1 a 12; inclui contexto, objetivos, escopo, quatro fluxos, contratos, erros, resiliência, observabilidade, dependências, integração, aceitação e riscos. |
| ACE-14 | FDD: Contratos públicos inclui pelo menos 4 endpoints com request, response e status codes | Atendido | [FDD §5](../../../docs/FDD.md#5-contratos-públicos): criação, PATCH, rotação e replay têm pares de blocos JSON de entrada/saída e códigos HTTP. Há sete rotas REST no total, além do contrato outbound; requests vazios estão explícitos como JSON. |
| ACE-15 | FDD: Matriz de erros usa prefixo WEBHOOK_ | Atendido | [FDD §6](../../../docs/FDD.md#6-matriz-de-erros-previstos): 11 códigos de domínio WEBHOOK_*. Erros genéricos de infraestrutura/autenticação permanecem compatíveis com o código existente. |
| ACE-16 | FDD: Integração referencia pelo menos 4 caminhos reais de código | Atendido | [FDD §10](../../../docs/FDD.md#10-integração-com-o-sistema-existente): 14 caminhos reais em 12 linhas de integração; existência e símbolos conferidos. |
| ACE-17 | FDD: Observabilidade cita métricas, logs e tracing | Atendido | [FDD §8](../../../docs/FDD.md#8-observabilidade): tempo HTTP separado da latência de notificação, métricas de fila, Pino/auditoria e correlação por UUID. |
| ACE-18 | ADRs: Contém entre 5 e 8 arquivos no padrão ADR-NNN-titulo-em-kebab-case.md | Atendido | [Índice dos ADRs](../../../docs/adrs/README.md): seis arquivos com nomes válidos. |
| ACE-19 | ADRs: Cada ADR contém Status, Contexto, Decisão, Alternativas Consideradas e Consequências | Atendido | Os [seis ADRs](../../../docs/adrs/README.md) possuem as cinco seções, alternativas reais/plausíveis e consequências positivas/negativas com trade-off. |
| ACE-20 | ADRs: O conjunto cobre pelo menos 5 das 6 decisões principais | Atendido | ADR-001 a ADR-006 cobrem 6/6: outbox, worker, retry/DLQ, HMAC/rotação, at-least-once e reuso. |
| ACE-21 | ADRs: Pelo menos 1 ADR referencia arquivos, módulos ou classes existentes | Atendido | Todos os [ADRs](../../../docs/adrs/README.md) citam código existente; ADR-006 detalha AppError, middlewares, composição e logger. |
| ACE-22 | Tracker: Arquivo existe e usa o formato de tabela obrigatório | Atendido | [Tracker](../../../docs/TRACKER.md): ID, Documento, Tipo, Conteúdo (resumo), Fonte e Localização; seis colunas em cada linha. |
| ACE-23 | Tracker: Pelo menos 80% dos itens identificáveis têm linha correspondente | Atendido | Inventário ampliado: 266 unidades, 266 correspondências (100%). Categorias e regras de consolidação estão no [método do Tracker](../../../docs/TRACKER.md#inventário-e-método-de-cobertura); denominador por documento abaixo. |
| ACE-24 | Tracker: Pelo menos 70% das linhas têm TRANSCRICAO e marcador válido | Atendido | 224/266 = 84,2%. Todos os marcadores e falantes existem literalmente na [transcrição](../../../TRANSCRICAO.md). A pertinência da fonte foi revisada separadamente. |
| ACE-25 | Tracker: Pelo menos 5 linhas têm CODIGO e arquivo real | Atendido | 42 linhas CODIGO; todos os caminhos existem. Caminho futuro não é usado como evidência. |
| ACE-26 | README: Contém todas as seções obrigatórias do requisito 6 | Atendido | [README](../../../README.md): sobre o desafio, ferramentas, workflow, prompts, iterações e navegação. |
| ACE-27 | README: Lista pelo menos 1 ferramenta de IA utilizada | Atendido | [README](../../../README.md#ferramentas-de-ia-utilizadas): Antigravity/Gemini, Spec Kit e Codex, com seus papéis. |
| ACE-28 | README: Mostra pelo menos 2 prompts customizados em blocos de código | Atendido | [README](../../../README.md#prompts-customizados): três prompts em blocos Markdown. |
| ACE-29 | README: Descreve pelo menos 2 iterações/ajustes concretos | Atendido | [README](../../../README.md#iterações-e-ajustes): quatro iterações iniciais, revisão do PR e auditoria de aceite, com problemas e correções. |
| ACE-30 | Consistência: Nenhum requisito, decisão ou restrição contradiz transcrição/código | Atendido | Revisão semântica: outbox condicionada a assinaturas, filtro na inserção, JWT do operador, ADMIN no replay, HMAC sobre o corpo, 24h, UUID/snapshot e exclusões alinhados. A interpretação do retry está explícita como derivação para revisão, sem atribuição de uma fala inexistente; não há garantias universais de latência/entrega. |
| ACE-31 | Consistência: Nenhum arquivo de código mencionado é inexistente | Atendido | Todos os caminhos concretos encontrados no README, docs e artefatos de suporte existem; componentes futuros são descritos logicamente. A auditoria do [quickstart](../quickstart.md) não usa exceções por prefixo. |

## Denominador da Cobertura do Tracker

O inventário foi ampliado antes da aferição de cobertura, incluindo decisões e consequências dos ADRs e seções de produto/escopo/dependências/testes sem ID na versão anterior. Cada unidade tem um identificador no documento indicado e uma linha no Tracker. Modelos/fluxos e seus exemplos/subpassos são unidades consolidadas, não contagens por palavra ou por campo de exemplo. Afirmações equivalentes repetidas na mesma unidade não inflacionam o denominador.

| Documento | Unidades identificáveis | Linhas correspondentes |
| --- | ---: | ---: |
| PRD | 63 | 63 |
| RFC | 22 | 22 |
| FDD | 83 | 83 |
| ADR-001 | 19 | 19 |
| ADR-002 | 18 | 18 |
| ADR-003 | 17 | 17 |
| ADR-004 | 17 | 17 |
| ADR-005 | 11 | 11 |
| ADR-006 | 16 | 16 |
| **Total** | **266** | **266 (100%)** |

As 224 fontes TRANSCRICAO representam 84,2%; as 42 fontes CODIGO representam 15,8%. O formato obrigatório mantém apenas essas duas categorias de fonte. O tipo e o texto distinguem decisões da reunião de propostas técnicas derivadas.

## Correções Feitas Nesta Auditoria

1. Ampliação do inventário e da cobertura para além dos IDs que já existiam; inclusão de contexto/decisões/consequências dos ADRs e problemas, personas, escopo, dependências, trade-offs e testes de produto.
2. Remoção das referências a arquivos futuros inexistentes; componentes futuros recebem nomes lógicos, sem criar código para satisfazer um critério documental. A auditoria de caminhos agora é estrita.
3. Requests vazios de rotação e replay exibidos em blocos JSON, comprovando quatro contratos com pares explícitos de payloads de entrada/saída e status HTTP.
4. Correção da afirmação absoluta de atomicidade no ADR-001 para incluir a condição de endpoint ativo interessado; retirada da afirmação residual de que polling sozinho cumpre a meta de latência no PRD. Melhoria do texto que separa fim do trimestre do pedido de prazo até novembro.
5. Sincronização do README, inventário, planejamento e auditorias com os números e critérios atuais.

## Método e Limites

Foram verificadas as seções obrigatórias, os exemplos JSON, as quantidades mínimas, os links locais, a correspondência entre unidades/documentos/Tracker, a existência de todas as referências de origem e a imutabilidade da aplicação/transcrição contra a base do desafio. A consistência semântica exige leitura humana/agêntica e não é comprovada por regex; a checklist registra essa leitura, com ressalvas de interpretação explicitadas nos documentos.

A feature continua documental: não foram implementados API/worker, migrações nem testes de carga. Aprovação futura das propostas pela equipe é parte do RFC, não uma aprovação inventada por esta auditoria. A extensão do RFC em páginas depende do renderizador; o texto permanece conciso e sem os detalhes de contrato do FDD.
