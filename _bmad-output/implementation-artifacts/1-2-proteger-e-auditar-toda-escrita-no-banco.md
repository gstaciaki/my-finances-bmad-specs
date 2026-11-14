# Story 1.2: Proteger e auditar toda escrita no banco

Status: done

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a responsável pelo sistema,
I want que toda escrita seja transacional e registrada,
So that eu saiba quem alterou o quê e nada seja gravado pela metade.


## Roteiro de implementação

Esta seção diz **o que fazer, em que ordem e como saber que terminou**. As seções seguintes (requisitos, critérios de aceite, decisões e notas técnicas) são a referência de apoio.

**Em uma frase:** depois desta história, toda escrita no banco passa por uma transação que registra quem mudou o quê, toda requisição HTTP deixa um registro, e os erros do banco viram respostas HTTP claras.

**Branch do backend:** `epic-1-story-1.2-audit-and-protect-writing-on-db`. Ao fim de cada etapa, rode `pnpm lint:check`, `pnpm tsc --noEmit` e `pnpm test`, e faça um commit.

| # | Etapa | Critérios de aceite |
|---|---|---|
| 1 | Criar as tabelas `AuditLog` e `RequestLog` | 6 |
| 2 | Preparar `src/database/`: cliente puro, actor e erro base | 3 |
| 3 | Criar a extensão do Prisma (bloqueios e auditoria) | 2, 3 |
| 4 | Trocar o tipo `PrismaClient` por `DbClient` | 3 |
| 5 | Criar o Unit of Work e o `FakeUnitOfWork` | 1 |
| 6 | Passar o actor adiante e migrar os 4 use cases | 7 |
| 7 | Traduzir erros do Prisma e completar o mapa de erros | 5 |
| 8 | Registrar as requisições (`RequestLog`) | 4 |
| 9 | Criar a regra de lint do `PrismaClient` | 3 |
| 10 | Escrever os testes e2e e conferir os critérios | 1 a 7 |
| 11 | Fechar a história | todos |

> **Atenção:** entre as etapas 3 e 6, as rotas que escrevem usuário (`POST /user`, `PATCH /user/:id`, `DELETE /user/:id` e `POST /user/change-password`) deixam de funcionar no ambiente de desenvolvimento, porque o cliente da aplicação passa a exigir actor. Isso é esperado. Os testes unitários continuam passando, porque usam repositórios falsos.

---

### Etapa 1: criar as tabelas `AuditLog` e `RequestLog`

**O que fazer:** adicionar os dois modelos em `prisma/schema.prisma` e gerar a migration.

- `AuditLog`: `id` (UUID), `at` (data, padrão agora), `actorId` (texto, **sem** chave estrangeira: guarda o id do usuário ou `system`), `model` (texto), `recordId` (texto), `operation` (texto: `create`, `update`, `upsert` ou `delete`), `before` e `after` (`Json?`).
- `RequestLog`: `id` (UUID), `at` (data, padrão agora), `method` (texto), `route` (texto), `status` (inteiro), `durationMs` (inteiro), `actorId` (texto opcional: vazio quando a requisição não foi autenticada).
- Sugestão de índices: `AuditLog` por `(model, recordId)` e por `at`; `RequestLog` por `at`.

**Como:** com o Compose no ar, rode `pnpm migrate:dev --name audit_and_request_log`. Depois rode `pnpm test:db:migrate` para o banco de teste.

**Pronto quando:** as duas tabelas existem nos bancos `myfinances` e `myfinances_test`, e o `pnpm prisma generate` não dá erro.

### Etapa 2: preparar `src/database/`

**O que fazer:**
- Criar `src/database/context.ts` com: o tipo `Actor` (`{ userId: string }`), a constante `SYSTEM` (actor `system`) e o `AsyncLocalStorage` que guarda `{ actor, tx }` durante uma transação.
- Criar o `ForbiddenOperationError` (em `src/errors/`). Ele é erro de programação, não regra de negócio: não entra no mapa de erros e, se escapar, vira 500.
- Em `src/database/index.ts`, exportar o cliente **puro** como `prismaRaw`. Por enquanto, `prisma` continua apontando para o mesmo cliente (temporário até a etapa 3).
- Trocar `seed.ts`, `seeds/`, `populate.ts`, `populates/user.populate.ts` e `truncate.ts` para usar `prismaRaw`.

