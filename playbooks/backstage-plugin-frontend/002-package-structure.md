# Guardrail 002 — Package structure and naming

<!--
  Source: giveme community playbook (backstage-plugin-frontend/002-package-structure.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

A publishable Backstage plugin that doesn't follow the community naming
and structure conventions creates friction for every developer who installs
it. They expect `@scope/backstage-plugin-xxx`, a predictable `src/index.ts`
entry point, and a clean separation between the public API and internal
implementation. Deviating from these conventions breaks tooling, confuses
integrators, and makes the plugin harder to maintain across Backstage
version upgrades.

## Decision

Publishable frontend plugins follow the Backstage community naming convention
exactly. The package structure separates public exports from internal
implementation. The `src/index.ts` is the only public surface — everything
else is internal unless explicitly exported there.

There are two build profiles — **standard** and **module-federation**:

- **Standard** (default): the plugin is bundled by the host Backstage app at
  build time. No special flags. This is the correct choice for the vast
  majority of publishable plugins.
- **Module Federation**: the plugin is built as an independent remote and
  loaded dynamically at runtime by the host app. This is a conscious
  architectural decision to treat the plugin as a microfrontend. It requires
  `--module-federation` in the build script and additional `backstage` fields
  in `package.json`. Do not use this unless the team has explicitly decided
  to adopt a microfrontend architecture.

## Rules

- MUST: Package name follows the pattern `@scope/backstage-plugin-<id>` where `<id>` is lowercase kebab-case
- MUST: Plugin ID matches the package name suffix — if package is `backstage-plugin-my-tool`, plugin ID is `my-tool`
- MUST: `src/index.ts` is the single public export surface — no other file is imported directly by consumers
- MUST: `package.json` `main` and `types` point to `src/index.ts` for local development
- MUST: `package.json` has a `publishConfig` field that overrides `main` and `types` to point to `dist/` for publishing
- MUST: `package.json` has `"files": ["dist"]` — only compiled output is published
- MUST: `package.json` has `"sideEffects": false`
- MUST: `@backstage/*` packages go in `dependencies` — only `react`, `react-dom`, and `react-router-dom` go in `peerDependencies`
- MUST: `backstage` field in `package.json` declares `role`, `pluginId`, `pluginPackages`, and `externalModuleFederation`
- MUST: `pluginId` in `package.json` `backstage` field matches the plugin ID used in `createFrontendPlugin` or `createPlugin`
- MUST: Scripts use `backstage-cli` — `build`, `lint`, `test`, `start`, `prepack`, `postpack`
- MUST NOT: Use `--module-federation` flag in `build` script unless the plugin is intentionally a microfrontend — document this decision in `plan.md`
- SHOULD: Standard build script is simply `backstage-cli package build` with no additional flags
- MUST: Plugin default export is the plugin instance from `src/index.ts`
- MUST NOT: Publish internal components that are not part of the plugin's public contract
- MUST NOT: Import from `src/` paths directly in documentation or README examples — always import from the package name
- SHOULD: Keep a `src/components/` folder for React components and a `src/api/` folder for API implementations
- SHOULD: Export types separately when consumers need them for TypeScript integration
- SHOULD NOT: Use barrel exports (`export * from`) beyond `src/index.ts` — be explicit about what is public

## Examples

### Correct

```
@scope/backstage-plugin-my-tool/
├── package.json
├── src/
│   ├── index.ts              ← only public surface
│   ├── plugin.ts             ← plugin definition
│   ├── routes.ts             ← route refs
│   ├── components/
│   │   ├── MyToolPage/
│   │   │   ├── index.ts
│   │   │   └── MyToolPage.tsx
│   │   └── index.ts
│   └── api/
│       ├── MyToolApi.ts      ← ApiRef definition
│       └── MyToolApiImpl.ts  ← implementation
└── dev/
    └── index.tsx             ← local dev harness
```

```json
// package.json
{
  "name": "@scope/backstage-plugin-my-tool",
  "version": "0.1.0",
  "license": "Apache-2.0",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "publishConfig": {
    "access": "public",
    "main": "dist/index.esm.js",
    "types": "dist/index.d.ts"
  },
  "backstage": {
    "role": "frontend-plugin",
    "pluginId": "my-tool",
    "pluginPackages": [
      "@scope/backstage-plugin-my-tool"
    ]
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
    "@backstage/core-components": "^0.18.0",
    "@backstage/core-plugin-api": "^1.12.0",
    "@backstage/frontend-plugin-api": "^0.16.0",
    "@backstage/theme": "^0.7.0"
  },
  "peerDependencies": {
    "react": "^16.13.1 || ^17.0.0 || ^18.0.0",
    "react-dom": "^16.13.1 || ^17.0.0 || ^18.0.0",
    "react-router-dom": "^6.0.0"
  },
  "devDependencies": {
    "@backstage/cli": "^0.35.0",
    "@backstage/dev-utils": "^1.1.0",
    "@backstage/test-utils": "^1.7.0",
    "@testing-library/jest-dom": "^6.0.0",
    "@testing-library/react": "^14.0.0",
    "@testing-library/user-event": "^14.0.0",
    "msw": "^1.0.0",
    "react": "^18.0.0",
    "react-dom": "^18.0.0",
    "react-router-dom": "^6.0.0"
  },
  "files": [
    "dist"
  ]
}
```

```typescript
// src/index.ts — explicit public surface
export { myToolPlugin as default } from './plugin';
export { MyToolPage } from './components/MyToolPage';
export { myToolApiRef } from './api/MyToolApi';
export type { MyToolApi } from './api/MyToolApi';
```

### Correct — module federation (microfrontend, explicit decision only)

```json
// package.json — only when the plugin is a microfrontend
{
  "backstage": {
    "role": "frontend-plugin",
    "pluginId": "my-tool",
    "pluginPackages": ["@scope/backstage-plugin-my-tool"],
    "externalModuleFederation": true   // ← only present in microfrontend plugins
  },
  "scripts": {
    "build": "backstage-cli package build --module-federation"  // ← explicit decision
  }
}
```

### Incorrect

```json
// ❌ Backstage peer deps listed as regular dependencies
{
  "dependencies": {
    "@backstage/core-plugin-api": "^1.0.0"  // move to peerDependencies
  }
}

// ❌ missing backstage role field
{
  "name": "@scope/backstage-plugin-my-tool"
  // no "backstage": { "role": "frontend-plugin" }
}
```

```typescript
// ❌ plugin ID doesn't match package name suffix
export const myToolPlugin = createFrontendPlugin({
  pluginId: 'mytool',     // should be 'my-tool' to match package name
  extensions: [],
});

// ❌ barrel export hiding what's actually public
export * from './components';   // be explicit — list each export
export * from './api';
```

## Verify signal

Reject if:
- `package.json` `name` field does not match `*backstage-plugin-*` pattern
- `package.json` lists `react`, `react-dom`, or `react-router-dom` under `dependencies` instead of `peerDependencies`
- `package.json` is missing the `"backstage"` field or `"backstage.role"` is not `"frontend-plugin"`
- `package.json` is missing `"publishConfig"` with `main` and `types` pointing to `dist/`
- `package.json` is missing `"files": ["dist"]`
- `package.json` is missing `"sideEffects": false`
- `package.json` `build` script includes `--module-federation` without a corresponding decision documented in `plan.md`
- `pluginId` in `package.json` `backstage` field does not match the ID in `createFrontendPlugin` or `createPlugin`
- `src/index.ts` does not exist

Flag for human review if:
- `src/index.ts` uses `export * from` more than once — may be exposing too much internal surface
- `backstage.pluginPackages` array has more than one entry — verify all packages are intentional
- The `dev/` folder is missing — plugin cannot be developed or demoed in isolation
- `devDependencies` pins `react` to a single version instead of a range — reduces compatibility