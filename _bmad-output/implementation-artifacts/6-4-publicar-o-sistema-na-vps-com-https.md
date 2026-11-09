# Story 6.4: Publicar o sistema na VPS com HTTPS

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 6). Em caso de divergência, vale o `epics.md`.

## História

As a responsável pelo sistema,
I want publicá-lo em um endereço seguro,
So that a casa use de verdade.


## Requisitos referenciados

- **NFR-6**: Produção: HTTPS, segredos fora do repositório, build e deploy reproduzíveis, 5 tentativas de login por 15 minutos por e-mail, backup diário com restauração testada antes do uso real.
- **AR-18**: Produção em VPS com Compose, proxy HTTPS, `trust proxy`, `helmet`, CORS fechado, `/health` com banco, deploy e rollback por imagem (AD-24).
- **AR-20**: `pnpm admin:create` idempotente; seeds de desenvolvimento sem dependência de `test/` (AD-24).

## Acceptance Criteria

**Given** a VPS com Docker Compose
**When** o ambiente sobe
**Then** o proxy serve `/` e `/api` na mesma origem por HTTPS, com Postgres sem porta pública e `restart: unless-stopped`

**Given** a API em produção
**When** recebe requisições pelo proxy
**Then** usa `trust proxy = 1`, `helmet` e CORS fechado, e o cookie sai com `Secure`

**Given** um deploy
**When** executo o procedimento documentado
**Then** a imagem do commit é baixada, o alvo `migrate` roda e a API sobe, e `/health` confirma o banco

**Given** uma versão com problema
**When** faço o rollback documentado
**Then** a imagem anterior volta a funcionar

**Given** segredos de produção
**When** inspeciono repositório e imagens
**Then** nenhum segredo está neles, e o `JWT_SECRET` é novo

**Given** o ambiente novo
**When** rodo `pnpm admin:create`
**Then** o primeiro Admin existe, e nenhum seed de desenvolvimento rodou
