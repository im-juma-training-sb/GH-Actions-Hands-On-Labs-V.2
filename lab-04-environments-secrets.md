# Lab 4: Environments and Secrets

**Duration:** 50-60 minutes  
**Level:** Beginner-to-Intermediate  
**Prerequisites:** Labs 1-3 completed, repository `actions-fundamentals-lab` available

---

## Overview

Workflows often need sensitive values — API keys, tokens, database credentials — and the ability to target different deployment stages. In this lab you will create repository secrets, use the built-in `GITHUB_TOKEN`, configure GitHub environments with their own secrets, add protection rules with required reviewers, and use environment variables alongside secrets. By the end you will understand how GitHub securely stores and injects sensitive data into workflow runs.

All exercises are performed directly in your GitHub repository.

---

## Exercise 1: Create and Use Repository Secrets (10 minutes)

### Store sensitive values that any workflow in the repo can access

1. In your `actions-fundamentals-lab` repository, go to **Settings**.

2. In the left sidebar, expand **Secrets and variables** and click **Actions**.

3. Click **New repository secret** and create the following secrets one at a time:

   | Name | Value (fake — for training only) |
   |------|----------------------------------|
   | `API_KEY` | `sk_test_abc123def456` |
   | `NOTIFY_URL` | `https://hooks.example.com/notify` |

   For each: enter the **Name**, paste the **Value**, click **Add secret**.

4. After adding both, notice:
   - The value column shows only the **last updated** date — you can never view a secret's value once saved.
   - You can **Update** or **Remove** a secret, but not read it back.

5. Create `.github/workflows/use-secrets.yml`:

```yaml
name: Use Secrets

on:
  workflow_dispatch:

jobs:
  demo:
    runs-on: ubuntu-latest

    steps:
      - name: Attempt to print a secret (redacted)
        run: |
          echo "API_KEY value: ${{ secrets.API_KEY }}"
          # ↑ GitHub automatically detects the secret value in log output
          #   and replaces it with *** — this is called "secret redaction"

      - name: Use secret as an environment variable
        env:
          MY_KEY: ${{ secrets.API_KEY }}
          # ↑ Assigns the secret to a step-scoped environment variable.
          #   Best practice: map secrets to env vars so the shell receives
          #   them through the environment rather than inline expansion.
        run: |
          echo "Key length: ${#MY_KEY}"
          # ${#MY_KEY} → shell parameter expansion that returns the string length
          #   This proves the secret is populated without revealing its value

      - name: Use secret in a conditional
        env:
          NOTIFY_URL: ${{ secrets.NOTIFY_URL }}
        run: |
          if [ -n "$NOTIFY_URL" ]; then
            echo "Notification URL is configured — would POST here"
            # -n tests if the string is non-empty
          else
            echo "No notification URL set — skipping"
          fi
```

6. Commit to **main**.

7. Go to **Actions** -> select **"Use Secrets"** -> **Run workflow**.

8. Open the run log for step "Attempt to print a secret (redacted)" and confirm the value shows as `***`.

### Key points

- Secrets are encrypted at rest and only decrypted when injected into a runner.
- Redaction is automatic — GitHub scans log output for any string matching a secret value.
- If you base64-encode or otherwise transform a secret before printing, the raw value is still redacted, but the transformed form may leak. Always avoid printing secrets.
- A missing secret resolves to an empty string — the workflow does not fail.

---

## Exercise 2: The Built-in `GITHUB_TOKEN` (10 minutes)

### Use the automatic token GitHub provides for every workflow run

Every workflow run receives a short-lived token in `secrets.GITHUB_TOKEN`. It authenticates as the **github-actions[bot]** user and can interact with the repository's own API without any manual secret setup.

1. Create `.github/workflows/github-token-demo.yml`:

```yaml
name: GITHUB_TOKEN Demo

on:
  workflow_dispatch:

permissions:
  contents: read
  issues: write
  # ↑ Explicitly declare the permissions this workflow needs.
  #   'contents: read'  → can clone and read repo files
  #   'issues: write'   → can create and comment on issues
  #   By default GITHUB_TOKEN has broad permissions, but best practice
  #   is to restrict to only what is needed (principle of least privilege).

jobs:
  token-info:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        # ↑ actions/checkout automatically uses GITHUB_TOKEN to clone the repo.
        #   You do not need to pass it explicitly — it is the default.

      - name: List repository files via API
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # ↑ The GitHub CLI (gh) reads the token from the GH_TOKEN env var.
          #   This is the recommended way to authenticate gh commands.
        run: |
          echo "Listing repository contents via the GitHub API:"
          gh api repos/${{ github.repository }}/contents \
            --jq '.[].name'
          # gh api <endpoint>   → call the GitHub REST API
          # --jq '.[].name'    → filter JSON response using jq syntax:
          #   .[]   → iterate over each array element
          #   .name → extract the 'name' field

      - name: Show token permissions
        run: |
          echo "Token is scoped to: ${{ github.repository }}"
          echo "Token expires at the end of this workflow run"
          echo "Permissions granted:"
          echo "  contents: read"
          echo "  issues: write"
```

2. Commit to **main**.

