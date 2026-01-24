---
name: sst-env-secrets
description: Manage SST environment variables and secrets without relying on .env files. Use when configuring environment values per stage, linking secrets, or setting values with the sst secret CLI.
---
# SST environment variables and secrets

## Scope
- Configure environment values in `sst.config.ts` by stage or dev mode.
- Use linked resources or `sst.Secret` instead of `.env` files.
- Set and consume secrets with the `sst secret` CLI.

## Workflow
1. Prefer resource linking for connection strings and resource metadata instead of `.env` files.
2. Create `sst.Secret` components for sensitive values and link them to consumers.
3. Set secret values using `sst secret set` before deploying or running locally.
4. For non-sensitive config, define values in `sst.config.ts` and pass via the `environment` prop.
5. Use `$app.stage` or `$dev` to pick stage-specific values.
6. If legacy `.env` usage is required, load via `process.env` and manually pass values into components.

## Outputs
- Updated `sst.config.ts` with `environment`, `sst.Secret`, and stage-aware config.
- Documented instructions for setting secrets and running with stage-specific values.
