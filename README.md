# erascrm — CRM for WhatsApp

> Self-hostable CRM for WhatsApp® — shared inbox, contacts, sales
> pipelines, broadcasts, and no-code automations.

[![License: MIT](https://img.shields.io/badge/License-MIT-violet.svg)](./LICENSE)
[![CI](https://github.com/empresaeras-del/erascrm/actions/workflows/ci.yml/badge.svg)](https://github.com/empresaeras-del/erascrm/actions/workflows/ci.yml)
[![Next.js 16](https://img.shields.io/badge/Next.js-16-black?logo=nextdotjs)](https://nextjs.org)
[![Supabase](https://img.shields.io/badge/Supabase-Postgres%20%2B%20Auth-3ecf8e?logo=supabase)](https://supabase.com)

## What you get out of the box

- **Shared inbox** on the official WhatsApp Business API — multiple
  agents working one number, per-conversation assignment, status, and
  notes.
- **Contacts + tags + custom fields**, CSV import, deduplication.
- **Sales pipelines** (Kanban) with deals linked to conversations.
- **Broadcasts** with Meta-approved templates, delivery + read
  tracking, per-recipient variable substitution.
- **No-code automations** — triggers on inbound messages, new
  contacts, keywords, or schedule; conditional branches, waits,
  tags, webhooks. Visual builder.
- **AI reply assistant** — bring your own OpenAI or Anthropic key
  (stored encrypted; no per-seat AI fee, your data stays yours).
  One-click AI-drafted replies in the inbox, plus an optional
  auto-reply bot with a per-conversation cap and clean human handoff.
  Add a **knowledge base** (FAQs, policies, product docs) and it
  answers from your own content — hybrid retrieval (Postgres full-text,
  or semantic pgvector when an embeddings key is set).
- **Real-time dashboard** — response times, daily volume, pipeline
  value, cross-module activity feed.
- **Team accounts** — invite teammates by link, role-based access
  (owner / admin / agent / viewer), ownership transfer. Every install
  is account-scoped, so one shared inbox can be staffed by a whole
  team. Solo use stays single-user with zero setup.
- **Account management** — email, password, avatar, global sign-out.
- **Public REST API** (`/api/v1`) with scoped, revocable API keys —
  build your own automations on top of your CRM. See
  [docs/public-api.md](./docs/public-api.md).
- **MCP server** — drive your CRM from Claude, Cursor, and other AI
  assistants over the [Model Context Protocol](https://modelcontextprotocol.io).
  Read-only by default, opt-in writes. See [docs/mcp.md](./docs/mcp.md)
  (server in [`mcp-server/`](./mcp-server)).

## Stack

- **App** — Next.js 16 (App Router), React 19, TypeScript, Tailwind v4.
- **Data** — Supabase (Postgres + Auth + Storage + RLS).
- **WhatsApp** — Meta Cloud API (official WhatsApp Business API).
- **Security** — token encryption (AES-256-GCM), RLS on every table,
  HMAC-verified webhooks, CSP, rate limiting, CI typecheck/build on
  every PR.

## Quick start

```bash
git clone https://github.com/empresaeras-del/erascrm.git
cd erascrm
npm install
cp .env.local.example .env.local   # fill in Supabase + Meta creds
npm run dev
```

Open <http://localhost:3000>. You'll be redirected to `/login` (or
`/dashboard` if already signed in).

Prefer containers? See [docs/docker.md](./docs/docker.md) for the
Dockerfile + Docker Compose setup.

## Deploying to production

This runs on any Node.js host — Hostinger Managed Node.js, Docker on
your own VPS, Vercel, Railway, or anywhere else. See
[docs/checklist-producao.md](./docs/checklist-producao.md) for the
full checklist (Supabase setup, WhatsApp Business API config, required
env vars, and a comparison of hosting options) before going live.

## Documentation

- [docs/checklist-producao.md](./docs/checklist-producao.md) — production
  readiness checklist.
- [docs/docker.md](./docs/docker.md) — running with Docker.
- [docs/public-api.md](./docs/public-api.md) — the public REST API.
- [docs/mcp.md](./docs/mcp.md) — driving the CRM from AI assistants over MCP.

## Contributing

Internal project — see [`CONTRIBUTING.md`](./CONTRIBUTING.md) for the
dev-loop and PR flow. Security issues: see
[`.github/SECURITY.md`](./.github/SECURITY.md).

## License

[MIT](./LICENSE).
