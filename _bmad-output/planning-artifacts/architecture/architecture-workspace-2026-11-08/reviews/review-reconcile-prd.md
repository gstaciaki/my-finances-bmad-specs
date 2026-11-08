# Revisão de reconciliação: Architecture Spine x PRD e addendum

**Data:** 2026-11-08
**Revisado:** `ARCHITECTURE-SPINE.md` e `.memlog.md` (esta pasta de arquitetura)
**Contra:** `prds/prd-workspace-2026-11-07/prd.md` e `addendum.md`
**Consulta de apoio:** `levantamento-my-finances-backend.md` e `my-finances-backend/prisma/schema.prisma`, só para confirmar o estado atual dos itens citados no addendum.

**Fora do relatório, por serem mudanças conhecidas e intencionais:** desativação fundida em um único `deletedAt`, auditoria ampliada a todas as escritas e requisições HTTP, `occurredAt` como timestamp.

## Veredito

**Não reconciliado: ajustar antes de aprovar o spine.**

A espinha cobre bem a maior parte do PRD: isolamento, sessão, dinheiro, fuso, unidade de trabalho e Balanço. Porém, os ADs foram escritos em torno de "dado de Conta acessado por Membro", e o que foge desse formato ficou sem regra ou em conflito com ela. Isso afeta o Admin, o convidado e as leituras por Usuário. Há ainda uma mudança de escopo não declarada (frontend opcional) e requisitos pequenos que sumiram por não terem AD próprio.

| Severidade | Quantidade |
| --- | --- |
| Alta | 4 |
| Média | 8 |
| Baixa | 9 |

## Achados de severidade alta

### A1. Não há regra de autorização nem origem para o Admin (FR-4, FR-5, FR-6)

- **PRD:** só o Admin cria, lista, edita, redefine a senha e exclui Usuários (FR-4, FR-5). O glossário define o Admin como um tipo de Usuário.
- **Spine:** o AD-2 vale para "todo use case de dado de Conta" e afirma "Middleware só autentica". Criar e gerir Usuários não é dado de Conta, então nenhuma regra exige que o ator seja Admin nas rotas `/admin/user`. O FR-4 nem aparece nos `Binds` do AD-2, embora o mapa de capacidades atribua FR-4 a FR-6 ao AD-2 e ao AD-9.
- **Lacuna adicional:** `actor { userId, isAdmin }` aparece no AD-2, mas nenhum AD define de onde `isAdmin` vem. O JWT do AD-5 só traz `sub` e `sessionId`, e o AD-2 lista como lidos do banco apenas "vínculo, Permissão e Dono". O campo não existe hoje (D-08 do levantamento: não há coluna nem papel, e o token sai sempre com `admin: false`), e o seed estrutural não o menciona.
- **Sugestão:** estender o AD-2 (ou criar um AD) com: coluna em `User`, leitura do banco a cada requisição, e verificação de Admin no use case de toda operação administrativa.

### A2. O convidado e as leituras por Usuário colidem com AD-2 e AD-3 (FR-8, FR-10, FR-5)

- **PRD:** o convidado vê seus Convites pendentes ao entrar e aceita ou recusa cada um (FR-10). O Usuário vê a lista das próprias Contas (FR-8).
- **Spine:**
  - O AD-2 responde 404 a não Membro em todo use case de dado de Conta, e o convidado ainda não é Membro. Nenhuma exceção está prevista, embora os `Binds` do AD-2 incluam o FR-10.
  - O AD-3 exige `accountId` em toda operação dos repositórios de Convite e Membro. Listar Convites do Usuário em `/api/invitation`, listar as Contas do Usuário (FR-8) e apagar todos os vínculos de um Usuário excluído (AD-9) são, por natureza, consultas por `userId` em várias Contas.
