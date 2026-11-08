---
title: Revisão adversarial do Architecture Spine, My Finances
target: ../ARCHITECTURE-SPINE.md
contra: ../../../prds/prd-workspace-2026-11-07/prd.md
date: 2026-11-08
método: construir duas unidades independentes que obedecem a todos os ADs ao pé da letra e ainda assim são incompatíveis
---

# Revisão adversarial: Architecture Spine do My Finances

## Veredito

**Reprovado para a quebra em histórias, com correção barata.** O spine é forte em camadas, auditoria, dinheiro e isolamento por parâmetro. Os buracos se concentram em quatro lugares que ele deixa implícitos: o **ciclo de vida do Convite**, o **contrato de leitura do Balanço e das datas**, **quem é o dono de cada transição concorrente** e a **fronteira entre Admin, leitura por Usuário e dados por Conta**.

Foram encontrados 14 pares incompatíveis, além de 7 itens menores: 1 crítico, 7 altos, 6 médios e 7 baixos. Nenhum exige mudar o paradigma. Todos se fecham com ADs novos ou com frases a mais em ADs existentes, e a redação proposta está em cada achado.

| # | Severidade | Buraco | PRD atingido | AD a mudar |
| --- | --- | --- | --- | --- |
| 1 | Crítica | Convite expirado continua no índice único, e o novo Convite dá 409 | FR-9, FR-10 | AD-16, novo AD-21 |
| 2 | Alta | Categoria de outra Conta aceita pela FK simples | NFR-1, FR-19, FR-23 | AD-3 |
| 3 | Alta | "Mês" e "hoje" calculados em lugares diferentes | FR-15, FR-17, FR-22 | AD-8 |
| 4 | Alta | Balanço e mês sem contrato; limite de 100 contra "todas" | FR-21, FR-22, FR-23 | AD-14, Listas |
| 5 | Alta | Rotação do refresh derruba a Sessão com duas abas | FR-1, UJ-1, UJ-2 | AD-5, AD-6 |
| 6 | Alta | Auditoria sai como `system` sem nenhum teste falhar | FR-18 | AD-10, AD-12 |
| 7 | Alta | Conta excluída bloqueia excluir Usuário e não se distingue de Conta desativada | FR-5, FR-6 | AD-9 |
| 8 | Alta | Transições de Membro sem exclusão mútua | FR-5, FR-10, FR-11, FR-13 | novo AD-22 |
| 9 | Média | AD-3 torna FR-10 e AD-9 impossíveis de escrever sem violá-lo | FR-10, FR-5 | AD-3 |
| 10 | Média | Admin tem duas superfícies de leitura | FR-6, FR-8 | AD-2, Rotas |
| 11 | Média | E-mail normalizado só no limite de login | FR-1, FR-2, FR-4, FR-9 | novo AD-23 |
| 12 | Média | AD-15 faz a resposta de FR-2 revelar a existência do e-mail | FR-2, FR-9 | AD-15 |
| 13 | Média | Erros de Prisma não mapeados e transação longa em linha a linha | NFR-4, FR-19 | AD-10, AD-17 |
| 14 | Média | Paginação instável, porque as datas empatam em 00:00 | FR-17, SM-2 | Listas |

Base factual: li o spine, o `.memlog.md`, o PRD, o addendum, trechos do levantamento e o código em `my-finances-backend` (`schema.prisma`, `core/use-case.ts`, `core/repository.ts`, `util/jwt.ts`, `routes/index.ts`). Nada foi executado. Comportamentos de PostgreSQL e Prisma citados são conhecimento da ferramenta, e o achado que depende de inferência diz isso.

---

## Crítica

### 1. Convite expirado segue ocupando o índice único

**Unidade A, "Enviar Convite" (FR-9).** Cria a linha com `status = PENDING` e `expiresAt = now + 7d`. Conforme AD-16, cria o índice único parcial `(accountId, inviteeUserId) WHERE status = 'PENDING'`.

