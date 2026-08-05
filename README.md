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
3. **Node runtime majors** — the one deliberate exception to "no routine
   bumps", and the only PRs that come from a datasource other than
   `endoflife-date`. A node major is pinned in several files at once, and the
   EOL bump alone only rewrites `.nvmrc`, so `package.json` `engines.node`, a
   `Dockerfile` `FROM node:` base image and `@types/node` are re-enabled and
   **grouped into a single `node` PR**. See [Node majors move
   together](#node-majors-move-together).

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

### Node majors move together

A node major is pinned in more places than `.nvmrc`, and the rest are **not**
`endoflife-date` deps — Renovate extracts them natively, under datasources the
catch-all "disable everything" rule swallows:

| Source string | File(s) | Datasource |
|---|---|---|
| `.nvmrc` Node version | `.nvmrc` | `endoflife-date` (custom manager above) |
| `engines.node`, `volta.node` | `package.json` | `node-version` |
| `.node-version` | `.node-version` | `node-version` |
| `FROM node:<major>` | `Dockerfile` | `docker` |
| `@types/node` (majors only) | `package.json` | `npm` |

Before these rules only the first row could open a PR.

That is exactly what happened for node 24: `classroomready_backend#300` moved
`.nvmrc` to 24 on its own, leaving `engines.node: "22"` printing
`npm warn EBADENGINE` on every npm invocation in CI and the `Dockerfile` still
building the shipped image on `node:22-alpine` — a different major than CI
tested. `classroomready_backend#413` cleaned that up by hand; these rules stop
the next major needing the same follow-up.

Four details worth knowing before editing the rules:

- **Never put `matchUpdateTypes` on a rule that re-enables something.** An
  `updateType` is only known *after* the version lookup, and the catch-all rule
  disables the dep *before* it — so `matchUpdateTypes` + `enabled: true` never
  fires and the dep stays `skipReason: "disabled"`. A dry-run caught this on the
  first draft of the `@types/node` rule. Filtering in the other direction works
  fine, which is why majors-only for `@types/node` takes two rules: one enables
  the dep, the next disables its `minor`/`patch`/`pin`/`digest` updates.
- **A `>=` engines range never bumps, by design.** `cms` has
  `engines.node: ">=22.0.0"`, which already permits every newer major, so
  Renovate reports no update for it (the dep *is* enabled — confirmed by
  dry-run — there is simply nothing to rewrite). Only an exact pin like
  backend's `"24"` moves. That is fine: a permissive range can't produce the
  `EBADENGINE` warning an exact stale pin does.
- **LTS-only, suffix-safe tags come for free.** `workarounds:nodeDockerVersioning`
  (inherited via `config:recommended`) applies `versioning: node` plus a
  `versionCompatibility` regex, so the suffix survives — a dry-run against `cms`
  proposed `node:22-alpine` -> `node:24-alpine`, not a bare `24`. LTS filtering
  holds too: with node 26.7.0 already published, backend's `engines.node: "24"`
  was offered **no** update, because 24 is the current LTS.
  `workarounds:typesNodeVersioning` does the same versioning job for
  `@types/node`. The `!calico/node` / `!kindest/node`
  negations mirror that upstream preset — the `/(?:^|\/)node$/` pattern is broad
  enough that an unrelated image path ending in `node` would otherwise match.
- **`groupName: "node"`** puts all of it in one branch, so the runtime, the
  image and the types can't land in separate PRs and diverge again.

Not covered: a `Dockerfile` that pins node via `ENV NODE_VERSION=` rather than
the `FROM` tag (needs a `# renovate:` comment annotation), and `engines.npm`,
which is deliberately left alone.

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
