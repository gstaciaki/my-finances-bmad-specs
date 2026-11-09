# Story 3.3: Alterar a Permissão e remover Membros

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 3). Em caso de divergência, vale o `epics.md`.

## História

As a Dono,
I want ajustar o que cada Membro pode fazer e tirar quem não deve mais acessar,
So that eu mantenha o controle da Conta.


## Requisitos referenciados

- **FR-11**: Dono remove qualquer Membro, menos a si; perda de acesso imediata.
- **FR-24**: Dono altera a Permissão de outro Membro, com efeito imediato.
- **NFR-8**: Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.

## Acceptance Criteria

**Given** outro Membro
**When** o Dono altera a Permissão entre Lançamento e Gestão
**Then** a mudança vale na próxima requisição desse Membro, mesmo com sessão aberta
**And** alteração e remoção rodam sob o lock da Conta

**Given** outro Membro
**When** o Dono o remove e confirma no `ConfirmDialog`
**Then** a API responde 204, e o removido recebe 404 na próxima requisição

**Given** o próprio Dono como alvo
**When** ele tenta se remover ou alterar a própria Permissão
**Then** recebe erro claro

**Given** um Membro que não é o Dono, mesmo com Gestão
**When** chama essas rotas
**Then** recebe 403

**Given** e2e com dois Usuários
**When** rodam os cenários de Lançamento, Gestão e Dono
**Then** cada Permissão recebe exatamente o que o PRD define nas rotas de Conta
