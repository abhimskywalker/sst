---
name: sst-resource-import-reference
description: Import or reference existing resources in SST apps. Use when migrating to SST, managing external resources without ownership, or sharing resources across stages with static get methods.
---
# SST resource import and reference

## Scope
- Import existing resources so SST manages them.
- Reference external resources without ownership.
- Share resources across stages using `static get` methods.

## Workflow
1. Decide whether the resource should be managed by SST (import) or remain external (reference).
2. For imports into SST components, use `transform` to set `opts.import` and align `args` to match the existing resource.
3. For imports into low-level provider resources, use the resource `import` option and adjust properties per error output.
4. For references, use the resource `static get` method or provider lookup methods.
5. Use `sst.Linkable` to expose properties from referenced resources for runtime access.
6. For stage sharing, use `static get` conditionally based on `$app.stage` and avoid sharing complex frontends.

## Outputs
- Updated `sst.config.ts` with import transforms or lookup references.
- Clear notes on ownership, lifecycle, and stage-sharing decisions.
