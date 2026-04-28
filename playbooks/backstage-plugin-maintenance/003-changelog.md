# Guardrail 003 — Changelog

<!--
  Source: giveme community playbook (backstage-plugin-maintenance/003-changelog.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

A changelog is the primary communication channel between a plugin
maintainer and its consumers. Operators decide whether to upgrade based
on the changelog. Template authors know whether their templates will break.
Module developers know whether their ExtensionPoint implementations need
updating.

A changelog that says "bug fixes and improvements" communicates nothing.
A changelog that lists every internal refactor buries the information
operators actually need. The right changelog is operator-first — it
answers "what do I need to do when I upgrade?" before answering
"what changed internally?"

Backstage plugins have a specific audience that generic npm changelogs
ignore: operators who manage app-config.yaml, template authors who
maintain Software Templates, and module developers who implement
ExtensionPoints. Each group needs different information.

## Decision

The changelog follows Keep a Changelog format with Backstage-specific
sections for operator impact and migration steps. Every release entry
documents the version, date, and categorized changes. Breaking changes
have their own section at the top with explicit migration instructions.
Internal changes that don't affect the public contract are grouped
under a collapsible section.

## Rules

- MUST: Follow [Keep a Changelog](https://keepachangelog.com) format — `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`
- MUST: Every release entry includes version number and release date — `## [1.5.0] — 2026-04-28`
- MUST: Breaking changes have a dedicated `### ⚠️ Breaking Changes` section at the top of the release entry
- MUST: Every breaking change entry includes explicit migration steps — not just what changed, but what the consumer must do
- MUST: Document config key changes with before/after examples in app-config.yaml format
- MUST: Document action ID changes with before/after template YAML examples
- MUST: Document ExtensionPoint interface changes with before/after TypeScript examples
- MUST: Keep an `## [Unreleased]` section at the top — new entries go here before release
- MUST NOT: Write "bug fixes and improvements" without specifying what was fixed and what improved
- MUST NOT: Include internal implementation details that don't affect the public contract in the main sections
- SHOULD: Add a `### 🎯 Operator impact` summary at the top of each release — one sentence per audience group
- SHOULD: Link each version header to the GitHub diff — `[1.5.0]: https://github.com/org/repo/compare/v1.4.2...v1.5.0`
- SHOULD: Internal-only changes go under `### Internal` at the bottom — keeping them separate from operator-relevant changes
- SHOULD NOT: Document changes that only affect `devDependencies` or test files in operator-facing sections

## Examples

### Correct

```markdown
# Changelog

All notable changes to `@scope/backstage-plugin-my-tool` are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Support for filtering entity cards by `spec.lifecycle` field

---

## [2.0.0] — 2026-04-28

### 🎯 Operator impact
- **App operators**: Update `app-config.yaml` — `myTool.apiUrl` renamed to `myTool.baseUrl`
- **Template authors**: Update action ID in all templates — `myorg:create-repo` → `myorg:create-repository`
- **Module developers**: `MyToolExtensionPoint.addProvider()` now requires a second `options` argument

### ⚠️ Breaking Changes

#### Config key renamed: `myTool.apiUrl` → `myTool.baseUrl`

Before:
```yaml
myTool:
  apiUrl: https://api.example.com
```

After:
```yaml
myTool:
  baseUrl: https://api.example.com
```

#### Action ID changed: `myorg:create-repo` → `myorg:create-repository`

Update all Software Templates that reference this action:

Before:
```yaml
steps:
  - id: create-repo
    action: myorg:create-repo
```

After:
```yaml
steps:
  - id: create-repo
    action: myorg:create-repository
```

#### ExtensionPoint: `addProvider()` now requires options

Before:
```typescript
myToolExtensionPoint.addProvider(provider);
```

After:
```typescript
myToolExtensionPoint.addProvider(provider, { priority: 'normal' });
```

### Added
- New `myorg:create-repository` action replaces the deprecated `myorg:create-repo`
- `MyToolExtensionPoint.addProvider()` accepts optional `priority` in options

### Removed
- `myorg:create-repo` action — use `myorg:create-repository` instead
- `myTool.apiUrl` config key — use `myTool.baseUrl` instead

### Internal
- Migrated from `@backstage/backend-common` to `coreServices`
- Updated test setup to use `mockServices` from `@backstage/backend-test-utils`

---

## [1.4.2] — 2026-03-15

### Fixed
- Entity card no longer shows blank state when `spec.owner` is undefined (#42)
- Corrected HTTP 500 response when catalog returns empty entity list (#38)

---

## [1.4.1] — 2026-02-28

### Fixed
- Corrected entity filter on `EntityCard` — was matching all kinds instead of `Component` only

### Security
- Updated `express` to 4.19.2 to address CVE-2024-29041

[Unreleased]: https://github.com/scope/my-tool/compare/v2.0.0...HEAD
[2.0.0]: https://github.com/scope/my-tool/compare/v1.4.2...v2.0.0
[1.4.2]: https://github.com/scope/my-tool/compare/v1.4.1...v1.4.2
[1.4.1]: https://github.com/scope/my-tool/compare/v1.4.0...v1.4.1
```

### Incorrect

```markdown
## [2.0.0] — 2026-04-28

### Changed
- Renamed config key                    ← no before/after, no migration steps
- Updated action ID                     ← template authors don't know what to change
- Various internal improvements         ← meaningless to operators
- Bumped dependencies                   ← not operator-relevant
```

## Verify signal

Reject if:
- A breaking change entry has no explicit migration steps — "what changed" is not enough, "what to do" is required
- A config key change has no before/after app-config.yaml example
- An action ID change has no before/after template YAML example
- A release entry is missing the date — `## [1.5.0]` without `— YYYY-MM-DD`
- `CHANGELOG.md` has no `## [Unreleased]` section — new entries have nowhere to go

Flag for human review if:
- A breaking change is documented without a corresponding major version bump in `package.json`
- The `## [Unreleased]` section is empty but `package.json` version was bumped — changelog may be missing entries
- A release entry has no `### ⚠️ Breaking Changes` section but the version is a major bump — verify nothing was missed
- More than 6 months have passed since the last release entry — plugin may be unmaintained