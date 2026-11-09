# Story 1.1: Preparar o ambiente de desenvolvimento e testes

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a desenvolvedor,
I want subir o banco e rodar a API e os testes com um comando,
So that o projeto evolua sobre versões corrigidas e um schema limpo.


## Requisitos referenciados

- **AR-1**: Histórico de migrations zerado; o schema recomeça do zero, sem `cpf`, e cada história adiciona a própria migration (AD-23, AD-25).
- **AR-11**: `app.ts` separado de `server.ts`; e2e com supertest e banco de teste protegido (AD-18, AD-20).
- **AR-12**: Atualizar Prisma 6.19.3, Express 5.3.0, Zod 4.6.7, jsonwebtoken 9.0.3 e uuid 11.1.1; adicionar `helmet`, `cookie-parser`, `express-rate-limit`, `date-fns`, `@date-fns/tz`, `tsc-alias`, `supertest`.
- **AR-15**: Compose só com banco em `127.0.0.1`, script do banco de teste, `.nvmrc`, `packageManager`, `.env.example` com placeholder; aposentar o script `run` (AD-18).
- **AR-20**: `pnpm admin:create` idempotente; seeds de desenvolvimento sem dependência de `test/` (AD-24).

## Acceptance Criteria

**Given** o repositório recém-clonado
**When** sigo o README e subo o Compose
**Then** só o banco sobe, em `127.0.0.1`, com um banco de desenvolvimento e um de teste, e a API roda no host
**And** as dependências estão nas versões do AR-12, com `.nvmrc`, `packageManager` e `.env.example` só com placeholders
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
