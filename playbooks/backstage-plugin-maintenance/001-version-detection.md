# Guardrail 001 — Version detection and compatibility context

<!--
  Source: giveme community playbook (backstage-plugin-maintenance/001-version-detection.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Maintenance tasks — bumping versions, writing changelogs, deprecating APIs,
updating peer dependency ranges — require knowing two things simultaneously:
the version of the plugin being maintained and the Backstage version range
it targets.

A maintainer who bumps a major version without checking whether the change
is actually breaking for the declared peer dependency range ships
unnecessary churn. A maintainer who widens a peer dependency range without
checking what changed in Backstage between the old and new bounds ships
a compatibility promise they cannot keep.

Unlike development playbooks where `backstage.json` tells the agent the
host app's version, maintenance tasks require reading the plugin's own
`package.json` to understand what it declares and what it owes to its
consumers.

## Decision

Before executing any maintenance task, the agent reads the plugin's own
`package.json` to detect its current version, its declared peer dependency
range for Backstage, and its published `backstage.role`. It also reads
the changelog to understand the recent history. The agent fetches the
Backstage release documentation for the bounds of the declared peer
dependency range to understand what changed between them.

## Rules

- MUST: Read `package.json` before any maintenance task — detect current version, role, and peer dependency ranges
- MUST: Read `CHANGELOG.md` before any maintenance task — understand recent history and last breaking change
- MUST: Identify the lower and upper bounds of the declared Backstage peer dependency range
- MUST: Fetch release docs for the lower bound — `https://backstage.io/docs/releases/v${lower.y.0}`
- MUST: Fetch release docs for the upper bound — `https://backstage.io/docs/releases/v${upper.y.0}`
- MUST: Document the detected version context in `spec.md` under `## Plugin version context`
- MUST NOT: Execute any versioning or changelog task without reading the current `package.json` and `CHANGELOG.md`
- MUST NOT: Widen a peer dependency range without reading what changed in Backstage between the bounds
- SHOULD: Check npm registry for the latest published version — `https://registry.npmjs.org/<package-name>` — to detect unpublished changes
- SHOULD NOT: Assume the latest git tag matches the latest published npm version — verify both

## Examples

### Correct

```bash
# Before any maintenance task, the agent reads:
# 1. package.json → name, version, peerDependencies, backstage.role
# 2. CHANGELOG.md → last 3-5 entries for context
# 3. npm registry → latest published version
# 4. Backstage release docs for peer dep bounds
```

```markdown
<!-- in spec.md -->
## Plugin version context
Package: @scope/backstage-plugin-my-tool
Current version (package.json): 1.4.2
Latest published (npm): 1.4.1 — 1.4.2 not yet published
Role: frontend-plugin
Backstage peer dep range: ^1.30.0
Lower bound docs: https://backstage.io/docs/releases/v1.30.0
Upper bound docs: https://backstage.io/docs/releases/v1.48.0 (latest)
Last changelog entry: 1.4.1 — fix: correct entity filter on EntityCard
Last breaking change: 1.0.0 — breaking: removed legacy createPlugin export
```

### Incorrect

```markdown
<!-- ❌ maintenance task started without reading changelog -->
## Plugin version context
Package: @scope/backstage-plugin-my-tool
Current version: 1.4.2
<!-- missing: changelog context, npm registry check, Backstage release docs -->
```

## Verify signal

Reject if:
- `spec.md` does not contain a `## Plugin version context` section before any maintenance task
- The plugin's `CHANGELOG.md` was not read before generating a changelog entry or version bump
- A peer dependency range is widened without fetching release docs for both bounds

Flag for human review if:
- The current `package.json` version is ahead of the latest npm published version — there may be unreleased changes
- The last changelog entry is more than 3 months old — the plugin may be unmaintained
- The Backstage peer dependency range spans more than 6 minor versions — compatibility guarantees become hard to verify