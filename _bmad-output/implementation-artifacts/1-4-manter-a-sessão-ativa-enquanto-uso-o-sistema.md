# Story 1.4: Manter a sessão ativa enquanto uso o sistema

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a Usuário,
I want continuar logado enquanto uso o sistema,
So that eu não seja interrompido a cada 15 minutos.


## Requisitos referenciados

- **FR-1**: Login com e-mail e senha; mesma mensagem para credenciais inválidas; sessão expira após 7 dias sem uso; logout vale imediatamente.
- **NFR-5**: Sessão: tokens de acesso e de renovação com finalidades distintas.
- **AR-7**: Sessão em banco, refresh opaco com rotação e janela de 30 s, cookie `httpOnly`, `X-Requested-With` (AD-5, AD-6).

## Acceptance Criteria

**Given** um cookie de renovação válido e o header `X-Requested-With`
**When** chamo `POST /api/refresh`
**Then** recebo novo token de acesso, o token de renovação é trocado, e a Sessão vale por mais 7 dias

**Given** duas abas renovando juntas
**When** o token anterior é usado em até 30 segundos
**Then** recebe novo token de acesso sem nova rotação

**Given** o token anterior usado depois de 30 segundos
**When** chamo a renovação
**Then** recebo 401 e a Sessão é revogada

**Given** a ausência do header, um token de acesso no lugar do de renovação ou uma Sessão com mais de 7 dias sem uso
**When** chamo a renovação
**Then** recebo 401
