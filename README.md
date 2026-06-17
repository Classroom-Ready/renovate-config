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

Preferred: set the org's **Mend Renovate `onboardingConfig`** (Mend dashboard)
to the snippet below, so every repo's onboarding PR auto-extends this preset
with **zero per-repo files**:

```json
{ "extends": ["local>Classroom-Ready/renovate-config"] }
```

Or add that same `renovate.json` to an individual repo manually.

> Renovate's `inheritConfig` is **disabled on the Mend cloud app** by default
> (it wastes API quota), so we use the shared-preset + `extends` pattern, which
> is what Renovate recommends anyway.

## What's tracked for EOL (auto-detected)

| Source string | File(s) | endoflife.date slug |
|---|---|---|
| `terraform_version: X.Y.Z` (setup-terraform input) | `.github/workflows/*.yml` | `terraform` |
| `.nvmrc` Node version | `.nvmrc` | `nodejs` |
| Lambda `Runtime: nodejsNN.x` | `template.yml` | `nodejs` |
| RDS/Aurora `engine_version = "..."` | `*.tf`, `*.tf.j2`, `*.tfvars` | `amazon-rds-postgresql` |

**Known false-positive surface (accepted for now):** the `engine_version`
manager assumes Postgres. A Redis/MySQL `engine_version` will be compared to
the wrong EOL table. Silence per-repo with a `packageRules` override pointing
`amazon-elasticache-redis` / `amazon-rds-mysql`, or narrow this preset later.

**Not on endoflife.date:** `directus`, `typesense`. Track those for freshness
with the `docker` datasource if/when wanted.

## Validate locally

```sh
npm install
npm run validate
```

CI (`.github/workflows/validate-config.yml`) runs the same validator on every
PR/push.
