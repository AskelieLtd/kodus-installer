# 0004. `api` crash-loops with Postgres auth failure after editing `.env` post-first-boot

- **Status:** Fixed
- **Severity:** Medium
- **Reported:** 2026-07-28
- **Resolved:** 2026-07-28

## Symptom

`api` (and dependents) restart continuously. `docker compose logs api` shows
migrations failing at boot:

```
Error during migration run:
error: password authentication failed for user "kodusdev"
    ... code: '28P01', severity: 'FATAL', routine: 'auth_failed'
```

## Expected

`api` should connect to Postgres successfully using the credentials in `.env`.

## Reproduction

1. `cp .env.example .env`, run the stack once (`docker compose up -d`) — Postgres
   initializes its data directory and sets the DB user's password from whatever
   `API_PG_DB_USERNAME`/`API_PG_DB_PASSWORD` were in `.env` **at that moment**.
2. Edit `.env` afterward (e.g. re-run `./scripts/generate-secrets.sh`, or change
   `API_PG_DB_PASSWORD` by hand) and restart the stack.
3. → `api` fails to authenticate; the Postgres container is still running with the
   *original* password baked into its persistent volume.

## Impact

Full outage of `api` and anything depending on it (`worker`, `webhooks` reuse the
same credentials). Workaround exists (see Fix), so not Critical — but blocks first
boot entirely until resolved.

## Root cause

The official Postgres Docker image only applies `POSTGRES_USER`/`POSTGRES_PASSWORD`
during **first initialization of an empty data directory** (via
`docker-entrypoint-initdb.d`). Once `pgdata` has data, Postgres ignores those env vars
on subsequent starts — the password baked into the volume from the very first boot is
authoritative until changed inside Postgres itself.

## Fix

For a fresh install with nothing worth preserving, wipe and reinitialize from the
current `.env`:

```
docker compose down
docker volume rm <project>_pgdata <project>_mongodbdata
docker compose up -d
```

(Find the exact volume names first with `docker volume ls | grep -E "pgdata|mongodbdata"`
— compose prefixes them with the project/directory name.)

If existing data must be preserved instead, `ALTER USER <user> PASSWORD '<new>';`
inside the running Postgres container to match `.env`, rather than wiping the volume.