**Unidade B, "Aceitar ou recusar" (FR-10).** Conforme AD-16, compara `expiresAt` no momento do uso, sem job, e responde "expirado". Não muda `status`, pois nenhum AD manda fazê-lo.

**Colisão.** O predicado de índice parcial em PostgreSQL só aceita expressão imutável, então `expiresAt > now()` não pode entrar nele. O Convite vencido continua `PENDING` no índice para sempre. O Dono tenta novo Convite, recebe `P2002` e AD-17 devolve 409.

**Defeito no PRD.** FR-10 diz que, após recusa ou expiração, "o Dono pode enviar um novo Convite". Ele nunca conseguirá. É o caminho feliz de uma regra explícita, e o spine o bloqueia por construção.

**Mais quatro pares na mesma entidade, todos sem regra:**

- **Reenvio contra unicidade.** AD-15 diz que o Dono "pode reenviar". Uma história implementa reenvio como `POST` novo, e o índice devolve 409. Outra cria rota `/resend`. O spine não diz qual, e a tabela de rotas não tem nenhuma das duas.
- **Convite por link contra Convite na tela.** AD-16 lista o Convite entre os itens com "tokens de link" guardados como hash. FR-10 diz que o convidado vê os Convites "ao entrar no sistema". Uma história grava `tokenHash` e faz `/invitation/:token` sem login; a outra lista por `inviteeUserId` e nunca usa token. Reenviar com só o hash guardado obriga a girar o token, o que invalida o e-mail anterior.
- **Aceite em Conta excluída.** FR-14 só grava `deletedAt` na Conta. O aceite só compara `expiresAt`. Resultado: nova linha em `UserAccount` de Conta excluída. O mesmo vale para quem já virou Membro por outro caminho.
- **AD-2 contra o convidado.** O convidado não é Membro, e AD-2 manda não Membro receber 404 em "todo use case de dado de Conta". Lido ao pé da letra, o aceite é impossível.

**Fechamento: tornar `AD-16` específico e criar o AD-21.**

> **AD-16 (acrescentar):** O predicado do índice parcial de Convite é `WHERE status = 'PENDING'`. Como o predicado não pode usar `now()`, `status` é materializado nas transições descritas no AD-21. Convite não tem token de link: identifica-se por `inviteeUserId` e é acessado depois do login. Só o link de recuperação de senha usa token.
>
> **AD-21: Ciclo de vida do Convite.** `Invitation` tem `status` em `PENDING | ACCEPTED | DECLINED | EXPIRED | SUPERSEDED`, com `inviteeUserId` obrigatório, sem e-mail copiado. `status` só muda nos use cases de `use-cases/invitation`. Enviar Convite, em um único Unit of Work, marca como `EXPIRED` o `PENDING` vencido do par (Conta, convidado) e só então cria o novo. Reenviar é `POST /api/account/:accountId/invitation/:id/resend`, permitido a Convite `PENDING` não vencido, e reenvia o e-mail sem alterar `expiresAt`. Aceitar e recusar são `POST /api/invitation/:id/accept` e `/decline`, com autorização própria: o Convite precisa ter `inviteeUserId = actor.userId`, do contrário 404. Aceitar, no mesmo Unit of Work e sob o bloqueio do AD-22, exige `PENDING` não vencido, Conta sem `deletedAt` e convidado ainda não Membro. Aceite vencido grava `EXPIRED` e retorna erro de regra. Excluir Conta marca seus Convites `PENDING` como `SUPERSEDED`. A leitura de Convites pendentes filtra também por `expiresAt > now()`.

---

## Altas

### 2. Categoria de outra Conta passa pela FK

**Unidade A, "Registrar Movimentação" (FR-15, FR-20).** Recebe `categoryId` do corpo e grava a FK escalar, como manda AD-11. O repositório de Movimentação recebe `accountId` obrigatório, como manda AD-3. O `categoryId` é só mais um valor do `data`.

