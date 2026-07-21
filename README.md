# achaditos-runner

Repo público que **solo ejecuta workflows programados** (crons de GitHub
Actions) para [achaditos.com](https://achaditos.com), un comparador de
precios de productos brasileños en Buenos Aires. El código de la aplicación
vive en un repo privado; estos workflows hacen checkout de ese repo con un
token de acceso mínimo y corren sus scripts.

Es automatización de un negocio real (actualización de precios y salud del
pipeline de datos), dentro del uso aceptable de GitHub Actions para repos
públicos.

## Patrón para agregar un cron nuevo

Todos los workflows siguen la misma receta; para sumar uno (alertas de
favoritos, newsletter semanal, etc.) copiar este esqueleto:

```yaml
on:
  schedule:
    - cron: "23 12 * * *" # evitar :00/:30 -- GitHub descarta ticks congestionados
  workflow_dispatch: {}

permissions:
  contents: read # el GITHUB_TOKEN de este repo no se usa para nada

jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with:
          repository: PabloNicolasAcevedo/brasil-precios-ba
          token: ${{ secrets.PRIVATE_REPO_TOKEN }}
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: package-lock.json
      - run: npm ci
      - run: node scripts/elScriptQueSea.js
        env:
          PGHOST: ${{ secrets.PGHOST }}
          # ... resto de secrets que necesite el script
```

Reglas del repo:

- **Nada de WhatsApp acá.** La sesión, su passphrase y el envío de alertas
  se quedan en los workflows del repo privado, siempre.
- Si un script crea issues de GitHub, pasarle `GITHUB_TOKEN:
  secrets.PRIVATE_REPO_TOKEN` y `NOTIFY_REPO:
  PabloNicolasAcevedo/brasil-precios-ba` para que los issues aterricen en el
  repo privado y no acá.
- Si el workflow commitea resultados al repo privado, usar `git pull
  --rebase` antes del `git push` (otros workflows también pushean).
- Los horarios de scraping están espejados en
  `web/lib/admin/scrapeSchedule.ts` del repo privado -- actualizar ambos.

## Secrets

| Secret | Para qué |
|---|---|
| `PRIVATE_REPO_TOKEN` | PAT fine-grained: solo el repo privado, Contents RW + Issues RW |
| `PGHOST` `PGPORT` `PGDATABASE` `PGUSER` `PGPASSWORD` `PGSSLMODE` | Postgres (Supabase, pooler) |
| `R2_ACCOUNT_ID` `R2_ENDPOINT` `R2_BUCKET` `R2_ACCESS_KEY_ID` `R2_SECRET_ACCESS_KEY` `R2_PUBLIC_BASE_URL` | Cloudflare R2 (imágenes) |
| `RESEND_API_KEY` `NOTIFY_EMAIL_FROM` `NOTIFY_FALLBACK_EMAIL` | Emails de salud (opcional: sin esto, el aviso queda solo in-app) |
