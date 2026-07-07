# 2026-07-07 — Exposed GitHub PAT Found and Rotated to SSH

## What happened
While checking which GitHub repo `money_buddy` lives under, ran `git remote -v` and found the origin URL had a classic GitHub personal access token embedded directly in it (`https://ghp_...@github.com/...`). The same token turned out to be reused across all three git-tracked projects on this server: `money_buddy` (`Transaction_Tracker`), `capitans_terps` (`Capitans-Terps`), and `server-runbook`.

## Why
Embedding a PAT in a remote URL puts the credential in plaintext in `.git/config` on disk, in shell history, and in the output of any command (or AI session) that inspects the remote. Because the same token was reused for all three repos, a single leak point compromised push/pull access to everything. Classic PATs are also long-lived and broad-scoped by default (this one had full `repo` access), unlike fine-grained tokens or deploy keys.

## Fix
1. Found an SSH keypair already existed at `~/.ssh/github` and was already registered on the GitHub account (added 2026-06-21), so no new key generation was needed.
2. Switched all three remotes from the token-embedded HTTPS URL to SSH:
   ```bash
   git remote set-url origin git@github.com:capitanminovel/Transaction_Tracker.git   # money_buddy
   git remote set-url origin git@github.com:capitanminovel/server-runbook.git
   git remote set-url origin git@github.com:capitanminovel/Capitans-Terps.git         # capitans_terps
   ```
3. Verified SSH auth worked with a plain `git fetch` on each (read-only, no data changed).
4. User manually revoked the leaked classic PAT via GitHub Settings → Developer settings → Personal access tokens (classic).
5. Confirmed dead by testing the exact leaked token against the API:
   ```bash
   curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: token <token>" https://api.github.com/user
   ```
   First check returned `200` (still live — user had deleted other tokens but missed this one). Second check after re-deleting returned `401` (confirmed revoked).

## What I learned
- Classic PATs give no way to introspect their own name/ID from the token itself via API — you can only tell it's "the right one" by testing whether it still authenticates, not by matching it to a row in the GitHub UI.
- `git remote -v` prints credentials in plaintext if they're embedded in the URL — worth grep-checking all repos on a server for this pattern (`grep -r 'ghp_' /opt/*/.git/config /root/*/.git/config`) rather than assuming it's isolated to one repo.

## Follow-up
- The `gh` CLI on this server is authenticated with a *separate* token that has near-full account scope (`admin:org`, `admin:enterprise`, `delete_repo`, etc.) — far broader than needed for git operations. Not addressed in this session; worth revisiting to scope it down or replace with a narrower token.
- Consider using per-repo GitHub deploy keys instead of one account-wide SSH key, so a compromise of one repo's key doesn't imply access to the others.