**Unidade B, "Balanço por Categoria" (FR-23).** Agrega por `categoryId` com `JOIN Category`, filtrando `Transaction.accountId`, como manda AD-3.

**Colisão.** AD-3 protege a **busca** por `{ id, accountId }`, não os **valores de FK** gravados. `Transaction.categoryId → Category.id` aceita qualquer UUID válido. Um Membro da Conta A informa a Categoria da Conta B: grava com sucesso, e o Balanço da Conta A exibe o nome de uma Categoria da Conta B.

**Defeito no PRD.** NFR-1 ("isso vale para toda a API") e FR-19 (Categoria por Conta). Também é o tipo exato de falha da issue #25, que AD-3 existe para impedir. Pior: excluir a Categoria na Conta B deixa de limpar corretamente, porque o `Restrict` passa a depender de Movimentações de outra Conta. Por fim, um `PATCH` com `accountId` no corpo mudaria a Movimentação de Conta.

**Fechamento (AD-3, acrescentar):**

> Referências entre dados de Conta são garantidas no banco por FK composta. `Category` tem `@@unique([accountId, id])`, e `Transaction` referencia `(accountId, categoryId) → Category(accountId, id)`, opcional, `onDelete: Restrict`. `accountId` vem só do caminho, nunca do corpo, e é imutável após a criação. Violação vira erro de regra de validação (ver AD-17), não 500.

A FK composta é preferível a um `findCategory` no use case, porque cobre qualquer história futura, incluindo a integração bancária.

### 3. "Mês" e "hoje" têm mais de um dono

Quatro unidades obedecem a AD-8 e divergem. AD-8 fala de "uma função única compartilhada", mas esse objeto não pode existir como escrito.

**Par 3a: SQL contra TypeScript.** AD-14 exige agregação SQL, então o agrupamento é `date_trunc('month', "occurredAt" AT TIME ZONE 'America/Sao_Paulo')`. O filtro de mês em FR-22 usa a função TypeScript de AD-8, com limites calculados em JS. São duas implementações do mesmo conceito, e a "função única" não roda dentro do `GROUP BY`. Basta uma delas usar offset fixo ou `<=` no fim do mês para o Balanço e a lista do mês divergirem.

**Par 3b: "hoje" no navegador.** O frontend preenche a data padrão com `new Date().toISOString().slice(0, 10)`. Não viola AD-8, que só governa o mês. Às 23h de 31/10 em São Paulo, isso produz `2026-11-01`, gravado como 00:00 de novembro. É **literalmente o caso que FR-22 proíbe**, e o teste e2e do backend passa, porque nele a data vem pronta. FR-15 fixa o padrão no fuso de São Paulo, e o spine não diz quem o calcula.

**Par 3c: filtro de período.** FR-17 filtra por período. Uma história define `to=YYYY-MM-DD` como `occurredAt <= 00:00` daquele dia em São Paulo. AD-8 permite entrada com hora ("Integração bancária: `occurredAt` já aceita hora"), então Movimentação às 15h do último dia sai do filtro. Outra história usa fim de dia. Nenhum formato de filtro é definido.

**Par 3d: exibição perdida na destilação.** O memlog registra a decisão "frontend exibe sempre nesse fuso". O spine não a carrega. Um formulário de edição que formata pelo fuso do navegador grava outro dia em qualquer máquina fora de São Paulo. Num `PATCH`, reenviar só a data também apaga a hora original.

**Fechamento (AD-8, substituir a frase da função única):**

> O mês tem uma definição, com duas implementações testadas uma contra a outra: `src/util/month.ts` (limites `[início, início do mês seguinte)` em `America/Sao_Paulo`, com biblioteca de fusos e sem offset fixo) e, em SQL, exatamente `to_char("occurredAt" AT TIME ZONE 'America/Sao_Paulo', 'YYYY-MM')`. Um teste e2e compara as duas nas viradas de mês. Filtros de período são `from` e `to` como `YYYY-MM-DD`, ambos inclusivos no calendário de São Paulo, traduzidos para o intervalo semiaberto `[from 00:00, to+1 00:00)`. Se `occurredAt` for omitido, o servidor usa a data de hoje em São Paulo, às 00:00. A API retorna `occurredAt` em ISO 8601 UTC, e o frontend sempre formata em `America/Sao_Paulo`, sem cortar string ISO. Data posterior a hoje em São Paulo é rejeitada com 400.

