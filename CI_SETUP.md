# CI Setup Guide: Snowflake-native dbt Pipeline

This guide covers everything needed to get the Snowflake-native CI pipeline running.

Compare to [cloud-adjacent-pipeline](https://github.com/sfc-gh-mwinkel/cloud-adjacent-pipeline) which runs dbt on GitHub Actions runners — this repo runs dbt **inside Snowflake** using `EXECUTE DBT PROJECT`.

---

## What's Different from cloud-adjacent-pipeline

| | cloud-adjacent-pipeline | this repo |
|---|---|---|
| dbt compute | GitHub Actions runner | Snowflake (`EXECUTE DBT PROJECT`) |
| Auth | RSA key pair | Username / password |
| `pip install dbt` per run | Yes (~30s) | No |
| Network policy for CI user | Required | Not required |
| Slim CI | Yes (`state:modified+`) | Not yet — see [Roadmap](#roadmap) |
| GHA runner minutes for build | High | Minimal (~5s for CLI install) |

---

## Step 1: Enable Personal Database

```sql
ALTER ACCOUNT SET ENABLE_PERSONAL_DATABASE = TRUE;
```

---

## Step 2: Create External Access Integration (for `dbt deps`)

Allows Snowflake to download dbt packages from hub.getdbt.com and GitHub when `dbt deps` runs:

```sql
CREATE OR REPLACE NETWORK RULE dbt_packages_egress
    MODE       = EGRESS
    TYPE       = HOST_PORT
    VALUE_LIST = ('hub.getdbt.com', 'codeload.github.com');

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION dbt_packages_integration
    ALLOWED_NETWORK_RULES = (dbt_packages_egress)
    ENABLED = TRUE;
```

---

## Step 3: Create Git API Integration

Allows Snowflake to pull project code from GitHub when a DBT PROJECT is executed:

```sql
CREATE OR REPLACE API INTEGRATION git_api_integration
    API_PROVIDER         = git_https_api
    API_ALLOWED_PREFIXES = ('https://github.com/<your-org>/');
```

> For private repos, you will be prompted for a Personal Access Token (PAT) with `repo` scope when creating the workspace in Step 4.

---

## Step 4: Create a Workspace and Deploy the DBT PROJECT Object

1. In Snowsight, navigate to **Projects → My Workspace**
2. Click **Create Workspace → From Git repository**
3. Enter:
   - Repository URL: `https://github.com/<your-org>/snowflake-native-dbt-pipeline`
   - API integration: `git_api_integration`
   - Authentication: your GitHub PAT (for private repos)
4. Open `profiles/profiles.yml` in the workspace and update the placeholder values:
   ```yaml
   warehouse: YOUR_WAREHOUSE
   database:  YOUR_DATABASE
   role:      YOUR_ROLE   # for both prod and check targets
   ```
5. Run `dbt deps` in the workspace terminal to install packages
6. Test with `dbt build` in the workspace
7. Deploy: click **Deploy** (or run `snow dbt deploy <name> --source .` via CLI)

Verify the deployment:

```sql
SHOW DBT PROJECTS IN DATABASE <your_database>;
```

---

## Step 5: Create a CI Service Account

```sql
-- Create a dedicated CI user (username/password auth)
CREATE USER ci_user
    PASSWORD          = '<strong-password>'
    DEFAULT_ROLE      = dbt_ci_role
    DEFAULT_WAREHOUSE = your_warehouse;

GRANT ROLE dbt_ci_role TO USER ci_user;

-- Grant EXECUTE privilege on the project
GRANT EXECUTE ON DBT PROJECT <your_database>.<your_schema>.<project_name>
    TO ROLE dbt_ci_role;
```

---

## Step 6: Add GitHub Repository Secrets

Go to **Settings → Secrets and variables → Actions → New repository secret**.

| Secret Name | Required | Value | Notes |
|---|---|---|---|
| `SNOWFLAKE_ACCOUNT` | ✅ | `orgname-accountname` | No `.snowflakecomputing.com` |
| `SNOWFLAKE_USER` | ✅ | `ci_user` | Service account created above |
| `SNOWFLAKE_PASSWORD` | ✅ | *(password)* | Set in Step 5 |
| `SNOWFLAKE_ROLE` | ✅ | `dbt_ci_role` | Role with EXECUTE on the project |
| `SNOWFLAKE_WAREHOUSE` | ✅ | `TRANSFORM_WH` | Warehouse for the CLI connection |
| `SNOWFLAKE_DATABASE` | ✅ | `ANALYTICS` | Database containing the DBT PROJECT |
| `SNOWFLAKE_CI_DATABASE` | ⬜ optional | `ANALYTICS_CI` | If set, CI schemas land here instead |
| `DBT_PROJECT_FQN` | ✅ | `ANALYTICS.OPS.PIPELINE` | Fully qualified DBT PROJECT name |

> **Note:** `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_USER`, and `SNOWFLAKE_PASSWORD` are used only by the Snowflake CLI on the GHA runner to trigger `EXECUTE DBT PROJECT`. The actual dbt execution inside Snowflake uses the session context — `profiles.yml` leaves `account` and `user` empty for this reason.

---

## Step 7: Enable Branch Protection

1. **Settings → Branches → Add branch protection rule**
2. Branch name pattern: `main`
3. **Require status checks to pass before merging** → add: `dbt build (CI)`

---

## How It Works

```
PR pushed
  │
  ├── GHA runner (Snowflake CLI only — no dbt installed)
  │     │
  │     ├── Install Snowflake CLI                  ← ~5s
  │     │
  │     ├── snow sql: EXECUTE DBT PROJECT ... seed  ─────────────────┐
  │     │     args='seed --target check                               │
  │     │           --vars {pr_number: 42}'                           │ Snowflake
  │     │                                           ◄─────────────────┤ compute
  │     └── snow sql: EXECUTE DBT PROJECT ... build ─────────────────┤
  │           args='build --target check                              │
  │                 --vars {pr_number: 42}'                           │
  │                                                 ◄─────────────────┘
  │
  └── Pass / Fail → GitHub status check → branch protection
```

The PR number is passed as a dbt variable (`--vars {pr_number: 42}`), which the `check` target in `profiles.yml` uses to build the per-PR schema name:
```
schema: "dbt_pr_{{ var('pr_number', 'ci') }}"  →  dbt_pr_42
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

### Slim CI (`state:modified+`)

Currently CI runs a full `dbt build` on every PR. Slim CI requires `manifest.json` from the last prod run to be accessible to Snowflake compute at execution time.

Planned approach:
1. After each prod build, store `manifest.json` in a Snowflake internal stage
2. Reference it in CI builds via `--state @<stage_path>`

This depends on Snowflake-native dbt supporting stage-based `--state` paths. The full build fallback is intentional — safe and correct, just not minimal.

---

## Troubleshooting

**`EXECUTE DBT PROJECT` fails: insufficient privileges**
```sql
GRANT EXECUTE ON DBT PROJECT <db>.<schema>.<name> TO ROLE dbt_ci_role;
```

**`dbt deps` fails in workspace**
Verify the external access integration is enabled and includes `hub.getdbt.com` and `codeload.github.com`:
```sql
DESCRIBE EXTERNAL ACCESS INTEGRATION dbt_packages_integration;
ALTER EXTERNAL ACCESS INTEGRATION dbt_packages_integration SET ENABLED = TRUE;
```

**Can't create workspace from GitHub**
Check the API integration allowed prefixes match your repo URL:
```sql
DESCRIBE API INTEGRATION git_api_integration;
```

**Schema not dropped after PR close**
Verify `SNOWFLAKE_PASSWORD` is set correctly — cleanup uses direct connector auth, not the CLI.

**`var('pr_number')` not resolving / wrong schema**
Confirm `--vars {pr_number: $PR_NUMBER}` is correctly interpolated in the SQL file written to `/tmp/build.sql` in the CI workflow step.
