# Story 1.5: Entrar pela tela de Login

Status: ready-for-dev

Fonte: `_bmad-output/planning-artifacts/epics.md` (Epic 1). Em caso de divergência, vale o `epics.md`.

## História

As a Usuário,
I want entrar por uma tela simples e continuar logado ao reabrir o app,
So that eu use o sistema pelo navegador.


## Requisitos referenciados

- **FR-1**: Login com e-mail e senha; mesma mensagem para credenciais inválidas; sessão expira após 7 dias sem uso; logout vale imediatamente.
- **NFR-7**: Plataforma: web para navegadores de computador.
- **AR-22**: Projeto Vue 3, Vite, TypeScript, Pinia, Vue Router com rotas manuais e guards, axios, Zod, Tailwind, pnpm; proxy do Vite para `/api`; Dockerfile de produção.
- **AR-23**: Cliente HTTP com token em memória, renovação compartilhada e retorno ao login; serviços e stores por domínio (AD-6).
- **UX-DR1**: Tokens de cor, tipografia, raio e espaçamento do `DESIGN.md` no Tailwind, em tema claro e escuro.
- **UX-DR2**: Alternância de tema pelo menu do usuário, seguindo o sistema na primeira visita e guardada no navegador.
- **UX-DR3**: `BaseButton` nas variantes primary, secondary e danger, com estados desabilitado e carregando.
- **UX-DR4**: `BaseInput` com rótulo, erro no campo, `aria-describedby` e `aria-invalid`.
- **UX-DR11**: `AppToast` com `aria-live="polite"`, sumindo em 4 segundos.
- **UX-DR13**: `AppHeader` com Conta atual, menu do usuário, Admin condicional e indicador de Convites.
- **UX-DR15**: Estados: abertura do app, carregamento em esqueleto, vazios, erro de validação, erro de rede, sessão expirada com rascunho preservado, 404 de Conta, 403 com recarga de Permissão, Convite com falha de e-mail.
- **UX-DR16**: As 13 telas e rotas da Arquitetura de Informação; ações sem Permissão ocultas.
- **UX-DR17**: Acessibilidade WCAG 2.2 AA, fluxo de lançamento só com teclado e sobra ou falta comunicadas por texto.
- **UX-DR19**: Microcopy do `EXPERIENCE.md`, incluindo "Sobrou" e "Faltou".

## Acceptance Criteria

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
