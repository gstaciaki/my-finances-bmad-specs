# Story 1.1: Preparar o ambiente de desenvolvimento e testes

Status: done

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a desenvolvedor,
I want subir o banco e rodar a API e os testes com um comando,
So that o projeto evolua sobre versões corrigidas e um schema limpo.


## Requisitos referenciados

- **AR-1**: Histórico de migrations zerado; o schema recomeça do zero, sem `cpf`, e cada história adiciona a própria migration (AD-23, AD-25).
- **AR-11**: `app.ts` separado de `server.ts`; e2e com supertest e banco de teste protegido (AD-18, AD-20).
- **AR-12**: Atualizar as dependências para as versões estáveis mais recentes (na implementação: Prisma 7 com `@prisma/adapter-pg`, Express 5.3, Zod 4.6, jsonwebtoken 9.0, uuid 14 e TypeScript 7); adicionar `helmet`, `cookie-parser`, `express-rate-limit`, `date-fns`, `@date-fns/tz`, `tsc-alias`, `supertest`. Enquanto `ts-jest` e `typescript-eslint` não suportarem o TypeScript 7, o Jest usa `@swc/jest` (sem checagem de tipos, que fica com `tsc`) e o ESLint usa o TypeScript 6 por alias.
- **AR-15**: Compose só com banco, sem restrição de interface por ser apenas o banco de desenvolvimento local, script do banco de teste, `.nvmrc`, `packageManager`, `.env.example` com placeholder; aposentar o script `run` (AD-18).
- **AR-17**: CI com pnpm, lint, `tsc` e testes com PostgreSQL real em cada PR (Epic 1, Story 1.1); o mesmo CI ganha o build e a publicação da imagem na `main` (Epic 6, Story 6.3) (AD-18, D-19).
- **AR-20**: `pnpm admin:create` idempotente; seeds de desenvolvimento sem dependência de `test/` (AD-24).

## Acceptance Criteria

**Given** o repositório recém-clonado
**When** sigo o README e subo o Compose
**Then** só o banco sobe, com um banco de desenvolvimento e um de teste, e a API roda no host
**And** as dependências estão nas versões mais recentes do AR-12, com `.nvmrc`, `packageManager` e `.env.example` só com placeholders
**And** o script `run` antigo não existe mais

**Given** o histórico atual de migrations
**When** aplico as migrations em um banco vazio
**Then** o histórico antigo não existe mais e uma nova migration inicial cria apenas o `User`, sem `cpf`
**And** o `cpf` sai também da API e dos seeds, e os seeds de desenvolvimento não dependem de `test/`

**Given** módulos existentes que dependem de tabelas ainda não recriadas
**When** o projeto é compilado
**Then** eles saem das rotas e da compilação, e voltam na história que recriar a tabela

**Given** `app.ts` separado de `server.ts`
**When** rodo os e2e
**Then** o supertest usa `DATABASE_URL_TEST`, e a limpeza recusa banco cujo nome não termine em `_test`

**Given** um pull request para a `main`
**When** o CI roda com pnpm e o Node do `.nvmrc`
**Then** executa o lint sem `--fix`, `tsc --noEmit` e os testes, com PostgreSQL real nos e2e
**And** qualquer passo que falhe reprova o PR

## Registro de implementação

Branch `epic-1-story-1/set-dev-and-test-environment`, repositório `my-finances-backend`.

### Requisitos

- [x] **AR-1**: histórico zerado; uma migration `init` cria só o `User` (com `password`, sem `cpf`). `cpf` removido da entidade, do DTO, do mapper, dos seeds e dos testes.
- [x] **AR-11**: `src/app.ts` exporta o Express sem escutar porta, e `server.ts` só inicia. Com `NODE_ENV=test` a API usa `DATABASE_URL_TEST`. `cleanDatabase` recusa banco cujo nome não termine em `_test` (teste em `test/helpers/base-test.spec.ts`). E2E com supertest em `test/e2e/`.
- [x] **AR-12**: dependências atualizadas e as novas adicionadas, **em versões mais novas que as da story** (ver Desvios). `supertest` ficou em `devDependencies`, com `@types/supertest`.
- [x] **AR-15**: Compose só com o banco, criando `myfinances` e `myfinances_test` por `docker/init-test-db.sql`; `pnpm test:db:migrate`; `.nvmrc`; `packageManager`; `.env.example` com placeholder; script `run` removido; README reescrito. A restrição a `127.0.0.1` foi retirada do requisito (ver Decisões de escopo).
- [x] **AR-17 (parte da 1.1)**: `.github/workflows/ci.yml` com `pnpm/setup`, Node pelo `.nvmrc` e `pnpm install --frozen-lockfile`. O job `test` (Postgres 16 como serviço) roda `prisma generate`, `pnpm test:db:migrate`, `pnpm tsc --noEmit` e `pnpm test`. O job `code-quality` roda `pnpm lint:check`. O CI passa.
- [x] **AR-20 (parte da 1.1)**: seeds e populate de desenvolvimento não dependem de `test/`, e o seed recusa rodar em produção. O `pnpm admin:create` fica para a Story 1.3.

### Ferramentas e qualidade

