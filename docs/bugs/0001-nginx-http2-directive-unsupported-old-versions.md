# 0001. nginx config fails with "unknown directive http2" on stock Ubuntu/Debian

- **Status:** Fixed
- **Severity:** High
- **Reported:** 2026-07-28
- **Resolved:** 2026-07-28
- **Fix:** commit `97b2415` on branch `feat/nginx-reverse-proxy-kodus-askelie-in`

## Symptom

`sudo nginx -t` fails immediately on a freshly-installed nginx:

```
nginx: [emerg] unknown directive "http2" in /etc/nginx/sites-enabled/kodus.askelie.in.conf:67
nginx: configuration file /etc/nginx/nginx.conf test failed
```

## Expected

The reverse-proxy config in `docker/nginx/kodus.askelie.in.conf` should parse cleanly on
whatever nginx version `apt install nginx` pulls in.

## Reproduction

1. Deploy `docker/nginx/kodus.askelie.in.conf` (initial version) to a host running
   nginx from Ubuntu/Debian's default apt repo (typically 1.18.x or 1.24.x).
2. Run `sudo nginx -t`.
3. → fails with `unknown directive "http2"`.

## Impact

Blocks the reverse proxy from starting at all — no workaround short of editing the
config, since the standalone `http2 on;` directive simply doesn't exist on these
nginx builds.

## Root cause

The config used the standalone `http2 on;` directive, introduced in nginx **1.25.1**.
Ubuntu/Debian's apt package is almost always older than that (1.18–1.24 depending on
release), so the directive is unrecognized and the whole config fails to parse.

## Fix

Switched to the older, broadly-compatible form: `listen 443 ssl http2;` /
`listen [::]:443 ssl http2;` instead of a separate `http2 on;` line. Verified against
nginx 1.18, 1.24, 1.27, and `stable` — passes on all four (1.25.1+ prints a harmless
deprecation warning recommending the new directive, but still loads correctly).
