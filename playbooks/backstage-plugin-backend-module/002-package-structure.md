# Guardrail 002 — Package structure and naming

<!--
  Source: giveme community playbook (backstage-plugin-backend-module/002-package-structure.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Backstage module naming follows a strict convention that encodes three
pieces of information in the package name: that it is a Backstage plugin,
that it targets the backend, that it is a module, which plugin it extends,
and what feature it provides. This verbosity is intentional — operators
scanning `node_modules` or a package registry need to understand what a
package does without reading its README.

The convention is:
`@scope/backstage-plugin-<target-plugin-id>-backend-module-<feature-id>`

A module that deviates from this pattern creates confusion for operators,
breaks the Backstage CLI's auto-discovery, and makes the package harder
to find in search results.

Modules also have a companion pattern distinct from REST plugins: they
depend on the target plugin's `-node` package for the ExtensionPoint
definition — not on the `-backend` package directly. Depending on
`-backend` instead of `-node` creates a circular dependency risk and
forces operators to install the full plugin implementation as a
transitive dependency of the module.

## Decision

Backend modules follow the Backstage naming convention exactly. The
package depends on the target plugin's `-node` package for ExtensionPoint
types, and on `@backstage/backend-plugin-api` for the module system.
The module factory is the default export of `src/index.ts`. The structure
mirrors the backend REST plugin but without a router or database layer.

## Rules

- MUST: Package name follows `@scope/backstage-plugin-<target-id>-backend-module-<feature-id>`
- MUST: `package.json` `backstage.role` is `"backend-plugin-module"`
- MUST: `package.json` `backstage.pluginId` matches the target plugin ID — not the module's own ID
- MUST: `package.json` `backstage.moduleId` is set to the feature ID — e.g. `"github-org-discovery"`
- MUST: The target plugin's `-node` package is in `dependencies` — not the `-backend` package
- MUST: `@backstage/backend-plugin-api` is in `dependencies`
- MUST: `main` and `types` point to `src/index.ts` for local development
- MUST: `publishConfig` overrides `main` and `types` to `dist/` for publishing
- MUST: `"sideEffects": false` and `"files": ["dist"]`
- MUST: `src/index.ts` exports the module factory as default and nothing else
- MUST NOT: Depend on the target plugin's `-backend` package — use `-node` for ExtensionPoint types
- MUST NOT: Export ExtensionPoint definitions from a module — those belong in the target plugin's `-node` package
- MUST NOT: Include a router, HTTP endpoints, or database access in a module — modules extend plugins, they don't serve HTTP
- SHOULD: Keep module logic in `src/module.ts` — not inlined in `src/index.ts`
- SHOULD NOT: Name the feature ID the same as the target plugin ID — it must describe what the module adds

## Examples

### Correct — naming examples

```
Target plugin: catalog
Feature: GitHub org discovery
Package: @scope/backstage-plugin-catalog-backend-module-github-org-discovery
pluginId: "catalog"
moduleId: "github-org-discovery"
```

```
Target plugin: scaffolder
Feature: Custom HTTP action
Package: @scope/backstage-plugin-scaffolder-backend-module-http-request
pluginId: "scaffolder"
moduleId: "http-request"
```

```
Target plugin: my-tool (custom internal plugin)
Feature: Slack notifications
Package: @scope/backstage-plugin-my-tool-backend-module-slack-notifications
pluginId: "my-tool"
moduleId: "slack-notifications"
```

### Correct — package.json

```json
{
  "name": "@scope/backstage-plugin-catalog-backend-module-github-org-discovery",
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
    "role": "backend-plugin-module",
    "pluginId": "catalog",
    "moduleId": "github-org-discovery"
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
    "@backstage/plugin-catalog-node": "^1.15.0"
  },
  "devDependencies": {
    "@backstage/cli": "^0.35.0",
    "@backstage/backend-test-utils": "^0.5.0",
    "@backstage/plugin-catalog-backend": "^1.26.0"
  },
  "files": [
    "dist"
  ]
}
```

```typescript
// src/index.ts — default export only
export { default } from './module';
```

### Correct — folder structure

```
@scope/backstage-plugin-catalog-backend-module-github-org-discovery/
├── package.json
└── src/
    ├── index.ts          ← default export only
    ├── module.ts         ← createBackendModule definition
    └── providers/
        └── GithubOrgEntityProvider.ts  ← the actual logic
```

### Incorrect

```json
// ❌ wrong role
{
  "backstage": {
    "role": "backend-plugin"    // should be "backend-plugin-module"
  }
}

// ❌ pluginId set to module's own id instead of target
{
  "backstage": {
    "pluginId": "github-org-discovery",  // should be "catalog"
    "moduleId": "github-org-discovery"
  }
}

// ❌ depending on -backend instead of -node
{
  "dependencies": {
    "@backstage/plugin-catalog-backend": "^1.26.0"  // use plugin-catalog-node instead
  }
}
```

```typescript
// ❌ business logic in index.ts
export default createBackendModule({   // move this to module.ts
  pluginId: 'catalog',
  moduleId: 'github-org-discovery',
  register(env) { ... },
});

// ❌ HTTP router in a module
import Router from 'express-promise-router';
const router = Router();               // modules don't serve HTTP — that's for plugins
```

## Verify signal

Reject if:
- `package.json` `name` does not match `*-backend-module-*` pattern
- `package.json` `backstage.role` is not `"backend-plugin-module"`
- `package.json` `backstage.pluginId` does not match the target plugin ID — e.g. `"catalog"` not `"github-org-discovery"`
- `package.json` is missing `backstage.moduleId`
- The target plugin's `-backend` package is in `dependencies` — use `-node` instead
- `src/index.ts` contains anything other than `export { default } from './module'`
- An Express router is instantiated anywhere in `src/`

Flag for human review if:
- `moduleId` in `package.json` does not clearly describe the feature — operators need to understand what the module does from the name alone
- The target plugin's `-node` package is in `devDependencies` instead of `dependencies` — it must be a runtime dependency
- `package.json` `files` includes `"migrations"` — modules typically don't own database schemas, verify this is intentional