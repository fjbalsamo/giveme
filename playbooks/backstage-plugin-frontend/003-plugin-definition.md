# Guardrail 003 — Plugin definition

<!--
  Source: giveme community playbook (backstage-plugin-frontend/003-plugin-definition.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Backstage has two frontend plugin systems that coexist today. The legacy
system uses `createPlugin` from `@backstage/core-plugin-api`. The new
system uses `createFrontendPlugin` from `@backstage/frontend-plugin-api`.
They are architecturally different — not just a rename.

In the legacy system, the plugin registers APIs directly and wires routes
in the app. In the new system, everything is an extension — APIs, pages,
nav items, entity cards — and the plugin just declares which extensions
it provides. The host app decides how to render them.

Starting a new plugin in the legacy system means planning a migration later.
The Backstage team has explicitly stated that legacy support will be dropped
in a future release.

## Decision

New plugins use `createFrontendPlugin` from `@backstage/frontend-plugin-api`
as their default export. Legacy plugins that still use `createPlugin` are
candidates for migration and must be identified as such. When both systems
need to be supported during a transition, the new system export is the
default and the legacy export lives under an `/alpha` sub-path.

## Rules

- MUST: New plugins use `createFrontendPlugin` from `@backstage/frontend-plugin-api`
- MUST: The plugin instance is the default export of `src/index.ts`
- MUST: Plugin ID is lowercase kebab-case and matches `package.json` `backstage.pluginId`
- MUST: All extensions (pages, nav items, APIs, entity cards) are declared in the `extensions` array of `createFrontendPlugin`
- MUST: When supporting both systems during migration, new system is default export and legacy is exported under `/alpha` sub-path
- MUST NOT: Use `createPlugin` from `@backstage/core-plugin-api` for new plugins
- MUST NOT: Register APIs directly in `createPlugin({ apis: [...] })` — APIs are extensions in the new system
- MUST NOT: Mix imports from `@backstage/core-plugin-api` and `@backstage/frontend-plugin-api` in the same plugin definition file
- SHOULD: Keep `plugin.ts` as a dedicated file for the plugin definition — not inlined in `index.ts`
- SHOULD: Keep `routes.ts` as a dedicated file for `createRouteRef` definitions
- SHOULD NOT: Export the plugin instance as a named export — default export only

## Examples

### Correct — new system

```typescript
// src/routes.ts
import { createRouteRef } from '@backstage/frontend-plugin-api';

export const rootRouteRef = createRouteRef();
```

```typescript
// src/plugin.ts
import { createFrontendPlugin } from '@backstage/frontend-plugin-api';
import { myToolPage } from './extensions/MyToolPage';
import { myToolNavItem } from './extensions/MyToolNavItem';
import { myToolApi } from './extensions/MyToolApi';
import { rootRouteRef } from './routes';

export const myToolPlugin = createFrontendPlugin({
  pluginId: 'my-tool',                    // matches package.json backstage.pluginId
  extensions: [
    myToolApi,                            // APIs are extensions now
    myToolPage,
    myToolNavItem,
  ],
  routes: {
    root: rootRouteRef,                   // available to other plugins
  },
});

export default myToolPlugin;             // default export
```

```typescript
// src/index.ts
export { default } from './plugin';      // default export is the plugin
export { myToolApiRef } from './api/MyToolApi';
export type { MyToolApi } from './api/MyToolApi';
```

### Correct — hybrid during migration (legacy app + new plugin)

```typescript
// src/index.ts — new system is default
export { default } from './plugin';      // createFrontendPlugin

// src/index.alpha.ts — legacy export for apps still on old system
export { myToolPluginLegacy as myToolPlugin } from './pluginLegacy';
```

### Incorrect

```typescript
// ❌ new plugin using legacy createPlugin
import { createPlugin, createRouteRef } from '@backstage/core-plugin-api';

export const myToolPlugin = createPlugin({
  id: 'my-tool',
  apis: [                                // APIs don't belong here in new system
    createApiFactory({
      api: myToolApiRef,
      deps: {},
      factory: () => new MyToolApiImpl(),
    }),
  ],
  routes: {
    root: rootRouteRef,
  },
});

// ❌ mixing imports from both systems in plugin.ts
import { createFrontendPlugin } from '@backstage/frontend-plugin-api';
import { createRouteRef } from '@backstage/core-plugin-api'; // wrong package for new system

// ❌ plugin ID doesn't match package.json
export const myToolPlugin = createFrontendPlugin({
  pluginId: 'mytool',                    // should be 'my-tool'
  extensions: [],
});

// ❌ plugin as named export only — missing default
export const myToolPlugin = createFrontendPlugin({ ... });
// missing: export default myToolPlugin
```

## Verify signal

Reject if:
- `createPlugin` from `@backstage/core-plugin-api` is used in a file that is not explicitly marked as legacy or alpha
- `createFrontendPlugin` and `createPlugin` both appear in the same `plugin.ts` file
- `pluginId` in `createFrontendPlugin` does not match `backstage.pluginId` in `package.json`
- `src/index.ts` does not have a default export
- `createPlugin({ apis: [...] })` is used with a non-empty `apis` array in a new plugin — APIs must be extensions

Flag for human review if:
- The plugin imports from both `@backstage/core-plugin-api` and `@backstage/frontend-plugin-api` — verify which is intentional
- `extensions` array in `createFrontendPlugin` is empty — plugin may be incomplete or still being scaffolded
- A legacy `createPlugin` export exists without a corresponding new system default export — migration may be incomplete