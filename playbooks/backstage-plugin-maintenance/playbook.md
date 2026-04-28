# Playbook — Backstage Plugin Maintenance

## Overview

This playbook covers the ongoing maintenance of publishable Backstage
plugins — packages distributed as npm packages that need to be kept
compatible, well-documented, and trustworthy for their consumers over time.

It targets plugin maintainers who need to version releases correctly,
communicate breaking changes clearly, manage Backstage compatibility as
the platform evolves, deprecate APIs gracefully, and follow a repeatable
release process.

This playbook applies to all three publishable plugin types:
`backstage-plugin-frontend`, `backstage-plugin-backend-rest`, and
`backstage-plugin-backend-module`. Maintenance concerns are the same
regardless of plugin type — the public contract surfaces differ, but
the discipline is identical.

## Stack version

| Tool | Version |
|---|---|
| Backstage | detected from `package.json` peer dep range |
| `@backstage/cli` | for `versions:check` |
| npm | for publish and pack dry-run |
| Keep a Changelog | 1.1.0 format |
| Semantic Versioning | 2.0.0 |

## Guardrails in this playbook

| # | Guardrail | What it protects |
|---|---|---|
| [001](./001-version-detection.md) | Version detection | Agent reads `package.json`, `CHANGELOG.md`, and npm registry before any task |
| [002](./002-semantic-versioning.md) | Semantic versioning | Full public contract evaluation — not just TypeScript exports |
| [003](./003-changelog.md) | Changelog | Keep a Changelog format, operator-first, migration steps required |
| [004](./004-backstage-compatibility.md) | Backstage compatibility | Peer dep ranges, tested bounds, README compatibility table |
| [005](./005-deprecation.md) | Deprecation | `@deprecated` JSDoc, migration window, runtime warnings |
| [006](./006-release-process.md) | Release process | Pre-release checklist, git tag, publish, post-publish verification |

## How to use this playbook

### Preparing a new release

Read guardrails 001 → 002 → 003 → 006 in order. Guardrail 001 gives
the agent the context it needs. Guardrail 002 determines the version
bump. Guardrail 003 generates the changelog entry. Guardrail 006 executes
the release.

### Updating Backstage compatibility

Read guardrails 001 → 004 in order. Guardrail 004 determines whether
the peer dep range needs widening and whether the README compatibility
table needs updating.

### Deprecating an API

Read guardrails 001 → 005 in order. Guardrail 005 generates the
`@deprecated` JSDoc, the changelog entry, and the runtime warning if
needed. Always ship deprecations as a minor bump — combine with guardrail 002.

### Auditing maintenance health

Run `/giveme:check` — it will evaluate the plugin against all six
guardrails and report issues grouped by guardrail.

## The full public contract of a Backstage plugin

The semver bump decision must consider all of these surfaces — not just
TypeScript exports:

| Surface | Who is affected | Breaking change example |
|---|---|---|
| TypeScript exports | Plugin consumers, module developers | Removing an exported type |
| Plugin ID | App operators | Renaming the plugin ID in `createFrontendPlugin` |
| Action IDs | Template authors | Renaming a `createTemplateAction` ID |
| Entity provider name | Catalog database | Changing `getProviderName()` return value |
| ExtensionPoint interface | Module developers | Adding a required method |
| Database schema | Operators | Migration requiring manual SQL |
| App-config keys | App operators | Renaming a config key |
| Peer dependency range | All consumers | Narrowing the supported Backstage version range |

Guardrail 002 covers every row of this table.

## The deprecation timeline

```
Minor N   → Deprecation announced
              @deprecated JSDoc added
              CHANGELOG ### Deprecated entry
              Runtime warning added (for non-TS surfaces)
              New replacement API shipped alongside deprecated one

Minor N+1 → Deprecation period (both old and new work)
              Consumers migrate at their own pace

Major X   → Breaking removal
              Deprecated API removed
              Migration guide in README
              ### ⚠️ Breaking Changes in CHANGELOG with migration steps
```

Guardrail 005 enforces the left column.
Guardrails 002, 003, and 006 enforce the right column.

## The release sequence

```
1. Pre-release checklist (guardrail 006)
2. Determine version bump (guardrail 002)
3. Write changelog entry (guardrail 003)
4. Verify Backstage compatibility (guardrail 004)
5. Open release PR (version bump + changelog together)
6. PR approved and merged
7. Git tag vX.Y.Z on merge commit
8. npm publish --dry-run
9. npm publish --access public
10. Verify on npm registry within 10 minutes
11. GitHub Release with changelog entry as notes
```

## What's not covered

**CI/CD automation** — automating the release process with GitHub Actions,
release-please, or semantic-release. This playbook covers the manual
process. Automation follows the same discipline but is not prescribed here.

**Monorepo releases** — releasing multiple packages together with
synchronized versions. This playbook assumes single-package releases.

**Plugin development** — creating new features, fixing bugs, adding
extensions. Covered in the stack-specific playbooks:
`backstage-plugin-frontend`, `backstage-plugin-backend-rest`,
`backstage-plugin-backend-module`.

**Security advisories** — CVE disclosure, coordinated vulnerability
disclosure, security patch process. Not covered in this playbook.

## Precedence

These guardrails are community defaults. Your team's guardrails always
take precedence. If your project has a `.specify/playbook/` folder,
those rules override anything defined here.

Guardrail 001 is non-negotiable — no agent should execute a maintenance
task without first reading the current `package.json`, `CHANGELOG.md`,
and the npm registry to understand the current state of the plugin.

## Maintainers

- [@fjbalsamo](https://github.com/fjbalsamo)

## Contributing

Found a guardrail that's wrong, incomplete, or doesn't account for a
specific release scenario? See [CONTRIBUTING.md](../../CONTRIBUTING.md).

When contributing to this playbook, always describe the specific release
scenario the change addresses — maintenance guardrails are context-sensitive
and edge cases matter.