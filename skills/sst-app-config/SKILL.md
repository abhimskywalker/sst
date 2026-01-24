---
name: sst-app-config
description: Define and configure SST apps in sst.config.ts, including components, providers, and transforms. Use when creating or editing SST infrastructure code, selecting AWS/Cloudflare or other providers, or tuning component configuration in sst.config.ts.
---
# SST app configuration

## Scope
- Create or update `sst.config.ts` to define app infrastructure in code.
- Select built-in or third-party providers and configure components.
- Apply component configuration and transforms for underlying resources.

## Workflow
1. Review the existing `sst.config.ts` (or create one) and confirm the app name, home provider, and stage expectations.
2. Define components for frontends and backends directly in `sst.config.ts` (functions, buckets, databases, services, etc.).
3. Prefer component configuration options first; use `transform` only when you need to adjust underlying resources.
4. If a provider is not covered by built-in components, use provider-specific resources (Pulumi/Terraform) alongside SST components.
5. Keep infrastructure changes in code; avoid manual changes to low-level resources managed by SST.

## Outputs
- Updated `sst.config.ts` with new or adjusted components and provider configuration.
- Notes describing any transforms or provider-specific resources added.