**Pronto quando:** `pnpm seed` e `pnpm populate` continuam funcionando.

### Etapa 3: criar a extensão do Prisma

**O que fazer:** em `src/database/extension.ts`, estender o cliente com `$extends({ query: ... })` para:
- **Bloquear** com `ForbiddenOperationError`: `createMany`, `createManyAndReturn`, `updateMany`, `updateManyAndReturn`, `deleteMany`, `$executeRaw`, `$executeRawUnsafe`, escrita aninhada de relação, e `$queryRaw`/`$queryRawUnsafe` que contenham `insert`, `update`, `delete`, `truncate`, `drop`, `alter` ou `create`.
- **Exigir actor** em `create`, `update`, `upsert` e `delete`: sem actor no contexto, lançar `ForbiddenOperationError`.
- **Auditar** esses quatro: ler o estado anterior (menos no `create`), executar a operação e gravar o `AuditLog` **pelo `tx` do contexto**, para ficar na mesma transação. Mascarar `password` e qualquer coluna terminada em `Hash`.
- Ignorar `AuditLog` e `RequestLog` (não são auditados).

Em seguida, exportar o cliente estendido como `prisma` (o da aplicação) e o tipo `DbClient` (cliente estendido ou cliente de transação do estendido). O esqueleto está no **Anexo A**.

**Pronto quando:** um teste de integração (contra o banco de teste) comprova: escrita sem actor é barrada, `createMany` é barrado, e um `create`/`update`/`delete` dentro de uma transação grava o `AuditLog` com estado anterior e posterior e a senha mascarada.

### Etapa 4: trocar `PrismaClient` por `DbClient`

**O que fazer:** nos arquivos abaixo, trocar o tipo `PrismaClient` por `DbClient` (importado de `@src/database`):
- `src/core/repository.ts` e `src/core/query.ts`
- `src/repositories/user/user.repository.ts`

`src/database/truncate.ts` e `test/helpers/base-test.ts` recebem o cliente **puro** e podem continuar com o tipo `PrismaClient`: ficam dentro de `src/database/` e `test/`, onde a regra de lint da Etapa 9 permite.

Em `BaseRepository.findWhere`, trocar o `$transaction([...])` em lote por duas consultas em sequência (`findMany` e depois `count`), como pede o AD-10.

**Pronto quando:** `pnpm tsc --noEmit` passa e os testes de repositório continuam verdes.

### Etapa 5: criar o Unit of Work

**O que fazer:** em `src/database/unit-of-work.ts`:
- Definir `IUnitOfWork` com `run(actor, fn, opts?)`, onde `fn` recebe os repositórios (`createRepositories(db)`, por enquanto só `user`) e devolve um `Either`.
- Implementar com `prisma.$transaction(...)`, `timeout: 30000` e `maxWait: 5000`, guardando `{ actor, tx }` no `AsyncLocalStorage` durante o callback.
- Se o callback devolver `Wrong`, desfazer a transação e devolver o mesmo `Wrong`. Faça isso lançando uma exceção interna dentro da transação e capturando-a do lado de fora (ver Anexo A). Se o callback lançar exceção, desfazer e deixar a exceção subir.
- Criar `test/helpers/fake-unit-of-work.ts` com o `FakeUnitOfWork`: recebe repositórios falsos e só executa o callback.

O parâmetro `lockAccounts` do AR-2 **não** é implementado agora (ainda não existe a tabela `Account`). Deixe `opts` extensível.

**Pronto quando:** há testes de integração para os três casos do AC 1: sucesso grava, `Wrong` desfaz tudo (inclusive o `AuditLog`) e exceção desfaz tudo.

### Etapa 6: passar o actor adiante e migrar os 4 use cases

