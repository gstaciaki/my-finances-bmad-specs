---
title: My Finances PRD
status: final
created: 2026-11-07
updated: 2026-11-08
---

# My Finances: PRD

## 0. Propósito do documento

Este PRD define a primeira versão do My Finances para quem vai implementá-la, e serve de entrada para a arquitetura e para as histórias. Ele parte do levantamento do backend (`_bmad-output/planning-artifacts/levantamento-my-finances-backend.md`), que documenta o que já existe e os defeitos conhecidos, e não repete esse conteúdo. Detalhes técnicos estão no `addendum.md`, nesta mesma pasta. Os termos seguem o Glossário (seção 3).

**Status:** revisado com Guilherme em 2026-11-07, sem premissas pendentes.

## 1. Visão

O dinheiro de uma casa raramente está em um lugar só: cada pessoa paga despesas pelo próprio banco, e nenhum extrato mostra o quadro completo. O My Finances é o lugar onde um casal junta tudo isso em uma única Conta da casa.

Cada pessoa registra as Entradas e Saídas que passaram pelo seu banco, normalmente em um momento fixo da semana. No fim de cada mês, o casal vê quanto entrou, quanto saiu, o que sobrou ou faltou, e quanto a casa guardou até ali. As Categorias mostram para onde o dinheiro foi, como os gastos com o carro.

O projeto também é um estudo do ciclo completo de uma aplicação, com a meta de chegar à produção. No futuro, a integração com os bancos poderá registrar as Movimentações automaticamente.

## 2. Usuários

### 2.1 Jobs to be done

- Juntar, em um só lugar, o dinheiro da casa que passa por bancos diferentes.
- Saber, ao fim de cada mês, se sobrou ou faltou dinheiro.
- Acompanhar quanto a casa guardou ao longo do tempo.
- Entender para onde o dinheiro foi, por Categoria, e se organizar para o mês seguinte.

### 2.2 Fora do público da v1

- O público geral: não há tela de cadastro. O Admin cria os logins pela área de administração.
- Uso no celular: a v1 é usada no computador, sem layout responsivo.

### 2.3 Jornadas principais

> **UJ-1. Alex lança os gastos da semana.**
> Alex e Sam dividem as despesas da casa, pagas em bancos diferentes. No momento fixo da semana, Alex abre o extrato do próprio banco e entra no My Finances, fazendo login se a sessão tiver expirado. Abre a Conta da casa e registra, uma por uma, as Movimentações da semana que saíram do banco dele, cada uma com a data real e a Categoria, quando fizer sentido. Ao terminar, as Movimentações de Alex estão na Conta da casa, junto com as que Sam registra da mesma forma.

> **UJ-2. Alex e Sam fecham o mês.**
> No começo de um novo mês, Alex e Sam abrem o My Finances juntos e veem a lista de meses, cada um com seu Balanço mensal. Abrem o mês que acabou: aparecem todas as Movimentações e o Resultado do mês, mostrando quanto sobrou ou quanto faltou. Ao lado, o Saldo acumulado mostra quanto a casa guardou até ali. Pelas Categorias, veem também quanto foi gasto com o carro, por exemplo. Usam isso para se organizar no mês seguinte.

## 3. Glossário

- **Usuário:** pessoa com login no sistema. O login é criado pelo Admin.
- **Admin:** Usuário com visão total do sistema e acesso à área de administração de Usuários.
- **Conta:** espaço que centraliza o dinheiro de uma casa, independentemente do banco de origem. Ex.: Conta da casa. Não é uma conta bancária. Um Usuário pode participar de várias Contas.
- **Membro:** Usuário com acesso a uma Conta. Todo Membro tem uma Permissão. Quem sai ou é removido da Conta, ou tem o login excluído, passa a ser **ex-Membro**: não acessa mais a Conta, mas a autoria do que lançou fica guardada.
- **Permissão:** o que um Membro pode fazer na Conta, definida pelo Dono. Há duas:
  - **Lançamento:** registrar, editar e excluir Movimentações e atribuir Categorias a elas.
  - **Gestão:** tudo o que a Permissão de Lançamento permite, mais gerenciar Categorias e editar o nome e o Saldo inicial da Conta.
