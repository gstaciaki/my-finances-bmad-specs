# PRD Quality Review: My Finances

## Overall verdict

O PRD tem tese clara: centralizar o dinheiro da casa espalhado por bancos e entregar o fechamento mensal. As funcionalidades derivam dessa tese, e as regras de cálculo do Balanço estão precisas e testáveis. O risco está nas bordas de permissão entre Admin, Dono e Membro, que deixam três situações sem comportamento definido, e em alguns requisitos sem consequência testável.

## Decision-readiness: adequate

As decisões aparecem como decisões, não como "considerações": uma Categoria por Movimentação, exclusão lógica, visão do Admin somente leitura, sem cadastro público. Porém, a escolha de deixar as Categorias só com o Dono (FR-19) cria uma tensão com a UJ-1 que não foi nomeada.

### Findings

- **high** Membro não consegue criar Categoria durante o lançamento (FR-19, UJ-1). Se Alex não for Dono e precisar de uma Categoria nova enquanto lança os gastos da semana, ele fica bloqueado até Sam criá-la. *Fix:* decidir se Membros também criam Categorias, ou registrar a troca de forma explícita.
- **high** Dono desativado deixa a Conta sem gestão (FR-5, FR-13). Se o Admin desativar o Dono, ninguém mais convida, remove Membros ou exclui a Conta. *Fix:* definir o que acontece, por exemplo o Admin transferir a titularidade ou a desativação ser bloqueada enquanto o Usuário for Dono.

## Substance over theater: strong

Não há personas decorativas: Alex e Sam carregam o contexto das duas jornadas. Os NFRs têm limites concretos (centavos, R$ 21.474.836,47, 7 dias, 1 hora, backup diário). A visão é específica a este produto.

## Strategic coherence: strong

A ordem de construção segue a tese (proteção dos dados, Movimentação, Balanço), e as métricas validam uso real e confiança no número, não atividade. Há contra-métrica, coerente com o conflito entre estudo e uso.

### Findings

- **low** SM-2 é parcialmente circular (seção 8). Comparar o Resultado do mês com "a soma dos extratos para as Movimentações lançadas" mede a conta do sistema, não se tudo foi lançado. *Fix:* comparar com o extrato completo dos bancos no mês, aceitando diferença zero ou explicada.

## Done-ness clarity: adequate

As funcionalidades de Movimentação e Balanço têm consequências testáveis fortes. Outros requisitos dependem só da frase principal.

### Findings

- **medium** Requisitos sem consequência testável explícita: FR-3, FR-7, FR-10, FR-11, FR-12 e FR-13. Por exemplo, FR-10 não diz o que acontece com Convites recusados ou expirados, e FR-11 não diz se o Membro removido perde o acesso na hora. *Fix:* adicionar ao menos uma consequência verificável em cada um.
- **medium** "Limite de tentativas no login" sem valor (NFR-6). *Fix:* definir um limite, como 5 tentativas por 15 minutos por e-mail.
- **medium** O perfil do próprio Usuário não está definido. Não está claro se o Usuário altera o próprio nome e e-mail, ou só o Admin (FR-3, FR-5). *Fix:* decidir e registrar.

## Scope honesty: strong

A seção "Fora do escopo" faz trabalho real, inclusive registrando que o modelo não deve impedir a integração bancária. As premissas foram revisadas e incorporadas.

### Findings

- **medium** Restauração de Conta excluída não definida (FR-14, FR-6). Com exclusão lógica, não está claro se o Admin vê Contas excluídas na visão total, nem se alguém pode restaurá-las. *Fix:* declarar se a restauração fica fora da v1 e o que o Admin enxerga.

## Downstream usability: adequate

Glossário presente e usado de forma consistente. IDs contíguos: FR-1 a FR-23, NFR-1 a NFR-8, UJ-1 e UJ-2, SM-1, SM-2, SM-C1. A ordem de construção já liga requisitos às issues.

### Findings

- **low** "Convidado" e "ex-Membro" são usados sem definição no Glossário (FR-5, FR-10). *Fix:* defini-los ou reescrever com termos existentes.

## Shape fit: strong

É um projeto pessoal em código já existente (brownfield) e com várias pessoas usando. O PRD tem duas jornadas, sem excesso, e aponta para o levantamento em vez de duplicá-lo. O tamanho é adequado ao rigor combinado.

## Mechanical notes

- Índice de premissas coerente: só a menção em "Propósito" usa `[ASSUMPTION]`, e não há premissa pendente.
- Perguntas em aberto: resta uma (provedor de e-mail), que é técnica e já está no addendum.
- O Saldo inicial pode ser negativo? O Glossário diz "valor que a casa já tinha guardado"; o levantamento previa permitir negativo.
