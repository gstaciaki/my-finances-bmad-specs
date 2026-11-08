---
name: My Finances
status: final
created: '2026-11-08'
updated: '2026-11-08'
sources:
  - _bmad-output/planning-artifacts/prds/prd-workspace-2026-11-07/prd.md
  - _bmad-output/planning-artifacts/architecture/architecture-workspace-2026-11-08/ARCHITECTURE-SPINE.md
---

# My Finances: Experience Spine

## Foundation

Aplicação web de página única, só para navegador de computador. Vue 3 com Tailwind e componentes `Base*` próprios, sem biblioteca de UI. `DESIGN.md` define a aparência; este arquivo define o comportamento. Termos seguem o Glossário do PRD.

## Information Architecture

| Tela | Rota | Chega por | Finalidade | Requisitos |
| --- | --- | --- | --- | --- |
| Login | `/login` | Sem sessão | Entrar | FR-1 |
| Esqueci a senha | `/senha/esqueci` | Login | Pedir link | FR-2 |
| Nova senha | `/senha/nova` | Link do e-mail | Definir senha | FR-2 |
| Minhas Contas | `/` | Após login, logo | Contas de que participa, Convites pendentes, criar Conta | FR-7, FR-8, FR-10 |
| Conta (meses) | `/contas/:id` | Minhas Contas | Lista de meses e ações da Conta | FR-21 |
| Lançar | `/contas/:id/lancar` | Botão na Conta | Registrar em sequência | FR-15, FR-20 |
| Mês | `/contas/:id/meses/:mes` | Linha do mês | Balanço e Movimentações do mês | FR-16, FR-17, FR-22, FR-23 |
| Categorias | `/contas/:id/categorias` | Ajustes da Conta | Gerenciar Categorias | FR-19 |
| Membros | `/contas/:id/membros` | Ajustes da Conta | Membros, Convites, Permissões, titularidade, sair | FR-9, FR-11 a FR-13, FR-24 |
| Ajustes da Conta | `/contas/:id/ajustes` | Conta | Nome, Saldo inicial e exclusão | FR-14 |
| Perfil | `/perfil` | Menu do usuário | Nome, e-mail e senha | FR-3 |
| Admin: Usuários | `/admin/usuarios` | Menu do usuário (Admin) | Criar, editar, redefinir senha, excluir | FR-4, FR-5 |
| Admin: Contas | `/admin/contas` | Menu do usuário (Admin) | Visão total, somente leitura, e transferir titularidade | FR-6 |

A barra superior traz logo, Conta atual e menu do usuário. Diálogos têm um nível só. Ações sem Permissão ficam ocultas, e a API continua sendo quem garante a regra.

## Voice and Tone

| Do | Don't |
| --- | --- |
| "Sobrou R$ 320,00" / "Faltou R$ 85,10" | "Resultado: -85.1" |
| "E-mail ou senha incorretos." | "Usuário não encontrado." |
| "Movimentação salva." | "Operação realizada com sucesso!" |
| "Excluir esta Conta? Ninguém mais terá acesso." | "Tem certeza?" |

Tratamento direto, em português, com os termos do Glossário.

## Component Patterns

| Componente | Uso | Comportamento |
| --- | --- | --- |
| `EntryForm` | Lançar | Linha única: Data, Descrição, Valor, `TypeToggle`, Categoria. Regras em *Lançamento por teclado* |
| `MoneyInput` | Valor, Saldo inicial | Digitação livre em reais (`12,50`), convertida para centavos só ao enviar; rejeita valor acima do limite do NFR-3 |
| `TypeToggle` | `EntryForm`, edição | Um único ponto de Tab; `S` ou `E` e as setas trocam o tipo |
| `BaseSelect` | Categoria, Permissão | Nativo, com busca por letra digitada; primeira opção "Sem categoria" |
| `BaseTable` | Listas | Linha clicável abre o destino; paginação abaixo, sem rolagem infinita |
| `AppDialog` | Edição de Movimentação, criar Conta, Convite | `Esc` fecha; foco vai ao primeiro campo e volta à origem ao fechar |
| `ConfirmDialog` | Toda exclusão, transferência, sair da Conta | Texto nomeia o alvo; foco inicial em Cancelar |
| `AppToast` | Confirmações | Some em 4 segundos, sem bloquear digitação |
| `SummaryCard` | Mês | Entradas, Saídas, Resultado do mês, Saldo acumulado |
| `MoneyValue` | Todos os valores | Formata centavos em `R$`, com sinal quando for resultado |
| `AppHeader` | Global | Mostra Admin só para Admin; indicador de Convites pendentes |

## State Patterns

