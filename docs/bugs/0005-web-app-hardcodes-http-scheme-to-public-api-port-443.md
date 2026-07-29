# 0005. Sign-up/login broken when `WEB_HOSTNAME_API`/`WEB_PORT_API` point at the public HTTPS API domain

- **Status:** Fixed
- **Severity:** High
- **Reported:** 2026-07-28
- **Resolved:** 2026-07-28

## Symptom

Sign-up and login fail. Browser network tab shows the web app's own
`/api/proxy/api/user/email` and `/api/proxy/.../auth/login` routes returning:

```
400 Bad Request
The plain HTTP request was sent to HTTPS port
```

(nginx's standard response when it receives non-TLS bytes on an `ssl` listener.)

`docker compose logs kodus-web` confirms the outgoing request itself is malformed:

```
url: 'http://api.kodus.askelie.in:443/auth/login'
...
protocol: 'http:',
host: 'api.kodus.askelie.in',
```

## Expected

Following the official reverse-proxy setup (`WEB_HOSTNAME_API=api.<domain>`,
`WEB_PORT_API=443` for an HTTPS deployment, per docs.kodus.io's reverse-proxy guide)
should let the web app reach the API successfully.

## Reproduction

1. Deploy behind a reverse proxy with the API on its own HTTPS subdomain, e.g.
   `api.kodus.askelie.in` → nginx (443, real TLS cert) → `api` container (3001).
2. Set, per the official docs:
   ```
   WEB_HOSTNAME_API=api.kodus.askelie.in
   WEB_PORT_API=443
   ```
3. Recreate `kodus-web`, then attempt sign-up or login in the browser.
4. → fails with the 400 above; `kodus-web`'s internal proxy route sent a **plain
   HTTP** request to nginx's TLS-only 443 listener for `api.kodus.askelie.in`.

## Impact

Breaks all authentication (sign-up, login) — a full-stop blocker for first use —
whenever `WEB_HOSTNAME_API`/`WEB_PORT_API` point at the public HTTPS domain, which is
exactly what the official reverse-proxy doc recommends. No workaround under that
config; the fix is to point these vars elsewhere (see Fix).

## Root cause

The Kodus web app's internal `/api/proxy/*` server-side API client hardcodes the
`http://` scheme for requests built from `WEB_HOSTNAME_API`/`WEB_PORT_API`,
regardless of the port value — it does not infer `https` from `PORT_API=443`. This
call is pure **server-to-server** traffic from the `kodus-web` container to the `api`
container/service and was never intended to leave the Docker network or cross a TLS
boundary; `NEXTAUTH_URL`/`API_FRONTEND_URL` (also set to the public `https://`
domain) are the vars actually meant for browser-facing, public-internet use.

## Fix

Point `WEB_HOSTNAME_API`/`WEB_PORT_API` at the API's **internal** Docker network
address instead of the public HTTPS domain:

```
WEB_HOSTNAME_API=api      # the docker-compose service name — resolvable via
                          # Docker's embedded DNS on the shared
                          # kodus-backend-services network
WEB_PORT_API=3001         # the API's internal plain-HTTP port
```

Leave `NEXTAUTH_URL` / `API_FRONTEND_URL` as the public `https://<domain>` — those
are unrelated (browser-facing OAuth/callback URLs) and unaffected by this bug.

Then recreate `kodus-web` to pick up the change: `docker compose up -d
--force-recreate kodus-web`.
