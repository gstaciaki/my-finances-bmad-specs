# Story 5.1: Ver a lista de meses da Conta

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 5). Em caso de divergência, vale o `epics.md`.

## História

As a Membro,
I want ver cada mês com o que entrou, saiu e sobrou,
So that eu acompanhe a casa ao longo do tempo.


## Requisitos referenciados

- **FR-21**: Lista de meses, do mais recente ao mais antigo, com Entradas, Saídas, Resultado e Saldo acumulado, da primeira Movimentação até o mês atual, incluindo meses vazios.
- **FR-14**: Membros com Gestão editam nome e Saldo inicial, o que recalcula o Saldo acumulado; só o Dono exclui, com confirmação explícita e exclusão lógica; sem restauração na v1.
- **NFR-8**: Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.
- **AR-10**: `src/util/month.ts` com `@date-fns/tz` e teste contra a expressão SQL (AD-8).
- **UX-DR8**: `MoneyValue` em `R$`, com sinal e cor apenas em resultados.
- **UX-DR15**: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.

## Acceptance Criteria

**Given** uma Conta com Movimentações
**When** abro `/contas/:id`
**Then** vejo os meses do mais recente ao mais antigo, com Entradas, Saídas, Resultado do mês e Saldo acumulado, em `MoneyValue`

**Given** um mês sem Movimentação entre a primeira e o mês atual
**When** a lista carrega
**Then** ele aparece com totais zero, e o Saldo acumulado é mantido

**Given** um Saldo inicial
**When** ele é editado, ou uma Movimentação com data passada é criada, alterada ou excluída
**Then** o Saldo acumulado do mês afetado e dos seguintes é recalculado

**Given** uma Conta sem Movimentações
**When** a API responde
**Then** retorna `{ data: [] }`, e a tela mostra o Saldo inicial, "Nenhuma Movimentação ainda." e o botão Lançar

**Given** uma Movimentação às 23h do último dia do mês em `America/Sao_Paulo`
**When** o Balanço é calculado
**Then** ela pertence a esse mês, e um e2e cobre as viradas
**And** um teste compara `src/util/month.ts` com a expressão SQL nessas viradas

**Given** somas acima do limite de `Int`
**When** a query agrega
**Then** os totais ficam corretos, usando `bigint`

**Given** um erro ao carregar
**When** a tela falha
**Then** nenhum total parcial aparece, e há "Tentar de novo"

**Given** um não Membro
**When** chama a rota de Balanço
**Then** recebe 404
