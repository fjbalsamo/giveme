# MCP — Jira

## What it does

Connects giveme to your Jira instance. When you run `/giveme:run --ticket TICKET-ID`,
the specify agent reads the ticket directly from Jira instead of relying on
the plain text intent alone. This gives the pipeline full context: acceptance
criteria, description, linked issues, and priority — without copy-paste.

## Required secrets

Add these to `giveme.env` in the root of your project:

```bash
# ─── Jira ───────────────────────────────
GIVEME_JIRA_URL=https://your-org.atlassian.net
GIVEME_JIRA_EMAIL=you@example.com
GIVEME_JIRA_API_TOKEN=your-token-here
```

**How to get your API token:**
1. Go to https://id.atlassian.com/manage-profile/security/api-tokens
2. Click "Create API token"
3. Copy the token — you won't see it again

## How agents use it

| Agent | What it reads from Jira |
|---|---|
| `specify` | Title, description, acceptance criteria, priority |
| `plan` | Linked issues, parent epic, labels |

The `implement` and `verify` agents do not use Jira — they work from
`spec.md` and `plan.md` only.

## Without this MCP

If Jira is not configured, `/giveme:run` falls back to plain text mode.
The intent you pass becomes the only source of context for the specify agent.

```bash
# with Jira — ticket is fetched automatically
/giveme:run "register a customer" --ticket NARX-1234

# without Jira — plain text only
/giveme:run "I want a POST /v1/customers endpoint that validates name and email, returns 201 with UUID"
```

Plain text mode works — but the more specific your intent, the better the output.

## Setup

Run `/giveme:init` after adding your secrets to `giveme.env`.
The init skill will detect the Jira MCP and validate the connection.