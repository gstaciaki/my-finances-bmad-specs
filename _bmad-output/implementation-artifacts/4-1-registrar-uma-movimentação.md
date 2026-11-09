# Story 4.1: Registrar uma Movimentação

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 4). Em caso de divergência, vale o `epics.md`.

## História

As a Membro,
I want registrar uma Entrada ou Saída,
So that o dinheiro da casa fique anotado.


## Requisitos referenciados

- **FR-15**: Membro registra Movimentação com tipo, valor positivo com até 2 casas, data não futura (padrão hoje em São Paulo), descrição e Categoria opcionais; valor acima do limite é rejeitado; formulário fica pronto para o próximo lançamento.
- **FR-18**: Sistema guarda autoria de criação, alteração e exclusão de Movimentações, além de toda escrita no banco e toda requisição HTTP, sem corpo nem segredos; não aparece nas telas da v1.
- **NFR-3**: Exatidão: valores em centavos; até R$ 21.474.836,47 por Movimentação.
- **AR-14**: Criação de Conta e vínculo do Dono em uma transação (D-14) e índice `(accountId, occurredAt)` (D-16).

## Acceptance Criteria

**Given** a migration desta história
**When** é aplicada
**Then** cria `Transaction` com `amount` inteiro em centavos, `type`, `occurredAt` `timestamptz`, `description` opcional, `createdById` e índice `(accountId, occurredAt)`

**Given** um Membro, com qualquer Permissão
**When** chama `POST /api/account/:accountId/transaction` com dados válidos
**Then** recebe 201 com `Location`, e `createdById` é o Usuário autenticado
**And** um e2e confirma o mesmo autor no `AuditLog`

**Given** `occurredAt` só com data, ou omitido
**When** salvo
**Then** vale o primeiro instante do dia informado, ou de hoje, em `America/Sao_Paulo`

**Given** uma data posterior a hoje nesse fuso
**When** salvo
**Then** recebo 400

**Given** um valor zero, negativo, não inteiro ou acima de 2.147.483.647 centavos
**When** salvo
**Then** recebo 400, nunca 500

**Given** um não Membro
**When** tenta registrar na Conta
**Then** recebe 404
