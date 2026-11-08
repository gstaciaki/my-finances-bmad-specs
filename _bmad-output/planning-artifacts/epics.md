---
status: final
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-workspace-2026-11-07/prd.md
  - _bmad-output/planning-artifacts/prds/prd-workspace-2026-11-07/addendum.md
  - _bmad-output/planning-artifacts/architecture/architecture-workspace-2026-11-08/ARCHITECTURE-SPINE.md
  - _bmad-output/planning-artifacts/ux-designs/ux-workspace-2026-11-08/DESIGN.md
  - _bmad-output/planning-artifacts/ux-designs/ux-workspace-2026-11-08/EXPERIENCE.md
  - _bmad-output/planning-artifacts/levantamento-my-finances-backend.md
---

# My Finances - Epic Breakdown

## Overview

Este documento decompõe os requisitos do PRD, do UX e da arquitetura do My Finances em histórias implementáveis.

## Requirements Inventory

### Functional Requirements

- FR-1: Login com e-mail e senha; mesma mensagem para credenciais inválidas; sessão expira após 7 dias sem uso; logout vale imediatamente.
- FR-2: Recuperação de senha por link enviado por e-mail, de uso único e válido por 1 hora; resposta igual exista ou não o e-mail; nova senha segue a política.
- FR-3: Usuário edita o próprio nome e e-mail e troca a senha informando a atual; e-mail único; não altera outros Usuários.
- FR-4: Admin cria Usuário com nome, e-mail e senha inicial; e-mail único; sem CPF.
- FR-5: Admin lista, edita, redefine a senha e exclui Usuários; excluído não faz login, vira ex-Membro e o e-mail fica reservado; Dono de Conta com outros Membros só é excluído após transferência; Dono único Membro leva a Conta junto.
- FR-6: Admin vê todas as Contas, Membros e Movimentações, inclusive Contas excluídas identificadas; somente leitura, exceto transferir titularidade.
- FR-7: Usuário cria Conta com nome e Saldo inicial opcional e se torna Dono; Conta aparece na lista.
- FR-8: Usuário vê só as Contas de que é Membro e não lê nem altera outras.
- FR-9: Dono convida Usuário existente por e-mail com Permissão; sem convidar Membro nem duplicar Convite pendente; expira em 7 dias.
- FR-10: Convidado vê Convites pendentes ao entrar e aceita ou recusa; recusado ou expirado não pode ser aceito, e o Dono pode enviar outro.
- FR-11: Dono remove qualquer Membro, menos a si; perda de acesso imediata.
- FR-12: Membro sai da Conta; Dono só após transferir.
- FR-13: Dono ou Admin transfere a titularidade para outro Membro; sempre um Dono; o antigo Dono escolhe a própria Permissão e continua Membro.
- FR-14: Membros com Gestão editam nome e Saldo inicial, o que recalcula o Saldo acumulado; só o Dono exclui, com confirmação explícita e exclusão lógica; sem restauração na v1.
- FR-15: Membro registra Movimentação com tipo, valor positivo com até 2 casas, data não futura (padrão hoje em São Paulo), descrição e Categoria opcionais; valor acima do limite é rejeitado; formulário fica pronto para o próximo lançamento.
- FR-16: Qualquer Membro edita ou exclui qualquer Movimentação da Conta.
- FR-17: Membro lista Movimentações ordenadas por data, com filtro por período e Categoria, nunca de outra Conta.
- FR-18: Sistema guarda autoria de criação, alteração e exclusão de Movimentações, além de toda escrita no banco e toda requisição HTTP, sem corpo nem segredos; não aparece nas telas da v1.
- FR-19: Membros com Gestão criam, renomeiam e excluem Categorias da Conta, com nome único; Lançamento só escolhe; excluir Categoria em uso deixa Movimentações sem Categoria.
- FR-20: Qualquer Membro atribui ou remove a Categoria de uma Movimentação.
- FR-21: Lista de meses, do mais recente ao mais antigo, com Entradas, Saídas, Resultado e Saldo acumulado, da primeira Movimentação até o mês atual, incluindo meses vazios.
- FR-22: Abrir mês mostra Balanço e todas as Movimentações, definido pelo calendário de São Paulo.
- FR-23: Balanço com totais por Categoria, separando Entradas e Saídas, e grupo "Sem categoria"; soma igual aos totais.
- FR-24: Dono altera a Permissão de outro Membro, com efeito imediato.

### NonFunctional Requirements

- NFR-1 Isolamento: Usuário nunca lê nem altera dados de Conta da qual não é Membro, exceto leitura do Admin, em toda a API.
- NFR-2 Dados sensíveis: nenhuma resposta contém senha ou hash; sem coleta de CPF.
- NFR-3 Exatidão: valores em centavos; até R$ 21.474.836,47 por Movimentação.
- NFR-4 Erros previsíveis: regra violada retorna erro claro, nunca 500.
- NFR-5 Sessão: tokens de acesso e de renovação com finalidades distintas.
- NFR-6 Produção: HTTPS, segredos fora do repositório, build e deploy reproduzíveis, 5 tentativas de login por 15 minutos por e-mail, backup diário com restauração testada antes do uso real.
- NFR-7 Plataforma: web para navegadores de computador.
- NFR-8 Testes: e2e cobrem isolamento, Permissões, poderes de Dono e Admin e cálculo do Balanço, incluindo a virada do mês.

### Additional Requirements

