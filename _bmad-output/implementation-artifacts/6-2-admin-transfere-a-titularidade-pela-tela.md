# Story 6.2: Admin transfere a titularidade pela tela

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 6). Em caso de divergência, vale o `epics.md`.

## História

As a Admin,
I want transferir uma Conta para outro Membro,
So that eu resolva uma Conta cujo Dono vai sair do sistema.


## Requisitos referenciados

- **FR-6**: Admin vê todas as Contas, Membros e Movimentações, inclusive Contas excluídas identificadas; somente leitura, exceto transferir titularidade.
- **FR-13**: Dono ou Admin transfere a titularidade para outro Membro; sempre um Dono; o antigo Dono escolhe a própria Permissão e continua Membro.

## Acceptance Criteria

**Given** uma Conta sem `deletedAt`, em Admin: Contas
**When** o Admin escolhe o novo Dono e a Permissão do antigo e confirma no `ConfirmDialog`
**Then** a tela usa a rota da Story 3.5 e mostra o novo Dono

**Given** uma Conta excluída
**When** o Admin a abre
**Then** a ação de transferir não aparece

**Given** a titularidade já transferida
**When** o Admin exclui o antigo Dono
**Then** a exclusão da Story 3.6 é aceita
