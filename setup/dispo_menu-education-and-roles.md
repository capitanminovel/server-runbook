# dispo_menu — Education tile and user roles (built 2026-09-25)

## Roles
| | Admin | Employee |
|---|---|---|
| Strain list, open a strain, copy text | yes | yes |
| Generate / redo / edit / link / archive / delete strains, Refresh live menu | yes | **no** |
| Brands, research sites (Sources tab) | yes | **no** (can see) |
| Trainings: view, open files, watch video | yes | yes |
| Trainings: add, edit, delete, reorder, sections | yes | **no** |
| Team page (logins) | yes | **no** |
- Enforced on the server: `require_admin` (`app/auth/session.py`) on every write/paid route -> 403 for employees even via direct
  API calls. The UI hiding buttons is only convenience. Verified: 10 employee attempts refused, reads allowed.
- **Accounts (decided): one shared employee login for now, individual logins later** — same Team page either way.
- **Team page** (Dashboard -> Team): add a login (role, email, password; a readable one is suggested), change password, remove.
  Changing a password bumps `staff_users.session_version`: every browser using that login is signed out at once. **When someone
  leaves and the login is shared, change its password.** Can't remove yourself or the last admin. Passwords min 10 characters.
- Login cookie: HttpOnly, Secure (HTTPS only), SameSite=Lax, 12 h. Login is looked up by email alone, so emails are unique app-wide.

## Education tile
- **Sections** = the tabs (create in the training form or rename/delete on the tab; deleting keeps its trainings -> "No section").
- **Training** = name, section, description (**at least 2 sentences**, checked on the server too), tags (fixed checkboxes: New Hire,
  Pinned, Need to Read — Pinned floats to the front), optional **YouTube/Vimeo** link, up to **10 files**.
