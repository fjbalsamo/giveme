# Playbook — NestJS

## Overview

This playbook covers backend microservices built with NestJS — the most
opinionated Node.js framework for building scalable server-side applications.
It targets teams that want structured, maintainable, and testable code without
debating conventions on every PR.

These guardrails represent decisions that have been validated in production.
They are not theoretical best practices — they are answers to real problems
that happen when you don't have them.

## Stack version

| Dependency | Version |
|---|---|
| Node.js | 22+ |
| TypeScript | 5.x with `strict: true` |
| NestJS | 11.x |
| Prisma | 6.x |
| Jest | default NestJS |
| Zod | 3.x via `nestjs-zod` |

## Guardrails in this playbook

| # | Guardrail | What it protects |
|---|---|---|
| [001](./001-module-structure.md) | Module structure and layer separation | Boundaries between domains and layers |
| [002](./002-validation-contracts.md) | Validation and contracts | HTTP input/output shape and validation |
| [003](./003-error-handling.md) | Error handling and logging | Consistent errors, traceable logs |
| [004](./004-testing-strategy.md) | Testing strategy | Coverage, isolation, and test-first discipline |
| [005](./005-data-patterns.md) | Data patterns | IDs, timestamps, migrations, optimistic locking |
| [006](./006-security.md) | Security | Secrets, CORS, rate limiting, audit trail |
| [007](./007-observability.md) | Observability | Health checks, tracing, metrics |
| [008](./008-api-design.md) | API design | Versioning, pagination, OpenAPI |

## Precedence

These guardrails are community defaults. Your team's guardrails always
take precedence. If your project has a `.specify/playbook/` folder,
those rules override anything defined here.

## What's not covered

**Async messaging** — events, queues, message brokers. If your service
publishes or consumes events, you need additional guardrails for that
contract. This playbook assumes synchronous HTTP only.

**Authentication** — this playbook covers authorization patterns (guards,
JWT validation structure) but does not prescribe an identity provider or
token issuance strategy. That decision belongs to your team.

**Multi-tenancy** — row-level security, tenant isolation, and scoped
queries are not covered. These require decisions that are too
context-specific to generalize.

**Caching** — Redis, in-memory, HTTP cache headers. Not covered here.

**Deployment and infrastructure** — Dockerfile, Helm, Kubernetes config.
These belong to your platform team's playbook, not the application layer.

## Maintainers

- [@fjbalsamo](https://github.com/fjbalsamo)

## Contributing

Found a guardrail that's too strict, too loose, or missing context?
Found a verify signal that produces false positives?
See [CONTRIBUTING.md](../../CONTRIBUTING.md) to propose changes.