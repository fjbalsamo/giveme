---
name: mcp-add
description: Adds a new MCP integration to giveme. Creates the mcp.json config, mcp-usage.md guide, and updates giveme.env with the required secrets template. Use when a developer wants to connect giveme to a new external service.
---

You are setting up a new MCP integration for giveme. Your job is to
create the config files, document how the agents use the MCP, and
generate the secrets template in `giveme.env`.

Be precise. MCPs that are misconfigured silently fail — the pipeline
runs but without the external context it should have.

---

## Your behavior rules

- Never write actual secret values — only placeholders.
- Always add secrets to `giveme.env` with the `GIVEME_` prefix.
- Always add `giveme.env` to `.gitignore` if not already there.
- Create both `mcp.json` and `mcp-usage.md` — never one without the other.
- Tell the dev exactly where to get their credentials.

---

## Phase 1 — Identify the MCP

Read `$ARGUMENTS` to get the MCP name (e.g. `linear`, `confluence`, `slack`).

If no argument provided, ask:
*"Which service do you want to connect? (e.g. linear, confluence, slack, notion)"*

Search for a known MCP server package for the requested service:
- Check `@modelcontextprotocol/server-<name>` on npm
- Check `@anthropic-labs/mcp-<name>` on npm
- If neither exists, ask the dev to provide the package name manually

---

## Phase 2 — Gather required info

Ask the dev:
*"What will giveme use [service] for in the pipeline?"*

Listen for which pipeline phase benefits:
- Specify phase → reading ticket/issue context
- Plan phase → reading additional docs or linked resources
- Run phase → opening issues, creating records
- Verify phase → checking external state

This determines what to document in `mcp-usage.md`.

---

## Phase 3 — Create the files

### `mcps/<service>/mcp.json`

```json
{
  "mcpServers": {
    "<service>": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "<mcp-package-name>"],
      "env": {
        "<ENV_VAR_EXPECTED_BY_PACKAGE>": "${GIVEME_<SERVICE>_<VAR>}"
      }
    }
  }
}
```

### `mcps/<service>/mcp-usage.md`

Follow the same structure as `mcps/jira/mcp-usage.md`:
- What it does
- Required secrets (with placeholder values)
- How to get credentials (step by step)
- How agents use it (table by phase)
- What happens without this MCP
- Setup instructions

### `giveme.env` — append secrets template

Append to `giveme.env` (create if it doesn't exist):

```bash
# ─── <Service> ──────────────────────────
GIVEME_<SERVICE>_<VAR>=your-value-here
```

Never overwrite existing entries. Always append.

### `.gitignore` — verify entry exists

Check if `giveme.env` is in `.gitignore`.
If not → add it. If yes → skip silently.

---

## Phase 4 — Summary

---

*"[Service] MCP is ready to configure.*

*Files created:*
- `mcps/<service>/mcp.json`
- `mcps/<service>/mcp-usage.md`

*Secrets added to `giveme.env`:*
- `GIVEME_<SERVICE>_<VAR>` — [where to get it]

*Next steps:*
1. Fill in your credentials in `giveme.env`
2. Run `/giveme:init` to activate the MCP
3. Run `/giveme:run` — the pipeline will use [service] automatically"*

---

## Arguments

`$ARGUMENTS` — the service name to add (e.g. `linear`, `confluence`, `slack`, `notion`).
If not provided, the skill asks interactively.