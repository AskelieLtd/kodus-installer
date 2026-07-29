# 0007. Reviews post a summary but zero inline comments — every suggestion silently discarded by the severity filter

- **Status:** Fixed
- **Severity:** High
- **Reported:** 2026-07-29
- **Resolved:** 2026-07-29

## Symptom

Code review completes successfully and posts its summary, but **no inline
comments ever appear** on the pull request. The run reports no error. Nothing in
the UI indicates anything was suppressed.

The stored review record shows suggestions were generated and then dropped:

```
priorityStatus: 'discarded-by-severity'
severity: 'high'
label: 'bug'
```

and the PR document carries:

```
syncedEmbeddedSuggestions: false
```

## Expected

Suggestions that the model generates — particularly `high`-severity `bug`
findings with valid diff-line anchors — should be posted as inline comments.

## Reproduction

1. Configure the code-review severity threshold above `high` (i.e. `critical`).
2. Open or update a PR with real reviewable changes.
3. Trigger a review. → summary posts; no inline comments; no error surfaced.
4. Inspect the stored suggestions:
   ```
   docker compose exec -T db_kodus_mongodb sh -c 'mongosh -u "$MONGO_INITDB_ROOT_USERNAME" \
     -p "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin \
     "$MONGO_INITDB_DATABASE" --quiet --eval "
   var p = db.pullRequests.findOne({}, {}, {sort: {updatedAt: -1}});
   (p.files||[]).forEach(function(f){ (f.suggestions||[]).forEach(function(s){
     print(s.severity + \"  \" + s.priorityStatus + \"  \" + f.path); }); });
   print(\"synced=\" + p.syncedEmbeddedSuggestions);
   "'
   ```
   → every suggestion reads `discarded-by-severity`.

## Impact

The review pipeline appears completely broken while being entirely healthy. The
failure is **indistinguishable from a broken LLM integration**: reviews run, the
summary is fine, and not one comment is produced — which is also exactly what a
misrouted provider endpoint, a rejected structured-output schema, or an
underpowered model looks like.

This cost a multi-hour investigation down the wrong path — Azure OpenAI base-URL
routing, `API_TRUST_JSON_SCHEMA_BASE_URLS` and the `json_object` fallback, and
model tier (`gpt-4o-mini`) were all suspected and changed before the real cause
surfaced. The generated suggestion turned out to be well-formed and correct
throughout: valid schema, valid line anchors, a genuine bug correctly described.

The reason the split (summary works, comments don't) is so misleading: the
summary is free-text and not severity-filtered, while inline comments are the
only output the filter touches.

## Root cause

The code-review configuration's severity threshold was set above the severity of
everything the model produced. Suggestions are generated, validated, persisted
with `priorityStatus: 'discarded-by-severity'`, and then never delivered.

The discard is **silent** — no warning at the default `API_LOG_LEVEL=error`, no
count of suppressed findings in the run summary, and no indication in the UI that
a filter removed anything. A "0 comments" result and a "12 comments suppressed by
your severity threshold" result are presented identically.

## Fix

Lower the severity threshold in the code-review configuration (web app → the
organization's or repository's code-review settings → severity level filter) to
`high`, or `medium` for broader coverage. Anything above `high` discards typical
model output wholesale.

## Verification

Re-run a review on a PR with real changes and confirm the disposition changed:

```
docker compose exec -T db_kodus_mongodb sh -c 'mongosh -u "$MONGO_INITDB_ROOT_USERNAME" \
  -p "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin \
  "$MONGO_INITDB_DATABASE" --quiet --eval "
var p = db.pullRequests.findOne({}, {}, {sort: {updatedAt: -1}});
(p.files||[]).forEach(function(f){ (f.suggestions||[]).forEach(function(s){
  print(s.severity + \"  \" + s.priorityStatus + \"  \" + f.path); }); });
print(\"synced=\" + p.syncedEmbeddedSuggestions);
"'
```

`priorityStatus` should be something other than `discarded-by-severity`, and
inline comments should appear on the PR.

## Diagnostic note for the next person

When reviews produce a summary but no comments, check disposition **before**
suspecting the LLM provider. In order of cost-to-check:

1. `priorityStatus` on the stored suggestions — is anything being generated at
   all, and what happened to it? This settles generation-vs-delivery in one query
   and would have short-circuited this entire investigation.
2. Whether the PR has reviewable files at all. Two cases legitimately report "No
   changed files in this pull request" and bail in well under a second, without
   ever contacting the LLM: PRs ingested by the install-time backfill (stored
   with an empty `files: []`), and dependency-manifest-only PRs such as
   dependabot bumps (single file, empty `patch`). Check with:
   `db.pullRequests.countDocuments({ files: { $size: 0 } })` versus
   `countDocuments({})`.
3. Whether the broker is blocked — a raised RabbitMQ alarm stops jobs reaching
   the worker entirely (see
   [0006](0006-rabbitmq-disk-free-limit-relative-false-alarm-high-ram.md)).
4. Only then the provider, schema, and model.

Also: raise `API_LOG_LEVEL` from its `error` default to `debug` first. At `error`
the pipeline's own stage logs — including discard reasons — are suppressed, so
log-based diagnosis is impossible until it's raised (and note
[0008](0008-docker-compose-restart-does-not-apply-env-changes.md): `docker
compose restart` will not apply that change).
