# Revisão por rubrica: Architecture Spine do My Finances

- **Documento revisado:** `ARCHITECTURE-SPINE.md` (status `draft`, altitude `initiative`, atualizado em 2026-11-08)
- **Comparado com:** `prd.md` e `addendum.md` (prd-workspace-2026-11-07), `levantamento-my-finances-backend.md`, `.memlog.md` do spine e o código de `my-finances-backend` em `c63fa8c`
- **Data da revisão:** 2026-11-08

## 1. Veredito

**Requer revisão antes de aprovar.** A base é boa: o paradigma ratifica o código real, os ADs têm forma verificável, e os defeitos críticos do levantamento (isolamento, tokens, dinheiro, data) têm dono. Mas o spine ainda não serve de substrato para histórias independentes, por quatro motivos:

1. Dois pontos de divergência centrais estão abertos ou têm regra que falha: o que acontece com um `wrong(...)` dentro do Unit of Work (A-1) e o contrato de `actor` (A-2).
2. Duas regras contradizem o PRD ou a si mesmas: o Convite expirado (A-3) e a fusão de "desativado" com "excluído" (A-4). A decisão de adiar o frontend também contradiz o PRD (A-5).
3. Duas regras não encaixam no código existente e não dizem como migrá-lo (A-6, A-7).
4. A dimensão de operação, que o próprio frontmatter coloca no escopo, está quase toda fora dos ADs (A-8, A-9). O documento não tem seção de perguntas em aberto.

Contagem: **9 altos, 12 médios, 8 baixos**. Nenhum exige mudar o paradigma. A maioria se resolve com uma ou duas frases em um AD existente.

### Resultado por critério

| Critério | Resultado | Achados |
| --- | --- | --- |
| Fixa os pontos reais de divergência, sem omitir nenhum | Parcial | A-1, A-2, A-6, M-1, M-2, M-3 |
| Toda Rule é aplicável e previne a divergência declarada | Parcial | A-1, A-3, A-7, M-4, M-5, M-6 |
| Nada em Deferred permite divergência entre unidades | Não | A-5, A-8, M-7 |
| Ratifica, sem contradizer, o código existente | Parcial | A-6, A-7, M-1, M-3, M-8 |
| Cobre as capacidades do PRD | Parcial | A-3, A-4, A-5, A-9, M-9, M-10 |
| Toda dimensão da altitude está decidida, adiada ou em aberto | Não | A-8, A-9, seção 4 |

## 2. Achados altos

### A-1. O destino de `wrong(...)` dentro de `uow.run` não está definido (AD-10, AD-1)

O contrato do AD-1 é retornar erro como valor. Uma transação interativa do Prisma só faz rollback quando o callback lança exceção. Se um use case grava a linha A, depois retorna `wrong(...)` dentro de `uow.run`, a transação **confirma** a escrita parcial, junto com a auditoria. É exatamente o que o AD-10 diz prevenir. Um use case pode lançar, outro pode retornar, e `AbstractUseCase.run` transformaria a exceção em `UnknownError` (500), violando o NFR-4.

**Correção:** definir no AD-10 que `uow.run` faz rollback quando o callback retorna `Wrong` e devolve esse `Wrong` ao chamador. Exceções continuam virando `UnknownError`. Cobrir com teste do próprio `uow`.

### A-2. O contrato de `actor` e a autorização administrativa têm lacunas (AD-2)

- `AbstractUseCase.run(input)` tem um único parâmetro, validado por Zod. O AD-2 não diz se `actor` entra no schema ou em um segundo argumento.
- Os controllers atuais espalham o corpo por último: `{ accountId, ...req.body }` e `{ id: userId, ...req.body }`. Replicar esse padrão com `actor` em `input` permite enviar `"actor": { "isAdmin": true }` no corpo. O mesmo padrão já permite sobrescrever `accountId` e `id` de hoje.
- A origem de `isAdmin` não está definida. O AD-5 deixa `sub` e `sessionId` no JWT, enquanto `TokenPayload` atual tem `user.admin`. O AD-2 só diz que vínculo, Permissão e Dono vêm do banco. Falta dizer também que `isAdmin` vem do banco, e o campo em `User` não aparece no spine.
- O AD-2 só cobre "dado de Conta". FR-4, FR-5 e FR-3 (`/admin/user`, `/me`) não têm regra, e "middleware só autentica". Falta dizer onde se verifica ser Admin. É a mesma classe de defeito do D-02.
- `GET /account` (FR-8) não tem `accountId`, então nem política por Conta nem AD-3 o alcançam. O código atual lista todas as Contas.
- O Admin aparece com dois caminhos de leitura possíveis: `/account/:id` ("passa pela política em leitura") e `/admin/account`. Falta escolher um.
- `AbstractController` não tem helper `forbidden` e nada mapeia para 403.

