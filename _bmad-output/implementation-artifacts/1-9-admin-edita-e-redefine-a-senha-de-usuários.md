# Story 1.9: Admin edita e redefine a senha de Usuários

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a Admin,
I want corrigir dados e redefinir senhas,
So that eu ajude quem perdeu o acesso.


## Requisitos referenciados

- **FR-5**: Admin lista, edita, redefine a senha e exclui Usuários; excluído não faz login, vira ex-Membro e o e-mail fica reservado; Dono de Conta com outros Membros só é excluído após transferência; Dono único Membro leva a Conta junto.

## Acceptance Criteria

**Given** um Usuário existente
**When** o Admin edita nome ou e-mail
**Then** os dados mudam, com as mesmas regras de unicidade do FR-3

**Given** uma nova senha que segue `zPassword`
**When** o Admin a redefine
**Then** todas as Sessões daquele Usuário são revogadas

**Given** um Usuário não Admin
**When** chama essas rotas
**Then** recebe 403 e nada muda
