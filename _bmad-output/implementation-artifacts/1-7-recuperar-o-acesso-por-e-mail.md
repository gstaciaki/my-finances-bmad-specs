# Story 1.7: Recuperar o acesso por e-mail

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a Usuário que esqueceu a senha,
I want receber um link para criar outra,
So that eu volte a entrar sem depender do Admin.


## Requisitos referenciados

- **FR-2**: Recuperação de senha por link enviado por e-mail, de uso único e válido por 1 hora; resposta igual exista ou não o e-mail; nova senha segue a política.
- **AR-9**: `EmailSender` com implementação falsa, `APP_BASE_URL` (AD-15).
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.

## Acceptance Criteria

**Given** a tela `/senha/esqueci`
**When** informo um e-mail, cadastrado ou não
**Then** a API responde 202 com corpo fixo, e a tela é igual nos dois casos
**And** uma falha no `EmailSender` só vai para o log

**Given** um link com menos de 1 hora, montado com `APP_BASE_URL`
**When** defino uma senha válida em `/senha/nova`
**Then** a senha muda, todas as Sessões são revogadas e vou ao Login

**Given** um link já usado ou vencido
**When** tento usá-lo
**Then** recebo 400 e a tela oferece pedir outro link

**Given** o banco
**When** inspeciono o token de recuperação
**Then** só existe o hash, e o token não aparece em logs
