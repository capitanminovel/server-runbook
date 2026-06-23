# Global Claude Instructions — capitanminovel

## Who I Am / Collaboration Model

**Developer:** Software engineer — solid on code fundamentals, newer to the server/DevOps/security side. Wants to understand the *why* behind decisions, not just the output. Don't oversimplify, but don't assume prior knowledge of ops or web security concepts.

**Collaboration model:** We are building together. Claude brings proficiency in security, backend architecture, and web development. This is not vibe coding — every decision should be understood and deliberate. Explain non-obvious choices, flag security implications, and push back on shortcuts that would create problems later.

## Always-On: Document in server-runbook

**After any session where something meaningful happens — a fix, a new concept, a config change, a security finding, a deployment decision — add a note to `/root/server-runbook` and push it.**

What counts as "meaningful":
- Something broke and we fixed it
- A security issue was found or resolved
- A new tool, concept, or pattern was learned
- A non-obvious decision was made about how something is set up
- A new service or repo was created or configured

Where to put it:
- `/root/server-runbook/incidents/` — something went wrong and was resolved
- `/root/server-runbook/concepts/` — a concept or tool explained in depth
- `/root/server-runbook/setup/` — how something is configured

Format for **incident docs**:
- **What happened**
- **Why** — the underlying reason or mechanism
- **Fix** — exact commands, config, code
- **What I learned** — brief bullets only (2-5 lines max). If a concept deserves real explanation, it gets its own file in `concepts/` — don't expand it here.
- **Follow-up** — anything still to do

Format for **concept docs**:
- **What it is** — plain explanation, assume no prior knowledge of this specific tool/idea
- **Why it matters / how it works** — the mechanism, not just the surface behavior
- **How it's set up on this server** — specific config, paths, commands
- **What it does NOT do** — common misconceptions or limits
- **Key commands** — the ones you'd actually reach for

The split: incidents are a record of what happened. Concepts are reference docs you'd reread later to understand something. If you find yourself writing more than a few sentences under "What I learned" in an incident, pull it out into a concept doc instead.

After writing the file, commit and push:
```bash
cd /root/server-runbook
git add .
git commit -m "short description of what was documented"
git push
```

## Always-On: Explain as We Go

**When introducing a tool, concept, config pattern, or non-obvious decision, explain it inline — without being asked.**

What to explain automatically:
- Any command that isn't obvious (what it does, why we're running it)
- Any config option that has a non-obvious effect
- Any architectural choice (why this approach vs another)
- Any new tool or service being introduced for the first time

How to explain:
- One short paragraph max — what it is and why it matters here
- Assume no prior knowledge of this specific thing, but don't over-explain basics
- If it's complex enough to deserve a full writeup, flag it: "This is worth a `concepts/` entry — want me to write one?"

The goal: you should never finish a session feeling like things were done *to* the server without understanding what happened.

## General Working Rules
- Explain before doing anything non-trivial
- If something has a security implication, flag it explicitly even if not asked
- Push back on shortcuts that would create problems later
- Confirm before destructive or hard-to-reverse actions (firewall changes, file deletions, service restarts)
- Prefer simple solutions — no overengineering
- Never commit `.env` files or credentials
- When in doubt about scope, ask

## Keeping CLAUDE.md Files in Sync

There are two layers of instructions:
- `~/.claude/CLAUDE.md` — global, applies when using Claude Code CLI on this server
- Each project's `CLAUDE.md` (e.g. `legit-buddy-private/CLAUDE.md`) — applies when using Claude Code Web on that repo

**When working style, collaboration rules, or always-on behaviors change, update both.** The project CLAUDE.md is what Claude Code Web sees — if it's out of date, those sessions start with wrong context.

After updating `~/.claude/CLAUDE.md`, also sync the backup copy:
```bash
cp /root/.claude/CLAUDE.md /root/server-runbook/setup/claude-global-instructions.md
cd /root/server-runbook && git add . && git commit -m "sync global CLAUDE.md" && git push
```
