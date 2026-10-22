# Guia de Auditoria do Pacote de Design Docs

A referência de aceite é o [enunciado preservado](./enunciado-original.md), idêntico ao README do commit base do desafio. O README atual descreve o processo. Os comandos abaixo verificam estrutura, rastreabilidade formal e imutabilidade; a revisão semântica da seção final complementa a automação.

## 1. Base da Comparação

Execute na raiz do repositório. A base deste PR é o commit abaixo; altere REVIEW_BASE apenas se estiver auditando outra base conhecida.

```bash
export REVIEW_BASE=e7f6311b96ee3b21bf8cb7077f53a6e4045143eb
```

## 2. Auditoria Executável

Este comando compara arquivos protegidos com a base, abrangendo alterações commitadas, staged e locais. Também detecta arquivos novos nos diretórios protegidos. Não usa git status como prova de imutabilidade e não depende de testes da aplicação.

```bash
python3 - <<'PY'
from pathlib import Path
from collections import Counter
import os
import re
import subprocess

base = os.environ.get('REVIEW_BASE', 'e7f6311b96ee3b21bf8cb7077f53a6e4045143eb')
def git(*args):
    return subprocess.check_output(['git', *args], text=True)
def require(condition, message):
    if not condition:
        raise SystemExit('FALHA: ' + message)

protected = ['src/', 'prisma/', 'tests/', 'TRANSCRICAO.md', 'package.json',
             'package-lock.json', 'tsconfig.json', 'tsconfig.build.json',
             'docker-compose.yml', '.env.example', '.eslintrc.json',
             '.prettierrc', '.prettierignore', '.gitignore', 'vitest.config.ts']
require(not git('diff', '--name-only', base, '--', *protected).strip(),
        'arquivo protegido difere da base (inclui commits e alterações locais)')
require(not git('ls-files', '--others', '--', *protected).strip(),
        'arquivo novo em caminho protegido')
require(git('show', base + ':README.md') ==
        Path('specs/001-webhook-design-docs/enunciado-original.md').read_text(),
        'enunciado preservado diverge do README da base')

adrs = sorted(Path('docs/adrs').glob('ADR-[0-9][0-9][0-9]-*.md'))
require(5 <= len(adrs) <= 8, 'quantidade de ADRs')
for p in adrs:
    for section in ['Status', 'Contexto', 'Decisão', 'Alternativas Consideradas', 'Consequências']:
        require('## ' + section in p.read_text(), str(p) + ': ' + section)
for name in ['PRD', 'RFC', 'FDD', 'TRACKER']:
    require(Path('docs', name + '.md').is_file(), 'documento ausente: ' + name)
require(len(re.findall(r'^## \d+\.', Path('docs/PRD.md').read_text(), re.M)) == 12,
        'seções do PRD')
require(len(re.findall(r'^### 5\.[1-7] ', Path('docs/FDD.md').read_text(), re.M)) >= 4,
        'quantidade de endpoints HTTP no FDD')

rows = []
for line in Path('docs/TRACKER.md').read_text().splitlines():
    if re.match(r'^\| (PRD|RFC|FDD|ADR)-', line):
        cells = [c.strip() for c in line.strip('|').split('|')]
        require(len(cells) == 6, 'linha do Tracker deve ter seis colunas')
        rows.append(cells)
require(bool(rows), 'Tracker vazio')
ids = [r[0] for r in rows]
require(len(ids) == len(set(ids)), 'IDs duplicados')
source_counts = Counter(r[4] for r in rows)
require(set(source_counts) <= {'TRANSCRICAO', 'CODIGO'}, 'fonte inválida')
require(source_counts['TRANSCRICAO'] / len(rows) >= .70, 'menos de 70% TRANSCRICAO')
require(source_counts['CODIGO'] >= 5, 'menos de cinco linhas CODIGO')
transcript = Path('TRANSCRICAO.md').read_text()
for id, doc, kind, summary, source, location in rows:
    require(Path(doc).is_file(), id + ': documento ausente')
    if source == 'TRANSCRICAO':
        refs = re.findall(r'\[\d{2}:\d{2}\] [A-Za-zÀ-ÿ]+', location)
        require(bool(refs), id + ': marcador ausente')
        for ref in refs:
            require(ref + ':' in transcript, id + ': origem ausente: ' + ref)
    else:
        require(Path(location).is_file(), id + ': código inexistente: ' + location)

pattern = r'\b(?:PRD|RFC|FDD)-[A-Z]+-\d{2}\b|\bADR-\d{3}(?:-(?:ALT|TRD)-\d{2})?\b'
docs = [Path('docs/PRD.md'), Path('docs/RFC.md'), Path('docs/FDD.md'), *adrs]
inventory = set()
for p in docs:
    inventory.update(re.findall(pattern, p.read_text()))
require(not inventory - set(ids), 'IDs sem origem no Tracker: ' + str(sorted(inventory - set(ids))))
require(not set(ids) - inventory, 'IDs do Tracker ausentes dos documentos')
for p in docs + [Path('docs/TRACKER.md'), Path('README.md')]:
    for target in re.findall(r'\[[^\]]*\]\(([^)]+)\)', p.read_text()):
        if not re.match(r'\w+://', target):
            require((p.parent / target.split('#')[0]).exists(), str(p) + ': link ausente: ' + target)
    for path in re.findall(r'(?:src|prisma)/[A-Za-z0-9_./-]+\.(?:ts|prisma)', p.read_text()):
        proposed = path == 'src/worker.ts' or path.startswith('src/modules/webhooks/')
        require(proposed or Path(path).is_file(), str(p) + ': código ausente: ' + path)
print('PASSOU: imutabilidade, enunciado, estrutura, links e referências formais.')
print(f'Cobertura de IDs explícitos: {len(inventory)}/{len(inventory)}; '
      f'TRANSCRICAO: {source_counts["TRANSCRICAO"]}/{len(rows)}; CODIGO: {source_counts["CODIGO"]}.')
print('A revisão semântica abaixo continua obrigatória; existência de origem não prova conteúdo.')
PY
```

