# 0003. Browser shows invalid cert after certbot succeeds — nginx never reloaded

- **Status:** Fixed
- **Severity:** Medium
- **Reported:** 2026-07-28
- **Resolved:** 2026-07-28
- **Related:** [0002](0002-certbot-bootstrap-live-directory-conflict.md)

## Symptom

Browser shows an invalid/untrusted certificate warning when visiting the site, even
though `certbot certonly` had already reported success. Confirmed via:

```
$ echo | openssl s_client -connect <domain>:443 -servername <domain> | openssl x509 -noout -subject -issuer -dates
subject=CN=<domain>
issuer=CN=<domain>          # <- self-signed, not Let's Encrypt
notBefore=<original dummy-cert bootstrap time>
```

```
$ curl -I https://<domain>
curl: (60) SSL certificate problem: self signed certificate
```

## Expected

Once certbot reports "Successfully received certificate", the site should serve that
real, CA-signed certificate.

## Reproduction

1. Follow the [0002](0002-certbot-bootstrap-live-directory-conflict.md) bootstrap
   (dummy self-signed cert → `certbot certonly --webroot`).
2. Certbot succeeds and writes the real cert to `/etc/letsencrypt/live/<domain>/`.
3. Skip (or forget) `sudo nginx -t && sudo systemctl reload nginx` afterward.
4. → nginx's running master process keeps serving the *original* dummy self-signed
   cert from memory indefinitely, since it only re-reads certificate files on
   reload/restart — it has no way to know the on-disk files changed.

## Impact

Site appears broken/insecure to every visitor despite a valid cert existing on disk.
Purely a missed manual step, not a config defect — but easy to skip since certbot's
own success message gives no indication nginx still needs a reload.

## Root cause

nginx loads TLS certificates into memory once, at start or reload, and never
re-reads them from disk on its own. `certbot certonly` (webroot mode) only issues the
certificate — unlike `certbot --nginx`, it does not touch or reload nginx.

## Fix

```
sudo nginx -t
sudo systemctl reload nginx
```

Confirmed via `openssl s_client` afterward showing `issuer=C=US, O=Let's Encrypt`
and a live `curl -I` returning proper HTTP/2 responses instead of a TLS error.
