# Guardrail 001 — Version detection and documentation lookup

<!--
  Source: giveme community playbook (backstage-plugin-backend-rest/001-version-detection.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

The Backstage backend system has evolved significantly across releases.
`coreServices` available in one version may be deprecated or replaced in
the next. The database migration API, the auth service signatures, and
the HTTP router contract have all changed across minor versions.

A backend plugin that hardcodes assumptions about service APIs without
checking the release documentation will generate code that compiles today
and breaks on the next `yarn backstage-cli versions:bump`.

Every Backstage project declares its version in `backstage.json`. Every
release has a dedicated documentation page at a predictable URL. These
two facts are the foundation of this entire playbook.

## Decision

Before generating any code for a Backstage backend plugin, the agent reads
`backstage.json` to detect the version, constructs the documentation URL
for that release, and consults it to verify that the `coreServices` and
APIs it intends to use are current and not deprecated. No assumption about
Backstage backend APIs is made from training data alone.

The backend system is separate from the frontend system — changes to one
do not imply changes to the other. This guardrail applies exclusively to
backend plugin development.

## Rules

- MUST: Read `backstage.json` before any code generation or architectural decision
- MUST: Extract `version` field from `backstage.json` in `x.y.z` format
- MUST: Construct the release documentation URL as `https://backstage.io/docs/releases/v${x.y.0}` — always use `0` as the patch segment
- MUST: Fetch and read the release documentation before selecting any `coreServices` dependencies
- MUST: Check the backend-specific changelog at `https://backstage.io/docs/backend-system/` for breaking changes in the detected version
- MUST: If `backstage.json` is missing, ask the developer for the Backstage version before proceeding
- MUST: Document the detected version in `spec.md` under a `## Backstage version` section
- MUST NOT: Rely solely on training data to determine whether a `coreServices` API is current
- MUST NOT: Generate code using backend services marked as deprecated in the release documentation
- MUST NOT: Assume that a `coreServices` reference available in one version exists in the next
- SHOULD: Verify that `@backstage/backend-plugin-api` version in `package.json` aligns with `backstage.json`
- SHOULD NOT: Use `@backstage/backend-common` APIs without checking if they have been superseded by `coreServices` equivalents — this package is being progressively deprecated

## Examples

### Correct

```typescript
// Before writing any plugin code, the agent:
// 1. reads backstage.json
// {
//   "version": "1.48.0"
// }
//
// 2. constructs the doc URL
// https://backstage.io/docs/releases/v1.48.0
//
// 3. fetches and reads the release notes for backend changes
// 4. checks coreServices availability for this version
// 5. checks @backstage/backend-common deprecation status
// 6. proceeds with current APIs only
```

```markdown
<!-- in spec.md — version always documented -->
## Backstage version
Detected: 1.48.0
Release docs: https://backstage.io/docs/releases/v1.48.0
Backend system: new (createBackendPlugin + coreServices)
backend-common status: deprecated APIs checked — none used
```

### Incorrect

```typescript
// ❌ using backend-common without checking deprecation status
import { PluginEndpointDiscovery } from '@backstage/backend-common';
// agent didn't check if this is still recommended for this version
// coreServices.discovery may be the current equivalent

// ❌ assuming coreServices shape from training data
import { coreServices } from '@backstage/backend-plugin-api';
deps: {
  tokenManager: coreServices.tokenManager,  // deprecated in favor of coreServices.auth
}
// agent should have checked the release docs first
```

## Verify signal

Reject if:
- `spec.md` does not contain a `## Backstage version` section
- `spec.md` `## Backstage version` section does not include the release docs URL
- Code uses APIs from `@backstage/backend-common` that are marked deprecated in the detected version's release notes
- `backstage.json` is absent and the agent proceeded without asking for the version
- `coreServices.tokenManager` is used — deprecated in favor of `coreServices.auth` since v1.24

Flag for human review if:
- The detected version is more than 4 months behind the latest release — backend plugins accumulate breaking change debt faster than frontend
- The `@backstage/backend-plugin-api` version in `package.json` does not align with `backstage.json` — version drift causes subtle runtime errors
- The agent found deprecation notices in `@backstage/backend-common` but proceeded anyway — review the justification