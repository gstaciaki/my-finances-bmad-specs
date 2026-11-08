# My Finances: addendum do PRD

Detalhes técnicos e contexto que não cabem no PRD, para a arquitetura e as histórias.

## Auditoria de Movimentações

- **Sugestão de Guilherme (2026-11-07):** registrar quem criou e quem alterou cada Movimentação em uma tabela de auditoria.
- **Fonte do requisito:** o casal quer saber quem lançou ou alterou uma Movimentação, para resolver lançamentos errados ou duplicados.

## E-mail transacional

- Convites e recuperação de senha exigem envio de e-mail. A escolha do provedor e a configuração de domínio (SPF, DKIM) ficam para a arquitetura.

## Integração bancária (futuro)

- Guilherme quer, no futuro, importar movimentações dos bancos automaticamente.
- O modelo de Movimentação não deve impedir isso. Por exemplo, um campo de origem ou um identificador externo pode ser adicionado depois para evitar duplicidade com o que foi lançado manualmente.

## Herança do backend atual

- O levantamento lista 23 defeitos conhecidos e as issues #25 a #29. Este PRD define o comportamento esperado; as correções aparecem como requisitos aqui, sem repetir a análise.
- O backend atual já tem login, refresh token, troca de senha e CRUD de Usuários, Contas e Movimentações. Esses itens serão reaproveitados e ajustados, não recriados.

## Itens técnicos do levantamento para a arquitetura

Vêm do levantamento e não aparecem no PRD por serem detalhes de implementação:

- Criação de Conta e vínculo do Dono em uma única transação de banco (D-14).
- Índice em `accountId` e na data da Movimentação (D-16).
- Scripts de build e start e Dockerfile de produção; o seed não deve depender de código de testes (D-17).
- CI usando pnpm em vez de yarn (D-19).
- Regex de símbolos da política de senha (D-20) e mensagens de erro com a entidade errada (D-21).
- Header `Location` nas respostas 201 (issue #24).
