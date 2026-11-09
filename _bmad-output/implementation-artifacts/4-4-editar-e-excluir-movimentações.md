# Story 4.4: Editar e excluir Movimentações

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 4). Em caso de divergência, vale o `epics.md`.

## História

As a Membro,
I want corrigir ou apagar qualquer lançamento da Conta,
So that os números fiquem certos.


## Requisitos referenciados

- **FR-16**: Qualquer Membro edita ou exclui qualquer Movimentação da Conta.
- **FR-18**: Sistema guarda autoria de criação, alteração e exclusão de Movimentações, além de toda escrita no banco e toda requisição HTTP, sem corpo nem segredos; não aparece nas telas da v1.

## Acceptance Criteria

**Given** um item de "Lançadas agora", lançado por qualquer Membro
**When** edito no `AppDialog`
**Then** a alteração é salva pelas mesmas validações, a tela reflete o novo valor e o autor fica no `AuditLog`

**Given** o `ConfirmDialog` que nomeia a Movimentação
**When** confirmo a exclusão
**Then** a API responde 204, e a remoção física fica registrada no `AuditLog` com o autor

**Given** um id de Movimentação de outra Conta na rota da minha Conta
**When** edito ou excluo
**Then** recebo 404 e nada muda

**Given** um Membro com Lançamento
**When** edita ou exclui Movimentação lançada por outro
**Then** a operação é aceita
