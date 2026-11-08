# Verificação de versões e tecnologias

Executada diretamente, depois de falha de API no revisor dedicado. Data: 2026-11-08.

- `pnpm audit --prod` nas versões do lockfile: 14 vulnerabilidades (1 crítica, 4 altas, 7 moderadas, 2 baixas).
- Com Prisma 6.19.3 e Express 5.3.0 instalados na cópia de verificação: restam 5 (2 moderadas, 3 altas), em jws, zod e uuid.
- Correção indicada: jsonwebtoken 9.0.3 (jws), Zod 4.6.7, uuid 11.1.1. O audit dessas três trocas não foi repetido.
- Prisma 6.19.3: extensão de auditoria, guarda, FK composta, advisory lock e rollback por sentinela testados com sucesso em Postgres 16.
- Mantidos em versão maior anterior por decisão: Prisma 6 (latest é pré-lançamento 8.0.0-rc) e TypeScript 5.9 (latest 7.0.2).
- pnpm 10 é compatível com lockfile v9, que o projeto usa; a versão exata é fixada em `packageManager` na implementação.
- Versões de frontend e bibliotecas novas confirmadas no registro npm; não foram instaladas juntas.