A última frase é uma decisão de produto a confirmar. FR-15 diz "pode estar no passado", e FR-21 fecha o intervalo no mês atual. Sem ela, uma Movimentação futura entra na lista de um mês que FR-21 não mostra.

### 4. Balanço sem contrato de saída, e limite de 100 contra "todas"

**Unidade A, API do Balanço.** AD-14 manda retornar "o intervalo completo de meses", então a resposta é um array sem paginação.

**Unidade B, cliente e convenção.** A convenção de Listas diz `{ data, pagination }`, "sempre", com `limit` máximo de 100. O serviço do frontend espera isso.

**Par 4b.** FR-22 exige "**todas** as Movimentações daquele mês". Uma história embute a lista em `GET /balance/2026-10`; outra chama `/transaction?month=` com `limit=100`. Com dois Membros lançando por semana, um mês com mais de 100 lançamentos é plausível. A tela corta a lista, e os totais deixam de bater com o que se vê, atacando SM-2 ("confere com os extratos").

**Par 4c, bordas.** Conta sem Movimentação: `[]` ou o mês atual com Saldo inicial? Abrir mês fora do intervalo: 404 ou mês zerado com Saldo acumulado? FR-21 define o intervalo a partir da primeira Movimentação e não resolve o vazio.

**Par 4d, formato.** "Sem categoria" é `categoryId: null` ou um grupo com nome? Entradas e Saídas de FR-23 ficam em linhas separadas ou no mesmo objeto? O mapper (Convenções) é obrigatório, mas não há formato comum.

**Fechamento (AD-14, acrescentar; a convenção de Listas ganha a exceção):**

> `GET /api/account/:accountId/balance` retorna `{ data: MonthlyBalanceSummary[] }`, sem paginação e do mês mais recente para o mais antigo, sendo a única lista isenta da convenção de paginação. Conta sem Movimentação retorna `{ data: [] }` e o frontend exibe o Saldo inicial. `GET .../balance/:month` retorna `MonthlyBalance`; mês fora do intervalo retorna 404. Formatos, em centavos:
> `MonthlyBalanceSummary = { month: "YYYY-MM", totalIncome, totalExpense, result, accumulatedBalance }`.
> `MonthlyBalance = MonthlyBalanceSummary & { categories: [{ categoryId: string|null, categoryName: string|null, income, expense }] }`, com um único item `categoryId: null` para "Sem categoria".
> As Movimentações do mês vêm de `GET .../transaction?from=&to=` com paginação normal. O frontend percorre todas as páginas antes de exibir o mês.

Alternativa aceitável: embutir as Movimentações em `MonthlyBalance` sem limite, pois o volume por mês é pequeno. O essencial é escolher uma.

### 5. Duas abas derrubam a Sessão

**Unidade A, backend.** AD-5: "o reuso de um valor já trocado revoga a Sessão".

**Unidade B, frontend.** Convenção: "renovação única compartilhada" por interceptor. AD-6: token de acesso só em memória, então **toda carga de página chama `/api/refresh`**.

**Colisão.** A renovação compartilhada vale por aba. Duas abas restauradas juntas, ou dois carregamentos próximos, enviam o mesmo cookie. A primeira gira o token, a segunda é reuso, e a Sessão é revogada. Nenhum AD foi violado, e o Usuário vai para o login. Resposta perdida na rede tem o mesmo efeito.

**Três pares próximos:**

