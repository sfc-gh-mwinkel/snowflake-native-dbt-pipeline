# CI Setup Guide: Snowflake-native dbt Pipeline

This guide covers everything needed to get the Snowflake-native CI pipeline running.

Compare to [cloud-adjacent-pipeline](https://github.com/sfc-gh-mwinkel/cloud-adjacent-pipeline) which runs dbt on GitHub Actions runners — this repo runs dbt **inside Snowflake** using `EXECUTE DBT PROJECT`.

---

## What's Different from cloud-adjacent-pipeline

| | cloud-adjacent-pipeline | this repo |
|---|---|---|
| dbt compute | GitHub Actions runner | Snowflake (EXECUTE DBT PROJECT) |
| Auth | RSA key pair | Username / password |
| pip install dbt per run | Yes (~30s) | No |
| Network policy for CI user | Required | Not required |
| Slim CI | Yes (state:modified+) | Not yet — see [Roadmap](#roadmap) |
| GHA runner minutes used | High | Minimal |

---

## Step 1: Create the DBT PROJECT Object in Snowflake

The `EXECUTE DBT PROJECT` command requires a DBT PROJECT object that is pre-configured with your GitHub repo. Run the following as ACCOUNTADMIN:

```sql
-- 1. Create external access integration to reach GitHub
CREATE OR REPLACE NETWORK RULE github_git_egress
    TYPE       = HOST_PORT
    MODE       = EGRESS
    VALUE_LIST = ('github.com:443', 'api.github.com:443');

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION github_git_integration
    ALLOWED_NETWORK_RULES = (github_git_egress)
    ENABLED = TRUE;

-- 2. Create a secret for your GitHub token (for private repos; skip for public)
CREATE OR REPLACE SECRET analytics.ops.github_pat
    TYPE = GENERIC_STRING
    SECRET_STRING = '<your-github-personal-access-token>';

-- 3. Create the DBT PROJECT object
CREATE OR REPLACE DBT PROJECT analytics.ops.pipeline
    GIT_REPOSITORY = 'https://github.com/<your-org>/snowflake-native-dbt-pipeline'
    GIT_BRANCH     = 'main'
    -- GIT_SECRET  = analytics.ops.github_pat   -- uncomment for private repo
    EXTERNAL_ACCESS_INTEGRATIONS = (github_git_integration);
```

> **Note:** DBT PROJECT objects require Snowflake to be able to reach `github.com` — hence the external access integration.

---

## Step 2: Create a CI Service Account

```sql
-- Create a dedicated CI user (username/password for simplicity)
CREATE USER ci_user
    PASSWORD         = '<strong-password>'
    DEFAULT_ROLE     = dbt_ci_role
    DEFAULT_WAREHOUSE = your_warehouse;

GRANT ROLE dbt_ci_role TO USER ci_user;

-- Grant EXECUTE privilege on the project to the CI role
GRANT EXECUTE ON DBT PROJECT analytics.ops.pipeline TO ROLE dbt_ci_role;
```

---

## Step 3: Add GitHub Repository Secrets

Go to **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

| Secret Name | Required | Value | Notes |
|---|---|---|---|
| `SNOWFLAKE_ACCOUNT` | ✅ | `orgname-accountname` | No `.snowflakecomputing.com` |
| `SNOWFLAKE_USER` | ✅ | `ci_user` | The user created above |
| `SNOWFLAKE_PASSWORD` | ✅ | *(password)* | The password set above |
| `SNOWFLAKE_ROLE` | ✅ | `dbt_ci_role` | Role that owns EXECUTE on the project |
| `SNOWFLAKE_WAREHOUSE` | ✅ | `TRANSFORM_WH` | Warehouse to use |
| `SNOWFLAKE_DATABASE` | ✅ | `ANALYTICS` | Default database |
| `SNOWFLAKE_CI_DATABASE` | ⬜ optional | `ANALYTICS_CI` | If set, CI schemas land here instead |
| `DBT_PROJECT_FQN` | ✅ | `ANALYTICS.OPS.PIPELINE` | Fully qualified DBT PROJECT name |

---

## Step 4: Verify the `profiles/profiles.yml` is Committed

The `profiles.yml` in this repo uses `{{ var('pr_number', 'ci') }}` for the CI schema name rather than an environment variable. The PR number is passed at execution time via `--vars`.

```
profiles/
└── profiles.yml   ← committed, no secrets
```

---

## Step 5: Enable Branch Protection

Require the CI check to pass before any PR can be merged:

1. **Settings** → **Branches** → **Add branch protection rule**
2. Branch name pattern: `main`
3. Enable **Require status checks to pass before merging**
4. Add: `dbt build (CI)`

---

## How It Works

```
PR pushed
  │
  ├── GHA runner (no dbt installed)
  │     │
  │     ├── Install Snowflake CLI          ← ~5s
  │     │
  │     ├── EXECUTE DBT PROJECT ... seed   ─────────────────┐
  │     │     args='seed --target check                      │ Snowflake compute
  │     │           --vars {pr_number: 42}'                  │ (dbt runs here)
  │     │                                  ◄─────────────────┘
  │     │
  │     └── EXECUTE DBT PROJECT ... build  ─────────────────┐
  │           args='build --target check                     │ Snowflake compute
  │                 --vars {pr_number: 42}'                  │ (dbt runs here)
  │                                        ◄─────────────────┘
  │
  └── Schema: <layer>_dbt_pr_<PR_NUMBER>   ← dropped on PR close
```

### Schema naming per PR

| Layer | CI schema (PR #42) |
|---|---|
| seeds | `seeds_dbt_pr_42` |
| staging | `staging_dbt_pr_42` |
| intermediate | *(ephemeral)* |
| marts | `marts_dbt_pr_42` |

---

## Roadmap

### Slim CI (state:modified+)

The current CI runs a full `dbt build` on every PR. Slim CI (only building changed models and their downstream dependencies) requires access to the previous production `manifest.json` at execution time.

With `EXECUTE DBT PROJECT`, dbt runs on Snowflake compute and cannot directly read from a GitHub Actions artifact. The planned approach:

1. After each prod build, write `manifest.json` to a Snowflake internal stage
2. Reference it in CI builds using `--state @<stage_path>`

This depends on Snowflake-native dbt supporting stage-based `--state` paths. Track progress in Snowflake release notes.

---

## Troubleshooting

**`EXECUTE DBT PROJECT` fails: insufficient privileges**
Ensure `dbt_ci_role` has `EXECUTE` privilege on the project object:
```sql
GRANT EXECUTE ON DBT PROJECT analytics.ops.pipeline TO ROLE dbt_ci_role;
```

**CI fails: `DBT PROJECT` cannot reach GitHub**
The external access integration for `github.com` may be missing or disabled:
```sql
SHOW EXTERNAL ACCESS INTEGRATIONS LIKE 'GITHUB_GIT_INTEGRATION';
ALTER EXTERNAL ACCESS INTEGRATION github_git_integration SET ENABLED = TRUE;
```

**Schema not dropped after PR close**
The `ci_user` needs `DROP SCHEMA` on the CI database. The cleanup uses username/password auth — verify `SNOWFLAKE_PASSWORD` secret is set correctly.

**Wrong PR number schema created**
Check that `DBT_PROJECT_FQN` is set to the correct fully qualified project name in secrets.
