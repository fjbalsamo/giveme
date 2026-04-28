# Guardrail 004 — Backstage version compatibility

<!--
  Source: giveme community playbook (backstage-plugin-maintenance/004-backstage-compatibility.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

Backstage releases a new minor version approximately every two weeks.
Each release may deprecate APIs, change service signatures, or move
functionality from one package to another. A plugin that declares
`"@backstage/core-plugin-api": "^1.0.0"` as a peer dependency is making
a compatibility promise for every Backstage version from 1.0 to the next
major — a range that today spans 48+ minor versions.

That promise is almost certainly false. No plugin maintainer has tested
against all 48 versions. The real question is: what is the minimum version
that works, and what is the maximum version the maintainer has verified?

Operators on the receiving end of this promise make real decisions based
on it. A plugin that claims to support Backstage 1.0 but actually requires
1.30 APIs will fail at runtime in ways that are very hard to diagnose.

## Decision

Plugins declare peer dependency ranges that reflect what they have actually
tested, not the widest range that might work. The lower bound is the minimum
version where all used APIs exist. The upper bound is kept current by
the maintainer — typically within 2-3 minor versions of the latest
Backstage release. The compatibility range is documented in README and
verified before every release.

## Rules

- MUST: Declare `@backstage/core-plugin-api` and/or `@backstage/frontend-plugin-api` as `peerDependencies` with a caret range — never an exact version
- MUST: The lower bound of the peer dependency range reflects the minimum Backstage version where all used APIs actually exist
- MUST: Verify the upper bound against the latest Backstage release before publishing — fetch release docs and check for deprecations
- MUST: Document the supported Backstage version range in `README.md` under a `## Compatibility` section
- MUST: Bump the upper bound of the peer dependency range within 4 weeks of a new Backstage minor release
- MUST: If a Backstage release deprecates an API the plugin uses, open an issue and label it `backstage-compat`
- MUST NOT: Declare a peer dependency range wider than 6 minor versions without a documented compatibility matrix
- MUST NOT: Use a `*` or `>=` range for Backstage peer dependencies — these make no compatibility promise
- MUST NOT: Pin to an exact version — `"^1.30.0"` not `"1.30.0"`
- SHOULD: Use `yarn backstage-cli versions:check` locally before each release to detect incompatibilities
- SHOULD: Maintain a compatibility table in README when the plugin supports multiple major Backstage versions
- SHOULD NOT: Support more than 2 Backstage major versions simultaneously — the maintenance burden is unsustainable

## Examples

### Correct

```json
// package.json — caret range, realistic bounds
{
  "peerDependencies": {
    "@backstage/core-plugin-api": "^1.30.0",
    "@backstage/frontend-plugin-api": "^0.9.0",
    "react": "^17.0.0 || ^18.0.0",
    "react-dom": "^17.0.0 || ^18.0.0",
    "react-router-dom": "^6.0.0"
  }
}
```

```markdown
<!-- README.md -->
## Compatibility

| Plugin version | Backstage version | Status |
|---|---|---|
| 2.x | 1.40.x — 1.48.x | ✅ Supported |
| 1.x | 1.30.x — 1.39.x | ⚠️ Security fixes only |
| 0.x | < 1.30.x | ❌ End of life |

**Current recommendation:** Use plugin 2.x with Backstage 1.44+.

To check compatibility in your project:
```bash
yarn backstage-cli versions:check
```
```

### Correct — widening peer deps for new Backstage support (minor bump)

```json
// before — supported up to 1.40
{
  "peerDependencies": {
    "@backstage/core-plugin-api": "^1.30.0"
  }
}

// after — widened to include 1.48 (minor bump — additive)
{
  "peerDependencies": {
    "@backstage/core-plugin-api": "^1.30.0"
  }
}
// Note: caret range already covers 1.48 if the APIs used still exist.
// If new APIs from 1.44 are adopted, bump lower bound and release as minor:
{
  "peerDependencies": {
    "@backstage/core-plugin-api": "^1.44.0"  // uses APIs added in 1.44
  }
}
```

### Incorrect

```json
// ❌ wildcard range — no compatibility promise
{
  "peerDependencies": {
    "@backstage/core-plugin-api": "*"
  }
}

// ❌ exact version — forces consumers to match exactly
{
  "peerDependencies": {
    "@backstage/core-plugin-api": "1.30.0"   // use ^1.30.0
  }
}

// ❌ lower bound too low — plugin uses APIs not available in 1.0
{
  "peerDependencies": {
    "@backstage/core-plugin-api": "^1.0.0"   // uses createFrontendPlugin from 1.30
  }
}

// ❌ range wider than tested
{
  "peerDependencies": {
    "@backstage/core-plugin-api": "^1.0.0"   // maintainer only tested 1.40+
  }
}
```

```markdown
<!-- ❌ missing compatibility section in README -->
# My Tool Plugin

Install: `yarn add @scope/backstage-plugin-my-tool`
<!-- no mention of which Backstage versions are supported -->
```

## Verify signal

Reject if:
- Any Backstage peer dependency uses `*` or `>=` range
- Any Backstage peer dependency is pinned to an exact version — no caret
- `README.md` has no `## Compatibility` section
- The lower bound of the peer dependency range is lower than the minimum Backstage version where all APIs used by the plugin actually exist
- The peer dependency range spans more than 8 minor versions without a documented compatibility matrix

Flag for human review if:
- The upper bound was last updated more than 4 weeks ago — may be out of date with latest Backstage
- The plugin uses APIs from `@backstage/backend-common` which is being deprecated — compatibility with future versions at risk
- The compatibility table in README shows a version as "supported" but the CI matrix does not test against it — the support claim is unverified
- Two major Backstage versions are supported simultaneously — verify the maintenance burden is sustainable