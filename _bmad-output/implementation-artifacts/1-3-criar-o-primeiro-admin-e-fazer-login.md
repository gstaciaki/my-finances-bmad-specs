# Story 1.3: Criar o primeiro Admin e fazer login

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a Usuário,
I want entrar com e-mail e senha e sair quando quiser,
So that eu use o sistema com segurança.


## Requisitos referenciados

- **FR-1**: Login com e-mail e senha; mesma mensagem para credenciais inválidas; sessão expira após 7 dias sem uso; logout vale imediatamente.
- **NFR-2**: Dados sensíveis: nenhuma resposta contém senha ou hash; sem coleta de CPF.
- **NFR-5**: Sessão: tokens de acesso e de renovação com finalidades distintas.
- **NFR-6**: Produção: HTTPS, segredos fora do repositório, build e deploy reproduzíveis, 5 tentativas de login por 15 minutos por e-mail, backup diário com restauração testada antes do uso real.
- **AR-4**: Contrato `useCase.run(input, actor)`, políticas em `src/policies/`, `requireAdmin`, entrada de controllers por campos explícitos (AD-2).
- **AR-7**: Sessão em banco, refresh opaco com rotação e janela de 30 s, cookie `httpOnly`, `X-Requested-With` (AD-5, AD-6).
- **AR-8**: `zEmail` e `zPassword` únicos, corrigindo o regex de símbolos (AD-23, D-20) e as mensagens com entidade errada (D-21).
- **AR-20**: `pnpm admin:create` idempotente; seeds de desenvolvimento sem dependência de `test/` (AD-24).
- **UX-DR20**: `GET /me` com `id`, `name`, `email`, `isAdmin`.

## Acceptance Criteria

**Given** a variável com a senha inicial
**When** rodo `pnpm admin:create` duas vezes
**Then** existe um único Admin, e a segunda execução não altera nada

**Given** e-mail e senha válidos, com o e-mail em qualquer caixa
**When** chamo `POST /api/login`
**Then** recebo um JWT de 15 minutos com `sub` e `sessionId` e o cookie de renovação com os atributos do AD-6
**And** nenhuma resposta contém senha ou hash

**Given** e-mail inexistente, senha errada ou Usuário com `deletedAt`
**When** tento entrar
**Then** a resposta é a mesma nos três casos, e o hash é comparado mesmo sem Usuário

**Given** 5 tentativas com o mesmo e-mail em 15 minutos
**When** faço a sexta
**Then** recebo 429

**Given** uma sessão ativa
**When** faço logout e reutilizo o token de acesso
**Then** recebo 401 imediatamente

**Given** um token válido
**When** chamo `GET /api/me`
**Then** recebo `id`, `name`, `email` e `isAdmin`
**And** o `actor` vem só do middleware, com `isAdmin` lido do banco
**And** as mensagens de validação citam a entidade correta (D-21)
