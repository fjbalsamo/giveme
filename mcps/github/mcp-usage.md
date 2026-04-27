# MCP — GitHub

## What it does

Connects giveme to your GitHub repository. When the pipeline reaches
Phase 5 (after verify returns COMPLIANT), the `giveme:run` skill uses
this MCP to create a branch, push the changes, and open a Pull Request
automatically — without the dev touching the terminal.

## Required secrets

Add these to `giveme.env` in the root of your project:

```bash
# ─── GitHub ─────────────────────────────
GIVEME_GITHUB_TOKEN=your-token-here
GIVEME_GITHUB_OWNER=your-org-or-username
GIVEME_GITHUB_REPO=your-repo-name
```

**How to get your Personal Access Token:**
1. Go to https://github.com/settings/tokens
2. Click "Generate new token (classic)"
3. Select scopes: `repo` (full control of private repositories)
4. Copy the token — you won't see it again

**Fine-grained tokens (recommended):**
1. Go to https://github.com/settings/personal-access-tokens/new
2. Set repository access to your target repo only
3. Grant permissions: `Contents` (read/write), `Pull requests` (read/write)

## How agents use it

| Phase | What it does via GitHub MCP |
|---|---|
| Phase 5 — PR | Creates branch `giveme/$TICKET`, pushes changes, opens PR |
| Phase 6 — Failure | Opens a GitHub issue with pipeline state and rejection reason |

The specify, plan, implement, and verify agents do not use GitHub.

## Without this MCP

If GitHub is not configured, the pipeline completes but cannot open a PR
automatically. After verify returns COMPLIANT, giveme will print the
branch name and the suggested PR description — you open the PR manually.

```
✅ Code is compliant. GitHub MCP not configured.

Branch: giveme/NARX-1234
To open the PR manually:
  git checkout -b giveme/NARX-1234
  git add .
  git commit -m "feat: add customer registration endpoint"
  git push origin giveme/NARX-1234
```

## PR format

When GitHub MCP is configured, giveme opens a PR with:

- **Branch:** `giveme/$TICKET`
- **Title:** derived from the intent (max 72 chars)
- **Body:** summary, link to spec.md, link to plan.md, guardrails applied, acceptance criteria
- **Label:** `giveme` (created automatically if it doesn't exist)

## Setup

Run `/giveme:init` after adding your secrets to `giveme.env`.
The init skill will detect the GitHub MCP and validate the connection.