3. Run the workflow manually. In the logs, confirm the API call returns your repo's file listing.

### `GITHUB_TOKEN` vs personal tokens

| Property | `GITHUB_TOKEN` | Personal Access Token (PAT) |
|----------|---------------|----------------------------|
| Created by | GitHub (automatic) | User (manual) |
| Lifetime | Single workflow run | Until revoked or expired |
| Scope | Current repository only | Any repo the user has access to |
| Rotation | Automatic every run | Manual |
| Stored as secret | No setup needed | Must be added as a repo/org secret |

---

## Exercise 3: Create Environments with Environment-Specific Secrets (10 minutes)

### Isolate secrets per deployment target

GitHub **Environments** let you attach secrets, variables, and protection rules to a named stage such as `development` or `staging`.

1. Go to **Settings** -> **Environments** -> **New environment**.

2. **Name**: `development` -> click **Configure environment**.

3. Under **Environment secrets**, click **Add secret**:

   | Name | Value |
   |------|-------|
   | `DEPLOY_URL` | `https://dev.internal.example.com` |
   | `DB_CONNECTION` | `Server=dev-db;Database=app;User=dev_user;` |

4. Leave all protection rules unchecked (development needs no gates). Click **Save protection rules** if prompted.

5. Go back to **Environments** -> **New environment**.

6. **Name**: `staging` -> click **Configure environment**.

7. Under **Environment secrets**, click **Add secret**:

   | Name | Value |
   |------|-------|
   | `DEPLOY_URL` | `https://staging.internal.example.com` |
   | `DB_CONNECTION` | `Server=staging-db;Database=app;User=staging_user;` |

8. Create `.github/workflows/env-secrets.yml`:

```yaml
name: Environment Secrets

on:
  workflow_dispatch:
    inputs:
      target:
        description: 'Which environment to deploy to'
        type: choice
        options:
          - development
          - staging
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.target }}
    # ↑ The 'environment' key tells GitHub which environment to pull secrets from.
    #   When set to 'development', secrets.DEPLOY_URL resolves to the development value.
    #   When set to 'staging', secrets.DEPLOY_URL resolves to the staging value.
    #   The same secret NAME returns DIFFERENT values depending on the environment.

    steps:
      - name: Show target environment
        run: |
          echo "Environment : ${{ inputs.target }}"
          echo "Branch      : ${{ github.ref_name }}"
          echo "Actor       : ${{ github.actor }}"

      - name: Deploy
        env:
          DEPLOY_URL: ${{ secrets.DEPLOY_URL }}
          DB_CONNECTION: ${{ secrets.DB_CONNECTION }}
        run: |
          echo "Deploying to $DEPLOY_URL ..."
          echo "DB connection string length: ${#DB_CONNECTION} characters"
          echo "Deployment complete."
```

9. Commit to **main**.

10. Run the workflow twice — once selecting **development**, once selecting **staging**. Compare the `DEPLOY_URL` output: each run prints `***` but at different lengths, confirming different secrets were injected.

### How environment secrets override repository secrets

If a repository secret and an environment secret share the same name, the **environment secret wins** when that environment is active. This lets you set a fallback at the repo level and override per environment.

---

## Exercise 4: Environment Protection Rules (10 minutes)

### Require manual approval before a job can proceed

1. Go to **Settings** -> **Environments** -> click **staging**.

2. Under **Environment protection rules**:
   - Check **Required reviewers**.
   - In the search box, type your own GitHub username and select it.
   - Click **Save protection rules**.

3. Create `.github/workflows/approval-gate.yml`:

```yaml
name: Approval Gate

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build application
        run: |
          echo "Compiling source..."
          echo "Running unit tests..."
          echo "Build artifact ready."

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    # ↑ Because 'staging' now has a required reviewer, this job will PAUSE
    #   after 'build' succeeds and wait for manual approval before executing.

    steps:
      - name: Deploy to staging
        env:
          DEPLOY_URL: ${{ secrets.DEPLOY_URL }}
        run: |
          echo "Deploying to staging at $DEPLOY_URL ..."
          echo "Deployment complete."

      - name: Run smoke tests
        run: |
          echo "Health check  ... OK"
          echo "API ping      ... OK"
          echo "DB connection ... OK"
```

4. Commit to **main**.

5. Run the workflow manually:
   - `build` executes immediately.
   - `deploy-staging` shows a yellow **"Waiting"** badge.

6. Click on the paused `deploy-staging` job -> click **Review deployments**.

7. Check the **staging** checkbox, optionally add a comment, and click **Approve and deploy**.

8. Watch the job resume and complete.

### What the reviewer sees

When a job is waiting for approval:
- A yellow banner appears on the workflow run page.
- Reviewers listed in the environment configuration receive a notification.
- Any listed reviewer can approve or reject.
- Rejected deployments mark the job as failed.

### Additional protection options

| Rule | Effect |
|------|--------|
| **Required reviewers** | Job pauses until an approved reviewer approves |
| **Wait timer** | Adds a delay (in minutes) even after approval before the job starts |
| **Deployment branches** | Restricts which branches can target this environment (e.g., only `main`) |

---

