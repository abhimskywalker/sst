---
name: sst-migration-v2
description: Plan and execute migration from SST v2 to v3. Use when converting sst.config.ts, adjusting workflow changes (sst dev, sst diff), and handling resource handoff between versions.
---
# SST v2 to v3 migration

## Scope
- Identify breaking changes between v2 and v3.
- Update `sst.config.ts` structure and resource definitions.
- Plan safe migration steps for non-prod and prod stages.

## Workflow
1. Review major changes: no CloudFormation/CDK, new `transform` usage, updated `sst.config.ts` structure, and `link` replacing `bind`.
2. Inventory v2 constructs that are not supported or require new provider resources.
3. Plan a staged migration: stand up v3 in a non-prod stage, validate behavior, then migrate prod with resource handoff.
4. Update app configuration to the v3 `export default $config({ app, run })` shape.
5. Update workflow steps: use `sst dev` multiplexer and `sst diff` in place of `sst build`.
6. Document any secrets, domain, or subscriber migrations required.

## Outputs
- Migration checklist with stage plan and unsupported constructs list.
- Updated `sst.config.ts` and notes on workflow changes for v3.
