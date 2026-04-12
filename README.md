# HandoffKit Plugin for Claude Code

Push AI session handoffs directly to [HandoffKit](https://handoffkit.com) from Claude Code. No more copy-pasting between your terminal and the browser.

## What it does

This plugin provides a one-command setup (`/handoffkit:setup`) that connects your Claude Code project to HandoffKit via MCP. Once connected, you gain a `push-handoff` tool that sends session handoff messages to HandoffKit as new cells.

- **One-time setup per project** — run `/handoffkit:setup`, paste a token, done
- **Per-project scoping** — each project directory links to its own HandoffKit project
- **Secure** — API keys stored by Claude Code, not in project files
- **Non-blocking** — if the push fails, your handoff is still complete locally

## Setup

1. Install this plugin in Claude Code
2. Go to [handoffkit.com/settings](https://handoffkit.com/settings) → **Plugin Integration** → **Generate Token**
3. Select the project you want to link
4. Copy the token and run `/handoffkit:setup` in Claude Code
5. Paste the token when prompted

That's it. Restart Claude Code and the `push-handoff` MCP tool is available.

## How it works

The setup skill exchanges your one-time token for a project-scoped API key, then runs `claude mcp add --scope local` to create a per-project MCP connection to HandoffKit's remote server at `handoffkit.com/api/mcp`. After setup, the plugin itself is optional — the MCP connection persists independently.

## Usage

After setup, you can:
- Add a push step to your `/handoff` command to auto-sync chat messages
- Ask Claude to "push this to HandoffKit" anytime
- The `push-handoff` tool sends only the concise chat message, never full handoff files

## Multiple projects

Run `/handoffkit:setup` in each project directory with a token generated for that HandoffKit project. Each directory gets its own isolated connection.

## Security

- Setup tokens expire in 10 minutes and are single-use
- API keys are project-scoped — can't access other projects
- Keys are stored by Claude Code in `~/.claude.json`, not in project files
- Revoke keys anytime at handoffkit.com/settings
- All communication is HTTPS

## Requirements

- A [HandoffKit](https://handoffkit.com) account (free tier works)
- Claude Code with MCP support
- `curl` available in your shell

## Links

- [HandoffKit](https://handoffkit.com)
- [Setup Guide](https://handoffkit.com/guides/plugin-setup)
- [Handoff Template](https://handoffkit.com/guides/handoff-template)
