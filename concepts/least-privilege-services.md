# Least privilege for a web service (why dispo_menu no longer runs as root)

## What it is
A service should run with only the permissions it needs. Until 2026-09-25 the dispo_menu API ran as `root` (a systemd
unit with no `User=` line runs as root), straight from the git working copy in `/root`. Any bug that let an attacker run
code — e.g. in the new file-upload feature — would have had full control of the whole server, including the other apps.

## How it works
Three layers, each independent:
1. **An unprivileged user.** `dispo` (system user, no password, no login shell). It owns only `/var/lib/dispo-menu`.
2. **Read-only code.** The service runs a *deployed copy* in `/opt/dispo-menu` owned by `root:dispo` with mode `750`/`640`:
   the service can read its code and `.env` but cannot change them. So a compromised app can't plant a backdoor in itself.
3. **systemd sandboxing.** Kernel-enforced limits declared in the unit file: `ProtectSystem=strict` (whole filesystem
   read-only except `ReadWritePaths`), `ProtectHome` (no `/root`, `/home`), `PrivateTmp`, `NoNewPrivileges` (can't gain
   privileges via setuid programs), empty `CapabilityBoundingSet` (no root capabilities at all), kernel/clock/cgroup
   protections, restricted socket types. `systemd-analyze security dispo-menu-api` scores it **4.2 OK** (a default root
   unit scores ~9.6 UNSAFE; lower is better).

## How it's set up on this server
| What | Where | Owner / mode |
|---|---|---|
| Deployed API code + `.env` | `/opt/dispo-menu/api` | root:dispo, 750/640 (read-only to the service) |
| Python venv | `/opt/dispo-menu/venv` | root:dispo |
| Headless Chromium for Sweed | `/opt/dispo-menu/ms-playwright` (`PLAYWRIGHT_BROWSERS_PATH`) | root:dispo — copied from `/root/.cache`, the download is geo-blocked here |
| Writable data (training uploads, browser profile) | `/var/lib/dispo-menu` | dispo:dispo, 700 |
| Working copy you edit | `/root/Dispo_menu` | root — **no longer what runs** |
| Unit file | `/etc/systemd/system/dispo-menu-api.service` (copy in this repo: `setup/dispo-menu-api.service.copy`) | |

**Workflow change:** editing files in `/root/Dispo_menu` does nothing until you deploy:
`/root/Dispo_menu/deploy.sh` (API + admin), `deploy.sh api`, or `deploy.sh admin`. It rsyncs the code, fixes ownership,
installs requirements if they changed, runs migrations *as dispo*, restarts, and waits for the API.
The test scripts (`apps/api/scripts/*.py`) still run from the working copy with its own venv.

Also changed: the session cookie is now `Secure` (only sent over HTTPS), `HttpOnly`, `SameSite=Lax`, 12-hour lifetime
(was 14 days, not Secure). Local http development: set `SESSION_COOKIE_SECURE=false` in a local `.env`.

## What it does NOT do
- It doesn't stop a bug *inside* the app from misusing what the app legitimately can do (read the DB, read uploads).
- The database user still has full rights on the dispo_menu database (a separate, later hardening step).
- `.env` secrets are readable by the service (it needs them); they're just not writable.

## Key commands
```bash
systemd-analyze security dispo-menu-api          # exposure score + which protections are on
ps -o user= -p $(systemctl show -p MainPID --value dispo-menu-api)   # -> dispo
/root/Dispo_menu/deploy.sh api                   # push working-copy changes live
journalctl -u dispo-menu-api -f                  # logs (look for "Permission denied" after a change)
runuser -u dispo -- <cmd>                        # test something with the service's permissions
```
Rollback: the previous unit is in `/root/backups/dispo-menu-api.service.<date>`; restore it, `systemctl daemon-reload`,
restart (it runs the old `/root` venv, which still exists).