**O que fazer:**
- Em `src/core/use-case.ts`, mudar o contrato para `run(input, actor)` e `execute(input, actor)`. As políticas, o `isAdmin` e a leitura do actor pelo banco continuam na Story 1.3.
- Criar uma função que monta o `Actor` a partir do `req.user.id` que o `authMiddleware` já coloca na requisição. Os controllers de user passam esse actor. Os controllers de sessão (login e refresh) passam `SYSTEM`.
- Migrar `create-user`, `update-user`, `delete-user` e `change-user-password`: receber o `IUnitOfWork` no construtor e fazer toda escrita dentro de `uow.run`. O hash da senha fica **fora** da transação. Se uma regra depende de uma leitura (por exemplo, e-mail duplicado), releia dentro do `uow.run`.
- Ajustar `src/factories/user.factory.ts` e `session.factory.ts`.
- Atualizar os specs: usar o `FakeUnitOfWork` e passar um actor (crie um `genActor()` em `test/prefab/`).

Os use cases `list-users` e `show-user`, e os de sessão, só leem e não migram.

**Pronto quando:** `POST /user`, `PATCH /user/:id`, `DELETE /user/:id` e `POST /user/change-password` voltam a funcionar, e cada uma deixa uma linha em `AuditLog` com o id do usuário autenticado.

### Etapa 7: traduzir erros do Prisma

**O que fazer:**
- Criar em `src/errors/`: `ConflictError` (409), `ServiceUnavailableError` (503), `ForbiddenError` (403), `TooManyRequestsError` (429) e um erro genérico de "não encontrado" que não exija entidade e campo (para o `P2025`).
- Em `AbstractController`, adicionar os métodos de resposta 403 e 429 e registrar as classes novas em `getErrorMap`.
- No `catch` de `AbstractUseCase.run` (único tradutor): `P2002` e `P2003` viram 409, `P2025` vira 404, `P2028` e `P2034` viram 503, e o resto continua `UnknownError`. O código do erro está em `error.code`.

**Pronto quando:** testes unitários cobrem cada código, e o corpo de erro continua `{ code, error, slug }`.

### Etapa 8: registrar as requisições

**O que fazer:** criar `src/middlewares/request-log.middleware.ts` e registrá-lo em `src/app.ts` antes das rotas. Ao disparar `res.on('finish')`, gravar em `RequestLog`:
- método, status e duração;
- o **padrão da rota**: `req.baseUrl` com cada valor de `req.params` trocado de volta por `:nome`, mais `req.route.path` (ou `UNMATCHED` se `req.route` não existir);
- o autor: `req.user?.id`, vazio se não houver;
- **nunca** o corpo, a query string ou os headers.

Gravar com `prismaRaw` (fora do Unit of Work), sem `await` bloqueante e com `try/catch`: uma falha ao gravar o log não pode alterar a resposta. O código está no **Anexo B**.

**Pronto quando:** `GET /api/user/<uuid>` grava `/api/user/:id`, uma rota inexistente grava `UNMATCHED`, e um log que falha não muda o status da resposta.

### Etapa 9: criar a regra de lint do `PrismaClient`

**O que fazer:** em `eslint.config.mjs`, adicionar `@typescript-eslint/no-restricted-imports` com `allowTypeImports: true`, proibindo o valor `PrismaClient` vindo de `@prisma/client` e de `@src/generated/prisma/client` (e de caminhos relativos para `generated/prisma/client`). A regra não vale para `src/database/**`, `test/**` e `jest.setup.ts`. Onde só o tipo for necessário, use `import type`.

**Pronto quando:** um `import { PrismaClient }` temporário em um use case faz o `pnpm lint:check` falhar. Remova-o depois do teste.

### Etapa 10: testes e2e e conferência dos critérios

**O que fazer:** em `test/e2e/`, com o supertest contra o banco de teste:
- Editar um usuário pela API grava `AuditLog` com o actor do token, o estado anterior e o posterior e a senha mascarada.
- A transação desfaz tudo quando a regra falha (por exemplo, e-mail duplicado não deixa auditoria órfã).
- O `RequestLog` grava método, padrão da rota, status, duração e autor, e não grava corpo nem query string.
- Erro de unicidade vira 409 e registro inexistente vira 404, nunca 500.

Depois, confira cada critério de aceite da tabela do início.

### Etapa 11: fechar a história

**O que fazer:** abrir o PR no backend, com CI verde. Depois, no repositório de specs: marcar as etapas concluídas nesta story, registrar o resultado e mover a story para `review` no `sprint-status.yaml`.

