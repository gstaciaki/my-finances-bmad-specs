# Story 3.5: Transferir a titularidade

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 3). Em caso de divergência, vale o `epics.md`.

## História

As a Dono,
I want passar a Conta para outro Membro,
So that ela continue tendo um responsável.


## Requisitos referenciados

- **FR-13**: Dono ou Admin transfere a titularidade para outro Membro; sempre um Dono; o antigo Dono escolhe a própria Permissão e continua Membro.
- **NFR-8**: Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.

## Acceptance Criteria

**Given** o Dono e outro Membro da Conta
**When** transfere com `{ newOwnerId, previousOwnerPermission }` e confirma
**Then** `ownerId` e as duas Permissões mudam em um único Unit of Work, sob o lock da Conta
**And** o novo Dono fica com Gestão, e o antigo fica com a Permissão escolhida

**Given** um Admin, mesmo sem ser Membro
**When** chama `POST /api/admin/account/:accountId/transfer`
**Then** a transferência funciona pelas mesmas regras, e `previousOwnerPermission` é definida pelo Admin

**Given** um alvo que não é Membro, ou uma Conta excluída
**When** a transferência é tentada
**Then** recebe erro claro e nada muda

**Given** um Membro que não é o Dono
**When** chama a transferência
**Then** recebe 403

**Given** duas transferências simultâneas
**When** ambas executam
**Then** a Conta termina com exatamente um Dono, que tem vínculo com Gestão