- **Corrida na rotação.** Duas requisições leem o mesmo hash, e ambas passam, se a troca for "ler e depois gravar". O caminho óbvio de compare-and-swap (`updateMany`) é proibido por AD-11. Um `update` com filtro no hash lança `P2025`, que AD-17 transforma em 404, mas `/refresh` deveria responder 401.
- **O mesmo vale para o uso único** do link de recuperação (AD-16).
- **Vida do cookie.** AD-6 lista os atributos sem `Max-Age`. Cookie sem ele morre ao fechar o navegador, e FR-1 define 7 dias sem uso. UJ-1 usa o sistema uma vez por semana, então o Usuário sempre veria o login.

**Fechamento (AD-5 e AD-6, acrescentar):**

> Rotação e consumo de token de uso único são compare-and-swap: `update` com `where: { id, <hash ou usedAt atual> }` dentro do Unit of Work; `P2025` nesse ponto é tratado pelo use case como token inválido (401 no refresh, 400 no link). A Sessão guarda o hash anterior e `rotatedAt`. Apresentar o hash anterior até 30 segundos depois da rotação não é reuso: responde 409 sem girar e sem revogar, e o frontend relê o token com uma nova chamada. Fora dessa janela, é reuso e revoga.
>
> O cookie tem `Max-Age` de 7 dias, renovado a cada rotação. O frontend coordena a renovação entre abas com Web Locks.

### 6. Auditoria grava `system` sem que nada falhe

**Unidade A, Unit of Work.** AD-10 fixa a assinatura `uow.run(repos => ...)`, sem actor. O contexto fica em `AsyncLocalStorage`.

**Unidade B, use case.** AD-2 passa `actor` como parâmetro. O autor na auditoria (AD-12) vem do contexto, com `system` para "fluxos sem Usuário autenticado".

**Colisão.** Há dois caminhos para o actor, e o spine não diz quem preenche o ALS: middleware ou UoW? Se o middleware esquecer, ou a história preencher só em alguns lugares, a extensão não encontra o actor e grava `system`, um valor **legítimo**. A auditoria parece certa, o teste de use case com repositório falso passa, e o FR-18 fica vazio em produção. Uma variante é a extensão fora do contexto gravar nulo, caso já observado no memlog ("fora de contexto actor e nulo").

**Defeito no PRD.** FR-18 e a origem dele no addendum: saber quem lançou ou alterou, para resolver lançamentos errados.

**Fechamento (AD-10 e AD-12):**

> A assinatura é `uow.run(actor, repos => ...)`, com `actor: Actor | SYSTEM` obrigatório, e o próprio `uow.run` é o único lugar que grava o actor no contexto. `SYSTEM` é uma constante explícita, usada só em login, renovação e recuperação de senha, e fica restrita a esses use cases por teste de arquitetura. A extensão lança erro se não houver actor no contexto, em vez de gravar um padrão. O NFR-8 ganha um teste e2e: criar, editar e excluir Movimentação geram `AuditLog.actorId` igual ao Usuário autenticado.

### 7. Estados de exclusão colidem

**Par 7a: Conta excluída bloqueia excluir Usuário.** A história de FR-14 grava `deletedAt` e mantém as linhas de `UserAccount`, pois dados "ficam guardados". A história de FR-5 aplica AD-9: "excluir um Dono de Conta com outros Membros é rejeitado até a transferência". Sem filtro por `deletedAt`, o Dono de uma Conta excluída com outro Membro **jamais pode ser excluído**: a transferência cai em 404 (AD-2), e nada resolve. O Admin fica sem saída.

**Par 7b: FR-6 exige o que AD-9 removeu.** FR-6 pede Contas "excluídas e desativadas, cada uma identificada como tal". AD-9 grava só `deletedAt`, então a história da visão Admin não consegue separar. Derivar por `owner.deletedAt` também falha: Conta excluída pelo Dono, que depois também foi excluído, é igual a Conta do Dono excluído. O memlog afirma que a decisão "altera FR-5, FR-6 e o termo Conta desativada", mas o PRD continua final com o texto antigo, e o spine declara `binds: FR-1..FR-24`.