- **Dono:** o Membro responsável pela Conta. Cada Conta tem exatamente um Dono, inicialmente quem a criou. O Dono tem todas as Permissões e é o único que convida e remove Membros, altera Permissões, transfere a titularidade e exclui a Conta.
- **Convite:** pedido enviado por e-mail pelo Dono para um Usuário, o **convidado**, se tornar Membro da Conta com uma Permissão definida. O convidado aceita ou recusa.
- **Conta excluída:** Conta excluída pelo Dono, ou cujo Dono, sendo o único Membro, teve o login excluído. A exclusão é lógica: nenhum Membro a acessa, mas os dados ficam guardados e o Admin continua vendo a Conta.
- **Movimentação:** registro de dinheiro que entrou (Entrada) ou saiu (Saída) de uma Conta, com valor, data, descrição opcional e Categoria opcional. Não registra o banco nem a pessoa de origem.
- **Categoria:** marcador para agrupar Movimentações. Cada Conta tem as próprias Categorias, gerenciadas pelos Membros com Permissão de Gestão. Uma Movimentação tem no máximo uma Categoria, e Categorias valem para Entradas e Saídas.
- **Resultado do mês:** total de Entradas menos total de Saídas de um mês. Positivo quando sobrou, negativo quando faltou.
- **Balanço mensal:** visão de um mês da Conta, com total de Entradas, total de Saídas, Resultado do mês, Saldo acumulado e totais por Categoria.
- **Saldo inicial:** valor que a casa já tinha guardado quando a Conta começou a ser usada. Opcional, padrão zero. Pode ser negativo, por exemplo quando a casa começa devendo no cheque especial.
- **Saldo acumulado:** dinheiro guardado pela casa até o fim de um mês: Saldo inicial mais os Resultados de todos os meses até ali.

## 4. Funcionalidades

### 4.1 Acesso

**Descrição:** o sistema só é usado por quem tem login. Não há tela de cadastro: os Usuários entram com e-mail e senha e podem recuperar o acesso sozinhos. Realiza UJ-1.

#### FR-1: Login

O Usuário entra com e-mail e senha. Realiza UJ-1.

- Credenciais inválidas retornam a mesma mensagem, seja o e-mail inexistente ou a senha errada.
- A sessão expira após 7 dias sem uso, e o Usuário volta ao login.
- O Usuário pode sair (logout), e a sessão encerrada deixa de valer imediatamente.

#### FR-2: Recuperar senha por e-mail

O Usuário que esqueceu a senha recebe um link por e-mail para criar uma nova.

- O link vale uma única vez e expira em 1 hora.
- A tela responde da mesma forma exista ou não o e-mail informado.
- A nova senha segue a política de senhas do sistema.

#### FR-3: Editar o próprio perfil

O Usuário logado edita o próprio nome e e-mail, e troca a senha informando a senha atual.

- O novo e-mail precisa ser único no sistema. Manter o próprio e-mail ao editar o nome não gera conflito.
- O Usuário não altera dados de outros Usuários.

### 4.2 Administração

**Descrição:** o Admin cria os logins e tem visão total do sistema, em uma área própria.

#### FR-4: Criar Usuário

O Admin cria um Usuário com nome, e-mail e senha inicial.

- O e-mail é único no sistema.
- Não é pedido CPF nem outro documento pessoal.

#### FR-5: Gerir Usuários

O Admin lista, edita, redefine a senha e exclui qualquer Usuário.

- A exclusão é lógica. O Usuário excluído não consegue fazer login, passa a ser ex-Membro das Contas de que participava, e o e-mail dele não pode ser reutilizado.
- O Admin só exclui o Dono de uma Conta com outros Membros depois de transferir a titularidade dela (FR-6).
- Se o Dono for o único Membro da Conta, a exclusão é permitida: a Conta também é excluída, e só o Admin a vê.

#### FR-6: Visão total

O Admin vê todas as Contas, seus Membros e suas Movimentações.

- A visão inclui as Contas excluídas, identificadas como tal.
- A visão é somente leitura, com uma exceção: o Admin pode transferir a titularidade de uma Conta para outro Membro dela. Para qualquer outra alteração, o Admin precisa ser Membro.

### 4.3 Contas e Membros

**Descrição:** qualquer Usuário cria Contas e compartilha cada uma com outras pessoas por meio de Convites. O Dono administra a Conta e define a Permissão de cada Membro.

#### FR-7: Criar Conta

O Usuário cria uma Conta com nome e Saldo inicial opcional, e se torna o Dono.

- A nova Conta aparece imediatamente na lista de Contas do Usuário.

#### FR-8: Ver as próprias Contas

O Usuário vê só as Contas das quais é Membro. Ele não lê nem altera Contas das quais não é Membro. Realiza UJ-1.

#### FR-9: Convidar por e-mail

O Dono convida um Usuário pelo e-mail, escolhendo a Permissão, e o convidado recebe um aviso por e-mail.