| Estado | Onde | Tratamento |
| --- | --- | --- |
| Abrindo o app | Global | Tenta renovar a sessão e mostra indicador central; sem sessão, vai ao Login |
| Carregando | Listas | Linhas de esqueleto; ações continuam desabilitadas |
| Sem Contas | Minhas Contas | "Você ainda não participa de nenhuma Conta." com botão Criar Conta |
| Sem Movimentações | Conta | Mostra o Saldo inicial, "Nenhuma Movimentação ainda." e botão Lançar |
| Mês sem Movimentação | Conta, Mês | Linha com totais zero; Mês mostra "Nenhuma Movimentação neste mês." |
| Sem Categorias | Lançar, Categorias | `BaseSelect` só com "Sem categoria"; quem tem Gestão vê link para criar |
| Erro de validação | Formulários | Mensagem no campo, foco no primeiro inválido e valores preservados |
| Erro de rede ou 5xx | Global | Toast "Não foi possível concluir. Tente de novo."; o formulário mantém os dados |
| Sessão expirada | Global | Ao falhar a renovação, vai ao Login com a rota de volta; o rascunho do `EntryForm` permanece em memória |
| Conta inexistente ou sem acesso (404) | Conta | "Conta não encontrada." com link para Minhas Contas |
| Sem Permissão (403) | Ações | Toast explicativo e recarrega a Permissão, pois ela pode ter mudado |
| Convite com falha de e-mail | Membros | Aviso "Convite criado, mas o e-mail não foi enviado." com Reenviar |
| Conta excluída | Admin: Contas | Etiqueta "Excluída" |

## Interaction Primitives

**Lançamento por teclado** (FR-15):

- A tela abre com foco em Data, preenchida com hoje em `America/Sao_Paulo`.
- A ordem de Tab segue o extrato: Data, Descrição, Valor, Tipo, Categoria.
- `Enter` em qualquer campo salva. Não existe botão a alcançar, embora o botão Salvar exista para mouse.
- Depois de salvar, o mesmo formulário continua na tela, sem abrir outro: Data e Tipo são mantidos, os demais campos são limpos e o foco volta para Data.
- Tipo começa em Saída.
- A Movimentação salva aparece no topo de "Lançadas agora", abaixo do formulário, com contador e opção de editar ou excluir.
- Em erro, nada é limpo e o foco vai ao campo inválido. Durante o envio, `Enter` não salva de novo.

**Tema:** segue o sistema na primeira visita; o menu do usuário alterna entre claro e escuro, e a escolha fica no navegador.

**Datas:** sempre exibidas como `dd/mm/aaaa` em `America/Sao_Paulo`; a data máxima é hoje.

**Banido:** rolagem infinita, arrastar, ações disponíveis só ao passar o mouse, diálogo sobre diálogo.

## Accessibility Floor

- WCAG 2.2 AA nas telas da v1.
- Todo o fluxo de lançamento funciona só com teclado, e a ordem de Tab segue a ordem visual.
- Campos têm `<label>`; erros usam `aria-describedby` e `aria-invalid`.
- `AppToast` usa `aria-live="polite"`.
- Sobra e falta são comunicadas por texto, não só por cor.

## Key Flows

### UJ-1. Alex lança os gastos da semana

1. Alex abre o extrato do banco e entra no My Finances; se a sessão expirou, faz login.
2. Em Minhas Contas, abre a Conta da casa e clica em Lançar.
3. Digita data, descrição e valor do primeiro gasto, usando Tab entre campos, escolhe a Categoria e aperta Enter.
4. **Clímax:** o gasto aparece em "Lançadas agora", o foco volta para Data, e Alex segue para o próximo sem tocar no mouse.
5. Repete até o fim do extrato e confere o contador com o número de linhas.

**Falha:** um valor inválido mostra o erro no campo, sem apagar nada. Se a sessão cair, depois do login Alex volta à tela com o que estava digitando.

### UJ-2. Alex e Sam fecham o mês

1. Alex e Sam abrem a Conta da casa e veem os meses, do mais recente ao mais antigo, com Entradas, Saídas, Resultado do mês e Saldo acumulado.
2. Abrem o mês que acabou.
3. **Clímax:** o topo do Mês diz "Sobrou" ou "Faltou" com o valor, ao lado do Saldo acumulado.
4. Veem os totais por Categoria, separando Entradas e Saídas, e as Movimentações do mês.
5. Corrigem um lançamento errado pelo diálogo de edição, e os números se atualizam.

**Falha:** se a lista não carregar, os totais não são mostrados parcialmente; aparece o erro com "Tentar de novo".

### Sam aceita o Convite

1. Sam faz login e vê em Minhas Contas o Convite pendente, com nome da Conta e Permissão.
2. **Clímax:** ao aceitar, a Conta entra na lista na hora.

**Falha:** Convite vencido mostra "Este Convite expirou. Peça um novo ao Dono."

## Necessidades da API

Descobertas no desenho das telas, para entrarem nas histórias:

| Necessidade | Motivo |
| --- | --- |
| `GET /me` retorna `id`, `name`, `email`, `isAdmin` | Ao abrir o app, só há o token renovado; o menu Admin depende disso |
| `GET /account` e `GET /account/:id` retornam `isOwner` e `permission` do Usuário atual | Ocultar ações sem Permissão |
| `GET /account/:id/category` disponível a qualquer Membro | Quem tem só Lançamento escolhe Categorias |
| `GET /account/:id/member` disponível a qualquer Membro, com nome | Tela Membros |
| `GET /invitation` retorna nome da Conta e Permissão | Mostrar Convite ao convidado, que não é Membro |
| Listas de Movimentação retornam `categoryId` e `categoryName` | Exibir sem consulta extra |
