# Guardrail 007 — Migration path from legacy to new frontend system

<!--
  Source: giveme community playbook (backstage-plugin-frontend/007-migration-path.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Most Backstage plugins in the wild today are legacy — they use `createPlugin`
from `@backstage/core-plugin-api`, register APIs directly, and export
components that the host app wires manually. The Backstage team has made
clear that legacy support will be dropped in a future release.

Migration is not a big-bang rewrite. Backstage supports a hybrid mode
where the host app runs the new system while individual plugins migrate
incrementally. A plugin can export both a new system default export and
a legacy named export under `/alpha` during the transition period.

The risk of staying in hybrid mode too long is real — the Backstage team
explicitly warns against it. This guardrail defines how to detect legacy
code, how to migrate it incrementally, and when to declare the migration
complete.

## Decision

Legacy plugins are migrated to the new frontend system incrementally,
one extension at a time. During migration, the new system export is the
default export and the legacy export lives under a transitional named
export. Migration is declared complete when no imports from
`@backstage/core-plugin-api` remain in `plugin.ts`, `routes.ts`, or
extension files — only in API implementation files where the core service
refs (`discoveryApiRef`, `fetchApiRef`, etc.) are still needed.

## Rules

- MUST: Before migrating, read `backstage.json` and fetch the migration guide for the detected version (guardrail 001)
- MUST: Detect legacy indicators before generating any migration code — see Verify signal
- MUST: Migrate one extension at a time — page first, then nav item, then entity cards, then APIs last
- MUST: During migration, new system export is the default export of `src/index.ts`
- MUST: During migration, legacy plugin is available as a transitional named export — not as default
- MUST: Update `src/index.ts` to export the new system plugin as default before declaring migration complete
- MUST: Remove `createPlugin` and all `createComponentExtension` / `createRoutableExtension` calls when migration is complete
- MUST NOT: Keep the host app in hybrid mode after all plugins it uses have migrated
- MUST NOT: Introduce new legacy `createPlugin` code during a migration — all new extensions use Blueprints
- SHOULD: Migrate APIs last — they require the most careful testing to ensure consumers are not broken
- SHOULD: Run the plugin in a local dev harness (`dev/index.tsx`) after each extension migration to verify behavior
- SHOULD NOT: Declare migration complete while any `createComponentExtension` or `createRoutableExtension` call remains

## Legacy detection checklist

Before generating migration code, the agent checks for these legacy indicators:

| Indicator | File | Legacy signal |
|---|---|---|
| `createPlugin` import | `src/plugin.ts` | Core legacy indicator |
| `createComponentExtension` | anywhere in `src/` | Legacy extension system |
| `createRoutableExtension` | anywhere in `src/` | Legacy extension system |
| `apis: [...]` in `createPlugin` | `src/plugin.ts` | APIs not yet migrated |
| `routes` defined in app code | `packages/app/` | Plugin not self-contained |
| No default export in `src/index.ts` | `src/index.ts` | New system not adopted |
| Import from `@backstage/core-plugin-api` in `plugin.ts` | `src/plugin.ts` | Mixed system |

If 3 or more indicators are present → plugin is fully legacy, full migration needed.
If 1-2 indicators are present → plugin is partially migrated, complete the migration.
If 0 indicators are present → plugin is already on the new system, no migration needed.

## Migration sequence

### Step 1 — Scaffold the new plugin definition

```typescript
// src/plugin.new.ts — new system alongside legacy
import { createFrontendPlugin } from '@backstage/frontend-plugin-api';

export const myToolPlugin = createFrontendPlugin({
  pluginId: 'my-tool',
  extensions: [],        // extensions added one by one in following steps
});

export default myToolPlugin;
```

### Step 2 — Migrate the page extension

```typescript
// Before — legacy
export const MyToolPage = myToolPlugin.provide(
  createRoutableExtension({
    name: 'MyToolPage',
    component: () => import('./components/MyToolPage').then(m => m.MyToolPage),
    mountPoint: rootRouteRef,
  }),
);

// After — new system
import { PageBlueprint } from '@backstage/frontend-plugin-api';

export const myToolPage = PageBlueprint.make({
  params: {
    path: '/my-tool',
    routeRef: rootRouteRef,
    loader: () =>
      import('./components/MyToolPage').then(m => <m.MyToolPage />),
  },
});
```

Add `myToolPage` to the `extensions` array and verify in `dev/index.tsx`.

### Step 3 — Migrate the nav item

```typescript
// Before — legacy (wired manually in app code)
// packages/app/src/App.tsx had: <Route path="/my-tool" element={<MyToolPage />} />

// After — new system (self-contained)
import { NavItemBlueprint } from '@backstage/frontend-plugin-api';

export const myToolNavItem = NavItemBlueprint.make({
  params: {
    routeRef: rootRouteRef,
    title: 'My Tool',
    icon: MyToolIcon,
  },
});
```

### Step 4 — Migrate entity cards and content tabs

```typescript
// Before — legacy
export const EntityMyToolCard = myToolPlugin.provide(
  createComponentExtension({
    name: 'EntityMyToolCard',
    component: {
      lazy: () => import('./components/EntityMyToolCard')
        .then(m => m.EntityMyToolCard),
    },
  }),
);

// After — new system
import { EntityCardBlueprint } from '@backstage/frontend-plugin-api';

export const myToolEntityCard = EntityCardBlueprint.make({
  name: 'my-tool',
  params: {
    filter: 'kind:component',
    loader: () =>
      import('./components/EntityMyToolCard').then(m => <m.EntityMyToolCard />),
  },
});
```

### Step 5 — Migrate APIs

```typescript
// Before — legacy
const myToolPlugin = createPlugin({
  id: 'my-tool',
  apis: [
    createApiFactory({
      api: myToolApiRef,
      deps: { discoveryApi: discoveryApiRef },
      factory: ({ discoveryApi }) => new MyToolApiImpl(discoveryApi),
    }),
  ],
});

// After — new system
import { ApiBlueprint } from '@backstage/frontend-plugin-api';

export const myToolApi = ApiBlueprint.make({
  name: 'my-tool',
  params: defineParams => defineParams({
    api: myToolApiRef,
    deps: { discoveryApi: discoveryApiRef },
    factory: ({ discoveryApi }) => new MyToolApiImpl(discoveryApi),
  }),
});
```

### Step 6 — Update exports and remove legacy code

```typescript
// src/index.ts — migration complete
export { default } from './plugin';           // new system is the only export
export { myToolApiRef } from './api/MyToolApi';
export type { MyToolApi } from './api/MyToolApi';
export { myToolApiFactory } from './api/MyToolApiFactory';
export { MyToolApiMock } from './api/MyToolApiMock';

// Delete: plugin.legacy.ts, any createComponentExtension calls
// Delete: any route wiring from packages/app/src/App.tsx
```

## Verify signal

Reject if:
- `createComponentExtension` exists in `src/` and the migration is declared complete in `plan.md`
- `createRoutableExtension` exists in `src/` and the migration is declared complete in `plan.md`
- `createPlugin` is used without a corresponding `createFrontendPlugin` default export — partial migration with no new system entry
- New `createComponentExtension` or `createRoutableExtension` calls are added during a migration — all new extensions must use Blueprints
- `src/index.ts` has no default export after migration is declared complete in `plan.md`

Flag for human review if:
- 3 or more legacy indicators are detected — full migration scope, estimate effort before starting
- The plugin has consumers in other packages that import legacy named exports — coordinate migration with consumers
- `packages/app/` still contains manual route wiring for this plugin after migration — app code cleanup needed
- The migration has been in hybrid mode for more than one Backstage minor version — risk of accumulating migration debt