# My Finances: levantamento do backend e refinamento do objetivo

- **Data:** 2026-11-07
- **Autora:** Mary (Business Analyst, BMAD) para Guilherme
- **Commit analisado:** `c63fa8c` (branch `main`, em dia com o GitHub)
- **Status:** o levantamento (seções 2 a 6) está verificado. O objetivo e o roadmap (seções 7 a 9) já refletem as 6 decisões de Guilherme (seção 10.1). Quatro decisões seguem em aberto, necessárias só a partir da Fase 3 (seção 10.2).

---

## 1. Resumo executivo

**O backend tem uma base técnica sólida, mas ainda não cumpre o objetivo do produto.** Dos 14 requisitos core do Notion, 13 estão marcados como concluídos, porém só 5 foram confirmados integralmente.

Três conclusões sustentam esse diagnóstico:

1. **A engenharia é boa.** A arquitetura em camadas é consistente, 188 testes passam, a cobertura de statements é de 93,66%, e `tsc` e ESLint estão limpos.
2. **O sistema ainda não registra uma saída.** Valores negativos são rejeitados e não existe tipo de transação. Também não existe data da movimentação, só `createdAt`. Sem isso, o resumo mensal (issue #20), que é o coração do objetivo, não pode ser construído.
3. **Não existe isolamento entre usuários.** Qualquer usuário logado lê e altera dados de todos. A listagem de transações mistura contas diferentes, e `GET /user/:id` devolve o hash da senha. Os testes não detectam isso porque, nesses fluxos, usam mocks, e um deles fixa o comportamento errado como esperado.

**Recomendação:** antes de novas funcionalidades, corrigir o isolamento de dados (Fase 0) e ajustar o modelo de transação (Fase 1). Só então construir o resumo mensal (Fase 2). O frontend ainda não existe e deve começar depois da Fase 2.

---

## 2. Fontes e método

| Fonte | Conteúdo | Situação |
|---|---|---|
| Repo `my-finances-backend` | Código, migrations, CI, Docker, 18 commits (2025-05-08 a 2025-12-07) | Lido |
| Notion: My Finances | Objetivo e requisitos (editada em 2026-11-06) | Lido |
| Notion: Escopos > Backend | Stack e arquitetura; seções do Core System vazias | Lido |
| Notion: Banco de Dados: Modelagem | Modelo planejado | Lido |
| Notion: Escopos > Frontend e Infra | Só "Under Construction" | Lido |
| GitHub Issues e PRs | 11 issues (5 abertas, 6 fechadas), nenhum PR aberto | Lido |
| GitHub Project 3 | Board citado no Notion | **Não lido:** token sem escopo `read:project` |
| Repo `my-finances-frontend` | Sem nenhum commit, local e no GitHub | Verificado |

**Como os achados foram verificados:** a API rodou em uma cópia isolada, com Postgres 16 descartável, e cada comportamento foi testado com requisições HTTP reais usando dois usuários. Os achados marcados como **[estático]** vêm só da leitura do código. Nenhum arquivo do repo foi alterado.

---

## 3. O que já existe

### 3.1 Stack

- **Linguagem e runtime:** TypeScript, Node 24
- **API:** Express 5, Zod 4, JWT (`jsonwebtoken`), `bcryptjs`
- **Dados:** PostgreSQL 16, Prisma 6
- **Qualidade:** Jest 30, ESLint, Prettier
- **Ambiente:** Docker Compose, script `./run`, GitHub Actions em PRs para `main`

### 3.2 Arquitetura

O fluxo é `routes → controllers → use-cases → repositories/queries → Prisma`, com `factories` para injeção de dependência.

| Padrão | Onde | Observação |
|---|---|---|
| Caso de uso abstrato | `src/core/use-case.ts` | Valida com Zod e captura erros inesperados |
| Either (`right`/`wrong`) | `src/util/either.ts` | Erros de negócio sem exceções |
| Controller abstrato | `src/core/controller.ts` | Mapeia tipo de erro para status HTTP |
| Repositório base | `src/core/repository.ts` | CRUD genérico e paginação |
| Queries de leitura | `src/queries/account/` | Separa leituras com relações |

### 3.3 Modelo de dados atual

| Tabela | Campos | Regras |
|---|---|---|
| `User` | id, name, email, password, cpf, createdAt, updatedAt | email e cpf únicos |
| `Account` | id, name, createdAt, updatedAt | |
| `UserAccount` | userId, accountId | PK composta; FKs com `ON DELETE RESTRICT` |
| `Transaction` | id, accountId, description?, amount (`Int`), createdAt, updatedAt | FK com `ON DELETE RESTRICT` |

`amount` guarda o valor multiplicado por 10.000 (4 casas decimais). No mesmo dia (2025-11-02) a coluna passou de `Decimal(10,2)` para `BigInt` e depois para `Int`. O motivo não está registrado.

### 3.4 API (19 endpoints)

Todos usam o prefixo `/api`, exceto `/health`.

| Módulo | Método e rota | Autenticação |
|---|---|---|
| Saúde | `GET /health` | Não |
| Sessão | `POST /login`, `POST /refresh` | Não |
| Usuário | `POST /user`, `GET /user`, `GET /user/:id`, `PATCH /user/:id`, `DELETE /user/:id`, `POST /user/change-password` | Sim |
| Conta | `POST /account`, `GET /account`, `GET /account/:id`, `PUT /account/:id`, `DELETE /account/:id` | Sim |
| Transação | `POST`, `GET` em `/account/:accountId/transaction`; `GET`, `PUT`, `DELETE` em `/account/:accountId/transaction/:id` | Sim |

### 3.5 Regras de negócio implementadas

- **Usuário:** valida CPF por dígito verificador e exige e-mail único.
- **Senha:** mínimo de 8 caracteres, com letra maiúscula, número e símbolo; armazenada com bcrypt.
- **Sessão:** access token de 15 minutos e refresh token de 7 dias.
- **Listagens:** paginação com limite de até 100 itens. Usuários e contas filtram por nome (e usuários também por e-mail), sem diferenciar maiúsculas.
- **Transação:** só é acessada dentro da conta à qual pertence, nas rotas por id.

### 3.6 Qualidade (medida)

| Métrica | Resultado |
|---|---|
| Testes | 188 aprovados, 25 suítes |
| Cobertura | Statements 93,66%, branches 78,68%, funções 91,32%, linhas 94,33% |
| `tsc --noEmit` | Sem erros |
| ESLint | Sem erros |
| Testes end-to-end | Nenhum: não há teste do fluxo HTTP completo com banco |

### 3.7 Issues do GitHub

- **Fechadas:** #8 contas, #9 transações, #10 troca de senha, #11 autenticação, #16 seeds e populate, #18 tipos depreciados do Zod.
- **Abertas:** #12 papéis de admin, #13 adicionar e remover usuários de contas, #19 Swagger, #20 resumo mensal, #24 header `Location` nas respostas 201.
- **Abertas a partir deste levantamento (2026-11-07):** #25 listagem por conta (D-01), #26 autorização (D-02), #27 hash da senha (D-03), #28 tipo de transação (D-04), #29 data da movimentação (D-05).

---

## 4. Requisitos do Notion versus realidade

| # | Requisito | Notion | Realidade | Evidência |
|---|---|---|---|---|
| 1 | Registrar movimentações | Feito | ⚠️ Parcial: só valores positivos, sem tipo e sem data | D-04, D-05 |
| 2 | Visualizar movimentações | Feito | ⚠️ Com defeito: mistura contas | D-01 |
| 3 | Editar movimentações | Feito | ✅ Confirmado | |
| 4 | Excluir movimentações | Feito | ✅ Confirmado | |
| 5 | Visão mensal com resumo | Pendente | ❌ Ausente (issue #20) | D-04, D-05 |
| 6 | Movimentação vinculada a conta | Feito | ✅ Confirmado | |
| 7 | Registrar contas | Feito | ✅ Confirmado (ressalvas em D-14) | |
| 8 | Visualizar contas | Feito | ⚠️ Parcial: todos veem todas | D-02 |
| 9 | Editar contas | Feito | ⚠️ Parcial: só o nome | D-07 |
| 10 | Excluir contas | Feito | ❌ Falha com erro 500 quando há usuário ou transação vinculada | D-06 |
| 11 | Conta admin | Feito | ❌ Só um usuário no seed, sem privilégio (issue #12) | D-08 |
| 12 | Conta com N movimentações | Feito | ✅ Confirmado | |
| 13 | N usuários em N contas | Feito | ⚠️ Parcial: o vínculo só ocorre na criação (issue #13) | D-07 |
| 14 | Autenticação JWT | Feito | ⚠️ Parcial: funciona, mas os tokens são intercambiáveis | D-09 |

**Placar:** 5 confirmados, 6 parciais, 3 não atendidos (entre eles, 2 marcados como feitos).

---

## 5. Defeitos e riscos

**Severidade:** Crítico = expõe dados ou impede o objetivo do produto. Alto = fluxo comum falha. Médio = erro limitado ou dívida relevante. Baixo = ajuste pontual.

### Críticos

| ID | Problema | Evidência | Local |
|---|---|---|---|
| D-01 | **A listagem de transações ignora a conta.** `accountId` é removido dos filtros, e o spec fixa `filters: {}` como esperado. | `GET` na conta A retornou `tx-de-A` e `tx-de-B` | `list-transactions.usecase.ts:31,42`; `.spec.ts:158` |
| D-02 | **Não há autorização.** O middleware valida o token, mas `req.user` não é usado em nenhum lugar. | Usuário B leu a conta de A (200), criou transação nela (201), listou todas as contas e excluiu outro usuário (200) | `src/controllers/`, `src/routes/` |
| D-03 | **Hash da senha exposto.** Criar, ver, editar e excluir usuário devolvem a entidade completa. CPFs também são visíveis para qualquer usuário logado. | `GET /user/:id` retornou `password` com hash bcrypt; `POST /user` também | `show-user.usecase.ts:31`, `create-user.usecase.ts:33`, `update-user.usecase.ts:45`, `delete-user.usecase.ts:34` |
| D-04 | **Não é possível registrar saída.** O regex rejeita negativos, não existe campo de tipo, e a mensagem de erro fala em casas decimais. | `amount: -50` retornou 400 com "no máximo 4 casas decimais" | `src/util/zod/currency.ts:9` |
| D-05 | **Não existe data da movimentação.** Só há `createdAt`, definido pelo servidor, então não dá para lançar uma despesa antiga. O filtro de data aceita só "a partir de" **[estático]**. | `createdAt` de 2020 enviado; gravada a data atual | `schema.prisma`, `parse-filters.ts:14` |

### Altos

| ID | Problema | Evidência | Local |
|---|---|---|---|
| D-06 | **Excluir conta ou usuário vinculado retorna 500.** As FKs são `RESTRICT` e o erro não é tratado. | Conta com vínculo: 500 (P2003). Conta sem vínculo: 200. Usuário vinculado: 500 | Migrations; `delete-account.usecase.ts`, `delete-user.usecase.ts` |
| D-07 | **Membros da conta não podem ser gerenciados.** Só o nome é editável, e o criador não é vinculado automaticamente. | `UpdateAccountSchema` só tem `name`; `usersIds: []` foi aceito | `use-cases/account/dtos.ts` (issue #13) |
| D-08 | **Admin não existe.** Não há campo ou papel no banco, o token sempre sai com `admin: false`, e nada lê esse valor. O CPF do seed é o texto literal `XXX.XXX.XXX-XX`. | Payload com `admin:false`; busca sem usos | `login.usecase.ts:37`, `user.seed.ts:14` (issue #12) |
| D-09 | **Access e refresh token são intercambiáveis.** O refresh dá acesso à API por 7 dias, e um access token renova a sessão em `/refresh`. Não há revogação nem logout. | Refresh usado como access: 200. Access em `/refresh`: 200 | `util/jwt.ts:17`, `refresh-token.usecase.ts:29` |
| D-10 | **Sem cadastro público.** `POST /user` exige token, então o primeiro acesso depende do seed. Pode ser intencional (decisão D3). | Sem token: 401 | `user.routes.ts:8` |
| D-11 | **Limite de valor por transação.** `Int` com 4 casas limita a R$ 214.748,3647, e acima disso a resposta é 500. | `300000`: 500. `214748.3647`: 201 | `schema.prisma:53`, `currency.ts:3` |

### Médios

| ID | Problema | Evidência | Local |
|---|---|---|---|
| D-12 | **CPF duplicado.** Sem máscara retorna 500, e o mesmo CPF com máscara é aceito como novo. Deixa de existir com a remoção do CPF (decisão D4). | 500 e 201 | `create-user.usecase.ts`, `dtos.ts` |
| D-13 | **Editar usuário com o próprio e-mail falha.** | `PATCH` retornou 409 | `update-user.usecase.ts:31` |
| D-14 | **Criação de conta não é atômica.** Uma falha ao vincular deixa a conta órfã. **[estático]** | Sem transação do banco | `create-account.usecase.ts:42` |
| D-15 | **Paginação sem ordenação.** A ordem não é garantida entre páginas. **[estático]** | `findMany` sem `orderBy` | `core/repository.ts:39` |
| D-16 | **Sem índices de consulta.** `Transaction.accountId` e datas não têm índice, o que afetará o resumo mensal. **[estático]** | Nenhum `@@index` | `schema.prisma` |
| D-17 | **Sem caminho de produção.** Não há scripts `build` e `start`, o Dockerfile roda `pnpm dev`, e o seed importa código de `test/`. **[estático]** | | `package.json`, `Dockerfile` |
| D-18 | **Endurecimento de segurança ausente.** CORS aberto, sem rate limit no login e sem headers de segurança. O `.env.example` traz um `JWT_SECRET` com valor. **[estático]** | | `server.ts:8`, `.env.example` |
| D-19 | **CI usa outro gerenciador.** O workflow usa `yarn --frozen-lockfile`, mas o repo só tem `pnpm-lock.yaml`. A execução real do CI não foi verificada. | | `.github/workflows/ci.yml:46` |

### Baixos

| ID | Problema | Local |
|---|---|---|
| D-20 | A regra de senha rejeita `-` e aceita `,`, ao contrário da lista de símbolos (`join()` usa vírgula e forma um intervalo no regex). Verificado: 400 e 201. | `util/password.ts:16` |
| D-21 | Erro de conta inexistente diz "usuário". | `delete-account.usecase.ts:27`, `update-account.usecase.ts:28` |
| D-22 | Sem documentação da API (issue #19) e sem header `Location` (issue #24). | |
| D-23 | Grafias: `ITransacationRepository`, `unknow.error.ts`, `Continuos Integration`. | |

### Por que os testes não pegaram isso

Os testes dos casos de uso usam repositórios com mock, e só 3 suítes de repositório tocam o banco. Nenhum teste percorre o caminho HTTP completo com dois usuários. A cobertura alta mede linhas executadas, não as regras de acesso.

---

## 6. Divergências entre Notion e código

| Tema | Notion | Código |
|---|---|---|
| `amount` | `Decimal(10,2)` no schema, `int` na tabela | `Int` com 4 casas |
| `Transaction.description` | Obrigatório | Opcional |
| `Transaction.updatedAt` | Na tabela, ausente no schema | Existe |
| `UserAccount.id` | Coluna `id` uuid na tabela | PK composta |
| `updatedAt` | `@default(now())` | `@updatedAt` |
| `Sessions` e `Roles` | Tabelas previstas, sem definição | Não existem |
| Core System (Entity, Repository, Use Case, Controller) | Seções vazias | Implementado |
| Repo do Frontend | Campo vazio | Repo existe, sem commits |
| Motivação | "Ambiente completo" | README: projeto pessoal de estudos |

**Atualização de 2026-11-07:** a modelagem do Notion agora descreve o modelo-alvo das seções 8 e 10, e a tabela acima registra as divergências encontradas antes dessa atualização. `Sessions` e `Roles` continuam sem definição.

**Ambiguidade de termo:** "conta" é usada em dois sentidos. Em "Deve existir uma conta admin" significa login. Em "Registrar contas" significa conta financeira. O glossário abaixo resolve isso.

| Termo proposto | Significado | Hoje no código |
|---|---|---|
| Usuário | Pessoa que faz login | `User` |
| Conta | Agrupamento financeiro (banco, carteira, conta da casa) | `Account` |
| Membro | Usuário com acesso a uma conta | `UserAccount` |
| Movimentação | Entrada ou saída de dinheiro | `Transaction` |

---

## 7. Objetivo refinado (proposta)

**Objetivo atual (Notion):** desenvolver um ambiente completo para um sistema de finanças, para registrar, editar e monitorar gastos de uma conta, com visão mensal de entradas, saídas, totais e saldo final.

**Objetivo proposto:**

> O My Finances permite que uma família registre entradas e saídas em contas compartilhadas e saiba, em poucos segundos, quanto entrou, quanto saiu, qual foi o saldo do mês e qual é o saldo acumulado, com a garantia de que só os membros de uma conta veem seus dados.

**Natureza do projeto:** o foco é o estudo do ecossistema completo de uma aplicação, e a meta final é colocar o sistema em produção. Por isso, quando houver escolha entre caminhos, vale preferir o que ensina mais, desde que não comprometa a segurança dos dados reais da família.

### Critérios de sucesso

| # | Critério | Medida |
|---|---|---|
| 1 | Isolamento | Um usuário não membro recebe 403 ou 404 em toda rota de conta, comprovado por testes end-to-end |
| 2 | Registro completo | Entrada e saída com data escolhida pelo usuário |
| 3 | Resumo correto | Entradas, saídas, saldo do mês e saldo acumulado batem com a soma das movimentações, inclusive na virada do mês |
| 4 | Integridade | Nenhum erro de regra de negócio conhecido retorna 500 |
| 5 | Em produção | Sistema publicado com HTTPS, backup testado e deploy reproduzível |
| 6 | Uso real | A família usa o sistema em produção, com dados reais, por um mês completo |

### Fora do escopo inicial (proposta)

Abertura para o público geral, integração bancária (Open Finance), investimentos, múltiplas moedas e aplicativo mobile nativo.

---

## 8. Requisitos propostos

### Must have

- **RF-01** Movimentação com tipo (entrada ou saída), valor positivo, descrição e data.
- **RF-02** Listagem restrita à conta informada, ordenada por data e filtrável por período.
- **RF-03** Resumo mensal por conta: entradas, saídas, saldo do mês, saldo acumulado e lista de movimentações.
- **RF-04** Toda operação em conta exige que o usuário seja membro.
- **RF-05** Quem cria a conta se torna membro automaticamente.
- **RF-06** Adicionar e remover membros, mantendo ao menos um por conta.
- **RF-07** Excluir conta com regra definida e resposta clara.
- **RF-08** Usuário só altera os próprios dados, exceto o admin.
- **RF-11** Conta com saldo inicial, usado no cálculo do saldo acumulado.
- **RNF-01** Nenhuma resposta da API contém senha ou hash.
- **RNF-02** Tokens de acesso e de refresh têm finalidades separadas e validadas.
- **RNF-03** Testes end-to-end cobrem o isolamento entre usuários.
- **RNF-05** Build e deploy de produção reproduzíveis.
- **RNF-06** Produção com HTTPS, segredos fora do repositório, rate limit no login e backup do banco com restauração testada.
- **RNF-07** O sistema não coleta nem armazena CPF.

### Should have

- **RF-09** Categorias de movimentação e resumo por categoria.
- **RF-10** Papel de admin com permissões definidas.
- **RF-12** Logout e revogação de refresh token.
- **RNF-04** Documentação OpenAPI.

### Could have

- **RF-13** Movimentações recorrentes e parcelas.
- **RF-14** Transferência entre contas.
- **RF-15** Orçamento mensal por categoria.
- **RF-16** Exportação em CSV.

### Modelo de transação proposto (a validar com o Architect)

```prisma
enum TransactionType {
  INCOME
  EXPENSE
}

model Transaction {
  id          String          @id @default(uuid()) @db.Uuid
  accountId   String          @db.Uuid
  type        TransactionType
  amount      Int             // sempre positivo, em centavos (decisão D5)
  description String?
  occurredAt  DateTime        @db.Date // data sem hora, informada pelo usuário (decisão D7)
  createdAt   DateTime        @default(now())
  updatedAt   DateTime        @updatedAt

  account     Account         @relation(fields: [accountId], references: [id])

  @@index([accountId, occurredAt])
}
```

**Efeitos da decisão por 2 casas decimais:**

- Com `Int` em centavos, o limite por movimentação sobe de R$ 214.748,36 para R$ 21.474.836,47, o que resolve D-11 para uso familiar. Valores acima disso devem retornar erro de validação, não 500.
- `DECIMAL_PLACES_LIMIT` passa de 4 para 2, e os specs que dependem dele precisam ser revistos.
- Dados já gravados precisam ser divididos por 100 na migration. Bancos locais com dados de teste podem simplesmente ser recriados.

**Efeito da decisão de mostrar os dois saldos:** `Account` ganha o campo `openingBalance` (`Int`, em centavos, padrão 0). O saldo acumulado no fim de um mês é o saldo inicial, mais todas as entradas, menos todas as saídas até aquela data. O saldo inicial pode ser negativo, por exemplo em conta com cheque especial.

**Efeitos da decisão de fuso:** como `occurredAt` é uma data sem hora, o filtro de mês compara datas do calendário, sem conversão de fuso. `America/Sao_Paulo` só é usado quando o sistema precisa definir "hoje", como no valor padrão de uma nova movimentação ou no mês atual do resumo. Os testes devem cobrir o último dia do mês à noite, quando UTC já está no mês seguinte.

**Efeitos da remoção do CPF:** sai a coluna `User.cpf`, por migration, junto com `src/entities/value-objects/cpf.ts`, a validação em `use-cases/user/dtos.ts`, o campo em `UserMapper`, o seed e `test/prefab/cpf.ts`. O e-mail continua como identificador único do usuário.

---

## 9. Roadmap proposto

| Fase | Objetivo | Itens | Issues | Saída |
|---|---|---|---|---|
| **0. Proteger os dados** | Fechar a exposição | D-01, D-02, D-03, D-09, D-12, RF-04, RF-05, RF-08, RNF-07, testes end-to-end | #25, #26, #27 | Critério 1 atendido |
| **1. Modelo financeiro** | Registrar a realidade | D-04, D-05, D-11, D-15, D-16, RF-01, RF-02, RF-11 | #28, #29 | Entrada e saída com data, em centavos |
| **2. Resumo mensal** | Entregar o valor central | RF-03 | #20 | Critério 3 atendido |
| **3. Gestão de contas** | Fechar o compartilhamento familiar | D-06, D-07, D-13, D-14, RF-06, RF-07 | #13 | Critério 4 atendido |
| **4. Preparar produção** | Publicar a API com segurança | D-10, D-17, D-18, D-19, RF-10, RF-12, RNF-04, RNF-05, RNF-06 | #12, #19, #24 | Critério 5 atendido |
| **5. Frontend** | Usar no dia a dia | Telas de login, contas, movimentações e mês | Criar novas | Critério 6 atendido |
| **6. Evolução** | Ampliar | RF-09, RF-13 a RF-16 | Criar novas | Conforme o uso |

Os defeitos D-20 a D-23 podem entrar em qualquer fase. A Fase 5 pode começar em paralelo à Fase 3, assim que o contrato da API estiver estável.

A Fase 0 é pré-requisito para qualquer dado real da família entrar no sistema. Como a meta é produção, as Fases 0 e 4 deixam de ser opcionais. As Fases 4 e 5 também são boas oportunidades de estudo: infraestrutura, deploy, observabilidade e frontend ainda estão vazios no Notion.

---

## 10. Decisões em aberto

### 10.1 Decididas por Guilherme (2026-11-07)

| # | Decisão | Resposta | Consequência |
|---|---|---|---|
| D1 | Quem usa? | Família | Contas compartilhadas são centrais; público geral fica fora do escopo |
| D2 | Estudo ou uso real? | Estudo, com meta final de subir em produção | Deploy e segurança viram requisitos obrigatórios (RNF-05, RNF-06) |
| D5 | Precisão de valores | 2 casas decimais, em centavos | `Int` em centavos; migration de valores; D-11 resolvido |
| D6 | "Saldo final" é do mês ou acumulado? | Mostrar os dois | RF-11 sobe para Must have; `Account` ganha saldo inicial |
| D4 | O CPF é necessário? | Não, remover | RNF-07 entra na Fase 0; D-12 deixa de existir |
| D7 | Qual fuso define o mês? | `America/Sao_Paulo` | `occurredAt` vira data sem hora; o fuso define "hoje" |

### 10.2 Ainda em aberto

| # | Decisão | Recomendação | Afeta | Necessária antes da |
|---|---|---|---|---|
| D3 | Cadastro público ou por convite? | Por convite, coerente com o uso familiar | D-10 | Fase 4 |
| D8 | Excluir conta com movimentações: bloquear, apagar tudo ou arquivar? | Arquivar (exclusão lógica) | RF-07 | Fase 3 |
| D9 | Todos os membros têm o mesmo poder na conta? | Sim no início; dono e membro depois | RF-06 | Fase 3 |
| D10 | O que o admin pode fazer? | Gerenciar usuários, sem ler dados financeiros | RF-10 | Fase 4 |

---

## 11. Limitações desta análise

- O GitHub Project 3 não foi lido, porque o token não tem o escopo `read:project`. O board pode conter itens que não aparecem aqui.
- A execução real do CI no GitHub Actions não foi verificada.
- Não há evidência de ambiente de produção. O impacto dos defeitos críticos depende de o sistema estar exposto.
- Pesquisa de mercado e de concorrentes não fez parte desta análise.
- O frontend não foi analisado porque não tem código.

---

## 12. Próximos passos

1. Responder às decisões da seção 10.2 antes de iniciar as Fases 3 e 4. Elas não bloqueiam o PRD.
2. Gerar o PRD com `bmad-prd`, usando este documento como entrada.
3. Definir as mudanças de modelo e autorização com `bmad-architecture`.
