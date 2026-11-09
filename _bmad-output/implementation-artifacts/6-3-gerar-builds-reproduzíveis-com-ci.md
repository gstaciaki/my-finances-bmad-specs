# Story 6.3: Gerar builds reproduzíveis com CI

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 6). Em caso de divergência, vale o `epics.md`.

## História

As a desenvolvedor,
I want que cada commit seja verificado e a `main` gere imagens,
So that eu publique sempre o que foi testado.


## Requisitos referenciados

- **NFR-6**: Produção: HTTPS, segredos fora do repositório, build e deploy reproduzíveis, 5 tentativas de login por 15 minutos por e-mail, backup diário com restauração testada antes do uso real.
- **AR-16**: Dockerfile multi-stage de produção com alvo `migrate`; scripts `build` e `start` (AD-18, D-17).
- **AR-17**: CI com pnpm, lint, `tsc`, testes e build; publicação da imagem na `main` (AD-18, D-19).

## Acceptance Criteria

**Given** um push ou PR no backend ou no frontend
**When** o CI roda com pnpm
**Then** executa lint, `tsc`, testes e build, com PostgreSQL real nos e2e do backend

**Given** um merge na `main`
**When** o CI termina com sucesso
**Then** publica as imagens marcadas pelo commit

**Given** o `Dockerfile` multi-stage do backend
**When** construo os alvos
**Then** há imagem de produção sem root e sem dependências de desenvolvimento, e um alvo `migrate` com a CLI do Prisma

**Given** `.env.example` e a configuração
**When** a API sobe com o `JWT_SECRET` de exemplo
**Then** ela recusa iniciar
