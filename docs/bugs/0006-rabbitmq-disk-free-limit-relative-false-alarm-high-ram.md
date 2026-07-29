# 0006. RabbitMQ blocks all publishing with generic "timeout" on high-RAM hosts

- **Status:** Fixed
- **Severity:** High
- **Reported:** 2026-07-28
- **Resolved:** 2026-07-29

## Symptom

`worker` fails to publish job messages, logged as a generic timeout with no obvious
mention of disk space:

```
ERROR: MessageBrokerService - Error publishing message to RabbitMQ
  exchange: "workflow.exchange"
  routingKey: "workflow.jobs.created.WEBHOOK_PROCESSING"
  error: { message: "timeout", stack: "Error: timeout at Timeout._onTimeout
    (.../amqp-connection-manager/dist/cjs/ChannelWrapper.js:212:32) ..." }
```

Webhook events are received and enqueued (`webhooks` service logs show delivery),
but the review never actually starts — jobs never reach the worker's processing
stage.

## Expected

Published messages should reach RabbitMQ and be consumed by `worker` normally.

## Reproduction

1. Deploy on a host with a large amount of RAM (observed on a GPU box, ~56GB RAM)
   where `/` (or wherever the `rabbitmq-data-prod` docker volume lives) has less
   free disk space than `1.2 × total RAM`.
2. Trigger a webhook-driven code review (e.g. open/update a PR on a connected repo).
3. `worker` attempts to publish the review job to RabbitMQ.
4. → publish call times out; no review ever runs. Confirmed via:
   ```
   docker compose exec rabbitmq rabbitmq-diagnostics check_alarms
   → Free disk space alarm on node rabbit@rabbitmq
   ```
   ...despite `df -h` showing tens of GB genuinely free (e.g. 56GB avail on a
   123GB `/`), which looks like more than enough at a glance.

## Impact

Silently blocks the entire code-review pipeline — the single most core feature of
Kodus — on any host where free disk (in absolute terms) happens to be less than
1.2x total system RAM. The error message gives no hint this is a disk-space issue,
making it easy to misdiagnose as a networking/RabbitMQ-connectivity bug.

## Root cause

`docker/rabbitmq/rabbitmq.conf` (baked into the `ghcr.io/kodustech/kodus-rabbitmq:4.2.2-kodus`
image) sets:

```
disk_free_limit.relative = 1.20
```

This requires free disk space ≥ 1.2× total system RAM before RabbitMQ will accept
publishes — a limit that scales with RAM, not with actual queue/message volume. On
a host with ~56GB RAM, that's a ~67GB free-disk requirement; a box with "only" 56GB
free (which is otherwise generous) trips the alarm and RabbitMQ silently blocks all
publishing connections until the alarm clears, surfacing to publishers as a bare
"timeout" rather than a disk-space error.

## Failed first attempt — `RABBITMQ_DISK_FREE_LIMIT` is inert on RabbitMQ 4.x

The first fix added an env var to the `rabbitmq` service in `docker-compose.yml`,
on the belief that it was an official image variable that takes precedence over
the baked-in config:

```yaml
- RABBITMQ_DISK_FREE_LIMIT=2GB   # DOES NOTHING — do not reinstate
```

**This is silently ignored.** RabbitMQ dropped configuration via `RABBITMQ_*`
environment variables in 3.9; 4.x images honour only a small set
(`RABBITMQ_DEFAULT_USER`/`_PASS`/`_VHOST`, `RABBITMQ_NODENAME`,
`RABBITMQ_ERLANG_COOKIE`, `RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS`, …).
`RABBITMQ_DISK_FREE_LIMIT` is not among them. The container boots clean, nothing
warns, and `disk_free_limit.relative = 1.20` stays in force — so the symptom
recurs on the next publish and looks like the fix "didn't hold".

The trap is that this variable *did* work on RabbitMQ 3.8 and earlier, so it is
still widely repeated in older guides and StackOverflow answers.

Always confirm the effective value rather than assuming a setting applied:

```
docker compose exec rabbitmq rabbitmq-diagnostics environment | grep -i disk_free_limit
docker compose exec rabbitmq rabbitmq-diagnostics check_alarms
```

## Fix

Mount a replacement config file over the image's baked-in one, with an absolute
floor instead of a RAM-relative limit. A file mount is the only reliable override
given the above.

1. `docker/rabbitmq/rabbitmq.prod.conf` — a copy of the image's config with
   `disk_free_limit.relative = 1.20` replaced by `disk_free_limit.absolute = 2GB`.
2. Mounted in `docker-compose.yml`, replacing the removed env var:

```yaml
    volumes:
      - rabbitmq-data-prod:/var/lib/rabbitmq
      - ./docker/rabbitmq/rabbitmq.prod.conf:/etc/rabbitmq/rabbitmq.conf:ro
```

Before recreating, check the mounted file against what the running container
actually ships, so a drifted image doesn't lose settings to the mount (the only
expected difference is the `disk_free_limit` line plus the file's header comment):

```
docker compose exec rabbitmq cat /etc/rabbitmq/rabbitmq.conf \
  | diff - docker/rabbitmq/rabbitmq.prod.conf
```

Then recreate and verify with the two diagnostics commands above:

```
docker compose up -d --force-recreate rabbitmq
```

An absolute floor is the right shape here: the limit should track queue/message
volume, not host RAM.

### Immediate unblock

Clears the alarm at runtime and lets queued jobs drain, without recreating the
container. Does **not** survive a restart — the config file is what makes it
permanent:

```
docker compose exec rabbitmq rabbitmqctl set_disk_free_limit "2GB"
docker compose exec rabbitmq rabbitmq-diagnostics check_alarms
```

## Verification

Both must hold after `docker compose up -d --force-recreate rabbitmq`:

```
docker compose exec rabbitmq rabbitmq-diagnostics environment | grep -i disk_free_limit
  → the absolute 2GB value, NOT a relative one

docker compose exec rabbitmq rabbitmq-diagnostics check_alarms
  → no alarms listed
```

Checking the effective value (rather than assuming the setting applied) is the
specific lesson of the failed attempt above — the env var produced a clean boot
and no warning while doing nothing at all.

## Knock-on effect worth knowing

A raised alarm blocks *all* publishing, so review jobs never reach the worker at
all. That presents downstream as reviews producing a summary but no inline
comments, and as pull-request records persisted with empty `suggestions: []` —
symptoms easily misattributed to the LLM provider, the model, or structured-output
schema handling. Clear this alarm before investigating any "reviews aren't
commenting" report.
