# Legit Buddy — Initial Server Deployment

**Date:** 2026-06-21

## What this is
Legit Buddy is a cannabis menu and staff tool for MN Legit Cannabis South Metro. It was previously hosted on GitHub Pages (public repo). Moved to the DigitalOcean droplet as a private repo to enable:
- Real API calls instead of Playwright fallback
- Private content (repo is now private)
- Server-side cron job instead of GitHub Actions
- Future multi-store expansion (owner wants the same system for a second dispensary — this upgrade is the demo)

**Repo:** https://github.com/capitanminovel/legit-buddy-private  
**Path on server:** `/opt/legit-buddy-private/`

## How it's set up

Same pattern as money_buddy and capitans_terps — FastAPI app behind nginx.

| Thing | Detail |
|---|---|
| Port | 8002 (internal only) |
| Service | `legit-buddy` (systemd) |
| Nginx | Proxies `http://167.99.235.92` → port 8002 |
| Venv | `/opt/legit-buddy-private/venv/` |
| Env file | `/opt/legit-buddy-private/.env` |
| Domain | None yet — IP only for development |

## Files created

**Systemd service** — `/etc/systemd/system/legit-buddy.service`:
```ini
[Unit]
Description=Legit Buddy FastAPI App
After=network.target

[Service]
WorkingDirectory=/opt/legit-buddy-private
EnvironmentFile=/opt/legit-buddy-private/.env
ExecStart=/opt/legit-buddy-private/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8002
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

**Nginx config** — `/etc/nginx/sites-available/legit-buddy`:
```nginx
server {
    listen 80;
    server_name 167.99.235.92;

    location / {
        proxy_pass http://127.0.0.1:8002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Symlinked to sites-enabled:
```bash
ln -s /etc/nginx/sites-available/legit-buddy /etc/nginx/sites-enabled/legit-buddy
```

## Setup commands (for reference / rebuilding)
```bash
cd /opt/legit-buddy-private
python3 -m venv venv
venv/bin/pip install -r requirements.txt

# Add real API key to .env
echo "ANTHROPIC_API_KEY=your_key_here" > .env

systemctl daemon-reload
systemctl enable legit-buddy
systemctl start legit-buddy
systemctl reload nginx
```

## Useful commands
```bash
systemctl status legit-buddy
journalctl -u legit-buddy -f        # live logs
systemctl restart legit-buddy       # after code changes
curl http://127.0.0.1:8002/         # test app directly
```

## URLs (while on IP-only)
- Menu UI: http://167.99.235.92/menu
- API root: http://167.99.235.92/
- API docs (Swagger): http://167.99.235.92/docs

## What was added to requirements.txt
`anthropic` and `python-docx` were missing from the original file — both are used by `enrich_strains.py` and `build_preview.py`. Added them before creating the venv.

## Follow-up
- [ ] Add real `ANTHROPIC_API_KEY` to `.env` when available
- [ ] Set up cron job to replace GitHub Actions daily scrape
- [ ] Pick a domain and add nginx vhost + SSL cert (certbot)
- [ ] Upgrade work: UI redesign, strain profile improvements, schedule image upload
