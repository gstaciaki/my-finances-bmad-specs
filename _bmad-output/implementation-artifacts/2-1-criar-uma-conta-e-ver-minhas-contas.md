# Story 2.1: Criar uma Conta e ver minhas Contas

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 2). Em caso de divergência, vale o `epics.md`.

## História

As a Usuário,
I want criar uma Conta e ver as Contas de que participo,
So that eu comece a organizar o dinheiro da casa.


## Requisitos referenciados

- **FR-7**: Usuário cria Conta com nome e Saldo inicial opcional e se torna Dono; Conta aparece na lista.
- **FR-8**: Usuário vê só as Contas de que é Membro e não lê nem altera outras.
- **NFR-1**: Isolamento: Usuário nunca lê nem altera dados de Conta da qual não é Membro, exceto leitura do Admin, em toda a API.
- **NFR-8**: Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.
- **AR-3**: `AccountScopedRepository`, exceções nomeadas por Usuário e FK composta de Categoria (AD-3).
- **AR-14**: Criação de Conta e vínculo do Dono em uma transação (D-14) e índice `(accountId, occurredAt)` (D-16).
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.
- **UX-DR21**: `GET /account` e `GET /account/:id` com `isOwner` e `permission` do Usuário atual.

## Acceptance Criteria

**Given** um Usuário em `/` sem Contas
**When** a tela carrega
**Then** vê "Você ainda não participa de nenhuma Conta." e o botão Criar Conta

**Given** o diálogo de criar Conta
**When** informo o nome e, se quiser, um Saldo inicial, que pode ser negativo
**Then** recebo 201 com `Location`, a Conta aparece na lista na hora e eu sou o Dono
**And** `Account.ownerId` e o vínculo com `MANAGEMENT` são gravados no mesmo Unit of Work

**Given** a lista de Contas
**When** chamo `GET /api/account`
**Then** recebo só Contas sem `deletedAt` das quais sou Membro, com `isOwner` e `permission`

**Given** a migration desta história
**When** é aplicada
**Then** cria `Account` e `UserAccount`, e `AccountScopedRepository` passa a exigir `accountId` em toda operação

**Given** uma Conta de outro Usuário
**When** chamo `GET /api/account/:id`, inclusive como Admin que não é Membro
**Then** recebo 404
**And** um teste e2e cobre esse isolamento
