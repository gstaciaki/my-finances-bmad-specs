# Story 2.2: Abrir a Conta e editar nome e Saldo inicial

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 2). Em caso de divergência, vale o `epics.md`.

## História

As a Membro com Gestão,
I want abrir a Conta e corrigir seu nome e Saldo inicial,
So that os dados fiquem certos.


## Requisitos referenciados

- **FR-14**: Membros com Gestão editam nome e Saldo inicial, o que recalcula o Saldo acumulado; só o Dono exclui, com confirmação explícita e exclusão lógica; sem restauração na v1.
- **NFR-1**: Isolamento: Usuário nunca lê nem altera dados de Conta da qual não é Membro, exceto leitura do Admin, em toda a API.
- **UX-DR15**: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.
- **UX-DR21**: `GET /account` e `GET /account/:id` com `isOwner` e `permission` do Usuário atual.

## Acceptance Criteria

**Given** uma Conta de que sou Membro
**When** abro `/contas/:id`
**Then** vejo o nome e o Saldo inicial, com "Nenhuma Movimentação ainda."
**And** `GET /api/account/:id` retorna `isOwner` e `permission`

**Given** a tela `/contas/:id/ajustes` e a Permissão de Gestão
**When** altero nome ou Saldo inicial
**Then** a alteração é salva e confirmada por toast

**Given** um Membro com Lançamento
**When** chama `PATCH /api/account/:id`
**Then** recebe 403, a Permissão é recarregada na tela e a ação fica oculta

**Given** um não Membro ou um id inexistente
**When** abro a rota ou chamo a API
**Then** recebo 404 e vejo "Conta não encontrada." com link para Minhas Contas

**Given** um `accountId` no corpo diferente do caminho
**When** edito a Conta
**Then** vale só o do caminho
