---
description: |
  Connect the current project to HandoffKit for automatic handoff syncing via MCP. Use when the user says "handoffkit setup", "connect to handoffkit", "link this project to handoffkit", "configure handoffkit", "set up handoffkit", or wants to push handoffs to HandoffKit.
tools:
  - Bash
  - Read
  - Write
---

# HandoffKit Setup

Connect this project to a HandoffKit project so handoffs can be pushed automatically via the `push-handoff` MCP tool.

## Prerequisites

The user must have:
1. A HandoffKit account at https://handoffkit.com (free tier works)
2. A setup token generated from HandoffKit Settings → Plugin Integration → Generate Token
3. The token looks like: `hk_setup_<random_string>` and expires in 10 minutes

## Setup Flow

### Step 1: Get the token

If the user hasn't provided a token, tell them:

> Go to **handoffkit.com/settings** → **Plugin Integration** → select your project → click **Generate Token**.
> Copy the token and paste it here. It expires in 10 minutes.

### Step 2: Exchange the token for an API key

Once you have the token, exchange it:

```bash
curl -s -X POST https://handoffkit.com/api/v1/auth/exchange \
  -H "Content-Type: application/json" \
  -d '{"setup_token": "TOKEN_HERE"}'
```

Expected success response:
```json
{
  "api_key": "hk_key_...",
  "project_id": "uuid-here",
  "project_name": "NewsplanetAI",
  "workspace_id": "uuid-here"
}
```

If the response contains an error:
- `"Token expired"` → Generate a new token from Settings
- `"Token already used"` → Generate a new token
- `"Invalid token"` → Check for typos or extra whitespace

### Step 3: Add the MCP connection

Use `claude mcp add` with **local scope** so the connection is project-specific:

```bash
claude mcp add handoffkit \
  --transport http \
  --scope local \
  --header "Authorization: Bearer API_KEY_HERE" \
  -- https://handoffkit.com/api/mcp
```

**Important:**
- Use `--scope local` — this scopes the connection to this project directory only
- The API key is stored securely by Claude Code in `~/.claude.json`, not in any project file
- Never display the full API key in output — show only the first 15 characters followed by `...`

### Step 4: Confirm

Tell the user:

> HandoffKit connected successfully via MCP.
> **Project:** {project_name}
> **Scope:** This project directory only
>
> Restart Claude Code to activate the `push-handoff` tool.
> After restart, your `/handoff` command can push chat messages directly to HandoffKit.
>
> To disconnect: `claude mcp remove handoffkit`
> To manage keys: handoffkit.com/settings

## Error Handling

- If the `claude mcp add` command fails, suggest the user check their Claude Code version (`claude --version`) — MCP support requires a recent version.
- If a HandoffKit MCP connection already exists, ask the user if they want to replace it. If yes, run `claude mcp remove handoffkit` first, then add the new one.
- If the curl command fails with a network error, suggest checking internet connectivity.
- Never display the full API key — show only the prefix (first 15 characters) followed by `...`

## Security Notes

- The setup token is single-use and time-limited (10 minutes)
- The API key is scoped to one HandoffKit project only
- The key is stored by Claude Code, not in any project file — nothing to gitignore
- Users can revoke API keys from handoffkit.com/settings at any time
- The MCP connection uses `--scope local` so it only applies to this project directory
