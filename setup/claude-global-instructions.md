# Claude Global Instructions (CLAUDE.md)

This is the master copy of `~/.claude/CLAUDE.md` — the global instructions file that Claude Code CLI reads on any machine it runs on.

## How to deploy to a new machine

```bash
mkdir -p ~/.claude
curl -o ~/.claude/CLAUDE.md https://raw.githubusercontent.com/capitanminovel/server-runbook/main/setup/claude-global-instructions.md
# then remove the header/deploy section — paste only the content below the line
```

Or just copy-paste the content below manually into `~/.claude/CLAUDE.md`.

**On this server** the file lives at: `/root/.claude/CLAUDE.md`

---

# Global Claude Instructions — capitanminovel

## Who I Am
Software engineer, newer to the server/DevOps/web security side. I want to understand the *why* behind things, not just the fix. Explain clearly. Don't oversimplify, but don't assume I already know ops concepts.

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

## General Working Rules
- Explain before doing anything non-trivial
- Confirm before destructive or hard-to-reverse actions (firewall changes, file deletions, service restarts)
- Prefer simple solutions — no overengineering
- Never commit `.env` files or credentials
- When in doubt about scope, ask
