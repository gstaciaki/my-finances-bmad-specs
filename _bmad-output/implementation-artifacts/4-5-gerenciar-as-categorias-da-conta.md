# Story 4.5: Gerenciar as Categorias da Conta

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 4). Em caso de divergência, vale o `epics.md`.

## História

As a Membro com Gestão,
I want criar, renomear e excluir Categorias,
So that os gastos fiquem agrupados do jeito da casa.


## Requisitos referenciados

- **FR-19**: Membros com Gestão criam, renomeiam e excluem Categorias da Conta, com nome único; Lançamento só escolhe; excluir Categoria em uso deixa Movimentações sem Categoria.
- **AR-3**: `AccountScopedRepository`, exceções nomeadas por Usuário e FK composta de Categoria (AD-3).
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.
- **UX-DR22**: Categorias e Membros legíveis por qualquer Membro; Membros com nome.

## Acceptance Criteria

**Given** a migration desta história
**When** é aplicada
**Then** cria `Category` com `@@unique([accountId, name])` e `Transaction.categoryId` com FK composta `(accountId, categoryId)`

**Given** `/contas/:id/categorias` e a Permissão de Gestão
**When** crio ou renomeio com nome válido
**Then** a mudança é salva, com `trim` no nome

**Given** um nome já usado na Conta
**When** salvo
**Then** recebo 409, com a mensagem no campo

**Given** uma Categoria em uso
**When** a excluo e confirmo
**Then** as Movimentações ficam sem Categoria, linha a linha no mesmo Unit of Work, e continuam existindo

**Given** um Membro com Lançamento
**When** chama as rotas de escrita
**Then** recebe 403, mas consegue listar, e as ações ficam ocultas
