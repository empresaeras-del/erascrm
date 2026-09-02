# Checklist de produção — erascrm

Este documento resume a análise deste repositório (fork do
[wacrm](https://github.com/ArnasDon/wacrm)) e o que falta configurar —
não no código, que já está saudável, mas no ambiente — para colocá-lo
em produção.

## 1. Estado atual do fork

Este fork ainda não tem nenhum commit próprio: o histórico é
integralmente o do upstream `ArnasDon/wacrm`. Validado localmente
nesta análise, com as mesmas variáveis dummy que o CI usa:

| Verificação | Resultado |
|---|---|
| `npm run lint` | 0 erros, 37 avisos (não bloqueiam build; ver §6) |
| `npm run typecheck` | OK |
| `npm test` (vitest) | 833 testes, 80 arquivos, todos passando |
| `npm run build` | build de produção completo sem erros |

Ou seja: **o código está pronto**. O que falta é a parte de
infraestrutura — provisionar Supabase, configurar a WhatsApp
Business API da Meta, escolher onde hospedar e preencher variáveis
de ambiente reais.

## 2. Stack e arquitetura

- **App**: Next.js 16 (App Router, Server Actions), React 19,
  TypeScript, Tailwind v4.
- **Dados**: Supabase (Postgres + Auth + Storage), com Row Level
  Security em todas as tabelas e 39 migrations versionadas em
  `supabase/migrations/`.
- **WhatsApp**: Meta Cloud API oficial (webhook HMAC-verificado em
  `/api/whatsapp/webhook`).
- **IA (opcional)**: bring-your-own-key (OpenAI ou Anthropic),
  armazenada criptografada por conta — sem chave global.
- **API pública**: `/api/v1` com API keys escopadas, ver
  `docs/public-api.md`.
- **MCP server**: `mcp-server/`, publicado como pacote npm próprio.
- **CI**: `.github/workflows/ci.yml` (lint/typecheck/test/build) e
  `.github/workflows/migrations.yml` (replay de todas as migrations
  contra um Postgres limpo a cada PR que toca `supabase/**`).

## 3. O que já vem pronto para produção

- Criptografia AES-256-GCM para tokens do WhatsApp (`ENCRYPTION_KEY`).
- Verificação HMAC-SHA256 do webhook da Meta (`META_APP_SECRET`).
- Row Level Security em todas as tabelas do Supabase.
- Cabeçalhos de segurança em toda resposta (`next.config.ts`):
  HSTS, `X-Content-Type-Options`, `X-Frame-Options`,
  `Permissions-Policy`, e uma CSP em modo *report-only*.
- Rate limiting (`src/lib/rate-limit.ts`) usado nas rotas sensíveis.
- Dockerfile multi-stage (standalone output, usuário não-root) +
  `docker-compose.yml` prontos (`docs/docker.md`).
- Deploy testado em três caminhos: Hostinger (Node.js gerenciado),
  Docker/VPS, ou qualquer host Node (Vercel, Railway) — é MIT e
  roda em qualquer lugar.

## 4. Checklist de configuração antes do go-live

### 4.1 Supabase
- [ ] Criar o projeto Supabase (produção — separado de qualquer
      projeto de teste).
- [ ] Rodar as 39 migrations em ordem (`supabase/migrations/`). O
      workflow `migrations.yml` já prova que elas aplicam limpo do
      zero; em produção, aplique via `supabase db push` ou a CLI.
- [ ] Conferir que a extensão `vector` está disponível caso queira
      busca semântica na base de conhecimento de IA (migration 030
      já tenta `CREATE EXTENSION IF NOT EXISTS vector`).
- [ ] Configurar backups automáticos do banco (Supabase Pro faz
      backup diário; no plano free não há — decidir o plano antes
      do go-live com dados reais).
- [ ] Copiar `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`
      e `SUPABASE_SERVICE_ROLE_KEY` (Project Settings → API).

### 4.2 WhatsApp Business API (Meta)
- [ ] Criar/usar um app em Meta for Developers com o produto
      WhatsApp habilitado.
- [ ] Registrar o número de telefone comercial (phone number ID).
- [ ] Configurar o webhook apontando para
      `https://<seu-domínio>/api/whatsapp/webhook` — exige HTTPS.
- [ ] Copiar `META_APP_SECRET` (e `META_APP_ID` se for usar templates
      com header de imagem).
- [ ] Submeter os message templates que serão usados em broadcasts.

### 4.3 Variáveis de ambiente
Baseado em `.env.local.example`:

**Obrigatórias**: `NEXT_PUBLIC_SUPABASE_URL`,
`NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`,
`ENCRYPTION_KEY` (gerar com
`node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`
— **nunca reaproveitar o valor dummy do CI**), `META_APP_SECRET`.

**Recomendadas**: `NEXT_PUBLIC_SITE_URL` (domínio final, sem barra
no fim), `NEXT_PUBLIC_APP_LOCALE`.

**Opcionais**: `ALLOWED_INVITE_HOSTS` (recomendado se o app ficar
exposto sem proxy confiável na frente), `AUTOMATION_CRON_SECRET`
(obrigatória apenas se usar passos "Wait" em automações — configurar
o pinger externo, ver `docs/automations-and-cron.md` no site),
`META_APP_ID`, `WHATSAPP_TEMPLATES_DRY_RUN` (deixar **desligada**
em produção).

- [ ] Confirmar que `WHATSAPP_TEMPLATES_DRY_RUN` não está setada
      (ou está `false`) no ambiente de produção.
- [ ] Nenhum segredo commitado no repositório — `.env.local` já está
      no `.gitignore`.

### 4.4 Hospedagem
Três caminhos, todos documentados:

| Opção | Quando escolher |
|---|---|
| **Hostinger** (recomendada pelo template, `README.md`) | Deploy por Git, SSL grátis, sem gerenciar Docker/infra. Mais rápido para ir ao ar. |
| **Docker / VPS próprio** | Controle total de infra; usar o `Dockerfile` + `docker-compose.yml` já prontos (`docs/docker.md`). |
| **Vercel / Railway / outro host Node** | Se a organização já tem esse provedor como padrão. |

- [ ] Decidir a hospedagem.
- [ ] Configurar domínio + HTTPS (obrigatório — o webhook da Meta
      exige TLS válido).
- [ ] Definir as variáveis de ambiente no painel do provedor (não em
      `.env` no servidor, exceto na rota Docker).

## 5. Segurança — itens a decidir antes do go-live

- [ ] **Ativar a CSP de verdade.** Hoje ela roda como
      `Content-Security-Policy-Report-Only` (`next.config.ts:39`) —
      só reporta violação no console, não bloqueia nada. Depois de
      confirmar que nenhuma página legítima dispara violação, trocar
      a chave para `Content-Security-Policy`.
- [ ] Guardar `SUPABASE_SERVICE_ROLE_KEY` e `ENCRYPTION_KEY` num
      cofre de segredos do provedor de hosting, não em texto puro.
- [ ] Definir uma política de rotação de `ENCRYPTION_KEY` — trocar o
      valor invalida tokens do WhatsApp já salvos (usuários precisam
      reconectar).
- [ ] Revisar `.github/CODEOWNERS` e proteção da branch `main` no
      GitHub (exigir CI verde antes de merge).

## 6. Itens de qualidade a limpar (não bloqueiam produção)

Os 37 avisos de lint atuais são de dependências de hooks React e um
`<img>` sem otimização — nenhum é erro. Vale um PR de limpeza
depois do go-live, sem urgência:
- `src/components/inbox/message-thread.tsx` (import não usado +
  dependência de hook)
- `src/components/settings/{ai-config,ai-knowledge,api-keys-settings,
  settings-overview,whatsapp-config}.tsx` (dependências de hook)
- `src/i18n/request.ts`, `src/middleware.ts` (variáveis não usadas)

## 7. Identidade do fork — decisão do time

Este repositório (`empresaeras-del/erascrm`) é hoje um fork
idêntico ao `wacrm`, sem nenhuma customização de marca. Antes de
divulgar publicamente como produto próprio, decidir:

- [ ] Nome do produto exibido (`package.json` → `name`, título das
      páginas, `README.md`).
- [ ] Remover/atualizar links que apontam para `ArnasDon/wacrm` e
      `wacrm.tech` no README, `package.json` (`homepage`,
      `repository`, `bugs`) e `.github/SECURITY.md` (email e link de
      advisory apontam para o mantenedor upstream).
- [ ] Manter o aviso de licença MIT e a atribuição ao autor original
      (`LICENSE`) — é exigido pela licença, independentemente da
      marca escolhida.

## 8. Resumo — pronto para ir ao ar quando

1. Projeto Supabase de produção criado e migrations aplicadas.
2. App da Meta configurado com webhook em HTTPS e número validado.
3. Todas as variáveis obrigatórias/recomendadas definidas no
   provedor de hosting escolhido.
4. CSP promovida de report-only para enforce.
5. Backups do Supabase configurados.
6. (Opcional, mas recomendado) decisões de marca do §7 resolvidas.

O código em si — CI, testes, build, segurança de aplicação — já
está pronto; nada nesta lista é bloqueado por bugs no template.
