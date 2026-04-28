# Guardrail 002 — Semantic versioning

<!--
  Source: giveme community playbook (backstage-plugin-maintenance/002-semantic-versioning.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Semantic versioning in Backstage plugins has nuances that don't exist in
generic npm packages. The plugin's public surface is not just TypeScript
exports — it includes the plugin ID (referenced in app-config.yaml), the
action ID (referenced in Software Templates), the entity provider name
(stored in the catalog database), the ExtensionPoint API (consumed by
modules), and the database schema (migrated in operator environments).

A change that looks like a refactor in the source code can be a breaking
change for operators who have the plugin ID in dozens of config files, or
for template authors who reference an action ID in hundreds of templates.

The most dangerous mistake is shipping a breaking change as a patch or
minor version because "the TypeScript API didn't change." TypeScript is
only one dimension of a plugin's public contract.

## Decision

Version bumps follow semver strictly, applied to the full public contract
of the plugin — not just its TypeScript exports. The full public contract
includes: exported TypeScript types and functions, plugin ID, action IDs,
entity provider names, ExtensionPoint interfaces, database schema, and
app-config.yaml keys. Any change to any of these dimensions is evaluated
against the breaking change rules before choosing a version bump.

## Rules

- MUST: Bump major version for any breaking change to the public contract
- MUST: Bump minor version for backwards-compatible new functionality
- MUST: Bump patch version for backwards-compatible bug fixes only
- MUST: Treat plugin ID changes as major — operators have the ID in app-config.yaml
- MUST: Treat action ID changes as major — template authors reference IDs in template YAML
- MUST: Treat entity provider name changes as major — the catalog stores the name as a location key
- MUST: Treat ExtensionPoint interface changes that add required methods as major
- MUST: Treat database schema changes that require manual operator intervention as major
- MUST: Treat removal of any exported TypeScript type, function, or constant as major
- MUST: Treat narrowing of a peer dependency range as major — consumers on the excluded version break
- MUST NOT: Ship a breaking change as minor or patch regardless of how small it appears
- MUST NOT: Bump major version for adding optional exports, optional config keys, or optional ExtensionPoint methods
- SHOULD: Treat widening of a peer dependency range as minor — new consumers can adopt, existing ones are unaffected
- SHOULD: Add `@deprecated` JSDoc before removing an export — give consumers at least one minor version to migrate
- SHOULD NOT: Bump major for changes that only affect `devDependencies` or internal implementation details

## Examples

### Breaking changes — always major

```typescript
// ❌ → major: removing an exported type
// before
export type { MyToolApi } from './api/MyToolApi';
// after — removed entirely

// ❌ → major: changing plugin ID
// before
createFrontendPlugin({ pluginId: 'my-tool' })
// after
createFrontendPlugin({ pluginId: 'mytool' })  // operators must update app-config.yaml

// ❌ → major: renaming action ID
// before
createTemplateAction({ id: 'myorg:create-repository' })
// after
createTemplateAction({ id: 'myorg:repo-create' })  // all templates using old ID break

// ❌ → major: adding required method to ExtensionPoint interface
// before
export interface MyToolExtensionPoint {
  addProvider(provider: Provider): void;
}
// after
export interface MyToolExtensionPoint {
  addProvider(provider: Provider): void;
  addValidator(validator: Validator): void;  // required — all existing modules break
}

// ❌ → major: narrowing peer dependency range
// before
"peerDependencies": { "@backstage/core-plugin-api": "^1.0.0" }
// after
"peerDependencies": { "@backstage/core-plugin-api": "^1.10.0" }  // consumers on 1.0-1.9 break
```

### Non-breaking changes — minor or patch

```typescript
// ✅ → minor: new optional export
export { MyToolDebugPanel } from './components/MyToolDebugPanel';  // additive

// ✅ → minor: new optional config key
// app-config.yaml: myTool.debug: true  // optional, has default

// ✅ → minor: adding optional method to ExtensionPoint
export interface MyToolExtensionPoint {
  addProvider(provider: Provider): void;
  addValidator?(validator: Validator): void;  // optional — existing modules unaffected
}

// ✅ → minor: widening peer dependency range
// before
"peerDependencies": { "react": "^17.0.0" }
// after
"peerDependencies": { "react": "^17.0.0 || ^18.0.0" }  // additive

// ✅ → patch: bug fix that doesn't change public contract
// fixing incorrect entity kind in provider output
// fixing wrong status code in REST endpoint
// fixing memory leak in provider scheduling
```

### Decision table

| Change type | Bump |
|---|---|
| Remove exported type, function, or constant | **major** |
| Change plugin ID, action ID, or provider name | **major** |
| Add required method to ExtensionPoint | **major** |
| Narrow peer dependency range | **major** |
| Database migration requiring operator action | **major** |
| Add new optional export | minor |
| Add new optional config key | minor |
| Add optional method to ExtensionPoint | minor |
| Widen peer dependency range | minor |
| Bug fix with no contract change | patch |
| Internal refactor with no contract change | patch |
| Dependency update with no contract change | patch |

## Verify signal

Reject if:
- A plugin ID, action ID, or entity provider name changes without a major version bump
- An exported TypeScript type or function is removed without a major version bump
- An ExtensionPoint interface gains a required method without a major version bump
- A peer dependency range is narrowed without a major version bump
- A `package.json` version bump is proposed without checking the full public contract — not just TypeScript exports

Flag for human review if:
- A major version bump is proposed for a change that only affects internal implementation — may be unnecessary churn
- An export is removed without a prior `@deprecated` JSDoc annotation in a previous version — consumers had no warning
- The version is still `0.x.y` — semver major bump rules apply from `1.0.0` onwards, `0.x` is considered unstable
- More than 3 breaking changes are bundled in a single major release — consider spreading them across versions