**Correção:** definir `run(actor, input)` ou equivalente, com `actor` vindo só do middleware e parâmetros de rota aplicados depois do corpo. Declarar `isAdmin` lido do banco, uma política `requireAdmin` em `src/policies/`, listagem de Contas por vínculo e um único caminho de leitura do Admin.

### A-3. O índice único parcial impede o novo Convite que o FR-10 garante (AD-16)

O AD-16 rejeita job de expiração e compara `expiresAt` no momento do uso. Um índice parcial não pode usar `now()`, porque o predicado precisa ser imutável. Logo ele só pode filtrar por status. Um Convite vencido continua `PENDING` para sempre e **bloqueia o novo Convite**, enquanto o FR-10 diz que o Dono pode reenviar. A falha viraria um 409 sem causa visível para o Dono.

O partial index também não se expressa em `schema.prisma`, então precisa de SQL manual na migration. O spine não registra isso.

**Correção:** na criação do Convite, no mesmo Unit of Work, marcar o Convite pendente vencido como expirado (ou removê-lo) antes de inserir. Registrar o SQL manual. Definir também o que acontece com Convite pendente quando o Usuário ou a Conta recebem `deletedAt`: o convidado não é Membro, então a política não o alcança.

### A-4. O AD-9 altera o PRD final sem registrar (FR-5, FR-6, Glossário)

O memlog admite que a fusão "altera FR-5, FR-6 e o termo Conta desativada", mas o spine não menciona isso e declara `binds: FR-1..FR-24`. O FR-6 exige que Contas excluídas e desativadas apareçam "cada uma identificada como tal". Com um único `deletedAt` isso não é distinguível de forma confiável: um Dono que exclui a Conta e depois é excluído fica igual ao Dono único excluído.

**Correção:** escolher uma das duas opções e registrá-la: (a) manter a fusão e corrigir FR-5, FR-6 e o Glossário do PRD; (b) guardar o motivo (`deletedReason` ou equivalente).

### A-5. "Decisão de construir o frontend" está adiada, mas o PRD a exige (Deferred)

O PRD §7 inclui tudo da seção 4 "publicado em produção" e lista o frontend como item 7. FR-15, FR-2 ("a tela", o link do e-mail), NFR-7 e SM-1 só se cumprem com interface. O spine marca o frontend como "se construído", com a justificativa "a API não depende dele". A API depende sim em um ponto: o link de FR-2 precisa apontar para uma página, e AD-6 e AD-18 assumem proxy e origem única.

O memlog mostra que a dúvida é real e do usuário, então não é erro de redação. Mas é uma mudança de escopo do PRD, não um item adiável.

**Correção:** ou confirmar o frontend na v1, ou corrigir o PRD e definir o que substitui a tela de redefinição de senha. Registrar também onde o frontend vive (`my-finances-frontend`) e como é publicado.

### A-6. O Unit of Work não encaixa em `BaseRepository`, `BaseQuery` nem nas factories (AD-10, AD-11, AD-1)

- O AD-11 restringe `PrismaClient` a `src/database/` por lint, mas ele é importado em `core/repository.ts`, `core/query.ts`, nos quatro repositórios, em `jest.setup.ts` e em `test/helpers/base-test.ts`. Os construtores são tipados com `PrismaClient`. O spine não define o tipo que os substitui, e o cliente de transação (`Prisma.TransactionClient`) não é atribuível a `PrismaClient`. A regra também precisa restringir só o nome `PrismaClient`, já que os repositórios importam tipos de modelo do mesmo pacote, e excetuar os testes.
- `BaseRepository.findWhere` chama `this.prismaClient.$transaction([...])`, e o cliente de transação não tem `$transaction`. A paginação quebra dentro do UoW.
- Hoje as factories injetam repositórios no construtor do use case. Com `uow.run(repos => ...)`, não está definido de onde vêm os repositórios de leitura, qual o formato de `repos`, se queries rodam dentro da transação, nem como os 25 specs existentes mocam o `uow`.
- O AD-2 diz que `actor` é parâmetro, enquanto o AsyncLocalStorage também carrega o autor para a auditoria (memlog). Falta dizer quem preenche esse contexto e como as duas fontes ficam iguais.

