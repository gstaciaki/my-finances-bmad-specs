# Story 3.6: Admin exclui um Usuário

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 3). Em caso de divergência, vale o `epics.md`.

## História

As a Admin,
I want excluir um login,
So that a pessoa não acesse mais o sistema.


## Requisitos referenciados

- **FR-5**: Admin lista, edita, redefine a senha e exclui Usuários; excluído não faz login, vira ex-Membro e o e-mail fica reservado; Dono de Conta com outros Membros só é excluído após transferência; Dono único Membro leva a Conta junto.
- **NFR-8**: Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.

## Acceptance Criteria

**Given** um Usuário sem Conta como Dono
**When** o Admin o exclui em `/admin/usuarios` e confirma
**Then** a API responde 204, o Usuário recebe `deletedAt`, suas Sessões são revogadas, seus vínculos apagados e seus Convites pendentes viram `CANCELLED`
**And** o e-mail fica reservado e a autoria permanece

**Given** um Usuário que é Dono de uma Conta com outros Membros
**When** o Admin tenta excluí-lo
**Then** recebe erro nomeando a necessidade de transferência, e nada muda em nenhuma das Contas

**Given** um Dono que é o único Membro da Conta
**When** o Admin o exclui
**Then** a Conta recebe `deletedAt` no mesmo Unit of Work e seus Convites pendentes viram `CANCELLED`

**Given** o próprio Admin como alvo
**When** tenta se excluir
**Then** recebe erro claro e nada muda
