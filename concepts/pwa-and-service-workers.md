# PWAs (installable web apps) and service workers

## What it is
A **Progressive Web App (PWA)** is a normal website that the phone lets you "install."
- It gets a home-screen icon, opens full-screen with no browser address bar, and shows up in the app switcher like a native app.
- It's still the same site loading from the server, so every deploy updates it automatically, with no app store involved.

## Why it matters / how it works
The browser offers "Install app" when three things are true:
1. **HTTPS.** Ours comes from Certbot / Let's Encrypt.
2. **A web app manifest.** This is a JSON file linked from each page with `<link rel="manifest">`. It gives the name, icons (192 px and 512 px, plus a *maskable* icon whose artwork stays inside a safe zone so Android can crop it into a circle or squircle), `start_url`, `display: standalone` (no browser UI) and colors.
3. **A service worker.** This is a JavaScript file the browser runs in the background, separate from the page. It sits between the app and the network: every request the page makes passes through its `fetch` handler. That's what makes offline apps possible, because it can answer requests from a cache.

On Android, Chrome turns the installed app into a "WebAPK," a real Android package that wraps the site. It shares Chrome's cookies, so you stay logged in.

## How it's set up on this server (money_buddy)
- **Manifest:** `/opt/money_buddy/static/manifest.webmanifest`. `scope: "/"` so the login redirects stay inside the app.
- **Icons:** `/opt/money_buddy/static/icons/` (rendered from SVG with headless Chromium).
- **Service worker:** `/opt/money_buddy/static/sw.js`, registered in `static/common.js`.
- **The service worker caches nothing on purpose.** Its fetch handler just passes every request to the network. It's a finance app, so no transaction data should sit in an extra on-device cache, and stale numbers would be worse than "no connection."
- **To install:** open https://money.withcapitan.com in Chrome or Samsung Internet, then menu → **Install app** / **Add to Home screen**.

## What it does NOT do
- It does **not** work offline. That would need caching, which we deliberately skipped.
- It's **not** more secure than the website; it *is* the website. Login, cookies and HTTPS are unchanged.
- The service worker's **scope** (what pages it controls) is the folder it's served from: `/static/` for us. The manifest `scope` is a separate setting that only decides which URLs count as "inside the app."
- **Updating:** changes to `sw.js` take effect on the next app launch. Our worker calls `skipWaiting()` and `clients.claim()`, so the new version takes over right away instead of waiting for every tab to close.

## Key commands
```bash
curl -sI https://money.withcapitan.com/static/manifest.webmanifest   # must be 200
curl -s  https://money.withcapitan.com/static/sw.js                  # served publicly (no data in it)
```
In desktop Chrome: DevTools → **Application** → Manifest / Service Workers shows install errors.
To remove the app from the phone: long-press the icon → Uninstall.
