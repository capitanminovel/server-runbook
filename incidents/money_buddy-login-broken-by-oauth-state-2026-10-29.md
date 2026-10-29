# money_buddy: login broken by the OAuth state check (2026-10-29)

## What happened
After the login hardening (commit `e0210aa`), signing in on the phone failed with
"Login expired or invalid — please try again". The logs show one attempt: `/auth/login` → 307,
then 5 seconds later `/auth/callback?state=…` → 400.

## Why
- The new check stored a random `state` in an `oauth_state` cookie and required the callback to send it back.
- The server set the cookie correctly (`HttpOnly; Secure; SameSite=lax`), but the phone's callback request didn't include it.
- Most likely cause: the login started in the installed app, and Google returned through a browser context with a different cookie jar (Samsung Internet vs Chrome, or the app's sign-in tab). Not confirmed.

## Fix
Removed the state cookie and its check (commit `c50e807`). These stay: TLS verification, no `access_type=offline`, the `Secure` session cookie.

## What I learned
- Login CSRF needs the app to *accept* the attacker's account. `ALLOWED_EMAILS` already rejects outsiders, so the check added little here.
- Cookie-based checks across an app → Google → app round trip can break when the phone hands off between browsers. Test real logins on the actual device before calling auth work done.

## Follow-up
- Confirm phone login works after the fix.
