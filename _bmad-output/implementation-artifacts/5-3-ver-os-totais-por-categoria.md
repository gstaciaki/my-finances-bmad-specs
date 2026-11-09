# Story 5.3: Ver os totais por Categoria

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 5). Em caso de divergência, vale o `epics.md`.

## História

As a Membro,
I want ver quanto foi para cada Categoria no mês,
So that eu me organize no mês seguinte.


## Requisitos referenciados

- **FR-23**: Balanço com totais por Categoria, separando Entradas e Saídas, e grupo "Sem categoria"; soma igual aos totais.
- **NFR-8**: Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.

## Acceptance Criteria

**Given** um mês com Movimentações categorizadas e sem Categoria
**When** abro o Mês
**Then** vejo os totais de cada Categoria, separando Entradas e Saídas, e um único grupo "Sem categoria"

**Given** os grupos
**When** somo seus valores
**Then** o resultado é igual ao total de Entradas e ao de Saídas do mês, e um e2e confirma

**Given** uma Categoria excluída
**When** o Balanço é recalculado
**Then** seus valores passam para "Sem categoria"

**Given** Categorias sem Movimentação no mês
**When** o mês é exibido
**Then** elas não aparecem