## Exercise 5: Workflow-Level Environment Variables (10 minutes)

### Use `env:` at workflow, job, and step levels alongside secrets

Environment variables (`env:`) hold non-sensitive configuration. They complement secrets — use `env` for values safe to display in logs and `secrets` for anything confidential.

1. Create `.github/workflows/env-variables.yml`:

```yaml
name: Environment Variables

on:
  workflow_dispatch:

env:
  APP_NAME: my-web-app
  REGION: us-east-1
  # ↑ Workflow-level env vars — available to ALL jobs and steps in this workflow

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      BUILD_CONFIG: release
      # ↑ Job-level env var — available to all steps in THIS job only

    steps:
      - name: Show inherited variables
        run: |
          echo "APP_NAME (workflow-level) : $APP_NAME"
          echo "REGION (workflow-level)   : $REGION"
          echo "BUILD_CONFIG (job-level)  : $BUILD_CONFIG"

      - name: Step with its own variable
        env:
          STEP_VAR: temporary-value
          # ↑ Step-level env var — exists only during this step
        run: |
          echo "STEP_VAR (step-level)     : $STEP_VAR"
          echo "APP_NAME still available  : $APP_NAME"
          echo "BUILD_CONFIG still here   : $BUILD_CONFIG"

      - name: Prove step-level var is gone
        run: |
          echo "STEP_VAR is now empty: '$STEP_VAR'"
          # ↑ Empty — STEP_VAR was scoped to the previous step only

  deploy:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Access workflow-level vars
        run: |
          echo "APP_NAME : $APP_NAME"
          echo "REGION   : $REGION"

      - name: Cannot access other job's vars
        run: |
          echo "BUILD_CONFIG: '$BUILD_CONFIG'"
          # ↑ Empty — BUILD_CONFIG was defined in the 'build' job, not here

      - name: Combine env vars with secrets
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          echo "Deploying $APP_NAME to $REGION"
          echo "API key length: ${#API_KEY}"
          echo "Non-sensitive config is visible; secrets remain redacted"
```

2. Commit to **main**.

3. Run the workflow and verify:
   - Workflow-level vars (`APP_NAME`, `REGION`) appear in both jobs.
   - Job-level var (`BUILD_CONFIG`) appears only in the `build` job.
   - Step-level var (`STEP_VAR`) appears only in its own step.
   - The secret `API_KEY` is redacted.

### Scope summary

| Level | Syntax location | Visible to |
|-------|----------------|------------|
| Workflow | Top-level `env:` | Every job and step |
| Job | Under `jobs.<id>.env:` | All steps in that job |
| Step | Under `steps[].env:` | That single step only |

---

## Exercise 6: Organization Secrets and Precedence (5 minutes)

### Understand the full secret hierarchy (conceptual)

This exercise is a review — no new workflow file is needed.

In a GitHub Enterprise organization, secrets can exist at **three levels**. When a workflow runs, GitHub resolves each `secrets.<NAME>` reference using the following precedence (highest wins):

```
1. Environment secret   (highest priority — overrides everything)
         |
2. Repository secret    (middle — overrides org secrets)
         |
3. Organization secret  (lowest — used as a shared default)
```

**Example scenario:**

| Secret name | Org-level value | Repo-level value | `staging` env value |
|-------------|----------------|------------------|---------------------|
| `API_KEY` | `org-key-000` | `repo-key-111` | `staging-key-222` |

- A workflow **without** an environment: resolves to `repo-key-111` (repo overrides org).
- A workflow **with** `environment: staging`: resolves to `staging-key-222` (environment overrides repo).

### Organization secrets setup (informational)

Organization admins configure org-level secrets at:

**Organization Settings** -> **Secrets and variables** -> **Actions** -> **New organization secret**

Each org secret has a **Repository access** policy:
- **All repositories** — every repo in the org can use it.
- **Private repositories** — only private and internal repos.
- **Selected repositories** — manually chosen list.

### When to use each level

| Level | Use case |
|-------|----------|
| Organization | Shared credentials (e.g., artifact registry token used by all repos) |
| Repository | Repo-specific keys (e.g., a deploy token for this app) |
| Environment | Stage-specific values (e.g., production DB password vs staging DB password) |

---

## Summary

| Concept | Key syntax / location | Purpose |
|---------|----------------------|---------|
| Repository secrets | Settings -> Secrets and variables -> Actions | Sensitive values available to all workflows |
| `GITHUB_TOKEN` | `secrets.GITHUB_TOKEN` | Auto-generated token scoped to the repo |
| `permissions:` | Workflow or job level | Restrict `GITHUB_TOKEN` to least privilege |
| Environments | Settings -> Environments | Group secrets and protection rules per stage |
| `environment:` key | `jobs.<id>.environment:` | Bind a job to an environment |
| Required reviewers | Environment protection rules | Pause a job until a human approves |
| `env:` (non-secret) | Workflow / job / step level | Non-sensitive configuration visible in logs |
| Secret precedence | Environment > Repository > Organization | Higher scope overrides lower |

You now know how to store sensitive data securely and use environments to control access and approval. These concepts apply directly when building deployment pipelines in later labs.
