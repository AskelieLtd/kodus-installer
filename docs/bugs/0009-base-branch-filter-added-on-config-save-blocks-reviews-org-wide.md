# 0009. Every review reports "No changed files" org-wide after a `baseBranches` filter appears in the saved config

- **Status:** Investigating   <!-- root cause identified (baseBranches introduced in code_review_config v7); fix authored but not yet confirmed, and one observed failure is not explained by it — see Unresolved. -->
- **Severity:** High
- **Reported:** 2026-07-29

## Symptom

Reviews complete almost instantly and post **"No changed files in this pull
request."** on PRs that demonstrably have changed files. Affects **multiple
repositories**, not one — so it presents as the whole instance breaking.

```
WARN  ResolveConfigStage - No files found in PR
      repository: "elie-workflow-service"   pullRequestNumber: 39
WARN  ResolveConfigStage - No files found in PR
      repository: "elie-konnect-service"    pullRequestNumber: 76
INFO  PipelineExecutor - Stage 'ResolveConfigStage' completed in 883ms
```

No error is raised. The stage resolves zero files and exits cleanly.

## Expected

PRs with changed files are reviewed. The webhook payload for
`elie-konnect-service#76` reports `"changed_files": 8, "additions": 188,
"deletions": 31` — GitHub has the files; Kodus resolved none.

## Reproduction

1. Have working reviews (suggestions being generated and persisted).
2. Save the code-review configuration in the web app.
3. Trigger a review on any PR with real changes — via push or `@kody start-review`.
4. → "No changed files in this pull request", in well under a second, with no
   LLM call and no error.

## Impact

Indistinguishable from a total outage, and the failure is *fast and silent*,
which sends diagnosis in the wrong direction. This one consumed a long
investigation across the LLM provider, structured-output mode, model tier,
RabbitMQ, webhook delivery, and the GitHub integration — all of which measured
healthy — because nothing surfaces the fact that a **configuration filter** is
excluding everything.

Note the misleading timing signal: `883ms` in the stage log is easy to misread as
a long duration. It is not — the stage is fast, which itself rules out retry
storms and unreachable dependencies and should be used to narrow the search
early.

## Root cause

`parameters.code_review_config` is **versioned**, and the version trail isolates
the change precisely:

| version | active | `severityLevelFilter` | `baseBranches` |
| ------- | ------ | --------------------- | -------------- |
| 4       | no     | `critical`            | —              |
| 5       | no     | `critical`            | —              |
| 6       | no     | `low`                 | —              |
| **7**   | **yes**| `low`                 | **`["dev"]`**  |

v6 changed only the severity threshold (the fix for
[0007](0007-severity-filter-discards-every-suggestion-summary-still-posts.md)).
**v7 added `baseBranches: ["dev"]`** and changed nothing else. That is the sole
difference between a config that reviewed PRs and one that resolves no files.

The restriction sits in the **global** config block, and all 20 repositories carry
`isSelected: true` with empty per-repo `configs: {}`, so every repository inherits
it. Any PR whose base branch falls outside the list is excluded instance-wide —
including all `main`-targeting PRs (e.g. the dependabot PRs on
`elie-security-service`, stored with `baseBranchRef: 'main'`).

## Unresolved

`elie-konnect-service#76` targets base `dev` — inside the allowlist — and still
resolved zero files. So either the filter is stricter than a plain base-branch
allowlist (e.g. matching the repository's default branch, `main`, rather than the
PR's actual base), or a second factor is in play. `baseBranches` remains the
correct first thing to remove: it is the only change between working and broken,
and it is trivially reversible.

If reviews still resolve no files after clearing it, the next candidate is the
spend limit — `organization_parameters.spend_limit_config` is
`{"enabled": true, "monthlyLimitUsd": 100, "modelPricing": {}}`. `modelPricing` is
**empty**, so there is no price data for the Azure BYOK deployments in use and
whatever spend figure is compared against the limit is derived from nothing.

## Fix

Clear the base-branch restriction in the web app's code-review settings, or add
`main` alongside `dev`. Prefer the UI over a direct SQL write so a proper v8 is
recorded and the version trail stays intact; v6 remains available as a known-good
reference.

## Verification

```
docker compose exec -T db_kodus_postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "
select version, active, jsonb_pretty(\"configValue\" - '\''repositories'\'')
from parameters
where \"configKey\"::text = '\''code_review_config'\''
order by version desc limit 2;"'
```

Then trigger a review and confirm the file count is non-zero:

```
docker compose logs --since 5m worker | grep -v 'Mongoose:' | grep -iE 'No files found|ResolveConfigStage'
```

## Diagnostic note — this is what actually cracked it

**`parameters.code_review_config` keeps a full version history.** Diffing the
active row against its predecessors is the fastest way to answer "what changed
just before this broke", and it is far more reliable than reading the settings UI
or reasoning about which field you *think* you edited:

```
select version, active, jsonb_pretty("configValue" - 'repositories') as config,
       jsonb_array_length("configValue" -> 'repositories') as repo_count
from parameters
where "configKey"::text = 'code_review_config'
order by version;
```

Strip `repositories` — it is the bulk of the bytes and rarely the interesting
part.

Two log-reading lessons from the same session, both of which cost rounds:

- **`API_LOG_LEVEL` defaults to `error`**, which suppresses the stage logs that
  name the cause. Raise it to `debug` first — and see
  [0008](0008-docker-compose-restart-does-not-apply-env-changes.md), because
  `docker compose restart` will not apply that change.
- **Mongoose query logging drowns everything.** Always `grep -v 'Mongoose:'`, and
  capture to a file before filtering rather than piping through `head` — startup
  output alone will fill a `head -50` and hide everything after it. Anchor greps
  on log text, not line numbers; the file shifts between commands.

## Related

- [0007](0007-severity-filter-discards-every-suggestion-summary-still-posts.md) —
  same class: a configuration filter silently discarding all output while the
  pipeline reports success. Together these two account for nearly every
  "Kodus is broken" symptom seen on this instance.
- [0008](0008-docker-compose-restart-does-not-apply-env-changes.md) — needed
  before any `API_LOG_LEVEL` change takes effect.
