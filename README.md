# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) configuration preset for the
**Classroom-Ready** org. Renovate auto-discovers an org repo named
`renovate-config`, and this is the single source of truth every repo extends.

## What this preset does

Deliberately **security-first + end-of-life only** — it does **not** open
routine minor/patch version-bump PRs (that noise was removed on purpose). It
opens PRs for:

1. **Security / vulnerability fixes** — via `vulnerabilityAlerts` +
   `osvVulnerabilityAlerts`. All packages are disabled by a catch-all
   `packageRules` entry, but Renovate force-overrides that for vulnerable deps,
   so security PRs still open. Like all PRs they open **within the schedule
   below** — when dev infra is up so the frontend preview→e2e chain passes on
   the first try. (There is no auto re-run of e2e when infra returns, so
   opening off-hours would leave checks red until a manual re-run.) Requires
   GitHub Dependabot/vulnerability alerts enabled on each repo (Settings → Code
   security).
2. **End-of-life / end-of-support** — for the version strings listed below,
   via the `endoflife-date` datasource. **Auto-detected, no annotation
   needed.** This is intentionally broad and tolerates false positives for now;
   if a given string gets noisy, add a per-repo override to silence it.

### Schedule

**All** PRs — including security — open only **Mon–Fri, 8am–5pm Mountain
time** (`America/Denver`, DST-aware), the window when the dev environment is up
so PR-triggered e2e tests run against live infra. For a fixed UTC-7 (no DST)
window, change `timezone` to `America/Phoenix`.

## How repos consume it

Org-wide with **zero per-repo files**, via Renovate **inherited config** on the
Mend hosted app: `org-inherited-config.json` in this repo is auto-applied to
every repo where the Mend app is installed. It extends `default.json` and sets
`onboarding: false` + `requireConfig: optional`, so Renovate runs on all repos
immediately — no "Configure Renovate" PRs to merge.

> Mend enables `inheritConfig` for the org only once it detects a commit of
> `org-inherited-config.json` while the app is installed on this repo (it is —
> installed org-wide). If it doesn't kick in, push a no-op commit to that file
> and wait a minute.

A single repo can still override with its own `renovate.json`:

```json
{ "extends": ["local>Classroom-Ready/renovate-config"] }
```

## What's tracked for EOL (auto-detected)

| Source string | File(s) | endoflife.date slug |
|---|---|---|
| `terraform_version: X.Y.Z` (setup-terraform input) | `.github/workflows/*.yml` | `terraform` |
| `.nvmrc` Node version | `.nvmrc` | `nodejs` |
| Lambda `Runtime: nodejsNN.x` | `template.yml` | `nodejs` |
| RDS/Aurora `engine_version = "..."` (in an `rds_master = { ... }` block) | `*.tf`, `*.tf.j2`, `*.tfvars` | `amazon-rds-postgresql` |
| ElastiCache `engine_version = "..."` (in a `redis = { ... }` block) | `*.tf`, `*.tf.j2`, `*.tfvars` | `amazon-elasticache-redis` |

Both `engine_version` managers use `matchStringsStrategy: recursive`: an outer
regex isolates the `rds_master = { ... }` / `redis = { ... }` block (bounded
by its own closing brace), then an inner regex finds `engine_version` within
just that block text. Only the inner match is ever rewritten, so a sibling
field in the same block (e.g. `description`) can't get corrupted, and a
Postgres/Redis `engine_version` pair in the same file can't collide into one
dependency. The Redis manager deliberately uses the `amazon-elasticache-redis`
slug, not community `redis` — ElastiCache's supported versions (currently 7.x)
lag community Redis (already on 8.x), so the community slug would propose
versions ElastiCache doesn't offer.

This previously (before both fixes above) produced a broken PR that bumped
Redis `7.0` to a Postgres major version and corrupted the block's
`description` field — see classroomready_infra#37.

**Not on endoflife.date:** `directus`, `typesense`. Track those for freshness
with the `docker` datasource if/when wanted.

## Validation

CI (`.github/workflows/validate-config.yml`) validates `default.json` on every
PR/push via
[`github-action-renovate-config-validator`](https://github.com/suzuki-shunsuke/github-action-renovate-config-validator)
in strict mode. **No dependencies are vendored in this repo** — that keeps it
free of the `renovate` package's transitive vulnerability surface (and the
security-update PR noise that would come with it).

To validate locally without committing anything:

```sh
npx --yes --package renovate -- renovate-config-validator --strict default.json
```
