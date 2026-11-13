# Story 1.2: Proteger e auditar toda escrita no banco

Status: in-progress

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a responsável pelo sistema,
I want que toda escrita seja transacional e registrada,
So that eu saiba quem alterou o quê e nada seja gravado pela metade.


## Requisitos referenciados

- **FR-18**: Sistema guarda autoria de criação, alteração e exclusão de Movimentações, além de toda escrita no banco e toda requisição HTTP, sem corpo nem segredos; não aparece nas telas da v1.
- **NFR-4**: Erros previsíveis: regra violada retorna erro claro, nunca 500.
- **AR-1**: Histórico de migrations zerado; o schema recomeça do zero, sem `cpf`, e cada história adiciona a própria migration (AD-23, AD-25).
- **AR-2**: `src/database/`: tipo `DbClient`, cliente puro (só para seeds, populates e scripts), extensão Prisma com bloqueio de operações e auditoria, `AsyncLocalStorage`, `uow.run(actor, fn, { lockAccounts })` com rollback em `Wrong`, `FakeUnitOfWork` (AD-10, AD-11, AD-12, AD-22).
- **AR-5**: Tradução única de erros Prisma e 403/429 no mapa de erros (AD-17).
- **AR-6**: `RequestLog` por middleware, sem corpo, com o padrão da rota (AD-13).

## Acceptance Criteria

**Given** um use case que escreve em `uow.run(actor, fn)`
**When** o callback retorna `Wrong` ou lança exceção
**Then** nada é confirmado, inclusive a auditoria

**Given** uma escrita bem-sucedida
**When** consulto `AuditLog`
**Then** há instante, autor, modelo, id, operação e estados anterior e posterior, com `password` e `*Hash` mascarados

**Given** o cliente da aplicação
**When** o código usa uma operação proibida do AD-11 ou escreve sem actor
**Then** recebe `ForbiddenOperationError`
**And** o valor `PrismaClient`, importado de `@src/generated/prisma/client` ou de `@prisma/client`, fora de `src/database/`, de `test/` e do `jest.setup.ts` falha no lint, e os tipos continuam liberados
**And** seeds, populates e scripts usam o cliente puro, exportado à parte por `src/database/`

**Given** qualquer requisição concluída
**When** consulto `RequestLog`
**Then** há método, padrão da rota (por exemplo `/api/account/:accountId/transaction/:id`, ou `UNMATCHED` quando nenhuma rota casou), status, duração e autor, sem corpo, query string ou headers
**And** o autor é o id do usuário do token, ou vazio se a requisição não foi autenticada
**And** uma falha ao gravar o log não altera a resposta

**Given** um erro Prisma ou de regra
**When** a API responde
**Then** o status segue o AD-17, com corpo `{ code, error, slug }`, e regra violada nunca retorna 500
**And** os erros novos de 409 e de 503 existem em `src/errors/` e estão no mapa de erros

**Given** o banco com a migration `init`
**When** aplico a migration desta história
**Then** `AuditLog` e `RequestLog` existem, sem chave estrangeira em `actorId`, e nenhum dos dois é auditado

**Given** os use cases que hoje escrevem: `create-user`, `update-user`, `delete-user` e `change-user-password`
**When** executam
**Then** escrevem só em `uow.run`, com o actor do usuário autenticado, e os testes usam o `FakeUnitOfWork`

## Decisões de escopo

- **Actor:** a 1.2 cria o tipo `Actor` e a constante `SYSTEM`. O autor é o id do usuário que o middleware JWT atual coloca em `req.user`, sem `isAdmin`. O contrato `useCase.run(input, actor)`, o `User.isAdmin` e as políticas continuam na Story 1.3 (AR-4), junto com a sessão em banco.
- **Use cases migrados:** os quatro que escrevem hoje (`create-user`, `update-user`, `delete-user` e `change-user-password`), porque, com o cliente estendido, escrever sem actor lança erro. Todas as rotas de user passam por `authMiddleware`. O login e o refresh não escrevem no banco e não migram aqui.
- **Rota no `RequestLog`:** o padrão da rota, não o caminho real. Requisição não autenticada tem autor vazio.
- **Retrabalho esperado:** parte desses use cases será reescrita nas histórias 1.6 a 1.9.

## Notas técnicas (spike no Prisma 7.10.1 com `@prisma/adapter-pg`, em 2026-11-13)

Verificado em um banco PostgreSQL real, com uma extensão `$extends` de `query`, `AsyncLocalStorage` e `$transaction` interativa:

- **Bloqueios (AD-11):** `createMany`, `$executeRaw`, `$executeRawUnsafe`, `$queryRawUnsafe` de escrita e escrita aninhada de relação foram barrados pela extensão. `$queryRaw` de leitura passou. As operações brutas são interceptáveis por chaves próprias (`$queryRaw`, `$executeRaw`, `$queryRawUnsafe`, `$executeRawUnsafe`) dentro de `query`.
- **Auditoria e rollback (AD-10 e AD-12):** o `uow.run` guarda `{ actor, tx }` no `AsyncLocalStorage`, e a extensão grava o `AuditLog` por esse `tx`, na mesma transação. `create`, `update`, `upsert` e `delete` foram auditados, com `before` e `after` e a senha mascarada. Exceção, `Wrong` (via exceção sentinela) e timeout desfizeram tudo, inclusive a auditoria. O `before` exige um `findUnique` extra por escrita.
- **Erros (AD-17):** `P2002` (unicidade), `P2003` (chave estrangeira), `P2025` (registro inexistente), `P2028` (timeout da transação) e `P2034` (conflito de escrita em `Serializable`) chegam como `PrismaClientKnownRequestError`, com o erro do driver em `meta.driverAdapterError`. Não testei `maxWait` esgotado.
- **Tipos:** `DbClient = ClienteEstendido | ClienteDeTransaçãoDoEstendido` compila e rejeita o `PrismaClient` puro. O tipo antigo `PrismaClient` dos repositórios, de `core/repository.ts`, de `core/query.ts` e do `truncate.ts` não aceita o cliente estendido e precisa trocar para `DbClient`.
- **`$transaction` aninhado:** no Prisma 7 o cliente de transação expõe `$transaction` e ele funcionou aninhado. Não verifiquei se roda na mesma transação. O AD-10 manda fazer a paginação sem `$transaction`, e isso continua valendo por prudência, mas o motivo não é mais que o método não exista.
- **Padrão da rota no Express 5:** `req.route.path` traz o padrão, mas o `req.baseUrl` de routers montados sob parâmetro (como `/account/:accountId`) traz o UUID real. O padrão completo sai trocando, em `req.baseUrl`, cada valor de `req.params` de volta por `:nome`, e a rota que não casou sai como `UNMATCHED`. Verificado no `res.on('finish')` com routers aninhados, `mergeParams` e 404.
- **Falso positivo possível:** a detecção de escrita aninhada olha chaves como `create` e `connect` em `data`. Colunas `Json` com essas chaves podem ser barradas por engano. O `AuditLog` é ignorado para evitar isso.

## Impacto na Story 1.1

`src/database/index.ts` exporta hoje um único `prisma`, que seeds, populates e `truncate.ts` usam com `createMany` e `$executeRawUnsafe`. Esses usos passam a exigir o cliente puro.
