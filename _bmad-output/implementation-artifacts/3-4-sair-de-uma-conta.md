# Story 3.4: Sair de uma Conta

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 3). Em caso de divergência, vale o `epics.md`.

## História

As a Membro,
I want sair de uma Conta,
So that eu deixe de participar quando fizer sentido.


## Requisitos referenciados

- **FR-12**: Membro sai da Conta; Dono só após transferir.

## Acceptance Criteria

**Given** um Membro que não é o Dono
**When** sai e confirma no `ConfirmDialog`
**Then** o vínculo é apagado sob o lock da Conta, perco o acesso imediatamente e volto para Minhas Contas

**Given** o Dono
**When** tenta sair
**Then** recebe erro explicando que precisa transferir a titularidade antes
