# Story 3.1: Convidar um Usuário para a Conta

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 3). Em caso de divergência, vale o `epics.md`.

## História

As a Dono,
I want convidar alguém por e-mail e definir sua Permissão,
So that eu divida a Conta com quem mora comigo.


## Requisitos referenciados

- **FR-9**: Dono convida Usuário existente por e-mail com Permissão; sem convidar Membro nem duplicar Convite pendente; expira em 7 dias.
- **UX-DR5**: `BaseSelect` sobre `<select>` nativo, com busca por letra.
- **UX-DR15**: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.
- **UX-DR22**: Categorias e Membros legíveis por qualquer Membro; Membros com nome.

## Acceptance Criteria

**Given** um Membro em `/contas/:id/membros`
**When** a tela carrega
**Then** vê os Membros com nome e Permissão, e só o Dono vê as ações de convite

**Given** o Dono no diálogo de Convite, com `BaseSelect` de Permissão
**When** informa o e-mail de um Usuário existente
**Then** recebe 201 com `{ emailDelivered }`, e o Convite fica pendente por 7 dias

**Given** um e-mail sem login, de Usuário excluído, de quem já é Membro ou com Convite pendente válido
**When** o Dono envia
**Then** recebe um erro claro e nenhum Convite é criado

**Given** um Convite pendente vencido para o mesmo Usuário
**When** o Dono envia outro
**Then** o antigo vira `EXPIRED` e o novo é criado no mesmo Unit of Work

**Given** uma falha no envio do e-mail
**When** o Convite é criado
**Then** ele permanece salvo e a tela mostra "Convite criado, mas o e-mail não foi enviado." com Reenviar
**And** reenviar não altera `expiresAt`

**Given** um Membro que não é o Dono
**When** chama a API de Convite
**Then** recebe 403

**Given** uma Conta com Convites pendentes
**When** o Dono a exclui
**Then** os Convites viram `CANCELLED`