- Só é possível convidar quem já tem login.
- Não é possível convidar quem já é Membro, nem ter dois Convites pendentes para a mesma pessoa na mesma Conta.
- O Convite expira em 7 dias.

#### FR-10: Aceitar ou recusar Convite

O convidado vê seus Convites pendentes ao entrar no sistema e aceita ou recusa cada um.

- Ao aceitar, torna-se Membro com a Permissão definida no Convite, e a Conta aparece na lista dele.
- Um Convite recusado ou expirado não pode mais ser aceito. O Dono pode enviar um novo Convite.

#### FR-11: Remover Membro

O Dono remove qualquer Membro, menos a si mesmo.

- O Membro removido perde o acesso à Conta imediatamente, inclusive em sessões já abertas.

#### FR-12: Sair da Conta

O Membro sai da Conta por iniciativa própria e perde o acesso imediatamente.

- O Dono só pode sair depois de transferir a titularidade.

#### FR-13: Transferir titularidade

O Dono transfere a titularidade para outro Membro. O Admin também pode fazer essa transferência (FR-6).

- Cada Conta tem sempre exatamente um Dono.
- Ao transferir, o antigo Dono escolhe a própria Permissão (Lançamento ou Gestão) e continua Membro.

#### FR-24: Alterar Permissão de um Membro

O Dono altera a Permissão de qualquer outro Membro entre Lançamento e Gestão.

- A mudança vale imediatamente, inclusive em sessões já abertas.

#### FR-14: Editar e excluir Conta

Membros com Permissão de Gestão editam o nome e o Saldo inicial da Conta. Só o Dono exclui a Conta.

- Editar o Saldo inicial recalcula o Saldo acumulado de todos os meses.
- A exclusão exige confirmação explícita e é lógica: nenhum Membro acessa mais a Conta nem suas Movimentações, mas os dados ficam guardados e continuam visíveis para o Admin (FR-6).
- A v1 não tem restauração de Conta excluída. Os dados guardados permitem adicionar isso no futuro.

### 4.4 Movimentações

**Descrição:** os Membros registram o que entrou e saiu, muitas vezes em lote, a partir do extrato da semana. Realiza UJ-1.

#### FR-15: Registrar Movimentação

O Membro registra uma Movimentação com tipo (Entrada ou Saída), valor, data, descrição opcional e Categoria opcional. Realiza UJ-1.

- O valor é sempre positivo, com até 2 casas decimais. O tipo define se o valor soma ou subtrai no Resultado do mês.
- A data é escolhida pelo Membro e pode estar no passado, mas não no futuro. O padrão é a data de hoje no fuso `America/Sao_Paulo`.
- Valores acima do limite suportado retornam erro de validação.
- Depois de salvar, o formulário fica pronto para o próximo lançamento, sem sair da Conta.

#### FR-16: Editar e excluir Movimentação

Qualquer Membro, com qualquer Permissão, edita ou exclui qualquer Movimentação da Conta, inclusive as lançadas por outro Membro.

#### FR-17: Listar Movimentações

O Membro lista as Movimentações da Conta, ordenadas por data, e filtra por período e por Categoria. A lista nunca inclui Movimentações de outra Conta.

#### FR-18: Registrar autoria

O sistema guarda quem criou cada Movimentação e cada alteração ou exclusão feita nela, com data e hora. Essa informação não aparece nas telas da v1. O sistema também registra toda escrita no banco e toda requisição HTTP, sem gravar corpo de requisição nem segredos.

### 4.5 Categorias

**Descrição:** cada Conta tem as próprias Categorias, que agrupam Entradas e Saídas.

#### FR-19: Gerenciar Categorias

Membros com Permissão de Gestão criam, renomeiam e excluem as Categorias da Conta. Os nomes são únicos dentro da Conta.

- Um Membro com Permissão de Lançamento só escolhe entre as Categorias existentes.
- Excluir uma Categoria em uso deixa as Movimentações dela sem Categoria.

#### FR-20: Atribuir Categoria

Qualquer Membro atribui ou remove a Categoria de uma Movimentação, ao registrar ou ao editar.

### 4.6 Balanço mensal

**Descrição:** o coração do produto. O casal acompanha os meses e entende o que sobrou, o que faltou e quanto a casa guardou. Realiza UJ-2.

#### FR-21: Lista de meses

O Membro vê a lista de meses da Conta, do mais recente para o mais antigo, cada um com total de Entradas, total de Saídas, Resultado do mês e Saldo acumulado. Realiza UJ-2.

- A lista vai do mês da primeira Movimentação até o mês atual, incluindo meses sem Movimentação, com Resultado zero.

