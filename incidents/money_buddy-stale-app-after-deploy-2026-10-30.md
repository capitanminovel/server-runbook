# money_buddy: phone kept showing the old app after a deploy (2026-10-30)

## What happened
A new version was deployed and the server was serving the new `app.js`, but the installed app on the phone still showed the old buttons after a refresh.

## Why
- FastAPI's `StaticFiles` sends `ETag` and `Last-Modified` but **no `Cache-Control`**.
- Without `Cache-Control`, browsers use *heuristic freshness*: a file can be reused without asking the server for about 10% of the time since it last changed. The old `app.js` was about a day old, so the phone could reuse it for hours.
- The service worker did `fetch(event.request)`, which goes through the same HTTP cache.

## Fix (commit after `78932e7`)
- `main.py` middleware sets `Cache-Control: no-cache` on every `/static/*` response. `no-cache` does **not** mean "don't store". It means "check with the server before using the stored copy". Unchanged files come back as a cheap `304 Not Modified` thanks to the ETag.
- `static/sw.js`: non-navigation requests use `fetch(req, { cache: "no-cache" })`. This gets the fix to phones that still hold old cached files without the header. It works because browsers check `sw.js` for updates on navigation even when it's cached, and the worker uses `skipWaiting` and `clients.claim`.
- Navigations pass through untouched. Rebuilding a navigate request changes its redirect handling, which could break the login redirect.

## Verify
```bash
curl -sI https://money.withcapitan.com/static/app.js | grep -i cache-control   # no-cache
curl -s -o /dev/null -w "%{http_code}\n" -H 'If-None-Match: "<etag>"' https://money.withcapitan.com/static/app.js  # 304
```

## What I learned
- No `Cache-Control` doesn't mean "not cached". It means "browser guesses".
- `no-cache` = revalidate every time. `no-store` = never keep. For app code you want `no-cache`.
- A service worker that "caches nothing" can still serve stale files through the normal HTTP cache.

## Follow-up
- None. If performance ever matters, put version hashes in the file names and cache them forever instead.