Arquivos propostos (`src/worker.ts` e `src/modules/webhooks/*`) não são apresentados como código existente nem usados como Fonte CODIGO. A validação acima distingue essas propostas da base real.

## 3. Revisão Semântica e Simulação dos Fluxos

Confronte cada requisito/decisão com sua fala ou implementação de origem. Para detalhes novos, verifique o rótulo de proposta derivada, a justificativa em FDD-DET-01 a FDD-DET-08 e o encaminhamento em RFC-REVIEW-01. Conferir também as seções obrigatórias, exemplos request/response, trade-offs e exclusões do enunciado; a contagem de títulos não verifica sua qualidade.

Na revisão operacional, simule:

1. Falha inicial e cinco retries: seis chamadas, esperas de 1m/5m/30m/2h/12h, DLQ somente após o último resultado falho; espera acumulada de 14h36m, além do tempo de execução.
2. Crash após reserva e após resposta sem persistência: bootstrap exclusivo recupera PROCESSING, preservando UUID; possível duplicação é esperada.
3. Eventos do mesmo destino/pedido com timestamps iguais e predecessor em retry/DLQ: sequência desempata e sucessores permanecem bloqueados; outros pedidos avançam.
4. Replay simultâneo da mesma DLQ: somente um novo ciclo; se houver novo esgotamento, outra linha (outboxId, cycle) preserva o histórico e permite novo replay.
5. Rotação e tentativa de nova rotação antes de 24h: mantém a chave antiga até expirar e retorna 409; receptor verifica qualquer digest válido do único formato X-Signature.
6. Código existente: OrderStatus vem de @prisma/client; ValidationError/NotFoundError fixam códigos; adaptador local faz os mapeamentos previstos.
7. Métricas: duração HTTP não substitui espera em fila + latência de notificação; primeira entrega, retry e replay são medidos separadamente.

A simulação do desenho não comprova comportamento de uma feature ainda não implementada. O [relatório de revisão](./review.md) registra o que foi efetivamente verificado nesta correção.
