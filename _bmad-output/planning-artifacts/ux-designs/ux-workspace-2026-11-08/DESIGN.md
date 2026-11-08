---
name: My Finances
description: Sistema web para uma casa centralizar Movimentações e acompanhar o Balanço mensal. Vue 3 com Tailwind CSS e componentes Base* próprios.
status: final
created: '2026-11-08'
updated: '2026-11-08'
sources:
  - _bmad-output/planning-artifacts/prds/prd-workspace-2026-11-07/prd.md
  - _bmad-output/planning-artifacts/architecture/architecture-workspace-2026-11-08/ARCHITECTURE-SPINE.md
colors:
  background: '#FFFFFF'
  surface: '#FFFFFF'
  surface-subtle: '#F7F6F3'
  text: '#37352F'
  text-muted: '#6B6A65'
  border: '#E9E9E7'
  primary: '#1F6FB8'
  primary-foreground: '#FFFFFF'
  focus-ring: '#1F6FB8'
  positive: '#2B7A5F'
  negative: '#C4372C'
  danger: '#C4372C'
  danger-foreground: '#FFFFFF'
  background-dark: '#191919'
  surface-dark: '#202020'
  surface-subtle-dark: '#2A2A2A'
  text-dark: '#E6E5E2'
  text-muted-dark: '#A5A49F'
  border-dark: '#373737'
  primary-dark: '#5AA3E0'
  primary-foreground-dark: '#191919'
  focus-ring-dark: '#5AA3E0'
  positive-dark: '#5FBFA3'
  negative-dark: '#FF8A7E'
  danger-dark: '#FF8A7E'
  danger-foreground-dark: '#191919'
typography:
  page-title:
    fontFamily: system-ui, sans-serif
    fontSize: 24px
    fontWeight: '600'
    lineHeight: '1.3'
  section-title:
    fontFamily: system-ui, sans-serif
    fontSize: 18px
    fontWeight: '600'
    lineHeight: '1.4'
  body:
    fontFamily: system-ui, sans-serif
    fontSize: 16px
    fontWeight: '400'
    lineHeight: '1.5'
  label:
    fontFamily: system-ui, sans-serif
    fontSize: 14px
    fontWeight: '500'
    lineHeight: '1.4'
  money:
    fontFamily: system-ui, sans-serif
    fontSize: 16px
    fontWeight: '500'
    lineHeight: '1.5'
rounded:
  sm: 4px
  md: 6px
  lg: 8px
spacing:
  '1': 4px
  '2': 8px
  '3': 12px
  '4': 16px
  '6': 24px
  '8': 32px
components:
  base-button-primary:
    background: '{colors.primary}'
    foreground: '{colors.primary-foreground}'
    radius: '{rounded.md}'
  base-button-secondary:
    background: '{colors.surface}'
    foreground: '{colors.text}'
    border: '{colors.border}'
    radius: '{rounded.md}'
  base-button-danger:
    background: '{colors.danger}'
    foreground: '{colors.danger-foreground}'
    radius: '{rounded.md}'
  base-input:
    background: '{colors.surface}'
    foreground: '{colors.text}'
    border: '{colors.border}'
    radius: '{rounded.sm}'
  base-table:
    background: '{colors.surface}'
    border: '{colors.border}'
    radius: '{rounded.lg}'
  confirm-dialog:
    background: '{colors.surface}'
    radius: '{rounded.lg}'
---

## Brand & Style

Inspirado na sobriedade do Notion, sem reproduzir sua marca: fundo limpo, texto em cinza quente, pouca cor e quase nenhuma moldura. É uma ferramenta doméstica, e os valores são o destaque. Tema claro é o padrão principal; tema escuro é uma opção com os mesmos componentes.

## Colors

Cada token tem par `-dark` para o tema escuro. Texto, ação e valores atingem contraste AA nos dois temas, sobre `{colors.surface}` e `{colors.surface-subtle}`.

- `{colors.background}`, `{colors.surface}`: fundo da página e de tabelas.
- `{colors.surface-subtle}`: linha em hover, cabeçalho de tabela e áreas de apoio.
- `{colors.primary}`: ação principal e links, usado com parcimônia.
- `{colors.positive}` e `{colors.negative}`: somente para valores, sempre com sinal ou palavra, nunca só a cor.
- `{colors.danger}`: ações destrutivas e mensagens de erro.
- `{colors.focus-ring}`: anel de foco, sempre visível, essencial no uso por teclado.

## Typography

Fonte do sistema, sem carregar fontes externas. `{typography.money}` usa algarismos tabulares (`tabular-nums`) e alinha à direita nas tabelas, para os valores ficarem comparáveis.

## Layout & Spacing

Layout de computador, com largura de conteúdo de até 1100px centralizada. Barra superior fixa e sem menu lateral. Margem de página `{spacing.6}`, separação entre blocos `{spacing.8}`.

## Elevation & Depth

Sem sombras em superfícies comuns; a separação vem de `{colors.border}` e de `{colors.surface-subtle}`, em linhas finas. Só o `ConfirmDialog` e o `AppDialog` têm sombra e fundo escurecido.

## Shapes

`{rounded.sm}` em campos, `{rounded.md}` em botões e `{rounded.lg}` em tabelas, cartões e diálogos.

## Components

| Componente | Visual |
| --- | --- |
| `BaseButton` | Variantes `primary`, `secondary` e `danger`, conforme tokens. Desabilitado com opacidade reduzida; carregando com indicador e largura mantida |
| `BaseInput` | Rótulo acima em `{typography.label}`; erro com borda e texto `{colors.danger}` abaixo do campo |
| `BaseSelect` | Mesmo visual do `BaseInput`, sobre `<select>` nativo |
| `BaseTable` | Sem moldura externa; cabeçalho em `{typography.label}` e `{colors.text-muted}`, linhas separadas por `{colors.border}`, hover em `{colors.surface-subtle}`, valores à direita |
| `TypeToggle` | Dois botões unidos, Saída e Entrada; o ativo usa `{colors.primary}` |
| `MoneyValue` | `{typography.money}`, com sinal; `{colors.positive}` ou `{colors.negative}` quando o valor é um resultado |
| `AppDialog` / `ConfirmDialog` | `<dialog>` nativo, largura até 480px, ação no canto inferior direito |
| `AppToast` | Canto inferior direito, fundo `{colors.text}`, texto `{colors.surface}`; invertido no tema escuro |
| `SummaryCard` | Rótulo `{typography.label}`, número `{typography.section-title}`, borda `{colors.border}` |
| `AppHeader` | Fundo `{colors.surface}`, borda inferior `{colors.border}` |

## Do's and Don'ts

| Do | Don't |
| --- | --- |
| Exibir valor sempre como `R$ 1.234,56` | Mostrar centavos crus ou separador americano |
| Indicar sobra ou falta com palavra e sinal, além da cor | Depender só de verde e vermelho |
| Manter o anel de foco visível | Remover `outline` |
| Usar só os tokens deste arquivo, em par claro e escuro | Criar cores por tela |
| Manter molduras finas e áreas limpas | Somar cartões, gradientes e sombras decorativas |
| Inspirar-se na sobriedade do Notion | Usar logotipo, nome, ícones ou ilustrações do Notion |
