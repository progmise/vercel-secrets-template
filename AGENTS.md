# AGENTS.md

Guide for working on Secrets components generated from
**vercel-secrets-template** — per-environment secret declaration repos that
deploy as `sensitive` env vars onto a Vercel project. No application code.

## Overview

A Secrets component is the deployable projection of an app's *secret*
runtime configuration: `secrets/<env>/secrets-<env>.yaml` declares the env
-var names only; `sec-deploy` resolves each name against the platform
credential store (Supabase Vault key `<NAME>__<ENV>`) and upserts it into
the target Vercel project as `sensitive`. The binding to the app is declared
in `deploy-manifest`: the secrets component lists the app's infra ids in
its `infra:` and the app lists it in `needs:` (secrets land before the app
deploys).

## Structure

- `secrets/<env>/secrets-<env>.yaml` — the only content that ships. A
  `secrets:` list of `- name: NAME` entries (scalar `- NAME` also OK).
  Names are uppercase env-var keys; values NEVER appear here.
- `version.txt` — the release version (semver). `release.yml` validates it,
  creates the tag + GitHub Release; manifest `tag:` must match.
- `.github/workflows/` — thin callers to `progmise/reusable-workflows@v1`
  (`sec-ci`, `sec-release`, `sec-deploy`). Keep them thin — never inline
  logic.

## Rules

- **NEVER commit real secret values.** This repo only declares names; the
  actual values live in the platform credential store (Supabase Vault,
  `platform_secrets` view) and are loaded through the dashboard's Seguridad
  view — role-gated (`developer`: `cert`/`pre`; `pro` needs `technical-lead`
  or `product-owner`).
- Keep names in sync across every env file — `sec-config.py validate`
  enforces parity (missing/extra names fail the PR).
- On any change to this repo, cut a release (`release.yml`) and bump the
  component's `tag` in `deploy-manifest` — the manifest PR validates the
  tag exists.
- Deploy fails if a declared name has no stored value for that env — load
  missing secrets in Seguridad before deploying.
- Upsert semantics only: removing a name from the file does **not** delete
  the var from Vercel — clean stale vars in the Vercel dashboard by hand.

## Git workflow

Single `main` (protected) or `main` + `development` per the component's
branch strategy. Work lands on `<type>/<snake_description>` + PR.

## Verify before done

```bash
python3 sec-config.py validate secrets   # script from reusable-workflows@v1
```
