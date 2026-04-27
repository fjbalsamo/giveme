# giveme

> *your codebase, your rules, enforced automatically.*

You write what you want to build. giveme knows how your team — or you — works best, and generates code that already follows those rules. No prompts to craft, no repeated review comments, no "we don't do it that way here."

For the freelancer who wants consistency across projects.  
For the team that wants the rules to live in the code, not just in the senior's head.

---

## The problem

AI coding tools are fast. Really fast. But fast in the wrong direction is just faster technical debt.

Every team has rules. Some are written down. Most live in someone's head. When AI generates code, it doesn't know your rules — it knows the internet's rules. So you get code that works but doesn't fit. Code that passes CI but fails review. Code that the senior has to fix. Again.

The problem isn't AI. The problem is that your standards aren't machine-readable yet.

**giveme fixes that.**

---

## How it works

```bash
# 1. install once, globally
claude plugin install github.com/fjbalsamo/giveme

# 2. in any repo, from any origin
/giveme:init

# 3. just say what you want
/giveme:run "I want an endpoint that registers customers"
```

That's it. giveme reads your project's guardrails, runs a fully automated pipeline, and opens a PR. You review and decide: **keep it or close it.**

---

## The pipeline

When you run `/giveme:run`, four isolated agents work in sequence — each one unaware of the others, each one focused on a single job:

```
your intent
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  specify agent   reads your intent + guardrails → spec.md   │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  plan agent      maps spec against guardrails → plan.md     │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  implement agent generates code within guardrail boundaries  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  verify agent    fresh context — finds violations, not excuses│
└──────────────────────────┬──────────────────────────────────┘
                           │
                     COMPLIANT?
                    /         \
                  yes          no
                   │            │
                   ▼            ▼
               open PR     loop back to implement
                               (max 3 retries)
```

The verify agent runs in a completely fresh session — no memory of what was implemented. This is intentional. It's the only real defense against an AI that confirms its own work.

---

## Skills

| Skill | What it does |
|---|---|
| `/giveme:init` | Detects your stack, downloads community guardrails, sets up `.specify/` |
| `/giveme:run` | Runs the full pipeline from intent to PR, unattended |
| `/giveme:community` | Interviews you about how you work, builds your custom playbook |
| `/giveme:check` | Validates existing code against your current guardrails |
| `/giveme:mcp-add` | Adds a new MCP integration — creates config, usage guide, and secrets template in `giveme.env` |

---

## Guardrails, not rules

We don't call them rules. Rules feel imposed. Guardrails feel protective.

A guardrail answers one question: *"why do we do it this way here?"* It has context, a decision, and a clear signal for what a violation looks like. It's machine-readable enough for an AI to enforce, and human-readable enough for a new dev to understand on day one.