**Par 7c: AD-4 contra AD-9.** "Todo Dono também tem linha em `UserAccount`" é falso depois que AD-9 apaga os vínculos do único Membro. Uma história que confia em AD-4, como o `JOIN` obrigatório da listagem do Admin, perde a Conta justamente em FR-6.

**Par 7d: remoção de Membro.** AD-9 define o efeito físico só para Usuário excluído. Uma história de FR-11 pode usar `removedAt`, para preservar "ex-Membro", e o PK `(userId, accountId)` quebra o reconvite.

**Fechamento (AD-9, acrescentar):**

> `Account` tem também `deletedReason` em `OWNER_DELETED | OWNER_DEACTIVATED`, gravado junto com `deletedAt`, e FR-6 exibe "excluída" ou "desativada" a partir dele. Regras de bloqueio e transferência consideram só Contas sem `deletedAt`, e Conta com `deletedAt` não aceita transferência, nem do Admin. Saída, remoção e exclusão de Usuário apagam fisicamente a linha de `UserAccount`; "ex-Membro" não é estado persistido. O AD-4 vale para Contas sem `deletedAt`.

Se a intenção for de fato aposentar "Conta desativada", o caminho alternativo é **alterar o PRD** em FR-5, FR-6 e no Glossário. Hoje o spine contradiz um documento que ele mesmo diz cumprir.

### 8. Transições de Membro sem exclusão mútua

AD-2 manda chamar a política "antes de qualquer outra ação", e o diagrama a mostra fora do Unit of Work. Portanto a checagem e a escrita não são atômicas. Combinações que obedecem a todos os ADs:

| Unidade A | Unidade B | Resultado |
| --- | --- | --- |
| Admin transfere para M (FR-13, FR-6) | Dono remove M (FR-11) | `ownerId = M` sem linha em `UserAccount`, violando AD-4 |
| Excluir Dono único Membro (FR-5) | Convidado aceita (FR-10) | Conta excluída com Membro ativo |
| Dono transfere para M | Dono altera M ou M sai (FR-12) | Dono que saiu da própria Conta |
| Duas transferências simultâneas | | A segunda sobrescreve, com a Permissão do antigo Dono errada |

O casal é poucos usuários, mas o Admin e o Dono são atores independentes e a invariante é absoluta: "cada Conta tem sempre exatamente um Dono" (FR-13).

**Agravante.** Um `SELECT ... FOR UPDATE` manual provavelmente é rejeitado pela guarda de AD-11. O memlog descreve a checagem como "SELECT/WITH sem palavra de escrita", e `UPDATE` está no texto. Inferência a confirmar no código da guarda.

**Fechamento (novo AD-22):**

> **AD-22: Transição de Membro serializada por Conta.** Transferência, remoção, saída, alteração de Permissão, aceite de Convite, exclusão de Conta e exclusão de Usuário rodam em `uow.run` com `lockAccounts: [accountId, ...]`. O Unit of Work, em `src/database/`, adquire `pg_advisory_xact_lock(hashtext(accountId))` em ordem crescente de id antes do callback, e o callback **relê** a política e as pré-condições dentro da transação. A checagem feita antes da transação serve só para a resposta rápida de 404 ou 403.

---

## Médias

### 9. AD-3 torna duas histórias impossíveis

AD-3 exige `accountId` em **toda** operação de Membro e Convite. FR-10 precisa listar Convites **do convidado**, em várias Contas. AD-9 precisa apagar todos os vínculos de um Usuário. Sem exceção escrita, cada história improvisa: uma pede um repositório novo, outra usa consulta solta, e outra percorre Contas. O resultado é o tipo de escape que AD-3 quer impedir.

**Fechamento (AD-3, acrescentar):**

> Uma operação de escopo de Usuário é permitida apenas quando o `userId` é parâmetro obrigatório e igual a `actor.userId`, ou quando o use case é de Admin. Únicas operações: `invitationRepository.findPendingByInvitee(userId)`, `invitationRepository.findByIdAndInvitee(id, userId)`, `memberRepository.findByUser(userId)` e `memberRepository.deleteByUser(userId)` (linha a linha).

