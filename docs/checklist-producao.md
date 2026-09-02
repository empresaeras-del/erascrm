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
- Rate limiting (`src/lib/rate-limit.ts`) cobre as rotas sensíveis:
  ações de conta/convite, envio/broadcast/reação do WhatsApp, todas
  as rotas de IA — e a **API pública `/api/v1`**, aplicado de forma
  centralizada por API key em `src/lib/auth/api-context.ts:99`, não
  rota por rota. Limitação a registrar: é em memória por processo
  (`RATE_LIMITS`, comentário no topo do arquivo) — não sobrevive a
  múltiplas instâncias/regiões; escalar horizontalmente exige trocar
  por Redis/Upstash mantendo a mesma assinatura de `check`.
- Dockerfile multi-stage (standalone output, usuário não-root) +
  `docker-compose.yml` prontos (`docs/docker.md`). Validei nesta
  análise que `npm run build` completa sem erro; não cheguei a
  validar um deploy real em nenhum dos três caminhos abaixo.
- Três caminhos de hospedagem documentados: Hostinger (Node.js
  gerenciado — é como o `main` do upstream é servido), Docker/VPS
  (usando o Dockerfile acima), ou qualquer host Node (Vercel,
  Railway) — é MIT, roda em qualquer lugar.

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
- [ ] Configurar backups automáticos do banco e **decidir o plano
      antes de colocar dados reais**: o plano Free do Supabase não
      tem backup automático (e pausa o projeto após 1 semana sem
      uso); o Pro inclui snapshot diário do Postgres com retenção de
      7 dias (não cobre Storage buckets nem Edge Functions), e PITR
      granular é um add-on pago no Pro/Team.
      [Fonte](https://axonbuild.com/blog/supabase-backup/).
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
exposto sem proxy confiável na frente), `AUTOMATION_CRON_SECRET`,
`META_APP_ID`, `WHATSAPP_TEMPLATES_DRY_RUN` (deixar **desligada**
em produção).

> `.env.local.example` aponta pra `docs/automations-and-cron.md` para
> mais detalhes sobre `AUTOMATION_CRON_SECRET` — esse arquivo **não
> existe neste repositório**, só no site de docs do upstream
> (`wacrm.tech`, que §7.2 já marca pra ser removido do README). Para
> não depender de um link que deixa de fazer sentido: a variável
> protege dois endpoints, `GET /api/automations/cron` e
> `GET /api/flows/cron` (`src/app/api/automations/cron/route.ts`,
> `src/app/api/flows/cron/route.ts`), que só existem porque passos de
> espera ("Wait") em automações/flows não têm um timer interno — algo
> externo (cron do provedor, Vercel Cron, ou um uptime pinger) precisa
> bater nessas rotas periodicamente com o header `x-cron-secret`
> igual ao valor da variável. Sem isso configurado, execuções pendentes
> com espera simplesmente nunca são retomadas.

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

- [x] **`CODEOWNERS` corrigido** — trocado de `* @ArnasDon` para
      `* @empresaeras-del` (`.github/CODEOWNERS`), a única conta com
      acesso a este repositório (confirmado via API do GitHub:
      `empresaeras-del` é o único collaborator, com role `admin`).
      Falta só: [ ] confirmar em Settings → Branches se existe (ou
      vai existir) uma regra de proteção do `main` exigindo aprovação
      de code owner — sem isso, o arquivo não tem efeito prático.
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
- [ ] Configurar proteção da branch `main` no GitHub para o time
      (exigir CI verde antes de merge) — depois de corrigir o
      `CODEOWNERS` acima, não antes.

## 6. Itens de qualidade a limpar (não bloqueiam produção)

Os 37 avisos de lint atuais são de dependências de hooks React e um
`<img>` sem otimização — nenhum é erro. Vale um PR de limpeza
depois do go-live, sem urgência:
- `src/components/inbox/message-thread.tsx` (import não usado +
  dependência de hook)
- `src/components/settings/{ai-config,ai-knowledge,api-keys-settings,
  settings-overview,whatsapp-config}.tsx` (dependências de hook)
- `src/i18n/request.ts`, `src/middleware.ts` (variáveis não usadas)

## 7. Identidade do fork

Este repositório (`empresaeras-del/erascrm`) é hoje um fork idêntico
ao `wacrm`, sem nenhuma customização. Decisões já tomadas pelo time:
**projeto interno** (sem fluxo de contribuição externa), **nome
"erascrm"**, **remover o link de afiliado Hostinger**. O que isso
implica, por categoria:

### 7.1 Obrigatório manter — é a licença, não branding
- [ ] `LICENSE` (texto MIT) permanece no repo. É a única exigência
      real da licença; atribuição ao autor original em outro lugar
      (README, etc.) é "appreciated but not required" — o próprio
      `CONTRIBUTING.md:101-104` do upstream confirma isso.

### 7.2 Branding visível — cosmético, baixo risco, renomear para "erascrm"
- [ ] `package.json`: `name`, `author`, `homepage`, `repository.url`,
      `bugs.url` (linhas 2, 7, 8, 11, 14).
- [ ] `src/app/layout.tsx:24-26` — título/descrição das páginas
      (`"wacrm"` / `"%s — wacrm"`).
