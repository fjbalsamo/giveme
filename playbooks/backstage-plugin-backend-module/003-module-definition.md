# Guardrail 003 — Module definition

<!--
  Source: giveme community playbook (backstage-plugin-backend-module/003-module-definition.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

A Backstage backend module is a unit of functionality that extends an
existing plugin via its published extension points. Unlike a plugin,
a module has no HTTP surface, no database schema, and no independent
lifecycle — it lives and dies with the plugin it extends.

The module system solves a specific problem: how do you let operators
add capabilities to a plugin without forking it? An operator who wants
GitHub org discovery in the catalog doesn't fork the catalog — they
install a module that registers an `EntityProvider` via the catalog's
`CatalogExtensionPoint`. The plugin doesn't know about the module at
compile time. The backend wires them together at startup.

The most common mistakes are: declaring the wrong `pluginId` (which
causes the module to never connect to its target), using `coreServices`
that belong to plugins not modules (like `httpRouter`), and putting
initialization logic in `register` instead of `registerInit`.

## Decision

Backend modules are defined with `createBackendModule` from
`@backstage/backend-plugin-api`. The module declares its target plugin
via `pluginId` and its own identity via `moduleId`. All registration
logic lives in `registerInit`. The module accesses the target plugin's
extension point via `deps` — it never imports the plugin implementation
directly.

## Rules

- MUST: Use `createBackendModule` from `@backstage/backend-plugin-api`
- MUST: `pluginId` matches the target plugin's ID exactly — e.g. `'catalog'`, `'scaffolder'`
- MUST: `moduleId` is lowercase kebab-case and matches `package.json` `backstage.moduleId`
- MUST: All registration logic lives inside `registerInit` — not in the `register` function body
- MUST: The target plugin's extension point is declared in `deps` via its exported ref
- MUST: The module factory is the default export of `src/module.ts`
- MUST: `src/index.ts` only re-exports the default from `src/module.ts`
- MUST NOT: Declare `coreServices.httpRouter` in module deps — modules don't serve HTTP
- MUST NOT: Declare `coreServices.database` in module deps unless the module explicitly manages its own persistence — rare
- MUST NOT: Import from the target plugin's `-backend` package — use `-node` for extension point refs
- MUST NOT: Call extension point methods outside of `registerInit`
- SHOULD: Accept an optional `options` parameter for module-level configuration
- SHOULD: Keep the module definition lean — delegate logic to provider or action classes
- SHOULD NOT: Depend on more than 3 `coreServices` — if more are needed, reconsider the module's scope

## Examples

### Correct

```typescript
// src/module.ts
import {
  createBackendModule,
  coreServices,
} from '@backstage/backend-plugin-api';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node';
import { GithubOrgEntityProvider } from './providers/GithubOrgEntityProvider';

export const catalogModuleGithubOrgDiscovery = createBackendModule({
  pluginId: 'catalog',                          // target plugin ID
  moduleId: 'github-org-discovery',             // this module's ID
  register(env) {
    env.registerInit({
      deps: {
        catalog: catalogProcessingExtensionPoint, // from -node package
        logger: coreServices.logger,
        config: coreServices.rootConfig,
        scheduler: coreServices.scheduler,
      },
      async init({ catalog, logger, config, scheduler }) {
        const provider = GithubOrgEntityProvider.fromConfig(config, {
          logger,
          scheduler,
        });
        catalog.addEntityProvider(provider);     // register via extension point
      },
    });
  },
});

export default catalogModuleGithubOrgDiscovery;
```

```typescript
// src/index.ts
export { default } from './module';
```

### Correct — with options

```typescript
export type GithubOrgDiscoveryOptions = {
  orgs?: string[];
  schedule?: SchedulerServiceTaskScheduleDefinition;
};

export const catalogModuleGithubOrgDiscovery = createBackendModule({
  pluginId: 'catalog',
  moduleId: 'github-org-discovery',
  register(env, options?: GithubOrgDiscoveryOptions) {
    env.registerInit({
      deps: {
        catalog: catalogProcessingExtensionPoint,
        logger: coreServices.logger,
        config: coreServices.rootConfig,
        scheduler: coreServices.scheduler,
      },
      async init({ catalog, logger, config, scheduler }) {
        const provider = GithubOrgEntityProvider.fromConfig(config, {
          logger,
          scheduler,
          orgs: options?.orgs,
          schedule: options?.schedule,
        });
        catalog.addEntityProvider(provider);
      },
    });
  },
});
```

### Incorrect

```typescript
// ❌ wrong pluginId — module will never connect to catalog
export const myModule = createBackendModule({
  pluginId: 'github-org-discovery',   // should be 'catalog'
  moduleId: 'github-org-discovery',
  ...
});

// ❌ importing from -backend instead of -node
import { catalogProcessingExtensionPoint }
  from '@backstage/plugin-catalog-backend';  // use plugin-catalog-node

// ❌ httpRouter in a module
env.registerInit({
  deps: {
    httpRouter: coreServices.httpRouter,    // modules don't serve HTTP
    catalog: catalogProcessingExtensionPoint,
  },
});

// ❌ logic in register body instead of registerInit
register(env) {
  const provider = new GithubOrgEntityProvider(); // too early — deps not injected yet

  env.registerInit({
    deps: { catalog: catalogProcessingExtensionPoint },
    async init({ catalog }) {
      catalog.addEntityProvider(provider);
    },
  });
},

// ❌ calling extension point outside registerInit
register(env) {
  env.registerInit({ deps: { catalog: catalogProcessingExtensionPoint }, ... });
  // trying to use catalog here — not available yet
}
```

## Verify signal

Reject if:
- `pluginId` in `createBackendModule` does not match `backstage.pluginId` in `package.json`
- `moduleId` in `createBackendModule` does not match `backstage.moduleId` in `package.json`
- `coreServices.httpRouter` appears in any module `deps` object
- Extension point methods are called outside of `registerInit`
- The target plugin's extension point ref is imported from a `-backend` package
- `src/index.ts` contains anything other than `export { default } from './module'`

Flag for human review if:
- `coreServices.database` is declared in module deps — modules rarely own persistence, verify the design
- `deps` declares more than 4 services — may indicate the module is taking on too much responsibility
- No `options` parameter is defined — verify whether the module needs any operator configuration
- The module instantiates provider or action classes directly in `registerInit` without a factory method — makes testing harder