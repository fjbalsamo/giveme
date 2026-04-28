# Guardrail 002 — Package structure and naming

<!--
  Source: giveme community playbook (backstage-plugin-backend-rest/002-package-structure.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

A backend plugin that doesn't follow Backstage naming conventions creates
friction for every operator who installs it. They expect
`@scope/backstage-plugin-<id>-backend`, a predictable entry point, and
a clean separation between the plugin package and any node packages it
exports for frontend consumption.

Backend plugins also have a companion pattern: a `-node` package that
exports types, `ServiceRef`s, and `ExtensionPoint`s that other plugins
or modules need to depend on without pulling in the full backend
implementation. Confusing what goes in `-backend` vs `-node` is one
of the most common structural mistakes in publishable backend plugins.

## Decision

Publishable backend REST plugins follow the Backstage community naming
convention exactly. The package name ends in `-backend`. Types and
service refs shared with other plugins live in a separate `-node` package.
The plugin entry point exports the plugin factory as default. Build and
tooling use `backstage-cli` exclusively.

## Rules

- MUST: Package name follows the pattern `@scope/backstage-plugin-<id>-backend`
- MUST: Plugin ID matches the infix — if package is `backstage-plugin-my-tool-backend`, plugin ID is `my-tool`
- MUST: `package.json` `backstage.role` is `"backend-plugin"`
- MUST: `package.json` `backstage.pluginId` matches the plugin ID
- MUST: `src/index.ts` exports the plugin factory as default
- MUST: `main` and `types` in `package.json` point to `src/index.ts` for local development
- MUST: `publishConfig` overrides `main` and `types` to point to `dist/` for publishing
- MUST: `"files": ["dist", "migrations"]` — migrations directory is included if the plugin uses a database
- MUST: `"sideEffects": false`
- MUST: Scripts use `backstage-cli` — `build`, `lint`, `test`, `clean`, `prepack`, `postpack`
- MUST: `@backstage/backend-plugin-api` is in `dependencies` — not `peerDependencies`
- MUST NOT: Export `ServiceRef`s or `ExtensionPoint`s from the `-backend` package — these belong in a `-node` package
- MUST NOT: Frontend components or `ApiRef`s in the `-backend` package
- SHOULD: Create a companion `@scope/backstage-plugin-<id>-node` package if the plugin exposes `ExtensionPoint`s or `ServiceRef`s for modules
- SHOULD: Keep a `migrations/` folder at the root for knex migration files if the plugin uses a database
- SHOULD NOT: Put business logic in `src/index.ts` — delegate to `src/plugin.ts` and `src/router.ts`

## Examples

### Correct

```
@scope/backstage-plugin-my-tool-backend/
├── package.json
├── migrations/                    ← knex migrations (if DB used)
│   ├── 20240101_init.js
│   └── 20240201_add_column.js
└── src/
    ├── index.ts                   ← default export only
    ├── plugin.ts                  ← createBackendPlugin definition
    ├── router.ts                  ← Express router factory
    ├── service/
    │   └── MyToolService.ts       ← business logic
    └── database/
        └── MyToolDatabase.ts      ← DB access layer
```

```json
// package.json
{
  "name": "@scope/backstage-plugin-my-tool-backend",
  "version": "0.1.0",
  "license": "Apache-2.0",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "publishConfig": {
    "access": "public",
    "main": "dist/index.cjs.js",
    "types": "dist/index.d.ts"
  },
  "backstage": {
    "role": "backend-plugin",
    "pluginId": "my-tool"
  },
  "sideEffects": false,
  "scripts": {
    "start": "backstage-cli package start",
    "build": "backstage-cli package build",
    "lint": "backstage-cli package lint",
    "test": "backstage-cli package test",
    "clean": "backstage-cli package clean",
    "prepack": "backstage-cli package prepack",
    "postpack": "backstage-cli package postpack"
  },
  "dependencies": {
    "@backstage/backend-plugin-api": "^0.9.0",
    "@backstage/backend-defaults": "^0.7.0",
    "express": "^4.18.0",
    "express-promise-router": "^4.1.0"
  },
  "devDependencies": {
    "@backstage/cli": "^0.35.0",
    "@backstage/backend-test-utils": "^0.5.0",
    "@types/express": "^4.17.0",
    "supertest": "^6.0.0"
  },
  "files": [
    "dist",
    "migrations"
  ]
}
```

```typescript
// src/index.ts — default export only, nothing else
export { default } from './plugin';
```

### Correct — companion node package (when ExtensionPoints exist)

```
@scope/backstage-plugin-my-tool-node/
├── package.json                   ← role: "node-library"
└── src/
    ├── index.ts
    └── extensions/
        └── MyToolExtensionPoint.ts  ← exported for modules to depend on
```

### Incorrect

```json
// ❌ wrong role
{
  "backstage": {
    "role": "frontend-plugin"    // should be "backend-plugin"
  }
}

// ❌ missing migrations in files
{
  "files": ["dist"]              // add "migrations" if DB is used
}

// ❌ @backstage/backend-plugin-api as peerDependency
{
  "peerDependencies": {
    "@backstage/backend-plugin-api": "^0.9.0"  // move to dependencies
  }
}
```

```typescript
// ❌ business logic in index.ts
import { createBackendPlugin, coreServices } from '@backstage/backend-plugin-api';

// index.ts should only re-export from plugin.ts
export default createBackendPlugin({   // this belongs in plugin.ts
  pluginId: 'my-tool',
  register(env) { ... },
});

// ❌ ExtensionPoint exported from -backend package
export { myToolExtensionPoint } from './extensions/MyToolExtensionPoint';
// this belongs in @scope/backstage-plugin-my-tool-node
```

## Verify signal

Reject if:
- `package.json` `name` does not end in `-backend`
- `package.json` `backstage.role` is not `"backend-plugin"`
- `package.json` lists `@backstage/backend-plugin-api` under `peerDependencies`
- `package.json` is missing `publishConfig` with `main` and `types` pointing to `dist/`
- `package.json` is missing `"sideEffects": false`
- `src/index.ts` contains anything other than re-exports from `plugin.ts`
- `ExtensionPoint` or `ServiceRef` definitions are exported from the `-backend` package

Flag for human review if:
- `package.json` `files` array does not include `"migrations"` but the plugin uses `coreServices.database`
- A `-node` companion package is missing but the plugin registers `ExtensionPoint`s in `plugin.ts`
- `express` is missing from `dependencies` — backend plugins serving HTTP need it explicitly