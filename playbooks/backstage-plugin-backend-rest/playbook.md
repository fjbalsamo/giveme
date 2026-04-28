# Playbook — Backstage Backend REST Plugin

## Overview

This playbook covers the development of publishable Backstage backend
plugins that expose REST HTTP endpoints — packages distributed as
`@scope/backstage-plugin-<id>-backend` that integrate into any Backstage
backend instance running the new backend system.

It targets developers building new backend plugins from scratch using
`createBackendPlugin` and `coreServices`. It does not cover legacy
`createRouter` + `PluginEnvironment` patterns — those are being phased out.

Like all giveme playbooks, this one is version-agnostic by design.
Guardrail 001 instructs every agent to read `backstage.json` and fetch
the release documentation for the exact version in use before making any
architectural decision. The `coreServices` API surface changes across
Backstage releases — the release docs always have the final word.

## Stack version

| Dependency | Version |
|---|---|
| Backstage | detected from `backstage.json` at runtime |
| `@backstage/backend-plugin-api` | aligned with `backstage.json` version |
| `@backstage/backend-defaults` | aligned with `backstage.json` version |
| `@backstage/backend-test-utils` | aligned with `backstage.json` version |
| `@backstage/cli` | for build, lint, test scripts |
| `express` | ^4.18.0 |
| `express-promise-router` | ^4.1.0 |
| `knex` | via `coreServices.database` — version managed by Backstage |
| TypeScript | 5.x strict |

## Guardrails in this playbook

| # | Guardrail | What it protects |
|---|---|---|
| [001](./001-version-detection.md) | Version detection | Agent reads `backstage.json` + release docs before coding |
| [002](./002-package-structure.md) | Package structure | Naming, `package.json`, `-node` companion, migrations in files |
| [003](./003-plugin-definition.md) | Plugin definition | `createBackendPlugin`, `registerInit`, `deps`, `coreServices` |
| [004](./004-http-router.md) | HTTP router | `createRouter`, `express-promise-router`, auth policies |
| [005](./005-database-access.md) | Database access | `coreServices.database`, knex, migrations, skip flag |
| [006](./006-auth-identity.md) | Auth and identity | `httpAuth`, `auth`, `userInfo`, deprecated services |
| [007](./007-testing.md) | Testing | `supertest`, `mockServices`, `startTestBackend`, `mockCredentials` |

## How to use this playbook

### Building a new plugin without a database

Start with guardrails 001 → 002 → 003 → 004 in order. Add 006 for
auth handling, and 007 for testing. Skip 005 entirely — no database,
no migrations.

### Building a new plugin with a database

Start with guardrails 001 → 002 → 003 → 004 → 005 in order. Then 006
for auth and 007 for testing. Pay close attention to the migration
skip flag pattern in 005 — it will save you in production.

### Auditing an existing plugin

Run `/giveme:check` — it will evaluate the plugin against all seven
guardrails and report violations grouped by guardrail with file and
line references.

## The three database profiles

| Profile | `coreServices.database` in deps | `migrations/` folder | When to use |
|---|---|---|---|
| No database | ❌ | ❌ | Plugin reads from external APIs or other plugins |
| Database, no migrations | ✅ | ❌ | Schema managed externally by the operator |
| Database with migrations | ✅ | ✅ | Plugin owns its schema — most common case |

Guardrail 005 covers all three. The verify agent detects which profile
is in use and applies the appropriate rules.

## The deprecated services checklist

These services from the legacy system must never appear in new plugins.
The verify agent rejects code that uses any of them:

| Deprecated | Replacement |
|---|---|
| `coreServices.tokenManager` | `coreServices.auth` |
| `coreServices.identity` | `coreServices.httpAuth` + `coreServices.userInfo` |
| `PluginEnvironment` from `@backstage/backend-common` | `registerInit` deps |
| `createRouter` from legacy pattern | `createRouter` factory in `src/router.ts` |
| `process.env` for config | `coreServices.rootConfig` |

## Companion packages

When a backend plugin needs to expose `ExtensionPoint`s or `ServiceRef`s
for modules to depend on, those contracts go in a separate
`@scope/backstage-plugin-<id>-node` package. The `-backend` package
never exports extension points. Guardrail 002 covers this separation.

## What's not covered

**Backend modules** — `createBackendModule`, `ExtensionPoint`s, extending
other plugins. Covered in a separate playbook: `backstage-plugin-backend-module`.

**Catalog EntityProviders** — ingesting entities from external systems.
Covered in `backstage-plugin-backend-module` as a specific module pattern.

**Scaffolder actions** — custom actions for the Software Templates system.
These have their own plugin role and registration pattern.

**Permission framework** — defining and enforcing fine-grained permissions
within a backend plugin. Requires coordination with the host app's
permission policy and a dedicated set of guardrails.

**Event system** — publishing and subscribing to Backstage events via
`coreServices.events`. Not covered in this playbook.

**Frontend plugin** — the companion frontend plugin that consumes this
backend's REST API. Covered in `backstage-plugin-frontend`.

**Plugin maintenance** — versioning strategy, changelog, deprecation notices,
breaking change communication. Covered in `backstage-plugin-maintenance`.

## Precedence

These guardrails are community defaults. Your team's guardrails always
take precedence. If your project has a `.specify/playbook/` folder,
those rules override anything defined here.

Guardrail 001 is non-negotiable — no agent should generate Backstage
backend plugin code without first reading `backstage.json` and the
release documentation for the detected version. The `coreServices` API
changes frequently enough that training data alone is unreliable.

## Maintainers

- [@fjbalsamo](https://github.com/fjbalsamo)

## Contributing

Found a guardrail that's wrong, incomplete, or generates false positives
for a specific Backstage version? See [CONTRIBUTING.md](../../CONTRIBUTING.md).

When contributing to this playbook, always specify which Backstage version
range the change applies to in the PR description. The backend system
evolves faster than the frontend — version context is critical.