---

### Anexo A: esqueleto da extensão e do Unit of Work

Validado no spike (Prisma 7.10.1). Adapte os nomes ao projeto.

```ts
const BLOCKED = ['createMany', 'createManyAndReturn', 'updateMany', 'updateManyAndReturn', 'deleteMany'];
const AUDITED = ['create', 'update', 'upsert', 'delete'];

export const prisma = prismaRaw.$extends({
  query: {
    $allModels: {
      async $allOperations({ model, operation, args, query }) {
        if (BLOCKED.includes(operation)) throw new ForbiddenOperationError(operation);
        if (model === 'AuditLog' || model === 'RequestLog' || !AUDITED.includes(operation)) {
          return query(args);
        }
        const ctx = als.getStore();
        if (!ctx) throw new ForbiddenOperationError('escrita sem actor');
        // (aqui: recusar escrita aninhada de relação em args.data)
        const delegate = ctx.tx[model[0].toLowerCase() + model.slice(1)];
        const before = operation === 'create' ? null : await delegate.findUnique({ where: args.where });
        const result = await query(args);
        await ctx.tx.auditLog.create({
          data: {
            actorId: ctx.actor.userId, model, recordId: result?.id ?? before?.id, operation,
            before: mask(before), after: operation === 'delete' ? null : mask(result),
          },
        });
        return result;
      },
    },
    async $executeRaw() { throw new ForbiddenOperationError('$executeRaw'); },
    async $executeRawUnsafe() { throw new ForbiddenOperationError('$executeRawUnsafe'); },
    // $queryRaw e $queryRawUnsafe: recusar se o SQL tiver palavra de escrita; senão, query(args)
  },
});

// Unit of Work
class Rollback extends Error { constructor(public readonly wrong: unknown) { super('rollback'); } }

async run(actor, fn, opts = {}) {
  try {
    return await prisma.$transaction(
      tx => als.run({ actor, tx }, async () => {
        const result = await fn(createRepositories(tx));
        if (result.isWrong()) throw new Rollback(result);
        return result;
      }),
      { timeout: 30_000, maxWait: 5_000, ...opts },
    );
  } catch (e) {
    if (e instanceof Rollback) return e.wrong;   // devolve o Wrong sem erro
    throw e;
  }
}
```

### Anexo B: padrão da rota no `RequestLog`

```ts
res.on('finish', () => {
  let base = req.baseUrl;
  for (const [key, value] of Object.entries(req.params ?? {})) {
    if (typeof value === 'string' && value) base = base.split(`/${value}`).join(`/:${key}`);
  }
  const route = req.route ? `${base}${req.route.path}` : 'UNMATCHED';

  prismaRaw.requestLog
    .create({ data: { method: req.method, route, status: res.statusCode, durationMs: Date.now() - start, actorId: (req as any).user?.id ?? null } })
    .catch(() => {});   // falha de log nunca altera a resposta
});
```
Verificado no Express 5: com routers aninhados e `mergeParams`, o `baseUrl` traz o UUID real, e a troca por `req.params` devolve `/api/account/:accountId/transaction/:id`.

---

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

- **Actor:** a 1.2 cria o tipo `Actor` e a constante `SYSTEM`. O autor é o id do usuário que o middleware JWT atual coloca em `req.user`, sem `isAdmin`. A assinatura `useCase.run(input, actor)` entra já na 1.2, porque o `uow.run` precisa do actor. O `User.isAdmin`, as políticas e a leitura do actor pelo banco continuam na Story 1.3 (AR-4), junto com a sessão em banco.
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
- **Falso positivo possível:** a detecção de escrita aninhada olha chaves como `create` e `connect` em `data`. Colunas `Json` com essas chaves podem ser barradas por engano. O `AuditLog` é ignorado para evitar isso. **Confirmado na implementação:** `update({ data: { name: { set: 'x' } } })`, sintaxe válida para coluna escalar, é barrado como escrita aninhada (`ForbiddenOperationError`). Nenhum código atual usa essa forma.

## Impacto na Story 1.1

`src/database/index.ts` exporta hoje um único `prisma`, que seeds, populates e `truncate.ts` usam com `createMany` e `$executeRawUnsafe`. Esses usos passam a exigir o cliente puro.