- **Risco:** aplicado à risca, o FR-10 não é implementável. Sem regra, cada implementador abrirá uma exceção diferente justamente no ponto que o AD-3 protege (issue #25).
- **Sugestão:** definir o caminho por Usuário: consultas sempre com `userId` do ator como parâmetro obrigatório, e autorização de Convite por "o ator é o convidado". Vale definir também que aceitar Convite de Conta com `deletedAt` retorna 404.

### A3. Convite expirado continua bloqueando novo Convite (FR-9, FR-10)

- **PRD:** "Um Convite recusado ou expirado não pode mais ser aceito. O Dono pode enviar um novo Convite."
- **Spine:** o AD-16 compara `expiresAt` no uso, sem job, e usa um índice único parcial contra dois Convites pendentes. Predicado de índice no PostgreSQL precisa ser imutável, logo não pode usar `now()`. Um Convite vencido continua com estado pendente na linha e faz o novo Convite falhar com 409 (AD-17), contrariando o FR-10.
- **Sugestão:** registrar que criar Convite primeiro encerra, na mesma transação, o pendente vencido do par Conta e Usuário. Definir também se o reenvio do AD-15 renova o `expiresAt` ou só repete o e-mail.

### A4. O spine torna o frontend opcional, mas o PRD o inclui na v1 (seção 7, NFR-7, SM-1)

- **PRD:** a v1 é "tudo o que está na seção 4, com os requisitos não funcionais da seção 5, publicado em produção". O item 7 da ordem de construção é "Frontend web", o NFR-7 define "aplicação web", e FR-2, FR-15, as jornadas e SM-1 pressupõem telas.
- **Spine e memlog:** o frontmatter diz "frontend web opcional", a seção Deferred adia a "Decisão de construir o frontend" e a convenção diz "se construído". O memlog registra as duas posições: "Escopo inclui o frontend web" (linha 14) e "ainda não decidida pelo usuário" (linha 53).
- **Observação:** é uma decisão legítima de Guilherme, mas não consta das mudanças conhecidas e o PRD não foi alterado. Sem frontend, FR-10 ("ao entrar"), UJ-1, UJ-2 e SM-1 não se cumprem como escritos.
- **Sugestão:** decidir: ou o spine trata o frontend como parte da v1, ou o PRD reduz o escopo e revê as métricas.

## Achados de severidade média

### M1. Não revelar existência de e-mail não virou regra, e o AD-15 pode quebrá-la (FR-1, FR-2)

O PRD exige a mesma mensagem para e-mail inexistente e senha errada, e a mesma resposta na recuperação exista ou não o e-mail. Nenhum AD trata disso. O AD-15, que vincula o FR-2, diz que em falha de envio "a resposta informa a falha", redigido para Convite. Se aplicado à recuperação, só e-mails existentes poderiam produzir essa resposta, revelando quais estão cadastrados. O envio direto dentro da requisição também cria diferença de tempo entre os dois casos.
**Sugestão:** AD ou cláusula própria: resposta idêntica no login e na recuperação, e falha de envio na recuperação só registrada no servidor.

### M2. Restauração testada do backup foi perdida (NFR-6)

O NFR-6 exige backup diário "com restauração testada ao menos uma vez antes do uso real". O backup aparece só no diagrama e na linha Deferred ("script de backup"), e a restauração não aparece em lugar nenhum. O mapa associa o NFR-6 a AD-18 e AD-19, que não falam de backup. A decisão do memlog (linha 47) também não virou AD. A justificativa "não afetam o código" não elimina o critério de saída.
**Sugestão:** incluir no AD-18 o backup diário e o teste de restauração como condição para uso real.

### M3. Violação de chave estrangeira segue virando 500 (NFR-4)

O AD-11 define FKs de negócio como `Restrict`, mas o AD-17 só mapeia `P2002` e `P2025`. `P2003` cairia no `UnknownError`, exatamente o defeito D-06 do levantamento (excluir vinculado retorna 500, com P2003), e também em Categoria inexistente ou corrida entre requisições.
**Sugestão:** acrescentar `P2003` ao AD-17, com status definido.

### M4. Remoção do CPF não aparece (NFR-2, FR-4)

O PRD diz que o sistema não coleta CPF e o FR-4 repete isso. O schema atual tem `cpf String @unique` obrigatório. O spine não menciona a remoção, e os `Binds` de NFR-2 apontam para ADs que tratam de log e acesso. Sem a migração, o FR-4 não funciona, pois a coluna é obrigatória.
**Sugestão:** registrar a remoção da coluna, do value object, do mapper e do seed (já detalhada no levantamento).

### M5. Exibição e data padrão no fuso de São Paulo ficaram de fora (FR-15, FR-22)

O memlog decidiu "frontend exibe sempre nesse fuso" (linhas 25 e 30), mas o AD-8 omite isso. Também não diz quem calcula o padrão "hoje em `America/Sao_Paulo`" do FR-15 nem o que ocorre quando a data é omitida. Um navegador em outro fuso quebra o exemplo do FR-22 (23h do último dia do mês).
**Sugestão:** completar o AD-8 com a exibição e a regra da data padrão.

### M6. Categoria de outra Conta na Movimentação não está impedida de forma explícita (FR-20, NFR-1)

O AD-3 obriga busca por `{ id, accountId }`, mas o `categoryId` recebido na escrita de Movimentação é uma FK escalar (AD-11) e nada exige validar que ela pertence à Conta da rota. Uma Categoria alheia passaria pela FK e vazaria seu nome pelo Balanço e pelas listas.
**Sugestão:** exigir a verificação no use case ou uma FK composta `(categoryId, accountId)`.

### M7. Política de senha e D-21 ficaram sem lugar (FR-2, FR-3, FR-4, addendum D-20, D-21)

O addendum envia à arquitetura a correção do regex de símbolos (D-20) e das mensagens com a entidade errada (D-21). O FR-2 também cita "a política de senhas do sistema". O spine não define a política, onde ela vive nem menciona os dois itens, que também não estão em Deferred.
**Sugestão:** uma linha nas convenções apontando schema Zod único de senha, e registrar D-20 e D-21 como correções da v1.

### M8. Seed de produção e primeiro Admin não têm caminho (addendum D-17, seção 2.2)

O addendum exige que o seed não dependa de código de testes, e isso não está no AD-18. Como não há cadastro público e só o Admin cria logins, a produção precisa de um Admin inicial, e o AD-18 prevê apenas `prisma migrate deploy`. O AD-11 cita seeds só para isentá-los da auditoria. Os scripts `build` e `start` do D-17 constam do memlog (linha 49), mas no spine não são explícitos.
**Sugestão:** incluir no AD-18 o processo de criação do primeiro Admin e o seed independente de `test/`.

## Achados de severidade baixa

| # | Requisito | Lacuna | Sugestão |
| --- | --- | --- | --- |
| B1 | FR-18 ("com data e hora") | Os campos listados no AD-12 não incluem instante nem id do registro | Acrescentar `occurredAt`/`createdAt` e `recordId` à lista |
| B2 | Rastreabilidade | O mapa dá FR-21 a FR-23 sem AD-2 e AD-3 (o texto de ambos cobre o Balanço; só o vínculo falta). NFR-7 está em `binds: NFR-1..NFR-8` sem AD nem linha no mapa. FR-5 não aponta para AD-5, que trata da revogação | Corrigir `Binds` e mapa |
| B3 | Addendum, e-mail | SPF e DKIM foram entregues à arquitetura; o Deferred cita só o provedor | Incluir na linha Deferred |
| B4 | FR-22 ("todas as Movimentações") | O `limit` máximo de 100 vale para toda lista; mês maior exige paginar ou exceção | Definir o comportamento na rota do mês |
| B5 | FR-19 | Unicidade de nome por Conta não aparece como índice `(accountId, name)`, e a sensibilidade a maiúsculas é indefinida | Declarar o índice e a regra |
| B6 | FR-15, FR-21 (ambiguidade do PRD) | O PRD só cita datas passadas e a lista termina no mês atual. Uma data futura, se aceita, fica fora da lista | Rejeitar ou definir como entra no Balanço |
| B7 | FR-13 com FR-6 (ambiguidade do PRD) | O antigo Dono "escolhe" a Permissão, mas não participa quando o Admin transfere | Definir quem escolhe ou um padrão |
| B8 | FR-5 | Login de excluído só é barrado por inferência (AD-5 trata do middleware, e o login não passa por ele; AD-9 restringe leituras) | Citar o login no AD-9 |
| B9 | FR-14, FR-2, FR-9, FR-15 | Confirmação explícita da exclusão (API ou tela?), prazos de 1 hora e 7 dias e formulário pronto para o próximo lançamento ficam só no PRD | Aceitável, desde que as histórias os carreguem; decidir se a confirmação é da API |

## Conferido sem divergência

- **Acesso e sessão:** expiração de 7 dias por renovação, logout imediato, separação dos tokens (FR-1, NFR-5).
- **Contas e Membros:** Dono único, transferência atômica, remoção e troca de Permissão com efeito imediato (FR-7, FR-11 a FR-13, FR-24); D-14 coberto por AD-4 e AD-10.
- **Movimentações e Balanço:** centavos e limite, autoria em `createdById`, cálculo na leitura e meses vazios (FR-15 a FR-18, FR-21 a FR-23, NFR-3); D-16 coberto pelo AD-14.
- **Operação e testes:** limite de login, segredos, CI com pnpm (D-19), `Location` (issue #24), testes e2e (NFR-6 parcial, NFR-8).
- **Futuro:** integração bancária e documentação da API (issue #19) estão em Deferred, de acordo com o PRD e o addendum.

## Nota sobre o memlog

A linha 54 lista regras derivadas na destilação "a confirmar com o usuário" (`Account.ownerId`, token opaco, escrita fora de UoW rejeitada, FKs `Restrict`, expiração por data, limite em memória, versões). Não há registro dessa confirmação. A1, A3 e M3 tocam diretamente nessas regras, então convém confirmá-las junto com as correções.
