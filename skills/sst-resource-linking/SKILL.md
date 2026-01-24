---
name: sst-resource-linking
description: Link SST resources to runtime code and access them via the SST SDK. Use when connecting components with the link prop, explaining sst dev linkage behavior, or working with generated sst-env.d.ts types.
---
# SST resource linking

## Scope
- Link resources (buckets, databases, secrets, etc.) to functions or frontends.
- Use the SST SDK `Resource` object in runtime code.
- Explain local vs deploy-time linking behavior and type generation.

## Workflow
1. Create the resource component to be linked (bucket, database, secret, etc.).
2. Add the `link` prop on the consuming component (function, Next.js, Remix, etc.).
3. Access linked values via the SST SDK `Resource` object in runtime code.
4. Use `sst dev` locally so links are injected and environment types are generated.
5. If working outside the multiplexer, wrap the frontend dev command with `sst dev`.
6. Check `sst-env.d.ts` output if type access or link visibility needs validation.
7. Use `sst.Linkable` when linking custom outputs or non-SST resources.

## Outputs
- Updated `sst.config.ts` with correct `link` definitions.
- Runtime code changes using `Resource.<Name>.*` accessors.
- Notes about local dev commands and generated types if relevant.