**Correção:** ampliar o AD-10 com o tipo `DbClient` exportado de `src/database/`, o formato de `repos` e o mock padrão. Definir a contagem de `findWhere` sem `$transaction` aninhado.

### A-7. O AD-3 tem exceções obrigatórias e um mecanismo inviável

O AD-3 exige `accountId` em toda operação de Membro e Convite, mas fluxos do próprio spine não têm Conta:

- `GET /invitation`: lista os Convites do convidado em todas as Contas, e a rota `/invitation` não tem `accountId` (FR-10).
- AD-9: excluir Usuário apaga seus vínculos em todas as Contas.
- `UserAccount` tem PK composta, sem `id`, então `{ id, accountId }` não se aplica.

O mecanismo também não fecha. Uma subclasse TypeScript não remove métodos públicos herdados, e sobrescrever `findById(id)` com outra assinatura não compila. Além disso `IBaseRepository` traz `update(id, data)` e `delete(id)` sem Conta, que a lista do AD-3 não cita.

**Correção:** dizer que esses repositórios não estendem `BaseRepository` (ou criar `AccountScopedRepository`). Listar as exceções nomeadas, com métodos específicos por `userId` que só aceitam o `actor`.

### A-8. Produção, backup e restauração não têm decisão (AD-18, Deferred, NFR-6)

- A decisão do memlog ("VPS única com Docker Compose (API, front, Postgres) e proxy com HTTPS") aparece só como diagrama no Structural Seed. Nenhum AD a fixa, e o AD-18 só fala de Compose em desenvolvimento.
- Deferred diz que proxy e script de backup "não afetam o código". O proxy afeta: terminação HTTPS, `trust proxy`, cookie `Secure`, origem única do AD-6.
- A **restauração testada antes do uso real**, exigida pelo NFR-6, não aparece em lugar nenhum. O backup não tem local fora da VPS nem retenção; um backup no mesmo disco não sobrevive à perda da VPS.
- "Deploy reproduzível" não tem mecanismo. O CI só faz build da imagem em PR, sem registro, tag ou gatilho de deploy. Os itens `.nvmrc` e `packageManager` foram aprovados no memlog e sumiram do spine. O `Dockerfile` atual usa `pnpm@latest`.
- `migrate deploy` "em passo separado" não diz onde roda. `prisma` está em `devDependencies`, então um estágio final só com dependências de produção não tem a CLI.
- Não há rollback nem regra de migration compatível com a versão anterior.
- O AD-19 só vale com uma instância, mas nenhum AD fixa essa instância única.

**Correção:** criar AD de topologia e entrega de produção, com backup fora da VPS e restauração como condição de entrada em produção. Separar em Deferred só provedor e dimensionamento.

### A-9. Não existe caminho para o primeiro Admin em produção (FR-4, D-08, D-17)

O FR-4 faz do Admin o único criador de logins, então o primeiro Admin é pré-requisito do sistema. O único caminho existente é `seed.ts`, que **apaga todo o banco** com `cleanDatabase`, importa de `test/` (o addendum pede o fim disso), grava senha `Password123!` e usa o CPF que sairá. Em produção, sem decisão, a execução do seed é destrutiva.

**Correção:** decidir um comando idempotente de criação do primeiro Admin, com senha por variável de ambiente e sem dependência de `test/`, e separar seed de desenvolvimento dele.

## 3. Achados médios

