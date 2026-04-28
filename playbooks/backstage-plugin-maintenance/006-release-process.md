# Guardrail 006 — Release process

<!--
  Source: giveme community playbook (backstage-plugin-maintenance/006-release-process.md)
  Copied: see project history
  Status: community default — edit freely to match your standards

  This file belongs to your project. giveme reads it from .specify/playbook/
  and will never overwrite it. Changes here affect only this project.
-->

## Context

A release is an irreversible act. Once a version is published to npm,
it cannot be unpublished without consequences for every consumer who
has already installed it. A release with a wrong version number, a missing
changelog entry, or an unpublished migration guide creates confusion that
takes multiple follow-up releases to resolve.

The release process for a Backstage plugin has more checkpoints than a
generic npm package because the plugin's public contract spans multiple
surfaces: TypeScript exports, config keys, action IDs, provider names,
and Backstage version compatibility. Each checkpoint exists to protect
consumers from avoidable breakage.

The most common release mistakes are: bumping the version in package.json
without moving the `## [Unreleased]` section to a dated entry, publishing
without running the full test suite against the declared Backstage version
range, and forgetting to create the git tag that links the changelog
diff URLs.

## Decision

Every release follows a fixed sequence: pre-release checklist, version
bump, changelog finalization, git tag, npm publish, and post-release
verification. No step is skipped. The release PR is the record of the
decision — it includes the version bump, the changelog update, and the
Backstage compatibility verification together.

## Rules

- MUST: Run the full pre-release checklist before bumping any version
- MUST: Move `## [Unreleased]` content to a dated release entry — `## [x.y.z] — YYYY-MM-DD`
- MUST: Add a new empty `## [Unreleased]` section above the dated entry
- MUST: Update the diff link at the bottom of `CHANGELOG.md` for the new version
- MUST: Create a git tag `vX.Y.Z` that matches the `package.json` version
- MUST: Run `npm pack --dry-run` before publish to verify the `files` field includes only intended files
- MUST: Verify the published package on npm within 10 minutes of publish — confirm the version appears and the dist files are present
- MUST: Open a release PR that includes version bump + changelog update together — never separate commits
- MUST NOT: Publish to npm directly from a local machine without running the full test suite
- MUST NOT: Skip the git tag — changelog diff URLs depend on it
- MUST NOT: Publish a version that is not tagged in git
- SHOULD: Use `npm publish --dry-run` before the real publish to catch packaging errors
- SHOULD: Verify `yarn backstage-cli versions:check` passes before publish
- SHOULD: Add a GitHub Release with the changelog entry as the release notes
- SHOULD NOT: Publish on a Friday or before a long weekend — if something goes wrong, consumers need maintainer availability

## Pre-release checklist

Before bumping the version, verify every item:

```markdown
## Pre-release checklist

### Versioning
- [ ] Identified all changes since last release
- [ ] Determined correct semver bump per guardrail 002
- [ ] All breaking changes have prior deprecation notices (guardrail 005)

### Changelog
- [ ] `## [Unreleased]` section is complete and accurate
- [ ] All breaking changes have migration steps (guardrail 003)
- [ ] All deprecated APIs are listed under `### Deprecated`

### Compatibility
- [ ] Peer dependency lower bound reflects actual minimum Backstage version used
- [ ] Verified against latest Backstage release (guardrail 004)
- [ ] `README.md` compatibility table is up to date

### Quality
- [ ] Full test suite passes locally
- [ ] `yarn backstage-cli versions:check` passes
- [ ] `npm pack --dry-run` output reviewed — no unexpected files included

### Release mechanics
- [ ] `package.json` version bumped
- [ ] `## [Unreleased]` moved to dated entry
- [ ] New empty `## [Unreleased]` section added
- [ ] Diff link added to bottom of CHANGELOG.md
- [ ] Release PR opened with version bump + changelog together
- [ ] Release PR approved and merged
- [ ] Git tag `vX.Y.Z` created on merge commit
- [ ] `npm publish` executed
- [ ] Published version verified on npm registry
- [ ] GitHub Release created with changelog entry as notes
```

## Examples

### Correct — release PR diff

```diff
// package.json
-  "version": "1.4.2",
+  "version": "1.5.0",

// CHANGELOG.md
+## [Unreleased]
+
+---
+
-## [Unreleased]
+## [1.5.0] — 2026-04-28
+
+### 🎯 Operator impact
+- **App operators**: No action required
+- **Template authors**: No action required
+
+### Added
+- Support for filtering entity cards by `spec.lifecycle` field (#47)
+
+### Fixed
+- Corrected HTTP timeout on catalog fetch — was 5s, now 30s (#45)

 ## [1.4.2] — 2026-03-15
 ...

+[1.5.0]: https://github.com/scope/my-tool/compare/v1.4.2...v1.5.0
 [1.4.2]: https://github.com/scope/my-tool/compare/v1.4.1...v1.4.2
```

### Correct — git tag and publish sequence

```bash
# after release PR is merged
git checkout main
git pull origin main

# create annotated tag
git tag -a v1.5.0 -m "Release v1.5.0"
git push origin v1.5.0

# dry run first
npm pack --dry-run
npm publish --dry-run

# real publish
npm publish --access public

# verify within 10 minutes
npm info @scope/backstage-plugin-my-tool version
# should return 1.5.0
```

### Incorrect

```bash
# ❌ publishing without tag
npm publish                            # tag must exist before publish

# ❌ publishing from feature branch
git checkout feature/my-new-feature
npm publish                            # publish from main only, after PR merge

# ❌ version bump without changelog update
# package.json: "version": "1.5.0"
# CHANGELOG.md: still has ## [Unreleased] with no dated entry
```

```markdown
<!-- ❌ missing diff link in CHANGELOG -->
## [1.5.0] — 2026-04-28
...
## [1.4.2] — 2026-03-15
...
<!-- no [1.5.0]: https://github.com/... link at the bottom -->
```

## Verify signal

Reject if:
- `package.json` version was bumped but `CHANGELOG.md` still has `## [Unreleased]` with content and no dated entry
- `CHANGELOG.md` has no `## [Unreleased]` section — new entries have nowhere to go after release
- A version appears in `package.json` with no corresponding git tag `vX.Y.Z`
- The diff link for the new version is missing from the bottom of `CHANGELOG.md`
- A major version is released without a `## Migration guides` section in `README.md`

Flag for human review if:
- `npm pack --dry-run` output includes files not in the `files` array — packaging may include unintended files
- The release PR changes version and changelog in separate commits — they must be atomic
- More than 30 days passed since the last release and there are unreleased changes — consumers may be waiting
- The release is on a Friday — maintainer availability risk if something goes wrong