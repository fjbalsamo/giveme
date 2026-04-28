# Playbook — Backstage Backend Module

## Overview

This playbook covers the development of publishable Backstage backend
modules — packages distributed as
`@scope/backstage-plugin-<target-id>-backend-module-<feature-id>` that
extend existing Backstage plugins without modifying them.

It covers three module types equally: generic ExtensionPoint modules,
EntityProvider modules for catalog ingestion, and ScaffolderAction modules
for Software Templates. All three assume the target plugin already exists
and is published — this playbook is about extending plugins, not creating them.

Like all giveme playbooks, this one is version-agnostic by design.
Guardrail 001 instructs every agent to read `backstage.json`, verify the
target plugin version compatibility, and fetch the release documentation
before making any architectural decision. ExtensionPoint APIs change across
releases — the release docs and the `-node` package types always have the
final word.

## Stack version

| Dependency | Version |
|---|---|
| Backstage | detected from `backstage.json` at runtime |
| `@backstage/backend-plugin-api` | aligned with `backstage.json` version |
| `@backstage/plugin-catalog-node` | for EntityProvider modules |
| `@backstage/plugin-scaffolder-node` | for ScaffolderAction modules |
| `@backstage/backend-test-utils` | for integration tests |
| `@backstage/cli` | for build, lint, test scripts |
| TypeScript | 5.x strict |

## Guardrails in this playbook

| # | Guardrail | What it protects |
|---|---|---|
| [001](./001-version-detection.md) | Version detection | Agent reads `backstage.json` + target plugin version before coding |
| [002](./002-package-structure.md) | Package structure | Naming convention, `-node` dependency, `backend-plugin-module` role |
| [003](./003-module-definition.md) | Module definition | `createBackendModule`, `pluginId`, `moduleId`, `registerInit` |
| [004](./004-extension-point-modules.md) | ExtensionPoint modules | ExtensionPoint boundary, `-node` only, `implements` contract |
| [005](./005-entity-provider-modules.md) | EntityProvider modules | `EntityProvider`, scheduling, `applyMutation`, provider name stability |
| [006](./006-scaffolder-action-modules.md) | ScaffolderAction modules | `createTemplateAction`, Zod schemas, action ID, secret safety |
| [007](./007-testing.md) | Testing | `createMockActionContext`, mock connection, `startTestBackend`, `msw` |

## How to use this playbook

### Building a generic ExtensionPoint module

Read guardrails 001 → 002 → 003 → 004 in order. Add 007 for testing.
Skip 005 and 006 — they cover specific module types.

### Building an EntityProvider module

Read guardrails 001 → 002 → 003 → 005 in order. Guardrail 004 provides
useful background on the ExtensionPoint pattern but 005 is the authoritative
source for EntityProviders. Add 007 for testing.

### Building a ScaffolderAction module

Read guardrails 001 → 002 → 003 → 006 in order. Add 007 for testing.
Skip 004 and 005 — they cover other module types.

### Auditing an existing module

Run `/giveme:check` — it will evaluate the module against all seven
guardrails and report violations grouped by guardrail with file and
line references.

## The three module types

| Type | Target plugin | ExtensionPoint | Guardrail |
|---|---|---|---|
| Generic | any plugin with an ExtensionPoint | plugin's `-node` package | 004 |
| EntityProvider | `catalog` | `catalogProcessingExtensionPoint` | 005 |
| ScaffolderAction | `scaffolder` | `scaffolderActionsExtensionPoint` | 006 |

## Naming convention

The module naming convention encodes everything an operator needs to know:

```
@scope/backstage-plugin-<target-plugin-id>-backend-module-<feature-id>
         ↑                ↑                              ↑
         org scope        which plugin it extends        what it adds
```

Examples:
- `@myorg/backstage-plugin-catalog-backend-module-github-org-discovery`
- `@myorg/backstage-plugin-scaffolder-backend-module-http-request`
- `@myorg/backstage-plugin-my-tool-backend-module-slack-notifications`

`backstage.pluginId` in `package.json` is always the **target plugin ID**,
not the module's own identifier. `backstage.moduleId` is the feature ID.
Guardrail 002 verifies both.

## The -node dependency rule

Modules depend on the target plugin's `-node` package — never on `-backend`.

| Package | Contains | Use in modules |
|---|---|---|
| `plugin-catalog-backend` | Full plugin implementation | ❌ Never |
| `plugin-catalog-node` | ExtensionPoint refs, interfaces, types | ✅ Always |

Depending on `-backend` creates circular dependency risk, inflates the
module's transitive dependency tree, and couples the module to the plugin's
internal implementation. Guardrail 002 rejects it.

## What's not covered

**Creating ExtensionPoints** — designing and publishing a new ExtensionPoint
in a plugin for others to consume. This requires changes to the target
plugin, not a module. Covered in `backstage-plugin-backend-rest`.

**Frontend module** — extending the Backstage frontend via the frontend
extension system. Covered in `backstage-plugin-frontend`.

**Permission framework** — adding permission checks to module-provided
capabilities. Requires coordination with the host app's permission policy.

**Event system** — publishing or subscribing to Backstage events from a
module. Not covered in this playbook.

**Plugin maintenance** — versioning strategy, changelog, deprecation notices,
breaking change communication when the module's own API changes. Covered
in `backstage-plugin-maintenance`.

## Precedence

These guardrails are community defaults. Your team's guardrails always
take precedence. If your project has a `.specify/playbook/` folder,
those rules override anything defined here.

Guardrail 001 is non-negotiable — no agent should generate Backstage
module code without first reading `backstage.json`, verifying the target
plugin version, and consulting the release documentation. ExtensionPoint
APIs evolve across releases and training data alone is unreliable.

## Maintainers

- [@fjbalsamo](https://github.com/fjbalsamo)

## Contributing

Found a guardrail that's wrong, incomplete, or generates false positives
for a specific Backstage version or target plugin version?
See [CONTRIBUTING.md](../../CONTRIBUTING.md).

When contributing to this playbook, always specify:
- Which Backstage version range the change applies to
- Which target plugin version the change applies to
- Whether the change affects all three module types or only specific ones