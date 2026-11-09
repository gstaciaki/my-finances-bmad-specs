# Story 3.2: Aceitar ou recusar um Convite

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 3). Em caso de divergência, vale o `epics.md`.

## História

As a convidado,
I want ver meus Convites e responder a cada um,
So that eu entre nas Contas certas.


## Requisitos referenciados

- **FR-10**: Convidado vê Convites pendentes ao entrar e aceita ou recusa; recusado ou expirado não pode ser aceito, e o Dono pode enviar outro.
- **UX-DR13**: `AppHeader` com Conta atual, menu do usuário, Admin condicional e indicador de Convites.
- **UX-DR23**: `GET /invitation` com nome da Conta e Permissão.

## Acceptance Criteria

**Given** Convites pendentes
**When** entro em Minhas Contas
**Then** vejo o nome da Conta e a Permissão de cada um, com o indicador no `AppHeader`

**Given** um Convite pendente e válido
**When** aceito
**Then** viro Membro com a Permissão definida, sob o lock da Conta, e a Conta entra na lista na hora

**Given** um Convite pendente
**When** recuso
**Then** ele não pode mais ser aceito, e o Dono pode enviar outro

**Given** um Convite vencido
**When** tento aceitar
**Then** vejo "Este Convite expirou. Peça um novo ao Dono."

**Given** um Convite de outro Usuário
**When** chamo aceitar ou recusar
**Then** recebo 404

**Given** um Convite de Conta excluída, ou um convidado que já virou Membro
**When** tento aceitar
**Then** recebo erro claro e nenhum vínculo é criado
