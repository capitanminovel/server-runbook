# CLAUDE.md Setup — Always-On Documentation

## What This Is

A layered CLAUDE.md setup so Claude Code automatically has server context and documentation instructions in every session — whether working on the server directly or inside a specific repo.

## How It Works

Claude Code reads CLAUDE.md files in order and stacks them all:

1. `~/.claude/CLAUDE.md` — global, applies to every session everywhere
2. `/root/CLAUDE.md` — applies to any session on this server
3. `/opt/<app>/CLAUDE.md` — applies when working inside that specific app

All three are active at once when working inside an app on this server.

## Files Created

### `~/.claude/CLAUDE.md` (global)
- Who I am and how I prefer to work
- **The always-on documentation rule** — after any meaningful session, write it up here and push
- General working rules (explain before doing, confirm before destructive actions, no overengineering)

### `/root/CLAUDE.md` (server-wide)
- Server overview — what's running, what ports, what's restricted
- Key file paths and useful commands
- Pointer to this runbook

### `/opt/money_buddy/CLAUDE.md` (already existed — project-specific)
- What the app does, tech stack, build stages, working rules

### `/opt/capitans_terps/CLAUDE.md` (created)
- What the app does, tech stack, project structure, how to run/restart

## Why This Matters

Without this setup, every new Claude session starts cold — no context about the server, no reminder to document things. With CLAUDE.md in place, Claude always knows:
- What's running on this server and why
- That the runbook exists and should be updated
- How to work in each specific project

## Runbook Location
- **Local:** `/root/server-runbook/`
- **GitHub:** https://github.com/capitanminovel/server-runbook
