# Contributing to giveme

First of all — thank you. giveme grows through the community, and the most valuable contribution isn't code. It's a well-crafted playbook for a stack that isn't covered yet.

This guide explains how to contribute a new stack playbook, improve an existing guardrail, or report a verify signal that doesn't work in practice.

---

## Ways to contribute

**Add a new stack playbook** — your stack isn't covered yet? That's the most impactful thing you can do. See [Adding a new playbook](#adding-a-new-playbook).

**Improve an existing guardrail** — you found a rule that's too strict, too loose, or missing context. See [Improving a guardrail](#improving-a-guardrail).

**Report a verify signal that doesn't work** — the AI enforces something incorrectly, or misses a real violation. Open an issue with the label `verify-signal`.

**Fix typos, improve clarity** — always welcome, no issue needed.

---

## Before you start

A few principles that guide every decision in this project:

**Guardrails should answer "why", not just "what".** A rule without context is just friction. Every guardrail needs enough context for a developer to understand the reasoning, not just the restriction.

**Verify signals must be concrete.** Vague signals produce false positives and lose developer trust fast. If you can't describe exactly what pattern to look for in the code, the guardrail isn't ready yet.

**Community playbooks are the floor, not the ceiling.** They represent solid defaults for a stack — not the one true way. Teams override them. That's expected and healthy.

**Your guardrail lives in the dev's project, not in giveme.** When `/giveme:init` runs, it copies your guardrail into `.specify/playbook/` of the dev's repo. From that point, it belongs to them. They edit it, version it, and make it match their company's standards. Your contribution is the starting point — not the final word.

This means: write guardrails that are correct for 80% of teams using that stack. Don't try to cover every edge case. The dev will customize the remaining 20%.

**Less is more.** A playbook with 5 well-crafted guardrails is more valuable than one with 20 mediocre ones. Resist the urge to cover everything.

---

## Adding a new playbook

### 1. Check what's already there

Browse `playbooks/` to see if your stack exists. If it does, check if it's marked `🚧 coming soon` in the README — it might just need someone to write the content.

### 2. Create the folder structure

```bash
playbooks/
└── your-stack/
    ├── playbook.md        ← index and overview
    ├── 001-<topic>.md     ← first guardrail
    ├── 002-<topic>.md     ← second guardrail
    └── ...
```

Use lowercase, hyphen-separated names for the folder. Use zero-padded numbers for guardrail files so they sort correctly.

Good folder names: `nestjs`, `spring-boot`, `golang`, `python-cli`, `django`, `rails`  
Avoid: `NestJS`, `my_stack`, `the-best-framework-ever`

### 3. Write the `playbook.md` index

```markdown
# Playbook — Your Stack

## Overview
One paragraph. What is this stack, who uses it, what kind of projects.

## Stack version
Which version(s) these guardrails apply to. Be specific.
Example: NestJS 11.x, Node.js 22+

## Guardrails in this playbook
- [001 — Module structure](./001-module-structure.md)
- [002 — Validation](./002-validation.md)
- ...

## What's not covered
Be honest about gaps. Better to document what's missing than to pretend it's complete.

## Maintainers
- @your-github-handle
```

### 4. Write each guardrail

Every guardrail file follows this exact schema. Don't skip sections.

```markdown
# Guardrail NNN — Title

## Context
Why does this decision matter? What problem does it solve?
What happens when teams ignore it? Keep it to 2-4 sentences.

## Decision
What did we decide to do, and why this option over the alternatives?
Be direct. One short paragraph.

## Rules
- MUST: [required behavior]
- MUST: [required behavior]
- MUST NOT: [forbidden pattern]
- MUST NOT: [forbidden pattern]
- SHOULD: [recommended behavior, but not enforced]
- SHOULD NOT: [discouraged pattern, but not blocked]

## Examples

### Correct
\`\`\`typescript
// show a real example of compliant code
\`\`\`

### Incorrect
\`\`\`typescript
// show a real example of what to avoid
\`\`\`

## Verify signal
The AI verify agent uses this section to detect violations.
Be as specific as possible — file patterns, code patterns, imports.

Reject code that contains:
- [concrete pattern 1 — e.g. `throw new Error(` outside *.spec.ts]
- [concrete pattern 2 — e.g. `console.log` anywhere in src/]
- [concrete pattern 3]

Flag for human review if:
- [ambiguous case that needs judgment]
```

### 5. Open a pull request

- Branch name: `playbook/<stack-name>`
- PR title: `playbook: add <stack-name> guardrails`
- PR description: briefly explain your experience with this stack and why you chose these specific guardrails over others

A maintainer will review with two questions in mind: *"Is the context clear enough for a junior dev to understand?"* and *"Is the verify signal specific enough for an AI to enforce without false positives?"*

---

## Improving a guardrail

If you find an existing guardrail that's wrong, incomplete, or generates false positives:

1. Open an issue first if the change is significant — explain the problem and your proposed fix
2. For small fixes (typos, clarity, missing examples) you can go straight to a PR
3. Branch name: `fix/<stack>/<guardrail-number>`
4. Always update the verify signal if you're changing a rule — they have to stay in sync

---

## Schema reference

### Rule levels

| Level | Meaning | Enforced by verify agent? |
|---|---|---|
| `MUST` | Required. No exceptions. | Yes — blocks the PR |
| `MUST NOT` | Forbidden. No exceptions. | Yes — blocks the PR |
| `SHOULD` | Strongly recommended | No — flagged for review |
| `SHOULD NOT` | Strongly discouraged | No — flagged for review |

### Verify signal patterns

Verify signals should be expressed as one of:

- **String patterns** — `throw new Error(`, `console.log`, `import * as`
- **File patterns** — `outside *.spec.ts`, `in src/ only`, `in *.controller.ts`
- **Structural patterns** — `more than one return statement in a controller method`
- **Import patterns** — `DataSource imported outside *.migration.ts`

Avoid signals that require understanding business logic — the verify agent can read code structure, not intent.

---

## Code of conduct

Be kind. Everyone here is trying to make developers' lives better. Disagreements about technical decisions are welcome and expected — that's how good guardrails get written. Personal attacks are not.

---

## Questions?

Open an issue with the label `question`. No question is too basic.

---

*Thank you for making giveme better for everyone.*