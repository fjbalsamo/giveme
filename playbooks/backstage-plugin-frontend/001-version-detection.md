# Guardrail 001 — Version detection and documentation lookup

<!--
  Source: giveme community playbook (backstage-plugin-frontend/001-version-detection.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Backstage evolves fast. APIs that are stable in one release are deprecated
in the next. A guardrail that hardcodes decisions for version 1.30 will
produce wrong code for version 1.48. The only way to write a version-agnostic
playbook is to make the agent read the actual release documentation before
making any architectural decision.

Every Backstage project declares its version in `backstage.json`. Every
release has a dedicated documentation page at a predictable URL. These
two facts are the foundation of this entire playbook.

## Decision

Before generating any code for a Backstage plugin, the agent reads
`backstage.json` to detect the version, constructs the documentation URL
for that release, and consults it to verify that the APIs it intends to
use are current and not deprecated. No assumption about Backstage APIs
is made from training data alone.

## Rules

- MUST: Read `backstage.json` before any code generation or architectural decision
- MUST: Extract `version` field from `backstage.json` in `x.y.z` format
- MUST: Construct the release documentation URL as `https://backstage.io/docs/releases/v${x.y.0}` — always use `0` as the patch segment
- MUST: Fetch the release documentation and check for deprecation notices affecting the APIs in scope
- MUST: If `backstage.json` is missing, ask the developer for the Backstage version before proceeding
- MUST: Document the detected version in `spec.md` under a `## Backstage version` section
- MUST NOT: Rely solely on training data to determine whether a Backstage API is current
- MUST NOT: Generate code using APIs marked as deprecated in the release documentation
- SHOULD: Check the Frontend System Changelog at `https://backstage.io/docs/frontend-system/architecture/migrations/` for breaking changes between the detected version and the previous major
- SHOULD NOT: Assume that an API available in one version is available in the next — always verify

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
// 3. fetches and reads the release notes
// 4. checks for deprecated APIs relevant to frontend plugins
// 5. proceeds with current APIs only
```

```json
// backstage.json — the single source of version truth
{
  "version": "1.48.0"
}
```

```markdown
<!-- in spec.md — version always documented -->
## Backstage version
Detected: 1.48.0
Release docs: https://backstage.io/docs/releases/v1.48.0
Frontend system: new (createFrontendPlugin)
Legacy compat: not required
```

### Incorrect

```typescript
// ❌ assuming API availability from training data
import { createPlugin } from '@backstage/core-plugin-api';
// agent didn't check if createPlugin is still recommended for this version

// ❌ hardcoding a version assumption in the playbook
// "use createFrontendPlugin introduced in v1.30"
// — this breaks when Backstage changes the API again
```

## Verify signal

Reject if:
- `spec.md` does not contain a `## Backstage version` section
- `spec.md` `## Backstage version` section does not include the release docs URL
- Code is generated using an API explicitly marked as deprecated in the release documentation for the detected version
- `backstage.json` is absent and the agent proceeded without asking for the version

Flag for human review if:
- The detected version is more than 6 months behind the latest release — the plugin may be accumulating migration debt
- The release docs URL returns a 404 — the version format may have changed, verify manually
- The agent found deprecation notices but proceeded anyway with a justification — review the justification