# Quickstart & Validation Guide: Auditoria do Pacote de Design Docs

**Feature**: `001-webhook-design-docs`
**Date**: 2026-10-21

Este guia descreve os comandos de verificação executáveis para validar de ponta a ponta que a entrega documental cumpre 100% dos critérios de aceite do `README.md` e da `constitution.md` sem violar a imutabilidade do código-fonte.

---

## 1. Pré-requisitos

- Repositório clonado localmente com Node.js $\ge 20$ e utilitários POSIX padrão (`git`, `ls`, `wc`, `grep`, `awk`).
- Contratos de referência consultáveis em:
  - [deliverables-contract.md](./contracts/deliverables-contract.md)
  - [webhook-http-contracts.md](./contracts/webhook-http-contracts.md)
  - [data-model.md](./data-model.md)

---

## 2. Cenários Executáveis de Validação End-to-End

### Cenário 1: Verificar Imutabilidade do Código-Fonte e da Transcrição (Princípio II)
Nenhum arquivo em `src/`, `prisma/`, `tests/`, configurações ou `TRANSCRICAO.md` pode ter sido modificado, adicionado ou removido.

```bash
git status --porcelain src/ prisma/ tests/ TRANSCRICAO.md package.json tsconfig.json
```
- **Resultado Esperado**: Saída vazia (`0` linhas).

### Cenário 2: Verificar Existência e Contagem dos Entregáveis (`docs/` e `README.md`)
Validar que os arquivos principais e entre 5 e 8 ADRs existem.

```bash
ls -la docs/PRD.md docs/RFC.md docs/FDD.md docs/TRACKER.md README.md
ls -1 docs/adrs/ADR-[0-9][0-9][0-9]-*.md | wc -l
```
- **Resultado Esperado**: Todos os arquivos listados existem e a contagem de ADRs está entre `5` e `8` (planejado: `6`).

### Cenário 3: Validar Prefixos de Erro `WEBHOOK_` e Caminhos de Arquivo Reais no `docs/FDD.md`
Confirmar que o `docs/FDD.md` utiliza códigos `WEBHOOK_*` e que todos os caminhos `src/...` e `prisma/...` mencionados realmente existem no repositório.

```bash
grep -E "WEBHOOK_[A-Z_]+" docs/FDD.md
grep -oE "(src|prisma)/[a-zA-Z0-9_./-]+(\.ts|\.prisma)" docs/FDD.md docs/RFC.md docs/PRD.md docs/adrs/*.md | cut -d: -f2 | sort -u | grep -vE "^src/(worker\.ts|modules/webhooks/)" | while read -r path; do
  if [ ! -f "$path" ]; then
    echo "MISSING FILE: $path"
  else
    echo "OK: $path"
  fi
done
```
- **Resultado Esperado**: Pelo menos 4 arquivos reais listados como `OK:` (planejados 12: `prisma/schema.prisma`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`) e `0` ocorrências de `MISSING FILE` (os arquivos novos propostos na reunião, `src/worker.ts` e `src/modules/webhooks/*`, são filtrados separadamente).

### Cenário 4: Validar Métricas Quantitativas do `docs/TRACKER.md`
Verificar o total de linhas da tabela, a proporção de linhas com `Fonte = TRANSCRICAO` ($\ge 70\%$) e a quantidade de linhas com `Fonte = CODIGO` ($\ge 5$).

```bash
TOTAL=$(grep -E "^\| (PRD|RFC|FDD|ADR)-" docs/TRACKER.md | wc -l | tr -d ' ')
TRANSCRICAO=$(grep -E "^\| (PRD|RFC|FDD|ADR)-.*\| *TRANSCRICAO *\|" docs/TRACKER.md | wc -l | tr -d ' ')
CODIGO=$(grep -E "^\| (PRD|RFC|FDD|ADR)-.*\| *CODIGO *\|" docs/TRACKER.md | wc -l | tr -d ' ')
echo "Total: $TOTAL | TRANSCRICAO: $TRANSCRICAO | CODIGO: $CODIGO"
```
- **Resultado Esperado**:
  - `TOTAL` cobre $\ge 80\%$ dos itens identificáveis nos documentos;
  - `TRANSCRICAO / TOTAL` $\ge 0.70$ (70%), todas com timestamp no formato `[hh:mm] Nome`;
  - `CODIGO` $\ge 5$ linhas com caminhos reais do repositório.