- Cards: bold name, description under it, tag chips; ✎ pencil opens the editor (delete is inside, with a confirm); **drag cards or
  section tabs to rearrange** (desktop mouse — HTML5 drag-and-drop doesn't work on phones; the order is saved).
- Viewer: **PDF** in the page (browser's own viewer: Chrome/Edge = PDFium, Firefox = PDF.js), **images** and **text** in the page,
  **Word/PowerPoint/Excel** = Download (decided: no server conversion — LibreOffice needs more RAM than this droplet has; tip shown
  to export as PDF for in-app viewing). Video: `youtube-nocookie.com` (privacy mode) / `player.vimeo.com?dnt=1`, sandboxed iframe.
  Known limit: iPhone Safari shows only the first page of a PDF inside a page — Download works; PDF.js (pdfjs-dist, Apache-2.0)
  can be added later if staff use phones.

## Upload security (details in `apps/api/app/education/files.py`)
- Allow-list: pdf, jpg/jpeg, png, gif, webp, txt, csv, md, docx, pptx, xlsx, doc, ppt, xls. **Checked by content** ("magic
  bytes"), not the name: an HTML page renamed `.pdf` is refused. **Never accepted:** HTML, SVG (can carry JavaScript), macro files
  (.docm/.pptm/.xlsm, or any Office file containing `vbaProject`), executables. Office zips are listed (never extracted) with
  zip-bomb limits.
- 25 MB per file (streamed, stopped at the limit), 2 GB per dispensary (`UPLOAD_QUOTA_MB`), nginx allows 30 MB only on
  `/api/education/trainings/<id>/files` with its own rate limit (`dispo_upload`, 20/min). Elsewhere nginx's 1 MB default stays.
- Stored in `/var/lib/dispo-menu/uploads/<dispensary_id>/<32 random hex>.<ext>`, mode 600, owner `dispo`. Original filename is a
  label only (a `../../etc/...` name is stripped to `evil.pdf`). Deleting a training/file deletes the file from disk.
- Served only by `GET /api/education/files/<id>` to logged-in staff of the **same dispensary** (another dispensary gets 404), with
  our own MIME type, `X-Content-Type-Options: nosniff`, `Content-Security-Policy: default-src 'none' …`, Office as a download.
- No antivirus: ClamAV needs ~1 GB RAM. Accepted because only admins upload and nothing is executed on the server.
- Tested: 23 attack checks (renamed HTML, SVG script, fake/macro PowerPoint, 26 MB file, path-trick name, employee upload,
  cross-dispensary read/edit) + 23 browser checks — all pass. `scripts/education_check.py` reruns the browser part free.

## Storage — decided: this server's disk for now
~9 GB free after the /opt copy. Rough capacity: training PDFs 1–5 MB, decks 5–20 MB. At ~5 MB average, 1 GB ≈ 200 files.
Later options (not decided): **DigitalOcean Spaces** (S3-compatible, ~$5/mo for 250 GB + 1 TB transfer — verify current pricing;
one Space can hold every dispensary in its own folder, our app keeps doing the permission check), or **Google Drive** via a
service account (works, but files are served through our app anyway to keep them private, Drive API quotas apply, and
"anyone with the link" sharing must never be used). Storage code is in one module (`education/files.py`) so switching is contained.

## Dependency security
`pip-audit` found 14 known vulnerabilities in Starlette 0.41.3 (under FastAPI) -> upgraded to FastAPI 0.141.1 / Starlette 1.7.0
(pinned in requirements.txt), re-audit clean; `npm audit` on the admin app: 0. Re-run both before each release:
`python3 -m venv /tmp/a && /tmp/a/bin/pip install pip-audit && /tmp/a/bin/pip-audit -r apps/api/requirements.txt`; `npm audit --omit=dev`.

## Look and feel (2026-09-25/26)
Brand taken from mnlegitcannabis.com's own stylesheet: action pink `#ea75b3` (buttons/bands, always with dark text — white on
that pink fails contrast), black/white, headings **Cormorant Garamond**, text **Inter** (both open-license, bundled via
@fontsource — no requests to Google; their script font Sign Painter is a paid license, not used). Login + dashboard: the store's
mural (web copy 324 KB of a 5.4 MB original) between pink bands (`?band=black` previews black bands). Inner pages: pink band with
logo (→ dashboard), quick links, user + Log out, on a light off-white page so forms and lists stay readable. Green is kept only
for the Active status badge (it carries meaning). Colours live as CSS variables in `apps/admin/src/index.css` (`--brand-*`).

## Viewing PDFs and video (2026-09-26)
- **PDFs are drawn by PDF.js** (`pdfjs-dist`, Apache-2.0, Mozilla's open-source reader), not the browser's built-in viewer. An `<iframe>` of a PDF is blank or download-only on iPhone and most Android phones; PDF.js draws each page onto a `<canvas>` so it looks the same everywhere. Pages render lazily as you scroll.
- **The work happens in each viewer's browser.** The server only hands over the file (auth-checked, same as before). 10 people reading at once ≈ 10 file downloads (10 × 3.7 MB ≈ 37 MB), versus about 1 TB/month of transfer included with the droplet. No server upgrade needed and no extra cost. It's free software we ship, not a paid service.
- **Video never touches our server.** YouTube and Vimeo stream it; we only embed their player.
- **nginx gotcha:** the PDF.js worker is a `.mjs` file. nginx's default `mime.types` doesn't know `.mjs`, so it sent `application/octet-stream` and browsers refused to run it as a module ("This PDF could not be shown here"). Fixed in `/etc/nginx/sites-available/dispo-admin.dev.withcapitan.com` (backup in `/root/backups`):
  ```nginx
  location ~* \.mjs$ {
      types { }
      default_type text/javascript;
  }
  ```
  Check: `curl -sI -u dispo:… https://dispo-admin.dev.withcapitan.com/assets/<pdf.worker…>.mjs | grep -i content-type`
- Fonts/cmaps/wasm for PDF.js are copied into the build by `apps/admin/scripts/copy-pdfjs-assets.mjs` (the `prebuild` step), so nothing loads from a CDN.
- **2 GB quota in practice:** ~550 PDFs the size of the 3.7 MB "Concentrates" test. Realistic mixes (PDFs 1–5 MB, slide decks 5–20 MB, photos <1 MB) land around 400–2,000 files. That's plenty for one dispensary's trainings. The quota is `UPLOAD_QUOTA_MB` in `.env`. The disk has about 9 GB free, so when several dispensaries fill up, the move is DigitalOcean Spaces (object storage).
