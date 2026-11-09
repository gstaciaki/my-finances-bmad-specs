# Story 5.2: Abrir um mês e saber se sobrou ou faltou

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 5). Em caso de divergência, vale o `epics.md`.

## História

As a Membro,
I want abrir um mês e ver o resultado e as Movimentações,
So that eu feche as contas com quem divide a casa.


## Requisitos referenciados

- **FR-22**: Abrir mês mostra Balanço e todas as Movimentações, definido pelo calendário de São Paulo.
- **FR-17**: Membro lista Movimentações ordenadas por data, com filtro por período e Categoria, nunca de outra Conta.
- **UX-DR12**: `SummaryCard` para Entradas, Saídas, Resultado e Saldo acumulado.
- **UX-DR15**: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.
- **UX-DR19**: Microcopy do `EXPERIENCE.md`, incluindo "Sobrou" e "Faltou".

## Acceptance Criteria

**Given** um mês da lista
**When** abro `/contas/:id/meses/:mes`
**Then** vejo os `SummaryCard` de Entradas, Saídas, Resultado do mês e Saldo acumulado, e "Sobrou" ou "Faltou" com o valor, por texto e não só por cor

**Given** um mês com Resultado zero
**When** abro
**Then** vejo "Sem sobra nem falta"

**Given** o mês aberto
**When** as Movimentações carregam
**Then** todas as do mês aparecem, com o cliente percorrendo todas as páginas de `GET .../transaction?from=&to=`

**Given** uma Movimentação
**When** a edito ou excluo pela tela Mês
**Then** os números e a lista se atualizam, e ela pode sair do mês se a data mudar

**Given** o filtro por Categoria
**When** escolho uma Categoria ou "Sem categoria"
**Then** só a lista é filtrada, e os totais continuam sendo os do mês inteiro
**And** a API de listagem aceita `categoryId=none` para Movimentações sem Categoria

**Given** um mês sem Movimentação dentro do intervalo
**When** abro
**Then** vejo totais zero e "Nenhuma Movimentação neste mês."

**Given** um mês fora do intervalo ou malformado
**When** abro
**Then** a API responde 404 ou 400, e a tela mostra o erro com link para a Conta
