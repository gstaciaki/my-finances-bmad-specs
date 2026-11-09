# Story 1.6: Editar meu perfil e trocar minha senha

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a Usuário,
I want manter meus dados atualizados e trocar minha senha,
So that eu continue com acesso seguro.


## Requisitos referenciados

- **FR-3**: Usuário edita o próprio nome e e-mail e troca a senha informando a atual; e-mail único; não altera outros Usuários.
- **AR-8**: `zEmail` e `zPassword` únicos, corrigindo o regex de símbolos (AD-23, D-20) e as mensagens com entidade errada (D-21).
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.

## Acceptance Criteria

**Given** a tela `/perfil`
**When** altero nome ou e-mail para um valor livre, ou mantenho o meu próprio e-mail
**Then** os dados são salvos sem conflito

**Given** um e-mail já usado, inclusive por Usuário excluído
**When** salvo
**Then** recebo 409, com a mensagem no campo

**Given** a senha atual correta e uma nova que segue `zPassword`, com qualquer símbolo
**When** troco a senha
**Then** todas as minhas Sessões são revogadas e volto ao Login

**Given** a senha atual errada
**When** tento trocar
**Then** recebo erro no campo e as Sessões continuam ativas

**Given** um `id` de outro Usuário no corpo
**When** edito o perfil
**Then** só meus próprios dados mudam
