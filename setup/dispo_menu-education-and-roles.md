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