**Base do backend (arquitetura):**
- AR-1: Histórico de migrations zerado; o schema recomeça do zero, sem `cpf`, e cada história adiciona a própria migration (AD-23, AD-25).
- AR-2: `src/database/`: tipo `DbClient`, extensão Prisma com bloqueio de operações e auditoria, `AsyncLocalStorage`, `uow.run(actor, fn, { lockAccounts })` com rollback em `Wrong`, `FakeUnitOfWork` (AD-10, AD-11, AD-12, AD-22).
- AR-3: `AccountScopedRepository`, exceções nomeadas por Usuário e FK composta de Categoria (AD-3).
- AR-4: Contrato `useCase.run(input, actor)`, políticas em `src/policies/`, `requireAdmin`, entrada de controllers por campos explícitos (AD-2).
- AR-5: Tradução única de erros Prisma e 403/429 no mapa de erros (AD-17).
- AR-6: `RequestLog` por middleware, sem corpo, com rota normalizada (AD-13).
- AR-7: Sessão em banco, refresh opaco com rotação e janela de 30 s, cookie `httpOnly`, `X-Requested-With` (AD-5, AD-6).
- AR-8: `zEmail` e `zPassword` únicos, corrigindo o regex de símbolos (AD-23, D-20) e as mensagens com entidade errada (D-21).
- AR-9: `EmailSender` com implementação falsa, `APP_BASE_URL` (AD-15).
- AR-10: `src/util/month.ts` com `@date-fns/tz` e teste contra a expressão SQL (AD-8).
- AR-11: `app.ts` separado de `server.ts`; e2e com supertest e banco de teste protegido (AD-18, AD-20).
- AR-12: Atualizar Prisma 6.19.3, Express 5.3.0, Zod 4.6.7, jsonwebtoken 9.0.3 e uuid 11.1.1; adicionar `helmet`, `cookie-parser`, `express-rate-limit`, `date-fns`, `@date-fns/tz`, `tsc-alias`, `supertest`.
- AR-13: Filtros explícitos no lugar de `autoParseFilters`; header `Location` em 201 (issue #24); `DELETE` responde 204.
- AR-14: Criação de Conta e vínculo do Dono em uma transação (D-14) e índice `(accountId, occurredAt)` (D-16).

**Infra e operação:**
- AR-15: Compose só com banco em `127.0.0.1`, script do banco de teste, `.nvmrc`, `packageManager`, `.env.example` com placeholder; aposentar o script `run` (AD-18).
- AR-16: Dockerfile multi-stage de produção com alvo `migrate`; scripts `build` e `start` (AD-18, D-17).
- AR-17: CI com pnpm, lint, `tsc`, testes e build; publicação da imagem na `main` (AD-18, D-19).
- AR-18: Produção em VPS com Compose, proxy HTTPS, `trust proxy`, `helmet`, CORS fechado, `/health` com banco, deploy e rollback por imagem (AD-24).
- AR-19: Backup diário fora da VPS e restauração testada (AD-24, NFR-6).
- AR-20: `pnpm admin:create` idempotente; seeds de desenvolvimento sem dependência de `test/` (AD-24).
- AR-21: Escolher provedor de e-mail e configurar SPF e DKIM (Deferred).

**Base do frontend (`my-finances-frontend`):**
- AR-22: Projeto Vue 3, Vite, TypeScript, Pinia, Vue Router com rotas manuais e guards, axios, Zod, Tailwind, pnpm; proxy do Vite para `/api`; Dockerfile de produção.
- AR-23: Cliente HTTP com token em memória, renovação compartilhada e retorno ao login; serviços e stores por domínio (AD-6).

### UX Design Requirements

- UX-DR1: Tokens de cor, tipografia, raio e espaçamento do `DESIGN.md` no Tailwind, em tema claro e escuro.
- UX-DR2: Alternância de tema pelo menu do usuário, seguindo o sistema na primeira visita e guardada no navegador.
- UX-DR3: `BaseButton` nas variantes primary, secondary e danger, com estados desabilitado e carregando.
- UX-DR4: `BaseInput` com rótulo, erro no campo, `aria-describedby` e `aria-invalid`.
- UX-DR5: `BaseSelect` sobre `<select>` nativo, com busca por letra.
- UX-DR6: `BaseTable` com hover, valores à direita, linha clicável e paginação.
- UX-DR7: `MoneyInput` com digitação em reais, conversão para centavos e limite do NFR-3.
- UX-DR8: `MoneyValue` em `R$`, com sinal e cor apenas em resultados.
- UX-DR9: `TypeToggle` com um ponto de Tab, teclas `S`/`E` e setas.
- UX-DR10: `AppDialog` e `ConfirmDialog` em `<dialog>`, com `Esc`, gestão de foco e foco inicial em Cancelar.
- UX-DR11: `AppToast` com `aria-live="polite"`, sumindo em 4 segundos.
- UX-DR12: `SummaryCard` para Entradas, Saídas, Resultado e Saldo acumulado.
- UX-DR13: `AppHeader` com Conta atual, menu do usuário, Admin condicional e indicador de Convites.
- UX-DR14: `EntryForm` em linha: Data, Descrição, Valor, Tipo, Categoria; `Enter` salva; mantém Data e Tipo, limpa o resto e volta o foco à Data; Tipo inicial Saída; sem envio duplicado; lista "Lançadas agora" com contador, editar e excluir.
- UX-DR15: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.
- UX-DR16: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.
- UX-DR17: Acessibilidade WCAG 2.2 AA, fluxo de lançamento só com teclado e sobra ou falta comunicadas por texto.
- UX-DR18: Datas em `dd/mm/aaaa` em São Paulo; "hoje" por `Intl` nesse fuso; data máxima hoje.
- UX-DR19: Microcopy do `EXPERIENCE.md`, incluindo "Sobrou" e "Faltou".
- UX-DR20 (API): `GET /me` com `id`, `name`, `email`, `isAdmin`.
- UX-DR21 (API): `GET /account` e `GET /account/:id` com `isOwner` e `permission` do Usuário atual.
- UX-DR22 (API): Categorias e Membros legíveis por qualquer Membro; Membros com nome.
- UX-DR23 (API): `GET /invitation` com nome da Conta e Permissão.
- UX-DR24 (API): Listas de Movimentação com `categoryId` e `categoryName`.

### FR Coverage Map

| Requisito | Epic | Observação |
|---|---|---|
| FR-1 | Epic 1 | Login, sessão e logout |
| FR-2 | Epic 1 | Recuperação de senha |
| FR-3 | Epic 1 | Perfil e troca de senha |
| FR-4 | Epic 1 | Admin cria Usuário |
| FR-5 | Epic 1 e Epic 3 | Listar, editar e redefinir senha no Epic 1; excluir no Epic 3, por depender de Membros e titularidade |
| FR-6 | Epic 6 | Visão do Admin, somente leitura |
| FR-7 | Epic 2 | Criar Conta |
| FR-8 | Epic 2 | Contas visíveis só a Membros |
| FR-9 | Epic 3 | Enviar Convite |
| FR-10 | Epic 3 | Aceitar ou recusar Convite |
| FR-11 | Epic 3 | Remover Membro |
| FR-12 | Epic 3 | Sair da Conta |
| FR-13 | Epic 3 | Transferir titularidade |
| FR-14 | Epic 2 | Editar e excluir Conta |
| FR-15 | Epic 4 | Registrar Movimentação |
| FR-16 | Epic 4 | Editar e excluir Movimentação |
| FR-17 | Epic 4 | Listar e filtrar Movimentações |
| FR-18 | Epic 1 e Epic 4 | Auditoria e `RequestLog` no Epic 1; autoria das Movimentações no Epic 4 |
| FR-19 | Epic 4 | Gerir Categorias |
| FR-20 | Epic 4 | Atribuir Categoria |
| FR-21 | Epic 5 | Lista de meses |
| FR-22 | Epic 5 | Detalhe do mês |
| FR-23 | Epic 5 | Totais por Categoria |
| FR-24 | Epic 3 | Alterar Permissão |
| NFR-1 | Epic 2 | Testado em cada epic seguinte |
| NFR-2, NFR-4, NFR-5 | Epic 1 | Mantidos nos demais epics |
| NFR-3 | Epic 4 | Centavos e limite por Movimentação |
| NFR-6 | Epic 6 | Produção |
| NFR-7 | Epic 1 | Web para computador, desde a base do frontend |
| NFR-8 | Epics 2 a 6 | Cada epic entrega os e2e do seu escopo; o Epic 5 cobre a virada do mês |
| AR-1, AR-2, AR-4 a AR-9, AR-11 a AR-13 | Epic 1 | AR-13 reaplicado nas listas seguintes |
| AR-3 | Epic 2 e Epic 4 | Repositório no Epic 2; FK composta no Epic 4 |
| AR-10 | Epic 5 | `src/util/month.ts` |
| AR-14 | Epic 2 e Epic 4 | Transação no Epic 2; índice no Epic 4 |
| AR-15 | Epic 1 | Ambiente de desenvolvimento |
| AR-16 a AR-21 | Epic 6 | Build, CI, produção, backup e e-mail; AR-20 tem `pnpm admin:create` no Epic 1 |
| AR-22, AR-23 | Epic 1 | Base do frontend |
| UX-DR1 a UX-DR4, UX-DR6, UX-DR10 (`AppDialog`), UX-DR11, UX-DR13, UX-DR20 | Epic 1 | Tema, componentes base e cabeçalho |
| UX-DR10 (`ConfirmDialog`) | Epic 2 | Primeiro uso na exclusão de Conta |
| UX-DR5 | Epic 3 | `BaseSelect`, primeiro uso na Permissão do Convite |
| UX-DR21 | Epic 2 | Conta com `isOwner` e `permission` |
| UX-DR22, UX-DR23 | Epic 3 e Epic 4 | Membros e Convites no Epic 3; Categorias no Epic 4 |
| UX-DR7, UX-DR9, UX-DR14, UX-DR18, UX-DR24 | Epic 4 | Lançamento |
| UX-DR8, UX-DR12 | Epic 5 | Valores e cartões do Balanço; `MoneyValue` já simples no Epic 4 |
| UX-DR15 a UX-DR17, UX-DR19 | Todos | Estados, telas, acessibilidade e microcopy de cada funcionalidade |

## Epic List

Cada epic entrega API e telas juntas e usa apenas os epics anteriores. O sistema fica utilizável em ambiente local ao fim do Epic 5; o uso real começa após o Epic 6.

### Epic 1: Entrar com segurança e gerir o próprio acesso
O primeiro Admin é criado, Usuários entram, renovam a sessão, recuperam a senha e editam o perfil, e o Admin cria e edita Usuários. Inclui a base técnica do backend e do frontend.
**FRs:** FR-1, FR-2, FR-3, FR-4, FR-5 (listar, editar, redefinir senha), FR-18 (auditoria e `RequestLog`)

### Epic 2: Criar e proteger as Contas da casa
Usuários criam, veem, editam e excluem Contas, com isolamento total entre elas.
**FRs:** FR-7, FR-8, FR-14

### Epic 3: Compartilhar a Conta
Donos convidam e gerem Membros e Permissões, Convidados aceitam ou recusam, Membros saem, a titularidade é transferida e o Admin exclui Usuários.
**FRs:** FR-5 (excluir), FR-9, FR-10, FR-11, FR-12, FR-13, FR-24

### Epic 4: Lançar os gastos da semana
Membros lançam em sequência pelo teclado, editam, filtram e categorizam Movimentações, com autoria registrada.
**FRs:** FR-15, FR-16, FR-17, FR-18 (autoria), FR-19, FR-20

### Epic 5: Fechar o mês
Membros veem a lista de meses e o Balanço, sabem quanto sobrou ou faltou e conferem os totais por Categoria.
**FRs:** FR-21, FR-22, FR-23

### Epic 6: Operar em produção com visão total
O Admin vê todas as Contas em leitura, e o sistema é publicado na VPS com HTTPS, backup com restauração testada e e-mail real.
**FRs:** FR-6; **NFRs:** NFR-6, fechamento do NFR-8

## Epic 1: Entrar com segurança e gerir o próprio acesso

O primeiro Admin é criado, Usuários entram, renovam a sessão, recuperam a senha e editam o perfil, e o Admin cria e edita Usuários. Inclui a base técnica do backend e do frontend.

### Story 1.1: Preparar o ambiente de desenvolvimento e testes

As a desenvolvedor,
I want subir o banco e rodar a API e os testes com um comando,
So that o projeto evolua sobre versões corrigidas e um schema limpo.

**Requisitos:** AR-1, AR-11, AR-12, AR-15, AR-20

**Acceptance Criteria:**

**Given** o repositório recém-clonado
**When** sigo o README e subo o Compose
**Then** só o banco sobe, em `127.0.0.1`, com um banco de desenvolvimento e um de teste, e a API roda no host
**And** as dependências estão nas versões do AR-12, com `.nvmrc`, `packageManager` e `.env.example` só com placeholders
**And** o script `run` antigo não existe mais

**Given** o histórico atual de migrations
**When** aplico as migrations em um banco vazio
**Then** o histórico antigo não existe mais e uma nova migration inicial cria apenas o `User`, sem `cpf`
**And** o `cpf` sai também da API e dos seeds, e os seeds de desenvolvimento não dependem de `test/`

**Given** módulos existentes que dependem de tabelas ainda não recriadas
**When** o projeto é compilado
**Then** eles saem das rotas e da compilação, e voltam na história que recriar a tabela

**Given** `app.ts` separado de `server.ts`
**When** rodo os e2e
**Then** o supertest usa `DATABASE_URL_TEST`, e a limpeza recusa banco cujo nome não termine em `_test`

### Story 1.2: Proteger e auditar toda escrita no banco

As a responsável pelo sistema,
I want que toda escrita seja transacional e registrada,
So that eu saiba quem alterou o quê e nada seja gravado pela metade.

**Requisitos:** FR-18, NFR-4, AR-2, AR-5, AR-6

**Acceptance Criteria:**

**Given** um use case que escreve em `uow.run(actor, fn)`
**When** o callback retorna `Wrong` ou lança exceção
**Then** nada é confirmado, inclusive a auditoria

**Given** uma escrita bem-sucedida
**When** consulto `AuditLog`
**Then** há instante, autor, modelo, id, operação e estados anterior e posterior, com `password` e `*Hash` mascarados

**Given** o cliente da aplicação
**When** o código usa uma operação proibida do AD-11 ou escreve sem actor
**Then** recebe `ForbiddenOperationError`, e o valor `PrismaClient` fora de `src/database/` e dos testes falha no lint

**Given** qualquer requisição concluída
**When** consulto `RequestLog`
**Then** há método, rota normalizada, status, duração e autor, sem corpo, query string ou headers
**And** uma falha ao gravar o log não altera a resposta

**Given** um erro Prisma ou de regra
**When** a API responde
**Then** o status segue o AD-17, com corpo `{ code, error, slug }`, e regra violada nunca retorna 500

### Story 1.3: Criar o primeiro Admin e fazer login

As a Usuário,
I want entrar com e-mail e senha e sair quando quiser,
So that eu use o sistema com segurança.

**Requisitos:** FR-1, NFR-2, NFR-5, NFR-6, AR-4, AR-7, AR-8, AR-20, UX-DR20

**Acceptance Criteria:**

**Given** a variável com a senha inicial
**When** rodo `pnpm admin:create` duas vezes
**Then** existe um único Admin, e a segunda execução não altera nada

**Given** e-mail e senha válidos, com o e-mail em qualquer caixa
**When** chamo `POST /api/login`
**Then** recebo um JWT de 15 minutos com `sub` e `sessionId` e o cookie de renovação com os atributos do AD-6
**And** nenhuma resposta contém senha ou hash

**Given** e-mail inexistente, senha errada ou Usuário com `deletedAt`
**When** tento entrar
**Then** a resposta é a mesma nos três casos, e o hash é comparado mesmo sem Usuário

**Given** 5 tentativas com o mesmo e-mail em 15 minutos
**When** faço a sexta
**Then** recebo 429

**Given** uma sessão ativa
**When** faço logout e reutilizo o token de acesso
**Then** recebo 401 imediatamente

**Given** um token válido
**When** chamo `GET /api/me`
**Then** recebo `id`, `name`, `email` e `isAdmin`
**And** o `actor` vem só do middleware, com `isAdmin` lido do banco
**And** as mensagens de validação citam a entidade correta (D-21)

### Story 1.4: Manter a sessão ativa enquanto uso o sistema

As a Usuário,
I want continuar logado enquanto uso o sistema,
So that eu não seja interrompido a cada 15 minutos.

**Requisitos:** FR-1, NFR-5, AR-7

**Acceptance Criteria:**

**Given** um cookie de renovação válido e o header `X-Requested-With`
**When** chamo `POST /api/refresh`
**Then** recebo novo token de acesso, o token de renovação é trocado, e a Sessão vale por mais 7 dias

**Given** duas abas renovando juntas
**When** o token anterior é usado em até 30 segundos
**Then** recebe novo token de acesso sem nova rotação

**Given** o token anterior usado depois de 30 segundos
**When** chamo a renovação
**Then** recebo 401 e a Sessão é revogada

**Given** a ausência do header, um token de acesso no lugar do de renovação ou uma Sessão com mais de 7 dias sem uso
**When** chamo a renovação
**Then** recebo 401

### Story 1.5: Entrar pela tela de Login

As a Usuário,
I want entrar por uma tela simples e continuar logado ao reabrir o app,
So that eu use o sistema pelo navegador.

**Requisitos:** FR-1, NFR-7, AR-22, AR-23, UX-DR1, UX-DR2, UX-DR3, UX-DR4, UX-DR11, UX-DR13, UX-DR15, UX-DR16, UX-DR17, UX-DR19

**Acceptance Criteria:**

**Given** o projeto `my-finances-frontend` criado com a stack do AR-22 e os tokens do `DESIGN.md`
**When** abro o app sem sessão
**Then** vejo o indicador de abertura e depois `/login`

**Given** credenciais inválidas
**When** envio o formulário
**Then** vejo "E-mail ou senha incorretos.", com os valores preservados e o erro anunciado ao leitor de tela

**Given** um erro de rede ou 5xx
**When** envio o formulário
**Then** vejo o toast "Não foi possível concluir. Tente de novo.", e os dados permanecem

**Given** login bem-sucedido
**When** entro
**Then** vou para `/`, com o `AppHeader` e o menu do usuário
**And** o token fica só em memória

**Given** várias requisições com token vencido
**When** todas recebem 401
**Then** há uma única renovação compartilhada; se ela falhar, vou ao Login com a rota de volta

**Given** o menu do usuário
**When** alterno o tema
**Then** a escolha fica no navegador; na primeira visita, o tema segue o sistema

**Given** a alternância e o Login
**When** uso só o teclado
**Then** tudo funciona, com `BaseButton`, `BaseInput` e `AppToast` conforme o `EXPERIENCE.md`

### Story 1.6: Editar meu perfil e trocar minha senha

As a Usuário,
I want manter meus dados atualizados e trocar minha senha,
So that eu continue com acesso seguro.

**Requisitos:** FR-3, AR-8, UX-DR16

**Acceptance Criteria:**

**Given** a tela `/perfil`
**When** altero nome ou e-mail para um valor livre, ou mantenho o meu próprio e-mail
**Then** os dados são salvos sem conflito

**Given** um e-mail já usado, inclusive por Usuário excluído
**When** salvo
**Then** recebo 409, com a mensagem no campo

**Given** a senha atual correta e uma nova que segue `zPassword`, com qualquer símbolo
**When** troco a senha
**Then** todas as minhas Sessões são revogadas e volto ao Login

**Given** a senha atual errada
**When** tento trocar
**Then** recebo erro no campo e as Sessões continuam ativas

**Given** um `id` de outro Usuário no corpo
**When** edito o perfil
**Then** só meus próprios dados mudam

### Story 1.7: Recuperar o acesso por e-mail

As a Usuário que esqueceu a senha,
I want receber um link para criar outra,
So that eu volte a entrar sem depender do Admin.

**Requisitos:** FR-2, AR-9, UX-DR16

**Acceptance Criteria:**

**Given** a tela `/senha/esqueci`
**When** informo um e-mail, cadastrado ou não
**Then** a API responde 202 com corpo fixo, e a tela é igual nos dois casos
**And** uma falha no `EmailSender` só vai para o log

**Given** um link com menos de 1 hora, montado com `APP_BASE_URL`
**When** defino uma senha válida em `/senha/nova`
**Then** a senha muda, todas as Sessões são revogadas e vou ao Login

**Given** um link já usado ou vencido
**When** tento usá-lo
**Then** recebo 400 e a tela oferece pedir outro link

**Given** o banco
**When** inspeciono o token de recuperação
**Then** só existe o hash, e o token não aparece em logs

### Story 1.8: Admin cria e lista Usuários

As a Admin,
I want criar logins e ver quem usa o sistema,
So that eu dê acesso às pessoas da casa.

**Requisitos:** FR-4, FR-5, AR-4, AR-13, UX-DR6, UX-DR10, UX-DR15, UX-DR16

**Acceptance Criteria:**

**Given** um Admin em `/admin/usuarios`
**When** crio um Usuário com nome, e-mail e senha inicial
**Then** recebo 201 com `Location`, o Usuário consegue entrar, e nenhum campo de CPF é pedido

**Given** um e-mail já usado, inclusive por Usuário excluído
**When** crio o Usuário
**Then** recebo 409, com a mensagem no campo

**Given** a lista de Usuários
**When** navego pelas páginas
**Then** vejo a `BaseTable` paginada, sem senha ou hash na resposta

**Given** um não Admin
**When** chamo `/api/admin/*` ou abro a rota
**Then** a API retorna 403 e a tela não fica acessível
**And** o item Admin do menu aparece só para Admin

**Given** uma lista carregando
**When** a tela aguarda a API
**Then** vejo linhas de esqueleto, padrão reutilizado nas demais listas

### Story 1.9: Admin edita e redefine a senha de Usuários

As a Admin,
I want corrigir dados e redefinir senhas,
So that eu ajude quem perdeu o acesso.

**Requisitos:** FR-5

**Acceptance Criteria:**

**Given** um Usuário existente
**When** o Admin edita nome ou e-mail
**Then** os dados mudam, com as mesmas regras de unicidade do FR-3

**Given** uma nova senha que segue `zPassword`
**When** o Admin a redefine
**Then** todas as Sessões daquele Usuário são revogadas

**Given** um Usuário não Admin
**When** chama essas rotas
**Then** recebe 403 e nada muda

## Epic 2: Criar e proteger as Contas da casa

Usuários criam, veem, editam e excluem Contas, com isolamento total entre elas.

### Story 2.1: Criar uma Conta e ver minhas Contas

As a Usuário,
I want criar uma Conta e ver as Contas de que participo,
So that eu comece a organizar o dinheiro da casa.

**Requisitos:** FR-7, FR-8, NFR-1, NFR-8, AR-3, AR-14, UX-DR16, UX-DR21

**Acceptance Criteria:**

**Given** um Usuário em `/` sem Contas
**When** a tela carrega
**Then** vê "Você ainda não participa de nenhuma Conta." e o botão Criar Conta

**Given** o diálogo de criar Conta
**When** informo o nome e, se quiser, um Saldo inicial, que pode ser negativo
**Then** recebo 201 com `Location`, a Conta aparece na lista na hora e eu sou o Dono
**And** `Account.ownerId` e o vínculo com `MANAGEMENT` são gravados no mesmo Unit of Work

**Given** a lista de Contas
**When** chamo `GET /api/account`
**Then** recebo só Contas sem `deletedAt` das quais sou Membro, com `isOwner` e `permission`

**Given** a migration desta história
**When** é aplicada
**Then** cria `Account` e `UserAccount`, e `AccountScopedRepository` passa a exigir `accountId` em toda operação

**Given** uma Conta de outro Usuário
**When** chamo `GET /api/account/:id`, inclusive como Admin que não é Membro
**Then** recebo 404
**And** um teste e2e cobre esse isolamento

### Story 2.2: Abrir a Conta e editar nome e Saldo inicial

As a Membro com Gestão,
I want abrir a Conta e corrigir seu nome e Saldo inicial,
So that os dados fiquem certos.

**Requisitos:** FR-14, NFR-1, UX-DR15, UX-DR16, UX-DR21

**Acceptance Criteria:**

**Given** uma Conta de que sou Membro
**When** abro `/contas/:id`
**Then** vejo o nome e o Saldo inicial, com "Nenhuma Movimentação ainda."
**And** `GET /api/account/:id` retorna `isOwner` e `permission`

**Given** a tela `/contas/:id/ajustes` e a Permissão de Gestão
**When** altero nome ou Saldo inicial
**Then** a alteração é salva e confirmada por toast

**Given** um Membro com Lançamento
**When** chama `PATCH /api/account/:id`
**Then** recebe 403, a Permissão é recarregada na tela e a ação fica oculta

**Given** um não Membro ou um id inexistente
**When** abro a rota ou chamo a API
**Then** recebo 404 e vejo "Conta não encontrada." com link para Minhas Contas

**Given** um `accountId` no corpo diferente do caminho
**When** edito a Conta
**Then** vale só o do caminho

### Story 2.3: Excluir uma Conta

As a Dono,
I want excluir uma Conta que não uso mais,
So that ninguém continue acessando.

**Requisitos:** FR-14, NFR-1, UX-DR10

**Acceptance Criteria:**

**Given** o Dono em Ajustes
**When** clica em Excluir
**Then** o `ConfirmDialog` mostra "Excluir esta Conta? Ninguém mais terá acesso.", com foco inicial em Cancelar

**Given** a confirmação
**When** a exclusão é concluída
**Then** a API responde 204, grava `deletedAt` sob o lock da Conta e os dados permanecem no banco

**Given** uma Conta excluída
**When** um Membro a lista, abre ou altera
**Then** ela não aparece na lista e as rotas respondem 404

**Given** um Membro que não é o Dono, mesmo com Gestão
**When** chama `DELETE /api/account/:id`
**Then** recebe 403, e o botão não aparece

## Epic 3: Compartilhar a Conta

Donos convidam e gerem Membros e Permissões, Convidados aceitam ou recusam, Membros saem, a titularidade é transferida e o Admin exclui Usuários.

### Story 3.1: Convidar um Usuário para a Conta

As a Dono,
I want convidar alguém por e-mail e definir sua Permissão,
So that eu divida a Conta com quem mora comigo.

**Requisitos:** FR-9, UX-DR5, UX-DR15, UX-DR16, UX-DR22

**Acceptance Criteria:**

**Given** um Membro em `/contas/:id/membros`
**When** a tela carrega
**Then** vê os Membros com nome e Permissão, e só o Dono vê as ações de convite

**Given** o Dono no diálogo de Convite, com `BaseSelect` de Permissão
**When** informa o e-mail de um Usuário existente
**Then** recebe 201 com `{ emailDelivered }`, e o Convite fica pendente por 7 dias

**Given** um e-mail sem login, de Usuário excluído, de quem já é Membro ou com Convite pendente válido
**When** o Dono envia
**Then** recebe um erro claro e nenhum Convite é criado

**Given** um Convite pendente vencido para o mesmo Usuário
**When** o Dono envia outro
**Then** o antigo vira `EXPIRED` e o novo é criado no mesmo Unit of Work

**Given** uma falha no envio do e-mail
**When** o Convite é criado
**Then** ele permanece salvo e a tela mostra "Convite criado, mas o e-mail não foi enviado." com Reenviar
**And** reenviar não altera `expiresAt`

**Given** um Membro que não é o Dono
**When** chama a API de Convite
**Then** recebe 403

**Given** uma Conta com Convites pendentes
**When** o Dono a exclui
**Then** os Convites viram `CANCELLED`

### Story 3.2: Aceitar ou recusar um Convite

As a convidado,
I want ver meus Convites e responder a cada um,
So that eu entre nas Contas certas.

**Requisitos:** FR-10, UX-DR13, UX-DR23

**Acceptance Criteria:**

**Given** Convites pendentes
**When** entro em Minhas Contas
**Then** vejo o nome da Conta e a Permissão de cada um, com o indicador no `AppHeader`

**Given** um Convite pendente e válido
**When** aceito
**Then** viro Membro com a Permissão definida, sob o lock da Conta, e a Conta entra na lista na hora

**Given** um Convite pendente
**When** recuso
**Then** ele não pode mais ser aceito, e o Dono pode enviar outro

**Given** um Convite vencido
**When** tento aceitar
**Then** vejo "Este Convite expirou. Peça um novo ao Dono."

**Given** um Convite de outro Usuário
**When** chamo aceitar ou recusar
**Then** recebo 404

**Given** um Convite de Conta excluída, ou um convidado que já virou Membro
**When** tento aceitar
**Then** recebo erro claro e nenhum vínculo é criado

### Story 3.3: Alterar a Permissão e remover Membros

As a Dono,
I want ajustar o que cada Membro pode fazer e tirar quem não deve mais acessar,
So that eu mantenha o controle da Conta.

**Requisitos:** FR-11, FR-24, NFR-8

**Acceptance Criteria:**

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

### Story 3.4: Sair de uma Conta

As a Membro,
I want sair de uma Conta,
So that eu deixe de participar quando fizer sentido.

**Requisitos:** FR-12

**Acceptance Criteria:**

**Given** um Membro que não é o Dono
**When** sai e confirma no `ConfirmDialog`
**Then** o vínculo é apagado sob o lock da Conta, perco o acesso imediatamente e volto para Minhas Contas

**Given** o Dono
**When** tenta sair
**Then** recebe erro explicando que precisa transferir a titularidade antes

### Story 3.5: Transferir a titularidade

As a Dono,
I want passar a Conta para outro Membro,
So that ela continue tendo um responsável.

**Requisitos:** FR-13, NFR-8

**Acceptance Criteria:**

**Given** o Dono e outro Membro da Conta
**When** transfere com `{ newOwnerId, previousOwnerPermission }` e confirma
**Then** `ownerId` e as duas Permissões mudam em um único Unit of Work, sob o lock da Conta
**And** o novo Dono fica com Gestão, e o antigo fica com a Permissão escolhida

**Given** um Admin, mesmo sem ser Membro
**When** chama `POST /api/admin/account/:accountId/transfer`
**Then** a transferência funciona pelas mesmas regras, e `previousOwnerPermission` é definida pelo Admin

**Given** um alvo que não é Membro, ou uma Conta excluída
**When** a transferência é tentada
**Then** recebe erro claro e nada muda

**Given** um Membro que não é o Dono
**When** chama a transferência
**Then** recebe 403

**Given** duas transferências simultâneas
**When** ambas executam
**Then** a Conta termina com exatamente um Dono, que tem vínculo com Gestão

### Story 3.6: Admin exclui um Usuário

As a Admin,
I want excluir um login,
So that a pessoa não acesse mais o sistema.

**Requisitos:** FR-5, NFR-8

**Acceptance Criteria:**

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

## Epic 4: Lançar os gastos da semana

Membros lançam em sequência pelo teclado, editam, filtram e categorizam Movimentações, com autoria registrada.

### Story 4.1: Registrar uma Movimentação

As a Membro,
I want registrar uma Entrada ou Saída,
So that o dinheiro da casa fique anotado.

**Requisitos:** FR-15, FR-18, NFR-3, AR-14

**Acceptance Criteria:**

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

### Story 4.2: Lançar em sequência só pelo teclado

As a Membro,
I want lançar o extrato inteiro sem tocar no mouse,
So that eu termine rápido.

**Requisitos:** FR-15, UX-DR7, UX-DR9, UX-DR14, UX-DR15, UX-DR16, UX-DR17, UX-DR18, UX-DR19

**Acceptance Criteria:**

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

### Story 4.3: Listar Movimentações com filtros

As a Membro,
I want consultar as Movimentações da Conta por período,
So that eu ache o que foi lançado.

**Requisitos:** FR-17, NFR-1, NFR-8, AR-13

**Acceptance Criteria:**

**Given** uma Conta com Movimentações
**When** chamo `GET /api/account/:accountId/transaction`
**Then** recebo a lista paginada, ordenada por `occurredAt desc, createdAt desc, id desc`

**Given** `from` e `to` em `YYYY-MM-DD`
**When** filtro
**Then** ambos são inclusivos no calendário de `America/Sao_Paulo`

**Given** Movimentações em outra Conta
**When** listo ou uso um id de outra Conta
**Then** elas nunca aparecem, e um e2e cobre esse isolamento

**Given** filtros
**When** a query é lida
**Then** eles vêm do schema Zod do use case, sem `autoParseFilters`

### Story 4.4: Editar e excluir Movimentações

As a Membro,
I want corrigir ou apagar qualquer lançamento da Conta,
So that os números fiquem certos.

**Requisitos:** FR-16, FR-18

**Acceptance Criteria:**

**Given** um item de "Lançadas agora", lançado por qualquer Membro
**When** edito no `AppDialog`
**Then** a alteração é salva pelas mesmas validações, a tela reflete o novo valor e o autor fica no `AuditLog`

**Given** o `ConfirmDialog` que nomeia a Movimentação
**When** confirmo a exclusão
**Then** a API responde 204, e a remoção física fica registrada no `AuditLog` com o autor

**Given** um id de Movimentação de outra Conta na rota da minha Conta
**When** edito ou excluo
**Then** recebo 404 e nada muda

**Given** um Membro com Lançamento
**When** edita ou exclui Movimentação lançada por outro
**Then** a operação é aceita

### Story 4.5: Gerenciar as Categorias da Conta

As a Membro com Gestão,
I want criar, renomear e excluir Categorias,
So that os gastos fiquem agrupados do jeito da casa.

**Requisitos:** FR-19, AR-3, UX-DR16, UX-DR22

**Acceptance Criteria:**

**Given** a migration desta história
**When** é aplicada
**Then** cria `Category` com `@@unique([accountId, name])` e `Transaction.categoryId` com FK composta `(accountId, categoryId)`

**Given** `/contas/:id/categorias` e a Permissão de Gestão
**When** crio ou renomeio com nome válido
**Then** a mudança é salva, com `trim` no nome

**Given** um nome já usado na Conta
**When** salvo
**Then** recebo 409, com a mensagem no campo

**Given** uma Categoria em uso
**When** a excluo e confirmo
**Then** as Movimentações ficam sem Categoria, linha a linha no mesmo Unit of Work, e continuam existindo

**Given** um Membro com Lançamento
**When** chama as rotas de escrita
**Then** recebe 403, mas consegue listar, e as ações ficam ocultas

### Story 4.6: Categorizar Movimentações

As a Membro,
I want escolher a Categoria ao lançar ou editar,
So that eu saiba para onde foi o dinheiro.

**Requisitos:** FR-20, FR-17, UX-DR15, UX-DR24

**Acceptance Criteria:**

**Given** o `EntryForm`
**When** chego à Categoria depois do Tipo
**Then** vejo o `BaseSelect` com "Sem categoria" primeiro e busca por letra, e a Categoria é limpa depois de salvar

**Given** uma Conta sem Categorias
**When** abro Lançar
**Then** só existe "Sem categoria", e quem tem Gestão vê link para criar

**Given** uma Movimentação existente
**When** envio `categoryId` com outro valor ou `null`
**Then** a Categoria é trocada ou removida, e campo ausente não muda nada

**Given** uma Categoria de outra Conta
**When** a atribuo
**Then** recebo 409 e nada muda

**Given** a lista de Movimentações
**When** chamo a API, com ou sem filtro `categoryId`
**Then** cada item traz `categoryId` e `categoryName`, e o filtro retorna só aquela Categoria

## Epic 5: Fechar o mês

Membros veem a lista de meses e o Balanço, sabem quanto sobrou ou faltou e conferem os totais por Categoria.

### Story 5.1: Ver a lista de meses da Conta

As a Membro,
I want ver cada mês com o que entrou, saiu e sobrou,
So that eu acompanhe a casa ao longo do tempo.

**Requisitos:** FR-21, FR-14, NFR-8, AR-10, UX-DR8, UX-DR15

**Acceptance Criteria:**

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

### Story 5.2: Abrir um mês e saber se sobrou ou faltou

As a Membro,
I want abrir um mês e ver o resultado e as Movimentações,
So that eu feche as contas com quem divide a casa.

**Requisitos:** FR-22, FR-17, UX-DR12, UX-DR15, UX-DR16, UX-DR19

**Acceptance Criteria:**

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

### Story 5.3: Ver os totais por Categoria

As a Membro,
I want ver quanto foi para cada Categoria no mês,
So that eu me organize no mês seguinte.

**Requisitos:** FR-23, NFR-8

**Acceptance Criteria:**

**Given** um mês com Movimentações categorizadas e sem Categoria
**When** abro o Mês
**Then** vejo os totais de cada Categoria, separando Entradas e Saídas, e um único grupo "Sem categoria"

**Given** os grupos
**When** somo seus valores
**Then** o resultado é igual ao total de Entradas e ao de Saídas do mês, e um e2e confirma

**Given** uma Categoria excluída
**When** o Balanço é recalculado
**Then** seus valores passam para "Sem categoria"

**Given** Categorias sem Movimentação no mês
**When** o mês é exibido
**Then** elas não aparecem

## Epic 6: Operar em produção com visão total

O Admin vê todas as Contas em leitura, e o sistema é publicado na VPS com HTTPS, backup com restauração testada e e-mail real.

### Story 6.1: Admin vê todas as Contas

As a Admin,
I want ver todas as Contas, seus Membros e Movimentações,
So that eu tenha a visão total do sistema.

**Requisitos:** FR-6, NFR-1, NFR-8, UX-DR15, UX-DR16

**Acceptance Criteria:**

**Given** um Admin em `/admin/contas`
**When** a tela carrega
**Then** vê todas as Contas, paginadas, e as excluídas têm a etiqueta "Excluída"

**Given** uma Conta, excluída ou não
**When** o Admin a abre
**Then** vê os Membros e as Movimentações, somente em leitura e pelas rotas `/api/admin/*`

**Given** um Admin que não é Membro da Conta
**When** chama as rotas normais de escrita ou leitura dela
**Then** recebe 404

**Given** um não Admin
**When** chama `/api/admin/account*` ou abre a rota
**Then** a API retorna 403 e a tela não fica acessível

### Story 6.2: Admin transfere a titularidade pela tela

As a Admin,
I want transferir uma Conta para outro Membro,
So that eu resolva uma Conta cujo Dono vai sair do sistema.

**Requisitos:** FR-6, FR-13

**Acceptance Criteria:**

**Given** uma Conta sem `deletedAt`, em Admin: Contas
**When** o Admin escolhe o novo Dono e a Permissão do antigo e confirma no `ConfirmDialog`
**Then** a tela usa a rota da Story 3.5 e mostra o novo Dono

**Given** uma Conta excluída
**When** o Admin a abre
**Then** a ação de transferir não aparece

**Given** a titularidade já transferida
**When** o Admin exclui o antigo Dono
**Then** a exclusão da Story 3.6 é aceita

### Story 6.3: Gerar builds reproduzíveis com CI

As a desenvolvedor,
I want que cada commit seja verificado e a `main` gere imagens,
So that eu publique sempre o que foi testado.

**Requisitos:** NFR-6, AR-16, AR-17

**Acceptance Criteria:**

**Given** um push ou PR no backend ou no frontend
**When** o CI roda com pnpm
**Then** executa lint, `tsc`, testes e build, com PostgreSQL real nos e2e do backend

**Given** um merge na `main`
**When** o CI termina com sucesso
**Then** publica as imagens marcadas pelo commit

**Given** o `Dockerfile` multi-stage do backend
**When** construo os alvos
**Then** há imagem de produção sem root e sem dependências de desenvolvimento, e um alvo `migrate` com a CLI do Prisma

**Given** `.env.example` e a configuração
**When** a API sobe com o `JWT_SECRET` de exemplo
**Then** ela recusa iniciar

### Story 6.4: Publicar o sistema na VPS com HTTPS

As a responsável pelo sistema,
I want publicá-lo em um endereço seguro,
So that a casa use de verdade.

**Requisitos:** NFR-6, AR-18, AR-20

**Acceptance Criteria:**

**Given** a VPS com Docker Compose
**When** o ambiente sobe
**Then** o proxy serve `/` e `/api` na mesma origem por HTTPS, com Postgres sem porta pública e `restart: unless-stopped`

**Given** a API em produção
**When** recebe requisições pelo proxy
**Then** usa `trust proxy = 1`, `helmet` e CORS fechado, e o cookie sai com `Secure`

**Given** um deploy
**When** executo o procedimento documentado
**Then** a imagem do commit é baixada, o alvo `migrate` roda e a API sobe, e `/health` confirma o banco

**Given** uma versão com problema
**When** faço o rollback documentado
**Then** a imagem anterior volta a funcionar

**Given** segredos de produção
**When** inspeciono repositório e imagens
**Then** nenhum segredo está neles, e o `JWT_SECRET` é novo

**Given** o ambiente novo
**When** rodo `pnpm admin:create`
**Then** o primeiro Admin existe, e nenhum seed de desenvolvimento rodou

### Story 6.5: Garantir backup com restauração testada

As a responsável pelo sistema,
I want backups diários que funcionem,
So that o histórico da casa não se perca.

**Requisitos:** NFR-6, AR-19

**Acceptance Criteria:**

**Given** o banco de produção
**When** passa um dia
**Then** um `pg_dump` é gerado e copiado para fora da VPS, e uma falha fica visível nos logs

**Given** um backup real
**When** executo o procedimento de restauração em um banco separado
**Then** Usuários, Contas e Movimentações voltam íntegros, e o Balanço confere
**And** a data do teste fica registrada, como condição para o uso real

### Story 6.6: Enviar e-mails reais e liberar para uso

As a Usuário,
I want receber Convites e links de recuperação de verdade,
So that eu use esses recursos em produção.

**Requisitos:** NFR-6, NFR-8, AR-21

**Acceptance Criteria:**

**Given** o provedor escolhido, com SPF e DKIM configurados
**When** a API envia um e-mail
**Then** a implementação real de `EmailSender` entrega Convite e recuperação com links de `APP_BASE_URL`, sem mudar os use cases

**Given** o provedor fora do ar
**When** um Convite é criado ou a recuperação é pedida
**Then** valem os comportamentos das Stories 3.1 e 1.7

**Given** a suíte de e2e
**When** confiro cada item do NFR-8
**Then** isolamento, Permissões, poderes de Dono e Admin e Balanço na virada do mês têm testes passando

**Given** a produção publicada
**When** percorro o checklist de liberação
**Then** login limitado a 5 tentativas, HTTPS, backup restaurado e e-mail real estão confirmados, e o SM-1 pode começar
