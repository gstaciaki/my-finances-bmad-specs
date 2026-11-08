# Reconciliação: levantamento versus PRD

- **Entrada:** `_bmad-output/planning-artifacts/levantamento-my-finances-backend.md`
- **Comparado com:** `prd.md` e `addendum.md`, em 2026-11-07

## Cobertos pelo PRD

| Item do levantamento | Onde no PRD |
|---|---|
| D-01 listagem mistura Contas (#25) | FR-17, NFR-1 |
| D-02 sem autorização (#26) | FR-8, NFR-1, NFR-8 |
| D-03 hash da senha exposto (#27) | NFR-2 |
| D-04 sem Saída (#28) | FR-15 |
| D-05 sem data da Movimentação (#29) | FR-15, FR-22 |
| D-06 exclusão retorna 500 | FR-14 (exclusão lógica), NFR-4 |
| D-07 Membros não gerenciáveis (#13) | FR-9 a FR-13 |
| D-08 admin inexistente (#12) | FR-4 a FR-6 |
| D-09 tokens intercambiáveis | NFR-5 |
| D-10 sem cadastro público | 2.2, FR-4 (decisão: manter sem cadastro) |
| D-11 limite de valor | NFR-3 |
| D-12 CPF duplicado | NFR-2 (CPF removido) |
| D-15 paginação sem ordem | FR-17 (ordenada por data) |
| D-18 endurecimento de segurança | NFR-6 |
| RF-11 Saldo inicial | FR-7, FR-14 |
| RF-12 logout | FR-1 |

## Lacunas

1. **Usuário editar o próprio perfil (D-13, RF-08):** o PRD só cobre a troca de senha (FR-3). Não diz se o Usuário altera o próprio nome e e-mail, ou se só o Admin altera.
2. **Deploy reproduzível (RNF-05, critério 5):** o NFR-6 cobre segurança e backup, mas não pede build e deploy reproduzíveis.
3. **Documentação da API (#19, RNF-04):** ausente do PRD e do escopo.
4. **Itens técnicos sem lugar no PRD:** criação atômica de Conta (D-14), índices (D-16), scripts de build e Dockerfile de produção (D-17), CI com gerenciador errado (D-19), regex de senha (D-20), mensagens de erro trocadas (D-21), header `Location` (#24). São detalhes de implementação e devem ir para o addendum, para a arquitetura e as histórias.
5. **Saldo inicial negativo:** o levantamento previa permitir, por exemplo com cheque especial. O Glossário define o Saldo inicial como "valor que a casa já tinha guardado" e não diz se ele pode ser negativo.
