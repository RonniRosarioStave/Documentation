# Deploy — docs-dev.stavecorp.com with Entra ID SSO

This directory holds the **reviewable** infrastructure for hosting the docsify site on
a self-managed Ubuntu box behind **nginx + oauth2-proxy**. The site's static files are
served by nginx; `/employee/` is gated by an Entra ID (Azure AD) OIDC login, while the
landing page and `/client/` stay public.

```
Browser --443/TLS--> nginx (host) --/, /client/ public--> static files
                          |
                          +--/employee/ : auth_request--> oauth2-proxy (127.0.0.1:4180) --OIDC--> Entra ID
```

## Files here

| File | Purpose |
|------|---------|
| `nginx.conf` | Server block: TLS, static files, `auth_request` gate on `^~ /employee/`, `/oauth2/*` proxy. |
| `oauth2-proxy.cfg.example` | Template for the oauth2-proxy config. **Copy to `oauth2-proxy.cfg` on the server and fill in real values.** |
| `docker-compose.yml` | Runs oauth2-proxy bound to `127.0.0.1:4180`. |
| `templates/error.html` | Branded "Access Denied" page oauth2-proxy renders instead of its default error page (see below). |

> **Secrets never live in the repo.** The real `oauth2-proxy.cfg` and any `.env` are
> git-ignored (see the root `.gitignore`). Client secret + cookie secret exist only on
> the server.

## Why the repo is split into two docsify roots

Docsify is client-side and its search plugin fetches every sidebar-listed page into the
browser. A single shared root would make a public visitor's browser fetch the internal
`employee/` markdown to build its search index. The site therefore has **two docsify
roots**:

- **Public root** (`/index.html`, `/_sidebar.md`) — links only to `client/`.
- **Protected root** (`/employee/index.html`, `/employee/_sidebar.md`) — the internal
  docs; reachable only after login.

The cover page's "Employee Access" is a real `<a target="_self">` navigation to
`/employee/` (not a docsify hash route), so an unauthenticated visit produces a
top-level 302 to Microsoft login instead of a silent AJAX failure.

---

## Custom "Access Denied" page

`templates/error.html` replaces oauth2-proxy's default error page with a branded one.
It only fires for pages oauth2-proxy renders itself through the full reverse proxy path
(`/oauth2/callback`, `/oauth2/sign_in`, etc.) — most importantly the 403 shown when
Microsoft login succeeds but the chosen account isn't `@stavecorp.com`. It does **not**
affect nginx's `auth_request /oauth2/auth` check on `^~ /employee/`: that lightweight
endpoint returns a bare 401/403 with no body by design (nginx only reads the status code
from it), and its 401 is what triggers the existing `error_page 401 = @sign_in` redirect
to Microsoft.

The page offers two actions:
- **Try another account** — posts to `/oauth2/sign_out` (clears the oauth2-proxy session
  cookie only, not the underlying Microsoft/Entra session) and returns to the page the
  user originally requested, then falls into the existing `prompt=select_account` flow.
- **Return to documentation** — links back to `/`, the public docs root.

**Wiring (server-only step):** `custom_templates_dir = "/templates"` is documented in
`oauth2-proxy.cfg.example`, but the real `oauth2-proxy.cfg` is git-ignored (secrets), so
add that same line to it by hand on the server, then `docker compose up -d` to pick up
both the new config line and the `deploy/templates/` volume mount in
`docker-compose.yml`.

## Part B — Entra ID app registration (Entra admin)

1. **App registrations → New registration**
   - Name: `docs-dev.stavecorp.com employee docs`
   - Supported account types: **Single tenant**
   - Redirect URI (Web): `https://docs-dev.stavecorp.com/oauth2/callback`
2. Record **Application (client) ID** and **Directory (tenant) ID**.
3. **Certificates & secrets → New client secret** → record the value (server only).
4. **Token configuration → Add optional claim → ID token → `email`.**
   *(Critical — Entra v2 tokens often omit `email`; without it oauth2-proxy rejects
   everyone. Fallback: use `oidc_email_claim = "preferred_username"`.)*
5. **API permissions**: `openid`, `profile`, `email` (delegated, Microsoft Graph) —
   grant admin consent.
6. Attach a **Conditional Access** policy for MFA / device / location rules.

## Part C — Ubuntu server bring-up

**Prereqs:** Docker + docker-compose, nginx, certbot, and DNS A/AAAA
`docs-dev.stavecorp.com` → server IP.

1. **Static files** → `/var/www/docs` (document root = `/var/www/docs/productdocsify`).
   A `git pull` release hook is enough for a docs site.
2. **oauth2-proxy**: copy `oauth2-proxy.cfg.example` → `oauth2-proxy.cfg`, fill in the
   real Entra values + a fresh `cookie_secret`
   (`openssl rand -base64 32 | tr '+/' '-_'`), then `docker compose up -d`.
   Verify: `docker compose logs oauth2-proxy` shows OIDC discovery success;
   `curl -I http://127.0.0.1:4180/ping` → 200.
   *(Gotcha: `http_address` must stay `0.0.0.0:4180` inside the container — Docker's
   port forwarding targets the container's bridge interface, not its loopback, so
   binding to `127.0.0.1` there makes the app unreachable even though the compose port
   mapping already restricts host exposure to loopback.)*
3. **TLS (webroot, keeps `nginx.conf` as source of truth)**: bootstrap a temporary
   HTTP-only site (`listen 80`, `root` = the docs root) so certbot's webroot challenge
   has something to serve from, then:
   `certbot certonly --webroot -w /var/www/docs/productdocsify -d docs-dev.stavecorp.com --deploy-hook "systemctl reload nginx"`
   (the `--deploy-hook` matters here — without it, unlike `certbot --nginx`, a renewed
   cert won't trigger an nginx reload). The cert lands at
   `/etc/letsencrypt/live/docs-dev.stavecorp.com/{fullchain,privkey}.pem`. Avoid
   `certbot --nginx`: it edits the live nginx config directly and will drift from this
   checked-in `nginx.conf`.
4. **nginx**: install `nginx.conf` as a site — uncomment the two `ssl_certificate`
   lines first (they're commented out in git since the cert doesn't exist until step 3)
   — replacing the temporary bootstrap site, then `nginx -t && systemctl reload nginx`.

## Verification

| Check | Expected |
|-------|----------|
| `curl -I https://docs-dev.stavecorp.com/` and `.../client/data-tools/README.md` | **200**, no redirect |
| `curl -I https://docs-dev.stavecorp.com/employee/` and `.../employee/data-tools/README.md` | **302** → `login.microsoftonline.com` |
| Browser: visit `/employee/` | Microsoft login → `@stavecorp.com` (+ MFA) → internal docs with sidebar + search |
| Sign in with a non-`stavecorp.com` account | oauth2-proxy denies (403) with the branded "Access Denied" page, not the default oauth2-proxy page |
| On the "Access Denied" page, click "Try another account" | Session cookie clears, Microsoft shows the account picker (`prompt=select_account`) again |
| On the "Access Denied" page, click "Return to documentation" | Lands on `/` (public docs root) |
| Public search for a string that exists only in an `employee/` page | **no result** (search indexes are isolated) |

## Rollout & rollback

- **Cutover**: validate on a temporary hostname or `/etc/hosts` override first; flip
  `docs-dev.stavecorp.com` DNS from GitHub Pages to the server IP only after the checks
  pass.
- **Rollback**: revert DNS to GitHub Pages. The two-root split is
  static-site-compatible, so the site still renders on Pages (just without auth) — a
  clean, instant fallback.
