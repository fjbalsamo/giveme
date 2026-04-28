# Playbook — Backstage Frontend Plugin

## Overview

This playbook covers the development and migration of publishable Backstage
frontend plugins — packages distributed as `@scope/backstage-plugin-<id>`
that integrate into any Backstage host app.

It targets two audiences equally: developers building a new plugin from
scratch using the new frontend system, and developers migrating an existing
legacy plugin (`createPlugin`) to the new extension-based architecture
(`createFrontendPlugin`).

These guardrails are version-agnostic by design. Guardrail 001 instructs
every agent to read `backstage.json` and fetch the release documentation
for the exact version in use before making any architectural decision.
The community defaults here represent patterns that have been stable across
multiple Backstage releases — but the release docs always have the final word.

## Stack version

| Dependency | Version |
|---|---|
| Backstage | detected from `backstage.json` at runtime |
| `@backstage/frontend-plugin-api` | aligned with `backstage.json` version |
| `@backstage/core-plugin-api` | legacy only — phase out during migration |
| `@backstage/frontend-test-utils` | for extension-level tests |
| `@backstage/test-utils` | for component-level tests |
| `@backstage/cli` | for build, lint, test scripts |
| TypeScript | 5.x strict |
| React | `^16.13.1 \|\| ^17.0.0 \|\| ^18.0.0` (peer) |

## Guardrails in this playbook

| # | Guardrail | What it protects |
|---|---|---|
| [001](./001-version-detection.md) | Version detection | Agent always reads `backstage.json` and release docs before coding |
| [002](./002-package-structure.md) | Package structure | Naming, `package.json`, exports, build profiles |
| [003](./003-plugin-definition.md) | Plugin definition | Legacy vs new system, `createFrontendPlugin`, hybrid export |
| [004](./004-extensions-blueprints.md) | Extensions and Blueprints | `PageBlueprint`, `NavItemBlueprint`, `ApiBlueprint`, lazy loading |
| [005](./005-api-design.md) | API design | `ApiRef`, interface/impl separation, factory export, mock |
| [006](./006-testing.md) | Testing | `renderInTestApp`, `TestApiProvider`, `createExtensionTester`, `msw` |
| [007](./007-migration-path.md) | Migration path | Legacy detection, incremental migration sequence, completion criteria |

## How to use this playbook

### Building a new plugin

Start with guardrails 001 → 002 → 003 → 004 in order. Then add 005 if
the plugin exposes an API, 006 for testing, and skip 007 entirely — it
only applies to legacy plugins.

### Migrating a legacy plugin

Start with guardrail 001 (version detection), then 007 (migration path)
to assess scope and sequence. Use guardrails 003, 004, and 005 as
reference during each migration step. Use 006 to verify behavior is
preserved after each step.

### Auditing an existing plugin

Run `/giveme:check` — it will evaluate the plugin against all seven
guardrails and report violations grouped by guardrail with file and
line references.

## Build profiles

This playbook covers two build profiles. Guardrail 002 defines when each applies:

| Profile | Build script | When to use |
|---|---|---|
| Standard | `backstage-cli package build` | Default — plugin bundled by host app |
| Module Federation | `backstage-cli package build --module-federation` | Explicit microfrontend architecture decision only |

## What's not covered

**Backend plugins** — `@backstage/backend-plugin-api`, `createBackendPlugin`,
service factories, database access. Covered in a separate playbook:
`backstage-plugin-backend`.

**Scaffolder actions** — custom actions for the Software Templates system.
These have their own plugin role and packaging conventions.

**Permission framework** — defining and enforcing permissions within a plugin.
This requires coordination with the host app's permission policy.

**Plugin maintenance** — versioning strategy, changelog, deprecation notices,
breaking change communication. Covered in a separate playbook:
`backstage-plugin-maintenance`.

**Backstage-internal plugins** — plugins that live inside `packages/` of a
company's Backstage monorepo and are not published to npm. These have
different conventions around naming, building, and distribution.

## Precedence

These guardrails are community defaults. Your team's guardrails always
take precedence. If your project has a `.specify/playbook/` folder,
those rules override anything defined here.

The one exception is guardrail 001 — version detection is non-negotiable.
No agent should generate Backstage plugin code without first reading
`backstage.json` and the release documentation for the detected version.

## Maintainers

- [@fjbalsamo](https://github.com/fjbalsamo)

## Contributing

Found a guardrail that's wrong, incomplete, or generates false positives
for a specific Backstage version? See [CONTRIBUTING.md](../../CONTRIBUTING.md).

When contributing to this playbook, always specify which Backstage version
range the change applies to in the PR description.