| ID | AD | Problema | Correção |
| --- | --- | --- | --- |
| M-1 | AD-17 | Corpo `{ "error": "<mensagem>" }` contradiz `util/http.ts`, que devolve `{ code, error, slug }` e, em validação, `error: { message, errors }`. O middleware usa `{ message }`. Sem marcar como mudança, os erros por campo somem e o frontend perde a validação por campo. | Ratificar o formato atual e padronizar o middleware, ou declarar a troca e dar forma ao erro de validação. |
| M-2 | Paradigma | `src/entities/` não aparece no spine e a frase "cada camada mapeia para pasta existente" é imprecisa (`policies/` e `gateways/` são novas). Hoje há três padrões de saída: `toOutput` na entidade, `UserMapper` e `right(account)` direto. "Sempre por mapper" não diz o papel das entidades nem se Categoria e Convite têm uma. | Declarar a camada de entidades e o papel dela. |
| M-3 | AD-17 | P2002 e P2025 são mapeados "antes do `UnknownError`", sem dizer onde. O `catch` de `AbstractUseCase.run` hoje captura tudo. P2003 (D-06) fica de fora, embora o AD-11 torne FKs `Restrict`. Timeout de transação interativa (5 s por padrão) também vira 500, e o AD-11 obriga atualização linha a linha. | Tradução única em `core/use-case.ts`, incluindo P2003. Definir timeout do UoW. |
| M-4 | AD-5 | "Reuso revoga a Sessão" exige guardar valores já trocados, e o modelo só indica `refreshTokenHash`. Sem isso o valor antigo é só desconhecido. Duas abas, ou recarga durante a renovação, disparam reuso e derrubam a Sessão; a "renovação única" do frontend só vale por aba, e o token em memória força renovação a cada recarga. | Definir armazenamento (hash anterior) e janela de tolerância, ou coordenação entre abas. |
| M-5 | AD-11 | Rule omite `$queryRawUnsafe`, que é escape direto da auditoria. O memlog admite que a checagem por palavra-chave é "rede de segurança", mas o spine diz apenas que lança erro. | Incluir `$queryRawUnsafe` e registrar o limite da checagem. |
| M-6 | AD-13 | O registro exige padrão de rota, mas `routes.use('/account/:accountId', transactionRouter)` faz `req.baseUrl` conter o UUID real, e `req.route.path` só tem o último trecho. Rotas sem correspondência não têm `req.route`. | Definir como reconstruir o padrão e o valor para rota não encontrada. |
| M-7 | AD-15, Deferred | Adiar o provedor é legítimo (PRD §9), mas SPF e DKIM (addendum) não têm dono e a URL base dos links não está definida. O formato de `EmailSender` não está fixado e FR-2 e FR-9 o consomem em histórias diferentes. | Fixar a interface e a variável de URL base. Citar SPF e DKIM no item adiado. |
| M-8 | AD-20 | O app usa `DATABASE_URL`, e só `prismaTest` usa `DATABASE_URL_TEST`. Com supertest local e `.env` de desenvolvimento, o e2e atinge o banco de dev, e `cleanDatabase` faz TRUNCATE nele. O CI mascara isso. | Fixar a seleção do banco de teste. |
| M-9 | AD-8, AD-14 | A "função única" do mês não se concilia com a agregação SQL do AD-14, que agrupa no próprio SQL. O memlog decidiu "frontend exibe sempre nesse fuso", e essa regra sumiu do AD-8. Em navegador fora do fuso, a data de 00:00 aparece no dia anterior. Também falta dizer quem calcula a data padrão do FR-15 e como tratar data futura, já que FR-21 termina no mês atual. | Fixar a expressão SQL canônica, a regra do frontend, o responsável pelo padrão e a data futura. |
| M-10 | Convenções | FR-22 pede "todas as Movimentações" do mês, e o limite máximo é 100. A convenção de listas também não diz se a lista de meses do FR-21 pagina. Faltam ainda rotas de fechamento do mês e de reenvio de Convite. | Definir exceção ou paginação, e rotas. |
| M-11 | AD-15 | Binds inclui FR-2, mas a falha só é definida para Convite. Envio direto só quando o e-mail existe cria diferença de tempo e possivelmente de resposta, contrariando "responde da mesma forma". | Resposta idêntica e envio sem aguardar em FR-2. |
| M-12 | Migração | Não há decisão sobre dados existentes: `ownerId` obrigatório em Contas sem dono, `occurredAt`, `type`, `createdById`, valores de 4 casas para centavos, remoção de `cpf`. O levantamento só sugere recriar bancos locais. | Registrar "sem dados a preservar, migrations podem recriar" ou a regra de preenchimento. |

## 4. Dimensões da altitude de iniciativa

