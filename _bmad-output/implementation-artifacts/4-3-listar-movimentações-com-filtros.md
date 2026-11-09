# Story 4.3: Listar Movimentações com filtros

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 4). Em caso de divergência, vale o `epics.md`.

## História

As a Membro,
I want consultar as Movimentações da Conta por período,
So that eu ache o que foi lançado.


## Requisitos referenciados

- **FR-17**: Membro lista Movimentações ordenadas por data, com filtro por período e Categoria, nunca de outra Conta.
- **NFR-1**: Isolamento: Usuário nunca lê nem altera dados de Conta da qual não é Membro, exceto leitura do Admin, em toda a API.
- **NFR-8**: Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.
- **AR-13**: Filtros explícitos no lugar de `autoParseFilters`; header `Location` em 201 (issue #24); `DELETE` responde 204.

## Acceptance Criteria

**Given** uma Conta com Movimentações
**When** chamo `GET /api/account/:accountId/transaction`
**Then** recebo a lista paginada, ordenada por `occurredAt desc, createdAt desc, id desc`

**Given** `from` e `to` em `YYYY-MM-DD`
**When** filtro
**Then** ambos são inclusivos no calendário de `America/Sao_Paulo`

**Given** Movimentações em outra Conta
**When** listo ou uso um id de outra Conta
**Then** elas nunca aparecem, e um e2e cobre esse isolamento

**Given** filtros
**When** a query é lida
**Then** eles vêm do schema Zod do use case, sem `autoParseFilters`
