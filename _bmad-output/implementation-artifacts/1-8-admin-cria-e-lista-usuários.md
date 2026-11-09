# Story 1.8: Admin cria e lista Usuários

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a Admin,
I want criar logins e ver quem usa o sistema,
So that eu dê acesso às pessoas da casa.


## Requisitos referenciados

- **FR-4**: Admin cria Usuário com nome, e-mail e senha inicial; e-mail único; sem CPF.
- **FR-5**: Admin lista, edita, redefine a senha e exclui Usuários; excluído não faz login, vira ex-Membro e o e-mail fica reservado; Dono de Conta com outros Membros só é excluído após transferência; Dono único Membro leva a Conta junto.
- **AR-4**: Contrato `useCase.run(input, actor)`, políticas em `src/policies/`, `requireAdmin`, entrada de controllers por campos explícitos (AD-2).
- **AR-13**: Filtros explícitos no lugar de `autoParseFilters`; header `Location` em 201 (issue #24); `DELETE` responde 204.
- **UX-DR6**: `BaseTable` com hover, valores à direita, linha clicável e paginação.
- **UX-DR10**: `AppDialog` e `ConfirmDialog` em `<dialog>`, com `Esc`, gestão de foco e foco inicial em Cancelar.
- **UX-DR15**: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.

## Acceptance Criteria

**Given** um Admin em `/admin/usuarios`
**When** crio um Usuário com nome, e-mail e senha inicial
**Then** recebo 201 com `Location`, o Usuário consegue entrar, e nenhum campo de CPF é pedido

**Given** um e-mail já usado, inclusive por Usuário excluído
**When** crio o Usuário
**Then** recebo 409, com a mensagem no campo

**Given** a lista de Usuários
**When** navego pelas páginas
**Then** vejo a `BaseTable` paginada, sem senha ou hash na resposta

**Given** um não Admin
**When** chamo `/api/admin/*` ou abro a rota
**Then** a API retorna 403 e a tela não fica acessível
**And** o item Admin do menu aparece só para Admin

**Given** uma lista carregando
**When** a tela aguarda a API
**Then** vejo linhas de esqueleto, padrão reutilizado nas demais listas
