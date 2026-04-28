# Guardrail 001 — Version detection and documentation lookup

<!--
  Source: giveme community playbook (backstage-plugin-backend-module/001-version-detection.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Backend modules interact with plugin extension points whose APIs evolve
across Backstage releases. The `CatalogExtensionPoint` API for entity
providers, the `ScaffolderExtensionPoint` API for template actions, and
the shape of `ExtensionPoint`s from third-party plugins all change across
minor versions.

A module that hardcodes assumptions about extension point APIs without
checking the release documentation will generate code that compiles today
and silently misbehaves or fails on the next
`yarn backstage-cli versions:bump`.

Unlike backend REST plugins, modules have an additional version concern:
the plugin they extend has its own version that must be compatible with
the Backstage version. Incompatible versions between a module and its
target plugin are one of the hardest runtime errors to diagnose.

## Decision

Before generating any code for a Backstage backend module, the agent reads
`backstage.json` to detect the Backstage version, constructs the
documentation URL for that release, and consults it to verify that the
extension point APIs it intends to use are current. The agent also checks
that the target plugin's version in `package.json` is compatible with
the detected Backstage version.

## Rules

- MUST: Read `backstage.json` before any code generation or architectural decision
- MUST: Extract `version` field from `backstage.json` in `x.y.z` format
- MUST: Construct the release documentation URL as `https://backstage.io/docs/releases/v${x.y.0}` — always use `0` as patch segment
- MUST: Fetch and read the release documentation before selecting any extension point APIs
- MUST: Verify the target plugin's package version in `package.json` is compatible with the Backstage version
- MUST: If `backstage.json` is missing, ask the developer for the Backstage version before proceeding
- MUST: Document the detected version and target plugin version in `spec.md` under `## Backstage version`
- MUST NOT: Rely solely on training data to determine the current shape of any extension point API
- MUST NOT: Generate module code without knowing which plugin it extends and that plugin's current version
- SHOULD: Check the target plugin's own changelog for breaking changes to its extension points
- SHOULD: Verify `@backstage/backend-plugin-api` version in `package.json` aligns with `backstage.json`
- SHOULD NOT: Assume that an extension point available in one version of a plugin exists in the next

## Examples

### Correct

```typescript
// Before writing any module code, the agent:
// 1. reads backstage.json → version: "1.48.0"
// 2. constructs: https://backstage.io/docs/releases/v1.48.0
// 3. fetches release notes — checks for ExtensionPoint API changes
// 4. reads package.json — finds @backstage/plugin-catalog-backend: "^1.26.0"
// 5. verifies catalog-backend@1.26.x is compatible with Backstage 1.48.0
// 6. proceeds with current CatalogExtensionPoint API
```

```markdown
<!-- in spec.md -->
## Backstage version
Detected: 1.48.0
Release docs: https://backstage.io/docs/releases/v1.48.0
Target plugin: @backstage/plugin-catalog-backend@^1.26.0
Compatibility: verified against release docs
ExtensionPoint API: CatalogExtensionPoint — addEntityProvider() current
```

### Incorrect

```typescript
// ❌ assuming CatalogExtensionPoint shape from training data
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node';

// agent didn't check if catalogProcessingExtensionPoint is the right
// extension point for entity providers in this version —
// it may have been renamed or split across versions

// ❌ no version compatibility check between module and target plugin
{
  "dependencies": {
    "@backstage/plugin-catalog-backend": "^1.0.0",  // potentially incompatible
    "@backstage/backend-plugin-api": "^0.9.0"
  }
}
// agent should have verified compatibility before generating code
```

## Verify signal

Reject if:
- `spec.md` does not contain a `## Backstage version` section
- `spec.md` does not document the target plugin name and version
- The target plugin version in `package.json` is not a caret range — pinned versions cause operator pain
- `backstage.json` is absent and the agent proceeded without asking for the version

Flag for human review if:
- The target plugin version range in `package.json` is broader than one major — e.g. `>=1.0.0` instead of `^1.26.0`
- The detected Backstage version is more than 4 months behind latest — module may depend on deprecated extension point APIs
- The target plugin is a third-party package not published by the Backstage team — its extension point API stability is not guaranteed