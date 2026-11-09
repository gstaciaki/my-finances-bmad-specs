# Story 6.6: Enviar e-mails reais e liberar para uso

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 6). Em caso de divergência, vale o `epics.md`.

## História

As a Usuário,
I want receber Convites e links de recuperação de verdade,
So that eu use esses recursos em produção.


## Requisitos referenciados

- **NFR-6**: Produção: HTTPS, segredos fora do repositório, build e deploy reproduzíveis, 5 tentativas de login por 15 minutos por e-mail, backup diário com restauração testada antes do uso real.
- **NFR-8**: Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.
- **AR-21**: Escolher provedor de e-mail e configurar SPF e DKIM (Deferred).

## Acceptance Criteria

**Given** o provedor escolhido, com SPF e DKIM configurados
**When** a API envia um e-mail
**Then** a implementação real de `EmailSender` entrega Convite e recuperação com links de `APP_BASE_URL`, sem mudar os use cases

**Given** o provedor fora do ar
**When** um Convite é criado ou a recuperação é pedida
**Then** valem os comportamentos das Stories 3.1 e 1.7

**Given** a suíte de e2e
**When** confiro cada item do NFR-8
**Then** isolamento, Permissões, poderes de Dono e Admin e Balanço na virada do mês têm testes passando

**Given** a produção publicada
**When** percorro o checklist de liberação
**Then** login limitado a 5 tentativas, HTTPS, backup restaurado e e-mail real estão confirmados, e o SM-1 pode começar
