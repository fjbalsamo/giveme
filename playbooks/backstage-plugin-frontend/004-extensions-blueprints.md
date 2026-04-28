# Guardrail 004 — Extensions and Blueprints

<!--
  Source: giveme community playbook (backstage-plugin-frontend/004-extensions-blueprints.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

In the legacy system, a plugin wired its own components directly into
the app via `createComponentExtension` or by exporting React components
that the app imported and placed manually. This created tight coupling —
the app had to know about every component the plugin provided.

The new system inverts this. Plugins declare extensions using Blueprints,
and the app decides how and where to render them. A `PageBlueprint` says
"I provide a page" — the app decides the route. A `NavItemBlueprint` says
"I provide a nav item" — the app decides where in the sidebar. This makes
plugins truly composable and overridable without touching the plugin code.

## Decision

Every UI contribution from a plugin is declared as an extension using
the appropriate Blueprint from `@backstage/frontend-plugin-api`. Extensions
are lazy-loaded. All extensions are registered in the `extensions` array
of `createFrontendPlugin`. The legacy `createComponentExtension` and
`createRoutableExtension` are not used in new plugins.

## Rules

- MUST: Use Blueprints from `@backstage/frontend-plugin-api` for all extension types
- MUST: Page components use `PageBlueprint.make()` with a `loader` that lazy-loads the component
- MUST: Nav items use `NavItemBlueprint.make()` with `routeRef`, `title`, and `icon`
- MUST: Entity cards use `EntityCardBlueprint.make()` with a `loader` and optional `filter`
- MUST: Entity content tabs use `EntityContentBlueprint.make()` with `title`, `path`, and `loader`
- MUST: APIs use `ApiBlueprint.make()` — not `createApiFactory` registered in `createPlugin`
- MUST: All extensions use lazy loading via dynamic `import()` in the `loader` function
- MUST: All extensions are registered in the `extensions` array of `createFrontendPlugin`
- MUST NOT: Use `createComponentExtension` from `@backstage/core-plugin-api`
- MUST NOT: Use `createRoutableExtension` from `@backstage/core-plugin-api`
- MUST NOT: Import React components eagerly in extension definitions — always use `loader`
- SHOULD: Keep each extension in its own file under `src/extensions/`
- SHOULD: Name extension files after the Blueprint type — `MyToolPage.tsx`, `MyToolNavItem.tsx`
- SHOULD NOT: Put business logic directly in Blueprint `loader` functions — delegate to components

## Examples

### Correct

```typescript
// src/extensions/MyToolPage.tsx
import { PageBlueprint } from '@backstage/frontend-plugin-api';
import { rootRouteRef } from '../routes';

export const myToolPage = PageBlueprint.make({
  params: {
    path: '/my-tool',
    routeRef: rootRouteRef,
    loader: () =>
      import('../components/MyToolPage').then(m => <m.MyToolPage />),
  },
});
```

```typescript
// src/extensions/MyToolNavItem.tsx
import { NavItemBlueprint } from '@backstage/frontend-plugin-api';
import { rootRouteRef } from '../routes';
import { MyToolIcon } from '../icons';

export const myToolNavItem = NavItemBlueprint.make({
  params: {
    routeRef: rootRouteRef,
    title: 'My Tool',
    icon: MyToolIcon,
  },
});
```

```typescript
// src/extensions/MyToolEntityCard.tsx
import { EntityCardBlueprint } from '@backstage/frontend-plugin-api';

export const myToolEntityCard = EntityCardBlueprint.make({
  name: 'my-tool',
  params: {
    filter: 'kind:component',             // only show for Component entities
    loader: () =>
      import('../components/MyToolEntityCard').then(m => <m.MyToolEntityCard />),
  },
});
```

```typescript
// src/extensions/MyToolApi.tsx
import { ApiBlueprint } from '@backstage/frontend-plugin-api';
import { myToolApiRef } from '../api/MyToolApi';
import { MyToolApiImpl } from '../api/MyToolApiImpl';

export const myToolApi = ApiBlueprint.make({
  name: 'my-tool',
  params: defineParams => defineParams({
    api: myToolApiRef,
    deps: {},
    factory: () => new MyToolApiImpl(),
  }),
});
```

```typescript
// src/plugin.ts — all extensions registered
import { createFrontendPlugin } from '@backstage/frontend-plugin-api';

export const myToolPlugin = createFrontendPlugin({
  pluginId: 'my-tool',
  extensions: [
    myToolApi,
    myToolPage,
    myToolNavItem,
    myToolEntityCard,
  ],
});
```

### Incorrect

```typescript
// ❌ legacy createComponentExtension
import { createComponentExtension } from '@backstage/core-plugin-api';

export const MyToolEntityCard = myToolPlugin.provide(
  createComponentExtension({
    name: 'MyToolEntityCard',
    component: {
      lazy: () => import('./components/MyToolEntityCard')
        .then(m => m.MyToolEntityCard),
    },
  }),
);

// ❌ eager import in extension loader
import { MyToolPage } from '../components/MyToolPage'; // eager — breaks lazy loading

export const myToolPage = PageBlueprint.make({
  params: {
    path: '/my-tool',
    routeRef: rootRouteRef,
    loader: () => Promise.resolve(<MyToolPage />), // not a dynamic import
  },
});

// ❌ extension not registered in createFrontendPlugin
export const myToolPage = PageBlueprint.make({ ... });

export const myToolPlugin = createFrontendPlugin({
  pluginId: 'my-tool',
  extensions: [],     // myToolPage is defined but not registered
});
```

## Verify signal

Reject if:
- `createComponentExtension` from `@backstage/core-plugin-api` appears in `src/`
- `createRoutableExtension` from `@backstage/core-plugin-api` appears in `src/`
- A Blueprint `loader` function does not use a dynamic `import()` — uses a direct import instead
- An extension is defined in `src/` but not present in the `extensions` array of `createFrontendPlugin`
- `ApiBlueprint` is not used for API extensions — `createApiFactory` registered directly in `createPlugin`

Flag for human review if:
- A `PageBlueprint` does not have a corresponding `NavItemBlueprint` — users may have no way to navigate to the page
- An `EntityCardBlueprint` has no `filter` — card will appear on all entity kinds, verify this is intentional
- More than 5 extensions are registered in a single plugin — consider splitting into multiple plugins or using extension points