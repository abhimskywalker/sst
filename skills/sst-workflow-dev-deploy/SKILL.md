---
name: sst-workflow-dev-deploy
description: Guide SST development and deployment workflow, including state management, local dev with sst dev, and safe updates. Use when planning or executing deploys, handling state, or explaining SST’s infrastructure-as-code workflow.
---
# SST workflow: develop and deploy

## Scope
- Explain SST’s infrastructure-as-code workflow and how state is managed.
- Run local development with `sst dev` and deploy with `sst deploy`.
- Handle imports, references, and stage-sharing considerations.

## Workflow
1. Confirm provider credentials are configured before deployment.
2. Describe the infrastructure-as-code flow: SST translates `sst.config.ts` into managed resources and tracks state.
3. Emphasize that low-level resources are managed by SST; avoid manual edits to prevent state drift.
4. Use `sst dev` for local development so linked resources are available to runtime code.
5. Use `sst deploy` to apply changes and rely on state for incremental updates.
6. When existing resources are required, decide between importing resources, referencing external resources, or sharing across stages.

## Outputs
- Clear deployment plan with `sst dev`/`sst deploy` steps.
- Notes on state considerations, imports, or shared-resource strategies.