#### FR-22: Abrir um mês

Ao abrir um mês, o Membro vê o Balanço mensal e todas as Movimentações daquele mês. Realiza UJ-2.

- O mês é definido pela data da Movimentação no calendário de `America/Sao_Paulo`.
- Uma Movimentação registrada às 23h do último dia do mês, com a data padrão, pertence a esse mês, e não ao seguinte.

#### FR-23: Totais por Categoria

O Balanço mensal mostra o total de cada Categoria no mês, separando Entradas e Saídas, e um grupo "Sem categoria". Realiza UJ-2.

- A soma dos grupos é igual ao total de Entradas e ao total de Saídas do mês.

**Regras de cálculo:**

- Resultado do mês = total de Entradas − total de Saídas do mês.
- Saldo acumulado ao fim de um mês = Saldo inicial + soma dos Resultados de todos os meses até ele, inclusive.
- Registrar, editar ou excluir uma Movimentação com data passada atualiza o Balanço daquele mês e o Saldo acumulado dos meses seguintes.

## 5. Requisitos não funcionais

- **NFR-1 Isolamento:** um Usuário nunca lê nem altera dados de uma Conta da qual não é Membro, exceto a leitura pelo Admin (FR-6). Isso vale para toda a API, não só para as telas.
- **NFR-2 Dados sensíveis:** nenhuma resposta da API contém senha ou hash de senha. O sistema não coleta CPF.
- **NFR-3 Exatidão:** os valores são calculados em centavos, sem arredondamento de ponto flutuante. Cada Movimentação suporta até R$ 21.474.836,47.
- **NFR-4 Erros previsíveis:** uma regra de negócio violada retorna um erro claro, nunca erro interno (500).
- **NFR-5 Sessão:** o token de acesso e o de renovação têm finalidades distintas, e um não substitui o outro.
- **NFR-6 Produção:** HTTPS, segredos fora do repositório, build e deploy reproduzíveis, limite de 5 tentativas de login a cada 15 minutos por e-mail, e backup diário do banco, com restauração testada ao menos uma vez antes do uso real.
- **NFR-7 Plataforma:** aplicação web para navegadores de computador.
- **NFR-8 Testes:** testes de ponta a ponta cobrem o isolamento entre Contas, as Permissões de Lançamento e Gestão, os poderes do Dono e do Admin, e as regras de cálculo do Balanço mensal, incluindo a virada do mês.

## 6. Fora do escopo (v1)

- Integração com bancos e importação automática de extratos. O modelo de dados não deve impedir essa integração no futuro (ver addendum).
- Cadastro público.
- Uso no celular e layout responsivo.
- Orçamento por Categoria, metas de economia e alertas.
- Movimentações recorrentes, parcelas e transferências entre Contas.
- Exibir nas telas o histórico de autoria e alterações.
- Várias moedas, investimentos e exportação de dados.

## 7. Escopo da v1

**Dentro:** tudo o que está na seção 4, com os requisitos não funcionais da seção 5, publicado em produção.

**Desejável, sem bloquear a v1:** documentação da API (issue #19).

**Ordem sugerida de construção**, que segue o levantamento:

1. Isolamento e proteção dos dados (NFR-1, NFR-2, FR-8), que corrigem as issues #25, #26 e #27.
2. Modelo de Movimentação (FR-15 a FR-17), que corrige as issues #28 e #29.
3. Balanço mensal (FR-21 a FR-23), que atende à issue #20.
4. Membros, Permissões e Convites (FR-9 a FR-13, FR-24), que atende à issue #13.
5. Categorias (FR-19, FR-20) e autoria (FR-18).
6. Acesso e administração (FR-1 a FR-6), que atende à issue #12, e preparação para produção (NFR-6).
7. Frontend web.

## 8. Métricas de sucesso

- **SM-1 Uso contínuo:** o casal registra as Movimentações da Conta da casa em pelo menos 4 semanas seguidas, em produção. Valida FR-15 e FR-21.
- **SM-2 Confiança no Balanço:** ao fim do primeiro mês, o Resultado do mês confere com os extratos completos dos bancos no mesmo mês, sem diferença ou com diferença explicada. Valida FR-15 e FR-21 a FR-23.
- **SM-C1 Contra-métrica, quantidade de funcionalidades:** não priorizar novas funcionalidades enquanto SM-1 e SM-2 não forem atingidas. Equilibra o objetivo de estudo com o de uso real.

## 9. Perguntas em aberto

1. Qual provedor de e-mail será usado para Convites e recuperação de senha? Fica para a arquitetura.
