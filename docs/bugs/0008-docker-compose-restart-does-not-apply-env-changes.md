# 0008. `.env` changes silently ignored after `docker compose restart`

- **Status:** Fixed
- **Severity:** Medium
- **Reported:** 2026-07-29
- **Resolved:** 2026-07-29

## Symptom

You edit `.env`, run `docker compose restart`, and the containers come back with
the **old** values. Nothing warns; the restart reports success.

```
$ docker compose exec worker printenv | grep -E 'TRUST_JSON|LOG_LEVEL'
API_TRUST_JSON_SCHEMA_BASE_URLS=          # just edited to a real value
API_LOG_LEVEL=error                       # just edited to debug
```

## Expected

A restart after editing `.env` applies the new values.

## Reproduction

1. Edit any variable in `.env` (e.g. `API_LOG_LEVEL=error` → `debug`).
2. `docker compose restart` (or `docker compose restart worker`).
3. `docker compose exec worker printenv | grep API_LOG_LEVEL`
4. → still `error`.

## Impact

Silently wastes debugging cycles: you believe a configuration change is live,
draw conclusions from behaviour that reflects the *old* config, and change
something else. Particularly costly when the variable you were setting was itself
a diagnostic (`API_LOG_LEVEL=debug`), because the absence of the expected logs
reads as "the code doesn't log that" rather than "my change never applied".

This is the same class of trap as
[0004](0004-postgres-password-auth-fails-after-env-changes-post-init.md) — a
`.env` edit that appears applied but isn't — though the mechanism differs. In 0004
Postgres had already persisted the password into its initialized data directory;
here the container's environment is simply frozen.

## Root cause

Environment variables are baked into a container at **creation** time. `docker
compose restart` stops and starts the *existing* container, so it returns with
exactly the environment it was created with — `env_file` is not re-read. Only
creating a new container picks up `.env` changes.

## Fix

Recreate rather than restart:

```
docker compose up -d --force-recreate worker api webhooks
```

`docker compose up -d` on its own usually detects the changed config and
recreates, but `--force-recreate` removes the ambiguity — worth being explicit
when a config change is what you are actually testing.

## Verification

Always confirm the value the process actually sees, rather than trusting that the
edit applied:

```
docker compose exec worker printenv | grep -E '<VAR_NAME>'
```

## Related

- [0004](0004-postgres-password-auth-fails-after-env-changes-post-init.md) — the
  other "`.env` edit didn't take" failure, via persisted Postgres state.
- [0006](0006-rabbitmq-disk-free-limit-relative-false-alarm-high-ram.md) — same
  underlying lesson at the RabbitMQ layer: a setting that looks applied but is
  read by nothing. Verify effective values, not intended ones.
