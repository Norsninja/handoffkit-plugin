---
description: |
  Connect the current project to HandoffKit for automatic handoff syncing via MCP. Use when the user says "handoffkit setup", "connect to handoffkit", "link this project to handoffkit", "configure handoffkit", "set up handoffkit", "relink handoffkit", or wants to push handoffs to HandoffKit.
tools:
  - Bash
  - Read
  - Write
  - Edit
---

# HandoffKit Setup

Connect this project to a HandoffKit project so handoffs can be pushed automatically via the `push-handoff` MCP tool.

## Modes

Detect which mode by checking the project root:

- **Fresh install**: no `.handoffkit.json` exists.
- **Migration**: `.handoffkit.json` exists AND contains an `api_key` field (legacy format from before MCP).
- **Relink**: `.handoffkit.json` exists AND has no `api_key` (already in marker format) — happens when MCP config was lost.

The flow below is the same for all three; only the cleanup at the end differs.

## Prerequisites

The user must have:
1. A HandoffKit account at https://handoffkit.com
2. A setup token generated from HandoffKit Settings → Plugin Integration → Generate Token
3. The token looks like `hk_setup_<random>` and expires in 10 minutes

If the user hasn't provided a token, tell them:

> Go to **handoffkit.com/settings** → **Plugin Integration** → select your project → click **Generate Token**.
> Copy the token and paste it here. It expires in 10 minutes.

## Setup Flow

### Step 1: Exchange the token for an API key

```bash
curl -s -X POST https://handoffkit.com/api/v1/auth/exchange \
  -H "Content-Type: application/json" \
  -d '{"setup_token": "TOKEN_HERE"}'
```

Expected success:
```json
{
  "api_key": "hk_key_...",
  "project_id": "uuid",
  "project_name": "Project Name",
  "workspace_id": "uuid"
}
```

On error responses (`Token expired`, `Token already used`, `Invalid token`): stop here, tell the user to generate a new token.

Capture `api_key`, `project_id`, `project_name` for the next steps. Never display the full `api_key` — show only the first 15 chars followed by `...`.

### Step 2: Verify the new key against the MCP endpoint

Before changing any local files or removing existing connections, prove the new key works against the MCP server:

```bash
curl -s -X POST https://handoffkit.com/api/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer NEW_API_KEY" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

The response must contain a tool named `push-handoff`. If the response is an error or doesn't list that tool, **stop here** — something is wrong with the new key. Don't touch any local files yet.

### Step 3: Add (or replace) the MCP connection

Check whether a HandoffKit MCP connection already exists for this project:

```bash
claude mcp list
```

- If `handoffkit` is listed: ask the user "An existing handoffkit connection is configured. Replace it?". If yes, run `claude mcp remove handoffkit` first.
- If not listed: skip removal.

Then add the new connection:

```bash
claude mcp add handoffkit \
  --transport http \
  --scope local \
  --header "Authorization: Bearer NEW_API_KEY" \
  -- https://handoffkit.com/api/mcp
```

`--scope local` is required — it scopes the connection to this project directory only. The key is stored by Claude Code in `~/.claude.json`, never in any project file.

### Step 4: Write the marker file (safe — only after Steps 1-3 succeed)

Write `.handoffkit.json` to the project root. **No API key.** This file is a marker that says "this project is linked to a HandoffKit project" — useful for the `/handoff` command and for recovery if the MCP connection is later lost.

```json
{
  "project_id": "<project_id from step 1>",
  "project_name": "<project_name from step 1>",
  "api_url": "https://handoffkit.com/api/v1",
  "linked_at": "<current ISO 8601 timestamp>"
}
```

If a `.handoffkit.json` already existed (migration mode), overwrite it with the new marker format. The old format had `api_key` and `workspace_id` fields — both removed in the new format.

### Step 5: Update .gitignore

Read `.gitignore` from the project root. If `.handoffkit.json` is not already listed, append it on its own line. If `.gitignore` doesn't exist, create one with just `.handoffkit.json`.

### Step 6: Update the /handoff command

If `.claude/commands/handoff.md` exists, check whether its push step (typically step 5) uses the legacy inline-curl pattern. The legacy pattern contains either of:
- the literal string `${API_URL}/cells`
- a `curl ... /api/v1/cells` line inside the handoff command

If a legacy pattern is detected, **and the rest of the file otherwise matches the standard handoff command template**, replace just the push step with the new MCP-aware version below. If the file looks heavily customized (other content differs), don't auto-edit — print the recommended diff and ask the user to apply it manually.

The new push step content (Step 5):

```markdown
5. **Push to HandoffKit (if linked)**:
   - If you have access to the `push-handoff` MCP tool, call it with `title` set to `Session Handoff - YYYY-MM-DD` (today's date) and `content` set to the chat message from Step 3. On success, tell the user "Handoff pushed to HandoffKit." On error, warn but don't block — the local handoff file and chat message are already complete.
   - Else if `.handoffkit.json` exists in the project root: tell the user "HandoffKit is linked but the MCP connection isn't loaded in this session — run `/handoffkit:setup` to relink." Don't fall back to inline curl.
   - Else: skip silently. Don't mention HandoffKit.
```

Apply the same update to `.config/commands/handoff.md` if it exists (some users mirror the command there).

### Step 7: Migration cleanup — revoke the old API key

Only applies in migration mode (the original `.handoffkit.json` had an `api_key` field).

Use the OLD api_key (captured before overwriting the file) to revoke itself:

```bash
curl -s -X POST https://handoffkit.com/api/v1/auth/revoke-self \
  -H "Authorization: Bearer OLD_API_KEY"
```

Expected success: `{"revoked": true, "key_prefix": "hk_key_xx..."}`. If this fails (e.g., the old key was already revoked), don't block — the migration's other steps already succeeded. Just note the failure to the user and suggest revoking from handoffkit.com/settings.

### Step 8: Confirm

Tell the user:

> HandoffKit connected via MCP.
> **Project:** {project_name}
> **Scope:** This project directory only
>
> Restart Claude Code to activate the `push-handoff` tool.
> After restart, your `/handoff` command will push chat messages automatically.
>
> To disconnect: `claude mcp remove handoffkit` and delete `.handoffkit.json`
> To manage keys: handoffkit.com/settings

If migration mode: also include "Old API key revoked." in the summary.

## Error Handling

- **`claude mcp add` fails**: check `claude --version` — MCP support requires a recent version. If failure is "already exists," go back to Step 3's removal path.
- **Step 2 verification fails**: stop. Tell the user to check that handoffkit.com is reachable. Don't proceed with file changes.
- **Network error on exchange**: tell user to check connectivity. Token may still be valid for retry within its 10-minute window.
- **Old key revoke fails (migration)**: don't block. The new key works; the old one will eventually be cleaned up. User can manually revoke at handoffkit.com/settings.

## Security

- Setup tokens: single-use, 10-minute expiry
- API keys: project-scoped, stored only in Claude Code's `~/.claude.json` after Step 3
- The marker `.handoffkit.json` contains no secrets — it's safe-by-default if accidentally committed, but Step 5 gitignores it anyway
- Never display the full API key in output — show only the first 15 characters followed by `...`
- Old keys are revoked server-side after migration so they can't be used even if the old `.handoffkit.json` was committed somewhere