- **TypeScript 7:** `tsconfig.json` sem `baseUrl` (removido no TS 7), com `paths` relativos e `types: ["node", "jest"]` declarados.
- **Jest:** `@swc/jest` no lugar do `ts-jest`. O `moduleNameMapper` é gerado a partir dos `paths` do `tsconfig.json`. `jest.setup.ts` roda como `setupFilesAfterEnv` e desconecta o client do Prisma no `afterAll`, para o Jest encerrar. O `config()` do `dotenv` usa `quiet: true` para não imprimir `injected env` como `console.error` nos testes.
- **ESLint:** `typescript-eslint` 8.72 recusa TypeScript 7. O pacote `typescript` é um alias para `@typescript/typescript6@6.0.2`, que é o que o `typescript-eslint` enxerga, e o TS 7 fica em `typescript7` (`npm:typescript@^7.0.2`), que fornece o binário `tsc`. A config passou a `eslint.config.mjs`. O script `lint:check` roda `eslint` sem `--fix` sobre a raiz do projeto (não só `src`) e é o usado no CI; `lint` é a versão com `--fix`. O `jest.config.ts` mantém o `require` do `tsconfig.json` com `eslint-disable-next-line @typescript-eslint/no-require-imports`, porque o Jest o carrega como CommonJS.
- **Erros de tipo após o upgrade do Express 5:** `req.params.id as string` no controller de user e `Map` tipado em `core/controller.ts`.
- **Schema:** o `User` tinha perdido a coluna `password` no reset; foi restaurada na migration `init`.
- **Histórico:** os commits do branch foram reescritos para corrigir o autor (e-mail inválido nos commits antigos e autor `agent` nos do sandbox) e o branch foi publicado novamente no `origin`.

### Módulos fora da compilação

`account` e `transaction` (controllers, use-cases, repositórios, rotas, entidades, factories, queries, populates e prefabs) continuam no disco, mas estão no `exclude` do `tsconfig.json` e no `testPathIgnorePatterns` do `jest.config.ts`. Voltam nas histórias que recriarem as tabelas.

### Decisões de escopo (já refletidas no `epics.md` e no AD-18)

- **Porta do banco:** o Compose publica `5432:5432`, sem `127.0.0.1`. O Compose só roda o banco local de desenvolvimento, então o AR-15 e o critério de aceite foram ajustados.
- **Versões:** mantidas as mais recentes em vez das pinadas no AR-12, que agora pede as versões estáveis mais recentes. Prisma `^7.10.1` com `@prisma/adapter-pg` (antes pinado em 6.19.3), uuid `^14.0.3` (antes pinado em 11.1.1), TypeScript `^7.0.2`. Express, Zod e jsonwebtoken ficaram nas versões do AR-12 ou acima.
- **Jest:** `ts-jest` não suporta TypeScript 7 (peer `typescript <7`). Trocado por `@swc/jest`, que só transpila. A checagem de tipos fica com `tsc --noEmit`. O client do Prisma 7 exige `--experimental-vm-modules`, que está no script `test`.
- **TypeScript:** `typescript` aponta para o TS 6 por causa do `typescript-eslint`, e o `tsc` vem do pacote `typescript7` (ver Ferramentas e qualidade).
- **CI:** o CI de qualidade e testes passa a ser entregue nesta história (AR-17, parte). O Epic 6 (Story 6.3) acrescenta build e publicação da imagem.

### Pendências

Nenhuma. Fechadas depois do último registro:

- [x] `tsc --noEmit` no CI (passo "Check for Type errors", depois do `prisma generate`).
- [x] Lint sobre toda a raiz do projeto, com os erros antigos de `test/` corrigidos.
- [x] Testes passam, no CI e localmente.
- [x] AD-18 e `epics.md` atualizados com as decisões de escopo.

### Fechamento

- **PR:** `gstaciaki/my-finances-backend#30`, mergeado na `main` em 2026-11-13 (commit `6ee02a5`). O merge foi feito pelo autor com bypass de admin, porque a `main` exige 1 aprovação e o autor não pode aprovar o próprio PR.
- **Revisão:** feita antes do merge, sem bloqueadores. Os dois pontos de ajuste (o `NODE_TLS_REJECT_UNAUTHORIZED=0` no passo do `prisma generate` e o comentário sobre `PNPM_VERSION` no `Dockerfile`) já não existem na `main`.

### Acompanhar nas próximas histórias

- **Story 1.2 (extensão do Prisma, AR-2):** a tabela de versões e a linha de risco do `ARCHITECTURE-SPINE.md` ainda citam Prisma 6.19 e dizem que a extensão foi testada nessa versão, deixando a migração de versão maior para depois da v1. O projeto está no Prisma 7. Rever antes de implementar a 1.2.
- **Epic 6 (AR-16):** a imagem de produção ainda leva dependências de desenvolvimento e roda `tsx`; o `JWT_SECRET` de exemplo ainda é aceito (Story 6.3).
- **Claude Code Review:** o workflow (PR #31) foi desativado temporariamente na `main`. A falha da primeira execução (Claude encerrava em 2 s, sem chamada de modelo) não foi diagnosticada. A action só roda se o workflow do PR for idêntico ao da `main`, então depurar exige mudar a `main` antes.
