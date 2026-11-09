# Story 6.1: Admin vê todas as Contas

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 6). Em caso de divergência, vale o `epics.md`.

## História

As a Admin,
I want ver todas as Contas, seus Membros e Movimentações,
So that eu tenha a visão total do sistema.


## Requisitos referenciados

- **FR-6**: Admin vê todas as Contas, Membros e Movimentações, inclusive Contas excluídas identificadas; somente leitura, exceto transferir titularidade.
- **NFR-1**: Isolamento: Usuário nunca lê nem altera dados de Conta da qual não é Membro, exceto leitura do Admin, em toda a API.
- **NFR-8**: Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.
- **UX-DR15**: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.

## Acceptance Criteria

**Given** um Admin em `/admin/contas`
**When** a tela carrega
**Then** vê todas as Contas, paginadas, e as excluídas têm a etiqueta "Excluída"

**Given** uma Conta, excluída ou não
**When** o Admin a abre
**Then** vê os Membros e as Movimentações, somente em leitura e pelas rotas `/api/admin/*`

**Given** um Admin que não é Membro da Conta
**When** chama as rotas normais de escrita ou leitura dela
**Then** recebe 404

**Given** um não Admin
**When** chama `/api/admin/account*` ou abre a rota
**Then** a API retorna 403 e a tela não fica acessível
