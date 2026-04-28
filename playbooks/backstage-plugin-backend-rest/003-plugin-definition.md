# Guardrail 003 — Plugin definition

<!--
  Source: giveme community playbook (backstage-plugin-backend-rest/003-plugin-definition.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

The new Backstage backend system replaces the legacy `createRouter` +
`PluginEnvironment` pattern with `createBackendPlugin` + `coreServices`.
In the legacy system, plugins received a fat `env` object with everything
pre-wired. In the new system, plugins declare exactly which services they
need via `deps` — the backend injects only what is requested.

This explicit dependency declaration is not just style. It enables the
backend to create isolated service instances per plugin, detect circular
dependencies at startup, and swap implementations without touching plugin
code. A plugin that depends on `coreServices.logger` gets a logger scoped
to its plugin ID automatically.

The most common mistake when migrating or writing new plugins is declaring
too many deps — pulling in services the plugin doesn't actually use, or
using `@backstage/backend-common` equivalents that have been superseded
by `coreServices` refs.

## Decision

Backend REST plugins are defined with `createBackendPlugin` from
`@backstage/backend-plugin-api`. The plugin declares only the `coreServices`
it actually uses in `deps`. All initialization logic lives in `registerInit`.
The plugin factory is the default export of `src/plugin.ts` and is
re-exported from `src/index.ts`.

## Rules

- MUST: Use `createBackendPlugin` from `@backstage/backend-plugin-api`
- MUST: `pluginId` is lowercase kebab-case and matches `package.json` `backstage.pluginId`
- MUST: Declare only services actually used in `deps` — no unused service dependencies
- MUST: All initialization logic lives inside `registerInit` — not in the `register` function body
- MUST: The plugin factory is the default export of `src/plugin.ts`
- MUST: `src/index.ts` only re-exports the default from `src/plugin.ts`
- MUST NOT: Import from `@backstage/backend-common` for services that have `coreServices` equivalents
- MUST NOT: Use `coreServices.tokenManager` — deprecated, use `coreServices.auth` instead
- MUST NOT: Put route handler logic inside `registerInit` directly — delegate to `createRouter` in `src/router.ts`
- MUST NOT: Access `process.env` directly — use `coreServices.rootConfig`
- SHOULD: Accept an optional `options` parameter for plugin-level configuration
- SHOULD: Pass all dependencies into `createRouter` as a typed `RouterOptions` object
- SHOULD NOT: Use `coreServices.rootHttpRouter` unless the plugin needs to mount at the backend root — use `coreServices.httpRouter` for plugin-scoped routes

## Examples

### Correct

```typescript
// src/plugin.ts
import {
  createBackendPlugin,
  coreServices,
} from '@backstage/backend-plugin-api';
import { createRouter } from './router';

export const myToolPlugin = createBackendPlugin({
  pluginId: 'my-tool',                        // matches package.json backstage.pluginId
  register(env) {
    env.registerInit({
      deps: {
        httpRouter: coreServices.httpRouter,   // plugin-scoped router
        logger: coreServices.logger,
        config: coreServices.rootConfig,
        database: coreServices.database,
        auth: coreServices.auth,
        httpAuth: coreServices.httpAuth,
      },
      async init({ httpRouter, logger, config, database, auth, httpAuth }) {
        httpRouter.use(
          await createRouter({               // delegate to router.ts
            logger,
            config,
            database,
            auth,
            httpAuth,
          }),
        );
        httpRouter.addAuthPolicy({           // declare auth policy explicitly
          path: '/health',
          allow: 'unauthenticated',
        });
      },
    });
  },
});

export default myToolPlugin;
```

```typescript
// src/index.ts
export { default } from './plugin';
```

### Correct — with options

```typescript
// src/plugin.ts
export type MyToolPluginOptions = {
  skipMigrations?: boolean;
};

export const myToolPlugin = createBackendPlugin({
  pluginId: 'my-tool',
  register(env, options?: MyToolPluginOptions) {
    env.registerInit({
      deps: {
        httpRouter: coreServices.httpRouter,
        logger: coreServices.logger,
        database: coreServices.database,
      },
      async init({ httpRouter, logger, database }) {
        if (!options?.skipMigrations) {
          await runMigrations(database);
        }
        httpRouter.use(await createRouter({ logger, database }));
      },
    });
  },
});
```

### Incorrect

```typescript
// ❌ using legacy PluginEnvironment pattern
export async function createPlugin(env: PluginEnvironment) {
  return createRouter({
    logger: env.logger,
    database: env.database,
  });
}

// ❌ unused service in deps
env.registerInit({
  deps: {
    httpRouter: coreServices.httpRouter,
    logger: coreServices.logger,
    scheduler: coreServices.scheduler,   // not used anywhere in init
    cache: coreServices.cache,           // not used anywhere in init
  },
  async init({ httpRouter, logger }) {   // scheduler and cache never used
    ...
  },
});

// ❌ deprecated tokenManager
env.registerInit({
  deps: {
    tokenManager: coreServices.tokenManager,  // deprecated — use coreServices.auth
  },
});

// ❌ route logic inline in registerInit
async init({ httpRouter, logger }) {
  httpRouter.use('/items', async (req, res) => {  // move this to router.ts
    res.json({ items: [] });
  });
},

// ❌ reading env directly
async init({ config }) {
  const secret = process.env.MY_SECRET;   // use config.getString('myTool.secret')
}
```

## Verify signal

Reject if:
- `createPlugin` or `PluginEnvironment` from `@backstage/backend-common` appears in `src/plugin.ts`
- `coreServices.tokenManager` is declared in any `deps` object
- `process.env` is accessed directly anywhere in `src/` — use `coreServices.rootConfig`
- Route handler functions are defined inline inside `registerInit` — must be in `router.ts`
- `pluginId` in `createBackendPlugin` does not match `backstage.pluginId` in `package.json`
- `src/index.ts` contains anything other than `export { default } from './plugin'`

Flag for human review if:
- `deps` declares more than 6 services — may indicate the plugin is doing too much
- `coreServices.rootHttpRouter` is used instead of `coreServices.httpRouter` — verify the plugin intentionally mounts at root
- No `httpRouter.addAuthPolicy` call exists — all routes will require authentication by default, verify this is intentional
- `options` parameter is typed as required — backend plugins should prefer optional options to ease installation