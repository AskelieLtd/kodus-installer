# 0002. Certbot bootstrap chicken-and-egg: dummy cert blocks `certbot --nginx`

- **Status:** Fixed
- **Severity:** High
- **Reported:** 2026-07-28
- **Resolved:** 2026-07-28
- **Related:** [0003](0003-nginx-not-reloaded-after-certbot-serves-stale-cert.md)

## Symptom

Two chained failures when first issuing a Let's Encrypt cert for a fresh nginx +
Kodus reverse-proxy setup:

1. `sudo nginx -t` fails before any cert exists:
   ```
   nginx: [emerg] cannot load certificate "/etc/letsencrypt/live/<domain>/fullchain.pem":
   BIO_new_file() failed (SSL: error:80000002:system library::No such file or directory)
   ```
2. After working around (1) with a throwaway self-signed cert at that path so nginx
   can start, `sudo certbot --nginx -d ...` then fails with:
   ```
   live directory exists for <domain>
   ```

## Expected

`certbot --nginx` should issue a real certificate and wire it into the running nginx
config without manual directory surgery.

## Reproduction

1. Write an nginx config with `ssl_certificate`/`ssl_certificate_key` pointing at
   `/etc/letsencrypt/live/<domain>/...` before any cert has been issued.
2. `sudo nginx -t` → fails (cert file missing) — nginx can't even start, so certbot's
   `--nginx` plugin (which needs nginx running on port 80 for the HTTP-01 challenge)
   can't proceed either.
3. Work around with a throwaway self-signed cert at the exact same path, `nginx -t` /
   `reload` now succeed.
4. Run `sudo certbot --nginx -d <domain>` → fails with `live directory exists for
   <domain>`, because certbot detects a `/etc/letsencrypt/live/<domain>/` directory
   with no certbot-managed lineage metadata (no matching `renewal/<domain>.conf`) and
   refuses to touch it.

## Impact

Blocks HTTPS setup entirely on the very first cert issuance for a new domain. No
workaround other than following the correct bootstrap sequence below.

## Root cause

Two separable facts combine into a bootstrap deadlock:

- nginx must have valid files at every `ssl_certificate`/`ssl_certificate_key` path
  referenced by *any* server block before it will start/reload at all — even to serve
  the unrelated port-80 ACME challenge for a *different* domain in the same file.
- Certbot's `--nginx` authenticator needs nginx already running to serve that
  challenge, and separately refuses to manage a `live/<domain>` directory it didn't
  create itself (no recognized lineage).

## Fix

1. Create a throwaway self-signed cert at the exact `ssl_certificate`/
   `ssl_certificate_key` path so nginx can start:
   ```
   sudo mkdir -p /etc/letsencrypt/live/<domain>
   sudo openssl req -x509 -nodes -newkey rsa:2048 -days 1 \
     -keyout /etc/letsencrypt/live/<domain>/privkey.pem \
     -out /etc/letsencrypt/live/<domain>/fullchain.pem \
     -subj '/CN=<domain>'
   sudo nginx -t && sudo systemctl reload nginx
   ```
2. Delete the unmanaged dummy directory (nginx's *running* master process keeps the
   cert loaded in memory — deleting the on-disk files doesn't crash it, only a future
   reload/restart would fail):
   ```
   sudo rm -rf /etc/letsencrypt/live/<domain> /etc/letsencrypt/archive/<domain>
   sudo rm -f /etc/letsencrypt/renewal/<domain>.conf
   ```
3. Request the real cert via **webroot**, not the `--nginx` plugin — this uses the
   still-running nginx to serve the challenge via the existing
   `/.well-known/acme-challenge/` location, without needing to re-validate the SSL
   config mid-process:
   ```
   sudo mkdir -p /var/www/certbot
   sudo certbot certonly --webroot -w /var/www/certbot -d <domain> [-d <other-domain> ...]
   ```
4. Reload nginx to pick up the now-real cert at the same path:
   ```
   sudo nginx -t && sudo systemctl reload nginx
   ```
5. Since `certonly` (not the `--nginx` installer) was used, add a renewal deploy hook
   so future auto-renewals reload nginx:
   ```
   sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy
   printf '#!/bin/sh\nsystemctl reload nginx\n' | sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
   sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
   ```