- [ ] `docker-compose.yml:1` — `name: wacrm` (nome do projeto Compose,
      só aparece em `docker ps` / `docker compose ls`).
- [ ] `README.md` — título, badges (`CI`, `Stars`, ambos hoje apontam
      pro repo `ArnasDon/wacrm`), seção "Deploy on Hostinger" e todos
      os links `wacrm.tech/docs/*` (não existe doc equivalente para
      vocês ainda — ou removem a seção ou substituem por instruções
      próprias, já que `docs/checklist-producao.md` cobre parte disso).
      **Remover** os links com `REFERRALCODE=WACRMHOST` (afiliado do
      autor original), conforme decidido.

### 7.3 Contato de segurança/conduta — trocar ou remover, não deixar como está
Hoje `.github/SECURITY.md` e `.github/CODE_OF_CONDUCT.md` apontam
para o e-mail pessoal do mantenedor upstream
(`a.donauskas@hostinger.com`) e para
`github.com/ArnasDon/wacrm/security/advisories`. Deixar assim **não é
neutro**: um relatório de vulnerabilidade nesse fork iria parar na
pessoa errada. Como o projeto é interno:
- [ ] Decidir se faz sentido manter um `SECURITY.md` formal (só é
      útil se houver gente de fora com acesso ao código/relatando
      bugs) ou substituí-lo por uma linha simples apontando pro canal
      interno de segurança/TI da empresa.
- [ ] Mesma decisão para `.github/CODE_OF_CONDUCT.md` — normalmente
      existe para projetos com colaboradores externos; num projeto
      interno pode ser removido sem perda.

### 7.4 Fluxo de contribuição externa — simplificar, já que é interno
Como decidido (projeto interno, não open-source colaborativo):
- [x] `.github/CODEOWNERS` — corrigido, ver §5.
- [ ] `.github/ISSUE_TEMPLATE/*.yml` — hoje pedem pra seguir
      `SECURITY.md`/`CONTRIBUTING.md` do upstream. Simplificar para o
      fluxo interno da equipe (ou remover, se usarem outro rastreador).
- [ ] `.github/dependabot.yml:12,50` — `reviewers: [ArnasDon]`. Trocar
      pelo usuário/time interno que deve revisar PRs do Dependabot
      (ou remover o campo, se não fizer sentido restringir revisor).
- [ ] `CONTRIBUTING.md` — hoje é um guia de "fork → PR upstream" e
      "se você mantém um fork público". Como não haverá contribuição
      externa, vale reescrever como guia interno de dev-loop (scripts,
      convenção de commit) e remover as seções voltadas a
      colaboradores de fora.

### 7.5 Identificadores funcionais com "wacrm" — não é só cosmético
Diferente do resto, estes têm efeito em contrato/protocolo, não só
em nome exibido. Ainda sem risco agora (fork pré-lançamento, sem
integrações externas usando o formato atual), mas tratar como decisão
separada do rebranding visual:
- [ ] `API_KEY_PREFIX = 'wacrm_live_'` (`src/lib/api-keys/keys.ts:25`)
      — prefixo de toda API key pública (`/api/v1`). Pensado para
      scanners de segredo (GitGuardian etc.) reconhecerem o padrão.
      Trocar para `erascrm_live_` é seguro agora, mas depois do
      go-live viraria uma mudança que invalida chaves existentes.
- [ ] Header `X-Wacrm-Signature` (assinatura HMAC dos webhooks —
      `src/lib/webhooks/sign.ts`, `src/app/api/v1/webhooks/route.ts`)
      — contrato que qualquer consumidor de webhook precisa checar.
      Mesma lógica: trocar agora é de graça, depois quebra integrações.
- [ ] Chaves de `localStorage` (`wacrm.theme`, `wacrm.mode`,
      `wacrm.flowEditor.view`, `wacrm:inbox:contact-panel-open`) —
      puramente internas ao navegador, nunca visíveis ao usuário.
      Baixíssima prioridade; só reseta preferências salvas.

### 7.6 Não mexer
- [ ] `CHANGELOG.md` — histórico factual de quando o código foi
      desenvolvido no upstream (links de issues/PRs `ArnasDon/wacrm`).
      Reescrever isso falsificaria o histórico; manter como está.
- [ ] `mcp-server/` — pacote separado (`wacrm-mcp`), publicado no npm
      pelo autor original. Só decidir algo aqui se vocês pretendem
      expor a API via MCP: aí sim é uma decisão à parte (usar o
      pacote npm oficial como está, ou fazer fork e publicar o de
      vocês sob outro nome).

## 8. Resumo — pronto para ir ao ar quando

0. ~~`.github/CODEOWNERS` apontando pro mantenedor upstream~~ —
   corrigido. Falta só confirmar a proteção de branch do `main` (§5).
1. Projeto Supabase de produção criado e migrations aplicadas.
2. App da Meta configurado com webhook em HTTPS e número validado.
3. Todas as variáveis obrigatórias/recomendadas definidas no
   provedor de hosting escolhido.
4. CSP promovida de report-only para enforce.
5. Backups do Supabase configurados.
6. Rebranding visual (§7.2-7.4) aplicado — decidido: interno, nome
   "erascrm", sem link de afiliado Hostinger.

O código em si — CI, testes, build, segurança de aplicação — já
está pronto; nada nesta lista é bloqueado por bugs no template.