| Dimensão | Situação | Observação |
| --- | --- | --- |
| Paradigma e camadas | Decidido | Falta entidades (M-2) |
| Modelo de dados | Decidido em parte | Só ER sem atributos; CPF não mencionado (B-1); migração (M-12) |
| Contrato de API | Decidido em parte | Erros (M-1); verbos e status (B-3) |
| Autenticação e autorização | Decidido em parte | A-2, M-4 |
| Segurança de borda | **Sem decisão** | D-18: `app.use(cors())` continua aberto, headers de segurança e `trust proxy` não aparecem |
| Ambientes | Decidido em parte | Dev e produção; falta dizer que não há homologação e listar variáveis por ambiente |
| Implantação | **Sem AD** | A-8 |
| Backup e restauração | **Adiado com justificativa errada** | A-8 |
| Observabilidade | **Ausente** | Só `console.error` e `RequestLog`. Sem destino de log, alerta ou monitoramento |
| Saúde e reinício | **Ausente** | `/health` não testa banco e o spine não o cita; sem política de reinício ou desligamento gracioso |
| Inicialização do sistema | **Ausente** | A-9 |
| Retenção de dados | Adiado em parte | `RequestLog` adiado; `AuditLog` e `Session` não citados |
| Concorrência | **Ausente** | Política lida fora da transação; sem nível de isolamento. Risco baixo com dois usuários, mas merece registro |
| Frontend | Adiado indevidamente | A-5 |
| Perguntas em aberto | **Seção inexistente** | O último evento do memlog lista regras derivadas "a confirmar com o usuário" (`ownerId`, refresh opaco, escrita fora de UoW, FKs `Restrict`, expiração por data, rate limit em memória, versões). O spine não as sinaliza. A pendência de atualizar issue #29, Notion e levantamento também não foi levada. |

## 5. Achados baixos

| ID | Local | Problema |
| --- | --- | --- |
| B-1 | NFR-2, FR-4 | O spine nunca menciona a remoção do CPF, embora `User.cpf` seja `NOT NULL @unique` e `UserMapper` o devolva. |
| B-2 | AD-12 | A lista de campos não inclui data e hora, exigidas pelo FR-18. A máscara é lista fixa, então um campo novo com hash vaza; preferir regra por sufixo. |
| B-3 | Convenções | Código mistura `PUT` e `PATCH`, e `DELETE` responde 200 com corpo; o spine não define. |
| B-4 | AD-7, AD-19 | `initialBalance` não tem limites, repetindo o D-11 como 500. "E-mail normalizado" vale só no limitador, e `User.email` é sensível a maiúsculas, permitindo duplicidade no FR-4. |
| B-5 | AD-18 | "O `JWT_SECRET` nunca é usado" não é verificável; o memlog previa placeholder, e `env.ts` não rejeita o valor. O `run`, o README e o `.env.example` (`@db:5432`) assumem API no contêiner. |
| B-6 | AD-2, AD-4 | "Antes de qualquer outra ação" conflita com `run`, que valida primeiro. O AD-4 diz que todo Dono tem `UserAccount`, e o AD-9 apaga esse vínculo no Dono único excluído. |
| B-7 | Rotas | `accountRouter.use(authMiddleware)` é montado sem caminho e alcança tudo que chega nele. Rotas públicas novas (`/password/*`) precisam ser montadas antes. |
| B-8 | Mapa | NFR-7 não aparece no mapa nem em Binds; FR-4 não é citado no Binds de nenhum AD além do `all`. Falta limite em `/password/forgot`, que dispara e-mail (o PRD não exige). |

## 6. O que está sólido

- AD-1 corresponde ao código (`AbstractUseCase`, `Either`, `handleError`, factories).
- AD-5 fecha D-09 de forma correta, e a sessão no banco atende logout e exclusão imediatos.
- AD-7 e AD-14 eliminam saldo divergente e erro de arredondamento.
- AD-10 a AD-12 têm base experimental registrada no memlog (Prisma 6.19.0).
- AD-16 elimina job de expiração, salvo o caso do índice (A-3).
- As versões de backend do Stack conferem com `pnpm-lock.yaml` (Prisma 6.19.0, TypeScript 5.9.3, Express 5.1.0, Zod 4.1.12, Jest 30.2.0, jsonwebtoken 9.0.2, bcryptjs 3.0.3).

## 7. Limites desta revisão

- Revisão estática. O repositório não tem `node_modules`, e nenhum código foi executado ou alterado.
- O comportamento do Prisma em A-1, A-3 e A-6 (rollback só por exceção, ausência de `$transaction` no cliente de transação, predicado imutável de índice) vem da semântica documentada de Prisma e PostgreSQL, sem teste aqui.
- As versões de frontend e de bibliotecas novas do Stack não foram conferidas no registro npm.
