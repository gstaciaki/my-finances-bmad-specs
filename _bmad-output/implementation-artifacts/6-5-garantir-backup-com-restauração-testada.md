# Story 6.5: Garantir backup com restauração testada

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 6). Em caso de divergência, vale o `epics.md`.

## História

As a responsável pelo sistema,
I want backups diários que funcionem,
So that o histórico da casa não se perca.


## Requisitos referenciados

- **NFR-6**: Produção: HTTPS, segredos fora do repositório, build e deploy reproduzíveis, 5 tentativas de login por 15 minutos por e-mail, backup diário com restauração testada antes do uso real.
- **AR-19**: Backup diário fora da VPS e restauração testada (AD-24, NFR-6).

## Acceptance Criteria

**Given** o banco de produção
**When** passa um dia
**Then** um `pg_dump` é gerado e copiado para fora da VPS, e uma falha fica visível nos logs

**Given** um backup real
**When** executo o procedimento de restauração em um banco separado
**Then** Usuários, Contas e Movimentações voltam íntegros, e o Balanço confere
**And** a data do teste fica registrada, como condição para o uso real
