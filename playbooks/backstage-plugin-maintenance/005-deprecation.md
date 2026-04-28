# Guardrail 005 — Deprecation

<!--
  Source: giveme community playbook (backstage-plugin-maintenance/005-deprecation.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Removing an API without warning is a breaking change that blindsides
consumers. A plugin that ships a major version with removed exports and
no prior deprecation notice forces operators and module developers to
discover the breakage at upgrade time — not during the grace period when
they still have time to migrate.

Deprecation is a two-phase process: first announce the intent to remove
(deprecation), then execute the removal (breaking change). The gap between
the two phases is the migration window — the time consumers have to update
their code before the upgrade becomes mandatory.

In Backstage plugins, deprecation has five distinct surfaces that each
require different communication strategies: TypeScript exports, plugin IDs,
action IDs, entity provider names, and config keys. Each surface has a
different audience and a different migration cost.

## Decision

APIs are deprecated before removal with at minimum one minor version
gap between the deprecation notice and the breaking change. TypeScript
exports use `@deprecated` JSDoc. Non-TypeScript surfaces (action IDs,
config keys, provider names) use README notices and changelog entries.
Every deprecation includes a migration path — not just a removal notice.

## Rules

- MUST: Add `@deprecated` JSDoc to every TypeScript export before removing it
- MUST: `@deprecated` JSDoc includes the version when removal is planned and the migration path
- MUST: Document non-TypeScript deprecations (action IDs, config keys, provider names) in `CHANGELOG.md` under `### Deprecated`
- MUST: Ship at least one minor version with the deprecation notice before the breaking removal
- MUST: Every deprecation entry in `CHANGELOG.md` includes the migration path — what to use instead
- MUST: Keep deprecated exports functional until the major version removes them — deprecation is a warning, not a removal
- MUST: Add a `## Migration guides` section to README when a major version removes previously deprecated APIs
- MUST NOT: Remove an export in the same version it is deprecated
- MUST NOT: Deprecate without providing a replacement or migration path
- MUST NOT: Ship a major version that removes APIs that were never marked deprecated in a prior release
- SHOULD: Support both the deprecated and new API simultaneously for at least one minor version
- SHOULD: Log a runtime warning when a deprecated config key or provider name is used — helps operators discover deprecations
- SHOULD NOT: Keep deprecated APIs beyond two major versions — the maintenance burden compounds

## Examples

### Correct — TypeScript export deprecation

```typescript
// src/api/MyToolApi.ts

/**
 * @deprecated Use `myToolApiRef` instead. Will be removed in v3.0.0.
 * Migration: replace `import { myApi } from '@scope/backstage-plugin-my-tool'`
 * with `import { myToolApiRef } from '@scope/backstage-plugin-my-tool'`
 */
export const myApi = myToolApiRef;

// the new name — available in the same version as the deprecation
export const myToolApiRef = createApiRef<MyToolApi>({
  id: 'plugin.my-tool.myToolApi',
});
```

### Correct — action ID deprecation (two action IDs in parallel)

```typescript
// src/actions/createRepositoryAction.ts

// new action ID — ship this in the same version as the deprecation
export function createCreateRepositoryAction(options: ActionOptions) {
  return createTemplateAction({
    id: 'myorg:create-repository',    // new stable ID
    // ...
  });
}

// deprecated action — still functional, logs a warning
export function createCreateRepoAction(options: ActionOptions) {
  return createTemplateAction({
    id: 'myorg:create-repo',          // deprecated ID
    async handler(ctx) {
      ctx.logger.warn(
        'Action myorg:create-repo is deprecated and will be removed in v3.0.0. ' +
        'Use myorg:create-repository instead.',
      );
      // delegate to new implementation
      return createCreateRepositoryAction(options).handler(ctx);
    },
  });
}
```

### Correct — config key deprecation with runtime warning

```typescript
// src/plugin.ts
async init({ config, logger }) {
  // support both old and new config key during deprecation window
  const baseUrl =
    config.getOptionalString('myTool.baseUrl') ??
    config.getOptionalString('myTool.apiUrl');  // deprecated key

  if (config.has('myTool.apiUrl') && !config.has('myTool.baseUrl')) {
    logger.warn(
      'myTool.apiUrl is deprecated and will be removed in v3.0.0. ' +
      'Rename it to myTool.baseUrl in your app-config.yaml.',
    );
  }
}
```

### Correct — CHANGELOG deprecation entry

```markdown
## [2.1.0] — 2026-04-28

### Deprecated
- `myApi` export — use `myToolApiRef` instead.
  Will be removed in v3.0.0.
  ```typescript
  // before
  import { myApi } from '@scope/backstage-plugin-my-tool';

  // after
  import { myToolApiRef } from '@scope/backstage-plugin-my-tool';
  ```

- Action ID `myorg:create-repo` — use `myorg:create-repository` instead.
  Will be removed in v3.0.0. Update all Software Templates that reference
  this action ID.

- Config key `myTool.apiUrl` — rename to `myTool.baseUrl` in `app-config.yaml`.
  Will be removed in v3.0.0.

### Added
- `myToolApiRef` export — replaces the deprecated `myApi`
- Action `myorg:create-repository` — replaces the deprecated `myorg:create-repo`
- Config key `myTool.baseUrl` — replaces the deprecated `myTool.apiUrl`
```

### Incorrect

```typescript
// ❌ @deprecated without migration path
/**
 * @deprecated
 */
export const myApi = myToolApiRef;  // consumer doesn't know what to use instead

// ❌ deprecation and removal in same version
// v2.0.0 removes myApi without any prior @deprecated notice

// ❌ deprecated export broken before removal
/**
 * @deprecated Use myToolApiRef instead.
 */
export const myApi = undefined;    // deprecated must remain functional until removed
```

```markdown
<!-- ❌ CHANGELOG deprecation without migration path -->
### Deprecated
- `myApi` export                    <!-- no replacement, no migration steps -->
```

## Verify signal

Reject if:
- A TypeScript export is removed without a `@deprecated` JSDoc in a prior version
- A `@deprecated` JSDoc has no planned removal version and no migration path
- A deprecated export is set to `undefined`, `null`, or throws before the planned removal version
- A `CHANGELOG.md` `### Deprecated` entry has no migration steps
- An action ID, config key, or provider name is removed in a major version without a prior `### Deprecated` changelog entry

Flag for human review if:
- A deprecated export has been in the codebase for more than 2 major versions — removal is overdue
- A runtime deprecation warning logs a sensitive value — verify the warning message is safe
- The deprecation window is shorter than one minor version gap — consumers may not have time to migrate
- More than 3 APIs are deprecated in a single release — may indicate a large architectural shift that warrants a migration guide