```markdown
# Guardrail: Error handling

## Decision
Use NestJS built-in exceptions. Never throw plain Error objects 
to the client.

## Rules
- MUST: Use NotFoundException, BadRequestException, ConflictException
- MUST NOT: throw new Error('something went wrong')
- MUST NOT: console.log — use Logger with module context

## Verify signal
Reject code that contains:
- `throw new Error(` outside of test files
- `console.log` or `console.error` anywhere in src/
```

---

## Community playbooks

giveme ships with community-curated guardrail sets for common stacks. When you run `/giveme:init`, it detects your stack and loads the right playbook automatically.

| Stack | Status |
|---|---|
| NestJS | ✅ available |
| Spring Boot | 🚧 coming soon |
| Go | 🚧 coming soon |
| Python CLI | 🚧 coming soon |
| Django | 🚧 coming soon |

Community playbooks are the **floor**, not the ceiling. Your team's guardrails always take precedence over the defaults.

**Want to contribute a playbook for your stack?** See [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## Your own playbook

Don't want to start from community defaults? Run the interview:

```bash
/giveme:community nestjs
```

giveme will ask you how you actually work — in plain language, no jargon. It listens, and turns your answers into a guardrail set that reflects your real conventions, not someone else's opinions.

For freelancers: your playbook lives in `~/.giveme/` and travels with you across projects.  
For teams: your playbook lives in `.specify/` and travels with the repo.

---

## Precedence

When guardrails conflict, this order wins:

```
team (.specify/playbook/) > personal (~/.giveme/) > community (giveme/playbooks/)
```

Your team's decisions always override community defaults. Your personal style overrides community defaults when working solo. Community defaults fill the gaps when nothing else is defined.

---

## Your project, your rules

When you run `/giveme:init`, giveme copies the community playbook into your
project's `.specify/playbook/` directory. From that moment, those files
belong to your project — not to giveme.

```
your-project/
└── .specify/
    ├── playbook/           ← your guardrails, versioned with your code
    │   ├── 001-module-structure.md     ← edit freely
    │   ├── 002-validation-contracts.md ← edit freely
    │   └── ...
    ├── orchestrator.md     ← your pipeline definition
    └── constitution.md     ← your project standards
```

**giveme never overwrites files in `.specify/playbook/`.**
If you run `/giveme:init` again, it skips files that already exist.
Your edits are always preserved.

This matters because two companies using NestJS can have completely
different standards. One uses `winston`, the other uses the NestJS Logger.
One requires optimistic locking everywhere, the other exempts read-heavy
services. The community playbook is a starting point — your `.specify/`
is the real thing.

### Updating community defaults

When giveme releases new or improved guardrails, they don't land in your
project automatically. You choose when to update:

```bash
/giveme:init --update
```

This shows you a diff of what changed in the community playbook since
you last initialized, and lets you accept changes file by file. You stay
in control.

### Committing `.specify/`

`.specify/` should be committed to version control. It's part of your
project — not a build artifact, not a secret. Your team reviews guardrail
changes in PRs just like any other code change. That's the point.

---

## Requirements

- [Claude Code](https://claude.ai/code) installed and authenticated
- Git repository (any origin — GitHub, GitLab, Bitbucket, local)
- That's it

No CI/CD setup required. No Backstage. No IDP. No configuration files before you start.

---

## Installation

```bash
claude plugin install github.com/fjbalsamo/giveme
```

Then in any repo:

```bash
/giveme:init
```

---

## FAQ

**Does this work with any repo, regardless of how it was created?**  
Yes. giveme doesn't care how the repo got there. It detects the stack and works from there.

**What if my team already has conventions written somewhere?**  
Run `/giveme:community --review`. giveme will read what you have and help you formalize it into guardrails the system can enforce.

**What if the generated code doesn't match what I wanted?**  
Close the PR. That's your single decision point. The pipeline ran, the code is compliant with your guardrails — but if it's not what you meant, close it and re-run with a more specific intent.

**Can I use this without Jira?**  
Yes. Jira integration is optional. You can run the pipeline with a plain text intent and no ticket reference.

**Is the verify agent really independent?**  
Yes. It runs in a fresh Claude Code session with no memory of the implementation phase. This is enforced by the plugin architecture, not by convention.

---

## Philosophy

giveme is built on one belief: **the best code review is the one that never needs to happen.**

If the guardrails are right, the code is right before the PR opens. The PR is a checkpoint for business logic, not a correction session for style, patterns, and conventions that should have been enforced from the start.

We're not trying to replace developers. We're trying to make sure that when an AI writes code, it writes *your* code — not generic code that happens to compile.

---

## Contributing

giveme grows with the community. The most valuable contribution isn't code — it's a well-crafted playbook for a stack that isn't covered yet.

See [CONTRIBUTING.md](./CONTRIBUTING.md) for how to add a new stack playbook, improve an existing guardrail, or report a violation signal that doesn't work in practice.

---

## License

MIT — use it, fork it, build on it.

---

*Built by [@fjbalsamo](https://github.com/fjbalsamo) · Powered by [Claude Code](https://claude.ai/code)*