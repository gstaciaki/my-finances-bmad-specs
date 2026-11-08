# My Finances: especificações

Este repositório guarda o planejamento do **My Finances** e a instalação do [BMad Method](https://github.com/bmad-code-org/BMAD-METHOD) usada para produzi-lo. Aqui não há código da aplicação.

## O que é o My Finances

O dinheiro de uma casa raramente está em um lugar só: cada pessoa paga despesas pelo próprio banco, e nenhum extrato mostra o quadro completo. O My Finances é uma aplicação web em que um casal junta tudo isso em uma única Conta da casa.

Cada pessoa registra as Entradas e Saídas que passaram pelo seu banco. No fim do mês, o casal vê quanto entrou, quanto saiu, o que sobrou ou faltou, quanto a casa guardou até ali e para onde o dinheiro foi, por Categoria.

O projeto também é um estudo do ciclo completo de uma aplicação, do planejamento à produção.

## Para que serve este repositório

- Manter em um só lugar as decisões de produto, arquitetura e experiência.
- Dar aos agentes de IA do BMad o contexto para implementar o sistema história por história.
- Registrar como cada decisão foi tomada, inclusive revisões e alterações.

## Repositórios do projeto

| Repositório | Conteúdo |
| --- | --- |
| `my-finances-bmad-specs` (este) | Especificações e instalação do BMad |
| [`my-finances-backend`](https://github.com/gstaciaki/my-finances-backend) | API em TypeScript, Express, Prisma e PostgreSQL |
| [`my-finances-frontend`](https://github.com/gstaciaki/my-finances-frontend) | Aplicação web em Vue 3, Vite e Tailwind |

Os dois repositórios de código são clonados dentro desta pasta, para que os agentes vejam tudo junto, mas ficam fora deste versionamento pelo `.gitignore`.

## Onde encontrar cada documento

Tudo está em `_bmad-output/planning-artifacts/`:

| Documento | Caminho |
| --- | --- |
| Levantamento do backend existente | `levantamento-my-finances-backend.md` |
| Requisitos do produto (PRD) | `prds/prd-workspace-2026-11-07/prd.md` |
| Detalhes técnicos do PRD | `prds/prd-workspace-2026-11-07/addendum.md` |
| Arquitetura | `architecture/architecture-workspace-2026-11-08/ARCHITECTURE-SPINE.md` |
| Identidade visual | `ux-designs/ux-workspace-2026-11-08/DESIGN.md` |
| Telas, estados e fluxos | `ux-designs/ux-workspace-2026-11-08/EXPERIENCE.md` |
| Epics e histórias | `epics.md` |

Os arquivos `.memlog.md` registram as decisões tomadas durante cada etapa, e as pastas `reviews/` guardam as revisões.

## Estrutura

```text
.
├── _bmad/          # configuração e scripts do BMad
├── _bmad-output/   # documentos gerados
└── .claude/skills/ # skills do BMad para o Claude Code
```

## Como usar

1. Clone este repositório.
2. Dentro dele, clone `my-finances-backend` e `my-finances-frontend`.
3. Abra o Claude Code na raiz, com `uv` instalado, e use as skills do BMad.

Se não souber por onde continuar, rode `/bmad-help`. Preferências pessoais ficam em arquivos `*.user.toml`, que não são versionados.

## Situação atual

O planejamento da v1 está concluído, com 6 epics e 33 histórias. O próximo passo é o Sprint Planning, seguido da implementação a partir da história 1.1.
