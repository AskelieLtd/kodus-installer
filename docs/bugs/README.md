# Bug Log

Bug reports for **kodus-installer**. Each row links to a full report under this
folder. Reports are append-only — resolved bugs stay listed with their status
updated, so the symptom → root cause → fix trail is never lost.

> Add one with the `log-bug` skill. Each file captures one bug: symptom, repro,
> impact, root cause, and fix.

| #    | Bug | Severity | Status | Reported | Symptom |
| ---- | --- | -------- | ------ | -------- | ------- |
| [0001](0001-nginx-http2-directive-unsupported-old-versions.md) | nginx config fails with "unknown directive http2" on stock Ubuntu/Debian | High | Fixed | 2026-07-28 | `nginx -t` fails: `unknown directive "http2"` on nginx < 1.25.1 |
| [0002](0002-certbot-bootstrap-live-directory-conflict.md) | Certbot bootstrap chicken-and-egg: dummy cert blocks `certbot --nginx` | High | Fixed | 2026-07-28 | nginx won't start without a cert; certbot then refuses an unmanaged `live/` dir |
| [0003](0003-nginx-not-reloaded-after-certbot-serves-stale-cert.md) | Browser shows invalid cert after certbot succeeds — nginx never reloaded | Medium | Fixed | 2026-07-28 | Site still serves the old self-signed dummy cert after a successful `certbot certonly` |
| [0004](0004-postgres-password-auth-fails-after-env-changes-post-init.md) | `api` crash-loops with Postgres auth failure after editing `.env` post-first-boot | Medium | Fixed | 2026-07-28 | `password authentication failed for user "kodusdev"` |
| [0005](0005-web-app-hardcodes-http-scheme-to-public-api-port-443.md) | Sign-up/login broken when `WEB_HOSTNAME_API`/`WEB_PORT_API` point at the public HTTPS API domain | High | Fixed | 2026-07-28 | `400 The plain HTTP request was sent to HTTPS port` on `/api/proxy/*` |
| [0006](0006-rabbitmq-disk-free-limit-relative-false-alarm-high-ram.md) | RabbitMQ blocks all publishing with generic "timeout" on high-RAM hosts | High | Fixed | 2026-07-28 | `MessageBrokerService - Error publishing message to RabbitMQ` (`timeout`); `check_alarms` shows a free-disk alarm despite ample free space |
| [0007](0007-severity-filter-discards-every-suggestion-summary-still-posts.md) | Reviews post a summary but zero inline comments — every suggestion discarded by the severity filter | High | Fixed | 2026-07-29 | Review succeeds, summary posts, no inline comments; stored suggestions read `priorityStatus: 'discarded-by-severity'` |
| [0008](0008-docker-compose-restart-does-not-apply-env-changes.md) | `.env` changes silently ignored after `docker compose restart` | Medium | Fixed | 2026-07-29 | `printenv` in the container still shows the pre-edit values after a successful restart |
| [0009](0009-base-branch-filter-added-on-config-save-blocks-reviews-org-wide.md) | Every review reports "No changed files" org-wide after a `baseBranches` filter appears in the saved config | High | Investigating | 2026-07-29 | `ResolveConfigStage - No files found in PR` in ~880ms across multiple repos, on PRs GitHub reports as having changed files |
<!-- newest rows appended; keep ordered by number ascending (open bugs may be listed first) -->