## Registro de implementação

Branch `epic-1-story-1.2-audit-and-protect-writing-on-db`, repositório `my-finances-backend`. Verificação no fim: `eslint` e `tsc --noEmit` limpos; Jest com **22 suítes, 220 testes passando e 1 `todo`** (escrita aninhada, que precisa de uma relação no schema), contra o banco de teste.

### Etapas

- [x] **1** Tabelas `AuditLog` e `RequestLog` (migration `20261113023435_audit_log_request_log`). `b0a9c12`
- [x] **2** `src/database/`: cliente puro (`prismaRaw`), `context.ts` (`Actor`, `SYSTEM`, `AsyncLocalStorage`), `ForbiddenOperationError`; seeds, populates e `truncate.ts` no cliente puro. `2907fe2`, `1444593`
- [x] **3** Extensão do Prisma (bloqueios, exigência de actor, auditoria com máscara) e tipo `DbClient`. `21098dd`
- [x] **4** `PrismaClient` trocado por `DbClient`; `findWhere` sem `$transaction`. `052c3b7`
- [x] **5** `UnitOfWork`, `IUnitOfWork`, `createRepositories` e `FakeUnitOfWork`. `37156fa`
- [x] **6** Actor adiante e os 4 use cases de escrita migrados; `actorFromRequest` em `src/util/actor.ts`; factories e controller ligados. `ef65d24`
- [x] **7** Tradução dos erros do Prisma e mapa de erros completo. `6084ada`
- [x] **8** `request-log.middleware.ts`, registrado em `app.ts`. `98aaacb`
- [x] **9** Regra de lint do valor `PrismaClient`. `df316b9`
- [x] **10** Testes e2e e conferência dos critérios. `27e5b11`
- [x] **11** PR aberto e mergeado com o CI verde; story movida para `done` (ver Fechamento).

### Conferência dos critérios de aceite

| AC | Onde está comprovado |
|---|---|
| 1. Rollback em `Wrong` e em exceção | `src/database/unit-of-work.spec.ts` (sucesso, `Wrong`, exceção, `P2002`, timeout `P2028`, execuções concorrentes) e o e2e de duas criações simultâneas com o mesmo e-mail (uma `AuditLog`, um usuário) |
| 2. `AuditLog` com estados e máscara | `src/database/extension.spec.ts` (`create`, `update`, `upsert`, `delete`, senha mascarada, rollback junto) e `test/e2e/user-audit.e2e.spec.ts` (as 4 rotas, actor do token) |
| 3. Operações proibidas, actor e lint | `extension.spec.ts` (bloqueios, sem actor, sem transação) e `test/lint/prisma-client-import.spec.ts`. Um `import { PrismaClient }` temporário em um use case falhou o lint, e foi removido. |
| 4. `RequestLog` | `test/e2e/request-log.e2e.spec.ts` (padrão da rota, `UNMATCHED`, routers aninhados com `mergeParams`, sem corpo/query/headers, falha de log sem efeito na resposta). **Ver a decisão sobre o autor, abaixo.** |
| 5. Erros e `{ code, error, slug }` | `translate-error.spec.ts`, `core/use-case.spec.ts`, `user.controller.spec.ts` e os e2e de 409, 404, 400 e 401. Nenhum retorna 500. |
| 6. Migration | `AuditLog` e `RequestLog` sem chave estrangeira em `actorId`; nenhum dos dois é auditado (`extension.spec.ts`). |
| 7. Use cases de escrita | Só escrevem em `uow.run`; os specs usam o `FakeUnitOfWork` e o e2e confere o actor do token. |

### Desvios do roteiro (decisões tomadas na implementação)

