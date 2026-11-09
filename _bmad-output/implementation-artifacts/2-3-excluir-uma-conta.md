# Story 2.3: Excluir uma Conta

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 2). Em caso de divergência, vale o `epics.md`.

## História

As a Dono,
I want excluir uma Conta que não uso mais,
So that ninguém continue acessando.


## Requisitos referenciados

- **FR-14**: Membros com Gestão editam nome e Saldo inicial, o que recalcula o Saldo acumulado; só o Dono exclui, com confirmação explícita e exclusão lógica; sem restauração na v1.
- **NFR-1**: Isolamento: Usuário nunca lê nem altera dados de Conta da qual não é Membro, exceto leitura do Admin, em toda a API.
- **UX-DR10**: `AppDialog` e `ConfirmDialog` em `<dialog>`, com `Esc`, gestão de foco e foco inicial em Cancelar.

## Acceptance Criteria

**Given** o Dono em Ajustes
**When** clica em Excluir
**Then** o `ConfirmDialog` mostra "Excluir esta Conta? Ninguém mais terá acesso.", com foco inicial em Cancelar

**Given** a confirmação
**When** a exclusão é concluída
**Then** a API responde 204, grava `deletedAt` sob o lock da Conta e os dados permanecem no banco

**Given** uma Conta excluída
**When** um Membro a lista, abre ou altera
**Then** ela não aparece na lista e as rotas respondem 404

**Given** um Membro que não é o Dono, mesmo com Gestão
**When** chama `DELETE /api/account/:id`
**Then** recebe 403, e o botão não aparece
