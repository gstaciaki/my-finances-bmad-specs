# Story 1.2: Proteger e auditar toda escrita no banco

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a responsável pelo sistema,
I want que toda escrita seja transacional e registrada,
So that eu saiba quem alterou o quê e nada seja gravado pela metade.


## Requisitos referenciados

- **FR-18**: Sistema guarda autoria de criação, alteração e exclusão de Movimentações, além de toda escrita no banco e toda requisição HTTP, sem corpo nem segredos; não aparece nas telas da v1.
- **NFR-4**: Erros previsíveis: regra violada retorna erro claro, nunca 500.
- **AR-2**: `src/database/`: tipo `DbClient`, extensão Prisma com bloqueio de operações e auditoria, `AsyncLocalStorage`, `uow.run(actor, fn, { lockAccounts })` com rollback em `Wrong`, `FakeUnitOfWork` (AD-10, AD-11, AD-12, AD-22).
- **AR-5**: Tradução única de erros Prisma e 403/429 no mapa de erros (AD-17).
- **AR-6**: `RequestLog` por middleware, sem corpo, com rota normalizada (AD-13).

## Acceptance Criteria

**Given** um use case que escreve em `uow.run(actor, fn)`
**When** o callback retorna `Wrong` ou lança exceção
**Then** nada é confirmado, inclusive a auditoria

**Given** uma escrita bem-sucedida
**When** consulto `AuditLog`
**Then** há instante, autor, modelo, id, operação e estados anterior e posterior, com `password` e `*Hash` mascarados

**Given** o cliente da aplicação
**When** o código usa uma operação proibida do AD-11 ou escreve sem actor
**Then** recebe `ForbiddenOperationError`, e o valor `PrismaClient` fora de `src/database/` e dos testes falha no lint

**Given** qualquer requisição concluída
**When** consulto `RequestLog`
**Then** há método, rota normalizada, status, duração e autor, sem corpo, query string ou headers
**And** uma falha ao gravar o log não altera a resposta

**Given** um erro Prisma ou de regra
**When** a API responde
**Then** o status segue o AD-17, com corpo `{ code, error, slug }`, e regra violada nunca retorna 500
