# Contribuindo

Este repositório é mantido internamente pela equipe da
`empresaeras-del`. Não há fluxo de contribuição externa (fork/PR de
terceiros) — o que segue é o guia de desenvolvimento do time.

## Rodando localmente

```bash
git clone https://github.com/empresaeras-del/erascrm.git
cd erascrm

cp .env.local.example .env.local   # preencha com credenciais Supabase + Meta
npm install
npm run dev
```

Setup completo (migrations do Supabase, WhatsApp Business API, deploy)
está em [`docs/`](./docs/), especialmente
[`docs/checklist-producao.md`](./docs/checklist-producao.md).

## Trazendo correções do upstream

Este repositório começou como um fork do
[wacrm](https://github.com/ArnasDon/wacrm). Vale a pena trazer
correções de bugs e segurança do upstream periodicamente:

```bash
git remote add upstream https://github.com/ArnasDon/wacrm.git  # uma vez
git fetch upstream
git checkout main
git merge upstream/main     # ou: git rebase upstream/main
# resolva conflitos (prováveis nas áreas já customizadas), depois:
git push origin main
```

Se as customizações forem pesadas, mesclar pode gerar conflito toda
vez. Uma alternativa é fixar numa tag específica do upstream e
atualizar no ritmo de vocês.

## Reportando problemas de segurança

Não abra issues públicas para problemas de segurança. Siga o fluxo
privado em [SECURITY.md](./.github/SECURITY.md).

## Fluxo de PR

- Branch a partir do `main` mais recente (não empurre commits para
  uma branch já mergeada — eles ficam órfãos).
- Rode `npm run typecheck` e `npm run format` localmente antes de
  abrir o PR.
- Preencha o template de PR, principalmente o **Test plan**.
- Uma mudança lógica por PR.
- Primeira linha da mensagem de commit no imperativo e curta; o
  corpo explica o *porquê* — o diff já mostra o *o quê*.

## Referência do dev-loop

| Comando | O que faz |
| --- | --- |
| `npm run dev` | Servidor de dev com Turbopack, porta 3000. |
| `npm run build` | Build de produção. O Next também roda seu próprio typecheck aqui. |
| `npm run typecheck` | `tsc --noEmit`. Passagem rápida só de tipos. |
| `npm run lint` | ESLint. |
| `npm run format` | Prettier, aplicando as mudanças. |
| `npm run format:check` | Prettier em modo checagem — usado no CI. |

## Licença

Este projeto é MIT ([`LICENSE`](./LICENSE)), herdado do template
original. O arquivo `LICENSE` permanece no repositório independente
do rebranding — é assim que as permissões do MIT viajam com o código.
