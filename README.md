# snowflake-native-dbt-pipeline

A reference implementation of dbt CI using **Snowflake-native execution** (`EXECUTE DBT PROJECT`) and GitHub Actions as a thin orchestrator.

This is a companion to [cloud-adjacent-pipeline](https://github.com/sfc-gh-mwinkel/cloud-adjacent-pipeline), which runs dbt on GitHub Actions runners. Use this repo when your team wants dbt compute to stay inside Snowflake.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  GitHub Actions  (orchestration only — no dbt compute here)     │
│                                                                 │
│  PR opened/updated                                              │
│    └── dbt-ci.yml                                               │
│          1. Checkout + git diff → compute changed models  (~5s) │
│          2. Install Snowflake CLI                         (~5s) │
│          3. EXECUTE DBT PROJECT ... seed  ──────────────────┐  │
│          4. EXECUTE DBT PROJECT ... build --select <changed>+┐ │
│             Result: PASS → merge / FAIL → blocked            │ │
│                                                          │   │   │
│  Push to main                                            ▼   ▼   │
│    └── dbt-prod.yml                               ┌────────────┐ │
│          EXECUTE DBT PROJECT ... seed build   ───►│  Snowflake │ │
│                                                   │  compute   │ │
│  PR closed                                        └────────────┘ │
│    └── dbt-cleanup.yml                                          │
│          DROP SCHEMA <layer>_dbt_pr_<N>                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Snowflake                                                      │
│                                                                 │
│  ANALYTICS.DBT_PROD           ← production                     │
│  ANALYTICS.SEEDS_DBT_PR_3     ← PR #3 seed layer               │
│  ANALYTICS.STAGING_DBT_PR_3   ← PR #3 staging layer            │
│  ANALYTICS.MARTS_DBT_PR_3     ← PR #3 marts layer              │
│                                                                 │
│  ANALYTICS.OPS.PIPELINE       ← DBT PROJECT object             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Compared to cloud-adjacent-pipeline

| | cloud-adjacent | this repo |
|---|---|---|
| dbt compute | GHA runner | Snowflake |
| Auth | RSA key pair | Username / password |
| `pip install dbt` per run | Yes | No |
| Network policy for CI user | Required | Not required |
| Slim CI | Yes (`state:modified+`) | Yes (git-diff model selection) |
| GHA runner minutes for build | High | Minimal |

Both repos share the same dbt project structure, macros, and schema isolation logic. Only the execution layer and auth differ.

---

## Data Model

```
raw_events (seed)
    └── stg_events  [view]
            └── int_user_event_summary  [ephemeral]
                        └── fct_user_revenue  [incremental table]
```

---

## Key Design Decisions

**`EXECUTE DBT PROJECT`** — dbt runs on Snowflake compute. GitHub Actions installs only the Snowflake CLI and passes SQL commands. No Python, no pip, no dbt on the runner.

**`var('pr_number')` instead of `env_var('PR_NUMBER')`** — since the dbt process runs inside Snowflake, environment variables from the GHA runner aren't available. The PR number is passed as a dbt variable via `--vars {pr_number: 42}` in the `args` string.

**Username/password auth** — simpler to set up than RSA key pairs; appropriate when the CI user's network access is controlled at the Snowflake level rather than via a network policy.

**Per-PR layer-namespaced schemas** — same `generate_schema_name` macro as cloud-adjacent-pipeline. Each layer gets its own schema (`seeds_dbt_pr_N`, `staging_dbt_pr_N`, `marts_dbt_pr_N`), all dropped on PR close.

**Slim CI via git diff** — Changed models are derived from `git diff origin/main...HEAD -- 'models/**/*.sql'`. Each is passed to `EXECUTE DBT PROJECT` as `--select model+ model2+` so dbt handles downstream selection natively. Falls back to a full build if no model files changed.

**`clone_incrementals_for_ci` hook** — Clones incremental tables from prod into the CI schema before the build. Uses `node.config.schema ~ '_' ~ target.schema` to resolve the correct layer-namespaced destination (e.g. `marts_dbt_pr_42`) rather than the raw `target.schema`.

---

## Setup

See **[CI_SETUP.md](./CI_SETUP.md)** for the full guide including DBT PROJECT object creation, service account setup, and GitHub secrets configuration.
