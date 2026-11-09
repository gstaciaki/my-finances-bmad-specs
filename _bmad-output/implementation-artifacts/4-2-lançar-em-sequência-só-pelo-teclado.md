# Story 4.2: Lançar em sequência só pelo teclado

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 4). Em caso de divergência, vale o `epics.md`.

## História

As a Membro,
I want lançar o extrato inteiro sem tocar no mouse,
So that eu termine rápido.


## Requisitos referenciados

- **FR-15**: Membro registra Movimentação com tipo, valor positivo com até 2 casas, data não futura (padrão hoje em São Paulo), descrição e Categoria opcionais; valor acima do limite é rejeitado; formulário fica pronto para o próximo lançamento.
- **UX-DR7**: `MoneyInput` com digitação em reais, conversão para centavos e limite do NFR-3.
- **UX-DR9**: `TypeToggle` com um ponto de Tab, teclas `S`/`E` e setas.
- **UX-DR14**: `EntryForm` em linha: Data, Descrição, Valor, Tipo, Categoria; `Enter` salva; mantém Data e Tipo, limpa o resto e volta o foco à Data; Tipo inicial Saída; sem envio duplicado; lista "Lançadas agora" com contador, editar e excluir.
- **UX-DR15**: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.
- **UX-DR17**: Acessibilidade WCAG 2.2 AA, fluxo de lançamento só com teclado e sobra ou falta comunicadas por texto.
- **UX-DR18**: Datas em `dd/mm/aaaa` em São Paulo; "hoje" por `Intl` nesse fuso; data máxima hoje.
- **UX-DR19**: Microcopy do `EXPERIENCE.md`, incluindo "Sobrou" e "Faltou".

## Acceptance Criteria

**Given** `/contas/:id/lancar`
**When** a tela abre
**Then** o foco está em Data, com hoje em `America/Sao_Paulo` e máximo hoje, e o Tipo começa em Saída
**And** as datas exibidas seguem `dd/mm/aaaa` nesse fuso

**Given** o `EntryForm`
**When** uso Tab
**Then** a ordem é Data, Descrição, Valor, Tipo, com `TypeToggle` em um único ponto de Tab e teclas `S`, `E` e setas

**Given** dados válidos
**When** aperto `Enter` em qualquer campo
**Then** a Movimentação é salva, Data e Tipo são mantidos, o resto é limpo e o foco volta à Data
**And** ela aparece no topo de "Lançadas agora", com o contador atualizado

**Given** um envio em andamento
**When** aperto `Enter` de novo
**Then** nada é enviado em duplicidade

**Given** um valor inválido
**When** tento salvar
**Then** vejo o erro no campo, com o foco nele e nada apagado

**Given** o `MoneyInput`
**When** digito `12,50`
**Then** a API recebe `1250`, e valores acima do limite são rejeitados na tela

**Given** a sessão expirada no meio do lançamento
**When** faço login de novo
**Then** volto à tela com o rascunho preservado
