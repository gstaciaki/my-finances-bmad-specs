# Story 4.6: Categorizar Movimentações

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 4). Em caso de divergência, vale o `epics.md`.

## História

As a Membro,
I want escolher a Categoria ao lançar ou editar,
So that eu saiba para onde foi o dinheiro.


## Requisitos referenciados

- **FR-20**: Qualquer Membro atribui ou remove a Categoria de uma Movimentação.
- **FR-17**: Membro lista Movimentações ordenadas por data, com filtro por período e Categoria, nunca de outra Conta.
- **UX-DR15**: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.
- **UX-DR24**: Listas de Movimentação com `categoryId` e `categoryName`.

## Acceptance Criteria

**Given** o `EntryForm`
**When** chego à Categoria depois do Tipo
**Then** vejo o `BaseSelect` com "Sem categoria" primeiro e busca por letra, e a Categoria é limpa depois de salvar

**Given** uma Conta sem Categorias
**When** abro Lançar
**Then** só existe "Sem categoria", e quem tem Gestão vê link para criar

**Given** uma Movimentação existente
**When** envio `categoryId` com outro valor ou `null`
**Then** a Categoria é trocada ou removida, e campo ausente não muda nada

**Given** uma Categoria de outra Conta
**When** a atribuo
**Then** recebo 409 e nada muda

**Given** a lista de Movimentações
**When** chamo a API, com ou sem filtro `categoryId`
**Then** cada item traz `categoryId` e `categoryName`, e o filtro retorna só aquela Categoria