- **Hierarquia de use cases.** Em vez de `run(input, actor)` em todo `AbstractUseCase`, há três classes em `src/core/`: `UseCasePipeline` (validação e tradução de erros), `AbstractUseCase` (leitura, `run(input)`) e `AbstractWriteUseCase` (`run(input, actor)`, abre o `uow.run` e chama `execute(input, repos)`). O actor é obrigatório só para quem escreve, e o compilador garante isso. Por isso os controllers de leitura e os de sessão (login e refresh) **não** passam `SYSTEM`: não escrevem.
- **Hash da senha dentro da transação.** Foi descartada uma etapa `prepare` fora do `uow.run`. Risco de desempenho aceito: o `bcrypt` segura uma conexão por cerca de 100 ms. Reintroduzir só se houver contenção no pool.
- **Autor do `RequestLog` sem autenticação é `system`, e não vazio.** A coluna `actorId` é `String` obrigatória e o middleware grava `SYSTEM.userId`. Diverge do AC 4 e do `epics.md` (linha 261, "ou vazio se a requisição não foi autenticada"), que ainda precisam ser ajustados. Efeito colateral: requisição anônima e ação interna do sistema ficam com o mesmo `actorId`.
- **Tradução dos erros do Prisma** fica em `UseCasePipeline.runPipeline`, e não em `AbstractUseCase.run`, para valer também para os use cases de escrita. A função é `translatePrismaError` (`src/util/prisma/translate-error.ts`); as mensagens são fixas, para não vazar tabela, coluna ou valor.
- **Nomes e formas.** A classe do 409 se chama `ConflitError` (a story dizia `ConflictError`). `NotFoundError` ganhou sobrecarga: sem argumentos serve ao `P2025`; com `(entidade, campo, valor)` é a forma antiga.
- **`BaseRepository` sem `any`.** O `model` passou de `any` a `ModelDelegate<T>` (interface estrutural mínima, atribuível tanto ao cliente puro quanto ao estendido), e `Repositories` é `{ user: IUserRepository }`, o que permite repositórios falsos no `FakeUnitOfWork`.
- **`findWhere` com `Promise.all`.** A story pedia consultas em sequência. `findMany` e `count` são leituras independentes, e o `$transaction` em lote também não dava snapshot único no isolamento padrão.
- **Testes de repositório com o cliente estendido.** Em vez de `prismaTest as DbClient`, os specs usam o cliente da aplicação dentro de `runAsActor` (`test/helpers/run-as-actor.ts`), que replica `{ actor, tx }` do Unit of Work.
- **Lint.** A regra também pegou três repositórios fora da compilação (`account`, `transaction` e `user-account`), trocados para `DbClient`. Como esses módulos não entram no `tsc`, a troca só será validada quando as tabelas voltarem.

### Observações para as próximas histórias

- **`POST /user` devolve a entidade inteira, inclusive `password` (o hash).** O corpo da resposta tem `password`. Falta um mapper de saída para os use cases de escrita; vale tratar junto com as histórias 1.6 a 1.9.
- **`PATCH /user/:id` com o mesmo e-mail responde 409.** O `update-user` consulta `findByEmail` e encontra o próprio usuário. Preexistente; o retrabalho dos use cases está previsto nas histórias 1.6 a 1.9.
- **Escrita aninhada sem teste real.** O schema atual não tem relações, então há um `it.todo` em `extension.spec.ts`; habilitar quando `Account` e `UserAccount` voltarem. O falso positivo de `{ set }` está nas Notas técnicas.
- **`maxWait` esgotado não foi testado** (o `timeout` da transação foi, e vira `P2028`).
- **Armadilha do `PrismaPromise`:** a query só executa no `await`. Dentro de `als.run`, o `await` precisa ficar dentro do callback; retornar a promise sem `await` roda a query fora do contexto e falha com "writing without actor".
- **`RequestLog` não é assíncrono de verdade:** a gravação não é aguardada, então testes precisam esperar a linha aparecer (`waitForRequestLogs` em `test/helpers/e2e.ts`).

### Fechamento

- **PR:** `gstaciaki/my-finances-backend#32`, mergeado na `main` em 2026-11-14 (commit `dad37dd`). CI verde: `Run Tests` e `Run ESLint`. O PR não tem revisão registrada.
- **Pendências que seguem abertas** (não bloqueiam o fechamento; estão em "Observações para as próximas histórias"): o `epics.md` (linha 261) e o AC 4 ainda dizem que o autor do `RequestLog` é vazio quando não há autenticação, e a implementação grava `system`; o corpo do `POST /user` devolve o hash da senha; o `PATCH /user/:id` com o mesmo e-mail responde 409.