### 10. Duas superfícies de leitura para o Admin

AD-2 deixa o Admin passar pela política em leitura, então `GET /account/:id/transaction` funciona para ele. As Rotas dão `/admin/{user,account}` para o mesmo fim. A história A serve a visão total por `/admin/account/:id/...`; a B deixa a política liberar as rotas normais. Depois:

- `GET /account` retorna, para o Admin, só suas Contas (FR-8) ou todas (FR-6)?
- AD-9 inclui linhas com `deletedAt` em "leituras do Admin", mas o Admin que é Membro comum, abrindo a própria Conta excluída, recebe 404 ou dados?
- Em FR-6, quem define a Permissão do antigo Dono quando o Admin faz a transferência? FR-13 diz que "o antigo Dono escolhe".

**Fechamento (AD-2, substituir a última frase):**

> A política de Membro nunca dá passe ao Admin. A visão total existe só em `GET /api/admin/account`, `/admin/account/:id`, `/admin/account/:id/member` e `/admin/account/:id/transaction`, com política própria (`requireAdmin`), e só essas leituras incluem `deletedAt`. A transferência é `POST /api/account/:accountId/transfer` para o Dono e `POST /api/admin/account/:accountId/transfer` para o Admin, ambas com `{ newOwnerId, previousOwnerPermission }` obrigatórios, sendo o valor decidido por quem executa. Admin que também é Membro usa as rotas normais com a Permissão do vínculo.

### 11. E-mail normalizado só no limite de login

AD-19 fala em "e-mail normalizado" e nenhum outro AD define isso. `User.email` é `@unique` sensível a caixa, e os schemas atuais usam `z.email()` puro. O Admin cria `Alex@x.com` (FR-4). O Convite por `alex@x.com` (FR-9) não acha o Usuário. O login falha com a mesma mensagem genérica de FR-1, sem pista. `alex@x.com` pode ser criado como segundo Usuário, violando "o e-mail é único" (FR-4). Além disso, FR-2 não envia o link.

**Fechamento (novo AD-23):**

> **AD-23: E-mail canônico.** Todo e-mail de entrada passa por um único `zEmail` em `src/util/zod/`, que aplica `trim` e minúsculas antes de validar; esse é o único formato gravado, buscado e usado como chave do limite de login. A migração normaliza os valores existentes. O limite de login roda depois do parser de JSON e conta todas as tentativas.

### 12. AD-15 vaza a existência do e-mail em FR-2

AD-15 liga FR-2 e FR-9 sob "a resposta informa a falha". Uma história de recuperação aplica isso e responde com erro quando o provedor falha. Só e-mails existentes chegam ao provedor, logo a resposta diferente confirma o e-mail. O envio direto também demora apenas nesse caso, o que abre canal de tempo. FR-2 exige resposta igual "exista ou não o e-mail". Em FR-9, o formato não está definido: 502 faz o cliente tratar como falha e tentar de novo, e a nova tentativa recebe 409.

**Fechamento (AD-15, substituir):**

> Em FR-9, o Convite é salvo e a resposta é 201 com `{ emailDelivered: boolean }`. Em FR-2, a resposta é sempre 202 com corpo fixo: a falha só vai para o log, e o envio acontece depois de responder (`setImmediate`, sem `await`). O login executa a comparação de hash mesmo para e-mail inexistente.

### 13. Erros de Prisma não mapeados e transação longa

AD-17 mapeia só `P2002` e `P2025`. Dois caminhos obedecem aos ADs e devolvem 500 (NFR-4, defeito D-06 de volta):

