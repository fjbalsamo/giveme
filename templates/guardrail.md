# Guardrail NNN — Title

<!--
  TEMPLATE — copy this file to start a new guardrail.
  Every section is required. Do not skip or rename them.
  The verify agent reads this structure literally.
-->

## Context

<!--
  Why does this decision matter?
  What problem does it solve?
  What goes wrong when teams ignore it?
  2-4 sentences max. Write for a developer who has never seen this codebase.
-->

## Decision

<!--
  What did we decide to do, and why this option over the alternatives?
  One short paragraph. Be direct.
-->

## Rules

<!--
  MUST / MUST NOT → enforced by the verify agent. No exceptions.
  SHOULD / SHOULD NOT → flagged for human review. Not a blocker.
  Keep each rule to one line. One behavior per rule.
-->

- MUST:
- MUST NOT:
- SHOULD:
- SHOULD NOT:

## Examples

### Correct

```
// A real example of compliant code.
// Show the pattern, not just the concept.
```

### Incorrect

```
// A real example of what to avoid.
// Show why it's wrong with a brief inline comment.
```

## Verify signal

<!--
  This is what the verify agent uses to detect violations.
  Be as specific as possible.
  Vague signals = false positives = developers stop trusting the tool.

  Express signals as one of:
  - String patterns   → `throw new Error(` outside *.spec.ts
  - File patterns     → `console.log` anywhere in src/
  - Import patterns   → DataSource imported outside *.migration.ts
  - Structural        → more than one DB call inside a *.controller.ts

  Do NOT write signals that require understanding business logic.
  The agent reads structure, not intent.
-->

Reject if:
-

Flag for human review if:
-