# lp-automation

Motor compartilhado pra automatizar a troca de turma vigente nas LPs da Impacta
(Claude Pro, Código Zero, ADV-IA, Lógica, QA Next, People AI Lab, etc).

## O que faz

Composite action que, quando chamada pelo workflow de uma LP:

1. Roda o script Node de switch da LP (`scripts/campaign-switch.mjs` por padrão)
2. Se o script commitou algo (SHA do HEAD mudou), dispara um redeploy no
   Coolify via API usando os secrets fornecidos
3. Se os secrets do Coolify não estiverem setados, apenas commita — o rebuild
   fica manual (igual sem a action)

## Por que isso existe

Cada LP tem **seu próprio** `campaign-switch.mjs` porque a estrutura dos
arquivos varia (ex: `claude-lp` tem `tracking-config.json` + `inject-*.js`;
`logica` tem config dentro de `app.js` + 2 checkouts por turma). Mas o
*motor* (cron + checkout + commit + push + redeploy Coolify) é o mesmo pra
todas — e vive aqui.

Pra bugfix no motor ou adicionar features (ex: notificar no Slack quando
switch rodar), basta um commit neste repo. Zero mudança nas LPs.

## Como uma LP usa

Adicione `.github/workflows/campaign-switch.yml` no repo da LP:

```yaml
name: Campaign switch
on:
  schedule:
    - cron: '5 3 * * *'   # 00:05 BRT diário
  workflow_dispatch: {}

permissions:
  contents: write

jobs:
  switch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: eduardoendo-hub/lp-automation/.github/actions/campaign-switch@main
        with:
          switch-script:          scripts/campaign-switch.mjs     # opcional (default)
          coolify-deploy-webhook: ${{ secrets.COOLIFY_DEPLOY_WEBHOOK }}
```

E dentro da LP, crie `scripts/campaign-switch.mjs` + `campaigns-schedule.json`
(formato específico da LP). Veja `claude-lp` como referência.

## Contrato esperado do `switch-script`

- Node >= 20
- Sem dependências externas (ou instale antes com um `npm ci` separado)
- Deve:
  - Decidir se há switch pra hoje (TZ America/Sao_Paulo)
  - Editar os arquivos da LP conforme o schedule
  - Fazer `git add`, `git commit` e `git push origin HEAD:main`
  - Remover o switch aplicado do schedule
- Idempotente: rodar 2x no mesmo dia não deve causar erro nem commit duplicado

## Inputs da action

| Input | Default | Descrição |
|---|---|---|
| `switch-script` | `scripts/campaign-switch.mjs` | Caminho do script Node da LP |
| `node-version` | `20` | Versão do Node |
| `coolify-deploy-webhook` | `''` | URL completa do deploy webhook da app no Coolify. A URL inteira é o segredo (não usa Bearer). Se vazia, o deploy é pulado. |

## Secret no repo da LP

Setar em `Settings → Secrets and variables → Actions`:

- `COOLIFY_DEPLOY_WEBHOOK` — URL de deploy webhook da app LP (achada no
  Coolify: app → Webhooks ou Resource Operations).

**Por que um só secret por LP?** A arquitetura antiga usava `COOLIFY_URL` +
`COOLIFY_TOKEN` + `COOLIFY_APP_UUID` (3 secrets) e tentava disparar deploy
via `/api/v1/deploy?uuid=...` com Bearer. Na instalação Coolify em uso,
esse endpoint não estava respondendo como API (retornava 302 → login pra
GET e 419 CSRF pra POST, mesmo com token root). O webhook de deploy
funciona em qualquer versão: cada app tem uma URL única em que a URL
inteira é o segredo. Simplifica a infra e dispensa gerenciamento de token.

## Visibilidade

Este repo está **privado**. GitHub Actions composite actions em repo privado
**não são acessíveis** por padrão de workflows em outros repos, mesmo da
mesma conta. Opções:

1. **Tornar público** (recomendado) — menor fricção, código é genérico e
   não contém segredo
2. **Mantê-lo privado** e, no workflow da LP, baixar este repo via
   `actions/checkout@v4` com um PAT (Personal Access Token) com escopo
   `repo`, e referenciar a action por caminho local

A opção 1 é bem mais simples.