- **`P2003`.** Excluir Categoria enquanto outra requisição atribui a mesma Categoria a uma Movimentação, com FK `Restrict` (AD-11).
- **`P2028`.** FR-19 exige limpar linha a linha (AD-11), e cada `update` audita com leitura anterior e inserção (AD-12). Uma Categoria com milhares de Movimentações pode estourar o tempo padrão de 5 s da transação interativa de Prisma, que nenhum AD ajusta. A exclusão falharia sempre a partir de certo volume.

Também não está dito onde o mapeamento acontece. `AbstractUseCase.run` converte qualquer exceção em `UnknownError`, então a tradução precisa ocorrer antes. E `P2025 → 404` genérico mascara casos como o do achado 5.

**Fechamento (AD-10 e AD-17):**

> `uow.run` define `timeout` de 30 s e `maxWait` de 5 s. O próprio `uow.run` traduz erros de Prisma para erros de `src/errors/`: `P2002` para 409, `P2025` para 404, `P2003` para 409, e `P2028` e `P2034` para 503 com mensagem clara. O NFR-8 cobre excluir Categoria com pelo menos 2.000 Movimentações.

### 14. Paginação instável com datas empatadas

A convenção exige "ordenação explícita". Uma história usa `orderBy: { occurredAt: 'desc' }` e cumpre. Mas AD-8 grava toda entrada só com data como 00:00, então todos os lançamentos de um mesmo dia **empatam**, situação normal em UJ-1. Entre páginas, itens se repetem ou somem (D-15 reaberto com outra aparência), e a soma visível deixa de bater (SM-2).

**Fechamento (convenção de Listas):**

> Ordenação de Movimentações é sempre `occurredAt desc, createdAt desc, id desc`. Toda ordenação inclui `id` como desempate final.

---

## Baixas (fechar junto, sem discussão longa)

| # | Par | Fechamento |
| --- | --- | --- |
| B1 | AD-4 deixa o novo Dono com `ENTRY` ou `MANAGEMENT`. A lista de Membros de uma história exibe o valor bruto, e o Dono aparece com Lançamento | O mapper de Membro retorna `isOwner`; o valor bruto não é usado na interface. O Dono recebe `MANAGEMENT` ao criar a Conta e ao receber a transferência |
| B2 | `PATCH` sem semântica para campo ausente. FR-20 exige remover Categoria | Campo ausente não altera; `categoryId: null` remove |
| B3 | AD-13 manda gravar o padrão da rota. Com `routes.use('/account/:accountId', ...)` (código atual), `req.baseUrl` traz o UUID real | Compor o padrão por constante da rota; requisição sem rota registra `UNMATCHED` |
| B4 | AD-12 mascara por nome de coluna; um `secretHash` futuro escapa | Regra: toda coluna de segredo termina em `Hash` e a extensão mascara `password` e `*Hash` |
| B5 | Queries usam o cliente global. Uma história que chama a query dentro de `uow.run`, para montar a resposta, lê o estado anterior | Queries só depois do commit, ou o resultado sai dos `repos` |
| B6 | `AbstractUseCase.run` valida Zod antes de `execute`, e AD-2 pede a política "antes de qualquer outra ação" | Escrever em AD-2: Zod vem antes; a política é a primeira ação de `execute` |
| B7 | Até 2018, São Paulo teve horário de verão, com dias sem 00:00 (por exemplo, 04/11/2018). Afeta só lançamentos muito antigos | Usar "primeiro instante válido do dia", já coberto pela biblioteca de fusos de AD-8 |

---

## Ordem sugerida para fechar

1. **Antes da primeira história de Convites:** achados 1, 8, 9.
2. **Antes do modelo de Movimentação e do Balanço (itens 2 e 3 da ordem do PRD):** achados 2, 3, 4, 14.
3. **Antes de qualquer escrita nova:** achados 6 e 13, pois o `uow.run` muda de assinatura.
4. **Decisões de produto a pedir a Guilherme:** data futura (achado 3), "Conta desativada" (achado 7) e a janela de 30 s de reuso (achado 5).
5. **Depois:** 5, 7, 10, 11, 12 e a tabela de itens baixos.
