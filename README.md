# vercel-secrets-template

Template repo for **Secrets components** on the progmise platform — the
Vercel equivalent of Gluon's `*-secrets` repos. Declares which secret env
vars exist per environment; deploying a version resolves each name from the
platform credential store (Supabase Vault) and upserts it as a `sensitive`
environment variable on the target Vercel project. No application code —
and **no secret values ever live here**.

## Layout

```
secrets/
  cert/secrets-cert.yaml      # declared secret names for 'cert'
  pre/secrets-pre.yaml
  pro/secrets-pro.yaml
version.txt                   # release version (semver, bumped per release)
.github/workflows/            # thin callers → reusable-workflows sec-* @v1
```

One folder per manifest environment (`secrets/<env>/secrets-<env>.yaml`).
Each file is a names-only list:

```yaml
secrets:
  - name: APIGEE_TOKEN_CLIENT_SECRET
  - name: REDIS_PASSWORD
```

## Where the values live

Secret values are loaded through the dashboard's **Seguridad** view into
Supabase Vault under the key `<NAME>__<ENV>` (env uppercased — e.g.
`REDIS_PASSWORD__PRO`). The deploy workflow reads them server-side via
`SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` and pipes them straight into
the Vercel upsert — they are never committed, logged, or returned to the
browser.

Environment access is role-gated: `developer` members can load `cert` and
`pre` values; `pro` requires `technical-lead` or `product-owner`.

## How it deploys

`deploy.yml` (manual dispatch or via `deploy-manifest`'s orchestrator) takes
`version`, `environment`, `ci_id` and `project`. The orchestrator derives
`project` from the manifest's `infrastructures[].properties.project` — the
Vercel project this component feeds. `sec-deploy` then:

- resolves every declared name from the credential store — a missing value
  **fails the deploy** (load it in Seguridad first);
- upserts each as a `sensitive` env var (write-only, masked in Vercel);
- `pro` → Vercel target `production`; any other env → `preview`.

## Rules

- **Never commit secret values** — the declaration files list names only.
  `sec-config.py validate` rejects any `value:`/`key:` entries.
- Keep the name set identical across all env files (CI enforces parity).
- On any change, cut a release (`release.yml`) and bump the component's
  `tag` in `deploy-manifest` — the manifest PR validates the tag exists.

## Flow

Seguridad view → values into the credential store. Separately: PR → `ci.yml`
(declaration validation) → merge → `release.yml` (tag + GitHub Release —
version lives in `version.txt`) → `deploy.yml` / orchestrator dispatches per
`ci_id` → secrets upserted into the target Vercel project.

Registering a secrets component in `deploy-manifest` looks like:

```yaml
  - name: loans-api-secrets
    repo: progmise/loans-api-secrets
    tag: "0.1.0"
    type: secrets               # no docker image — the deployable unit is secrets/
    needs: []
    infra: [loans-api-pre, loans-api-pro]   # the app's infra ids
  - name: loans-api
    ...
    needs: [loans-api-secrets]  # secrets land before the app deploys
```

Required repo secrets/vars: `VERCEL_TOKEN` + `SUPABASE_SERVICE_ROLE_KEY`
(secrets), `VERCEL_ORG_ID` + `SUPABASE_URL` (vars).
