# Lab 5: Reusable Workflows

**Duration:** 45-55 minutes  
**Level:** Intermediate  
**Prerequisites:** Labs 1-4 completed, repository `actions-fundamentals-lab` available

---

## Overview

As your organization adopts GitHub Actions, you will notice identical workflow logic repeated across many repositories — linting, testing, security scanning, deployments. Reusable workflows let you define a workflow once and call it from other workflows, just like calling a function. In this lab you will create reusable workflows with inputs, secrets, and outputs, call them from a separate caller workflow, and understand the constraints that apply.

All exercises are performed directly in your GitHub repository.

---

## Exercise 1: Create a Basic Reusable Workflow (10 minutes)

### Define a workflow that other workflows can call

A reusable workflow uses the special trigger `workflow_call` instead of events like `push` or `pull_request`.

1. Create `.github/workflows/reusable-greeting.yml`:

```yaml
name: Reusable Greeting

on:
  workflow_call:
    # ↑ This is the key difference: 'workflow_call' makes this workflow callable
    #   by other workflows. It cannot be triggered directly by events.

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Print greeting
        run: |
          echo "Hello from the reusable workflow!"
          echo "Called at: $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
```

2. Commit to **main**.

3. Create a caller workflow `.github/workflows/call-greeting.yml`:

```yaml
name: Call Greeting

on:
  workflow_dispatch:

jobs:
  invoke-reusable:
    uses: ./.github/workflows/reusable-greeting.yml
    # ↑ 'uses' at the JOB level (not step level) invokes a reusable workflow.
    #   ./ prefix means "same repository". The path is relative to the repo root.
    #   You can also reference another repo:
    #     uses: org-name/repo-name/.github/workflows/file.yml@main
```

4. Commit to **main**.

5. Go to **Actions** -> **"Call Greeting"** -> **Run workflow**.

6. Click into the run. Notice the job graph shows `invoke-reusable` which expands into the reusable workflow's internal jobs.

### Key constraints

- A reusable workflow is referenced at the **job** level with `uses:`, not at the step level.
- The caller job cannot define its own `steps:` — it delegates entirely to the reusable workflow.
- You must specify a ref (`@main`, `@v1`, `@<sha>`) when calling across repositories.

---

## Exercise 2: Inputs and Secrets in Reusable Workflows (10 minutes)

### Pass parameters and sensitive data to the called workflow

1. Create `.github/workflows/reusable-deploy.yml`:

```yaml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        description: 'Target deployment environment'
        required: true
        type: string
        # ↑ Supported types: string, boolean, number
      dry-run:
        description: 'Simulate without deploying'
        required: false
        type: boolean
        default: false
    secrets:
      deploy-token:
        description: 'Token used to authenticate the deployment'
        required: true
        # ↑ Secrets must be explicitly declared and passed by the caller.
        #   The reusable workflow cannot access the caller's secrets implicitly.

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Validate inputs
        run: |
          echo "Environment : ${{ inputs.environment }}"
          echo "Dry run     : ${{ inputs.dry-run }}"
          echo "Token length: ${#TOKEN}"
        env:
          TOKEN: ${{ secrets.deploy-token }}
          # ↑ Access passed secrets via secrets.<name-declared-above>

      - name: Perform deployment
        if: inputs.dry-run == false
        run: |
          echo "Deploying to ${{ inputs.environment }}..."
          echo "Authenticating with deploy token..."
          echo "Deployment complete."

      - name: Dry run output
        if: inputs.dry-run == true
        run: |
          echo "DRY RUN: Would deploy to ${{ inputs.environment }}"
          echo "No changes made."
```

2. Commit to **main**.

3. Create a caller `.github/workflows/call-deploy.yml`:

```yaml
name: Call Deploy

on:
  workflow_dispatch:
    inputs:
      target-env:
        description: 'Environment to deploy'
        type: choice
        options:
          - development
          - staging

jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: ${{ inputs.target-env }}
      # ↑ 'with' passes input values to the reusable workflow's declared inputs
      dry-run: false
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
      # ↑ 'secrets' passes secrets explicitly.
      #   The name on the left (deploy-token) must match the declared secret name
      #   in the reusable workflow. The value on the right references the caller's secret.
```

4. Commit to **main**.

5. Before running, ensure you have a repository secret named `DEPLOY_TOKEN` (create one in Settings -> Secrets and variables -> Actions with any fake value like `deploy_abc_123`).

6. Run **"Call Deploy"**, select **development**, and verify the reusable workflow receives the inputs.

### Alternative: `secrets: inherit`

If you want the reusable workflow to receive ALL of the caller's secrets without listing each one:

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
    secrets: inherit
    # ↑ Passes every secret available to the caller.
    #   Convenient but less explicit — use with care in enterprise settings
    #   where least-privilege is preferred.
```

---

## Exercise 3: Outputs from Reusable Workflows (10 minutes)

### Return data from the reusable workflow back to the caller

1. Create `.github/workflows/reusable-build.yml`:

```yaml
name: Reusable Build

on:
  workflow_call:
    inputs:
      build-config:
        type: string
        default: 'release'
    outputs:
      artifact-name:
        description: 'Name of the generated build artifact'
        value: ${{ jobs.build.outputs.artifact }}
        # ↑ Maps a job-level output to a workflow-level output.
        #   The caller accesses this via needs.<job>.outputs.artifact-name
      build-version:
        description: 'Computed build version string'
        value: ${{ jobs.build.outputs.version }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact: ${{ steps.set-output.outputs.artifact }}
      version: ${{ steps.set-output.outputs.version }}
      # ↑ Job-level outputs pull from step-level outputs

    steps:
      - name: Generate build info
        id: set-output
        run: |
          VERSION="1.0.${{ github.run_number }}"
          ARTIFACT="app-${{ inputs.build-config }}-${VERSION}.tar.gz"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "artifact=$ARTIFACT" >> "$GITHUB_OUTPUT"
          # ↑ Writing to $GITHUB_OUTPUT sets step outputs.
          #   Format: name=value (one per line)

      - name: Simulate build
        run: |
          echo "Building version ${{ steps.set-output.outputs.version }}..."
          echo "Config: ${{ inputs.build-config }}"
          echo "Build complete."
```

2. Commit to **main**.

3. Create `.github/workflows/call-build-and-report.yml`:

```yaml
name: Build and Report

on:
  workflow_dispatch:

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      build-config: release

  report:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Display build outputs
        run: |
          echo "Artifact: ${{ needs.build.outputs.artifact-name }}"
          echo "Version : ${{ needs.build.outputs.build-version }}"
          # ↑ Access reusable workflow outputs via needs.<caller-job-id>.outputs.<output-name>
          #   'build' is the job ID defined in THIS file, not the reusable workflow.
```

4. Commit to **main**.

5. Run **"Build and Report"** and verify the `report` job prints the artifact name and version generated by the reusable workflow.

### Output data flow

```
Reusable workflow step (GITHUB_OUTPUT)
     |
     v
Reusable workflow job output (jobs.<id>.outputs.<name>)
     |
     v
Reusable workflow-level output (on.workflow_call.outputs.<name>.value)
     |
     v
Caller accesses via: needs.<caller-job>.outputs.<output-name>
```

---

## Exercise 4: Nesting and Chaining Reusable Workflows (10 minutes)

### Call multiple reusable workflows in sequence

1. Create `.github/workflows/call-full-pipeline.yml`:

```yaml
name: Full Pipeline

on:
  workflow_dispatch:

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      build-config: release

  deploy-dev:
    needs: build
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: development
      dry-run: false
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}

  deploy-staging:
    needs: deploy-dev
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      dry-run: true
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
    # ↑ The same reusable workflow called multiple times with different inputs.
    #   Each invocation gets its own job in the graph.
```

2. Commit to **main**.

3. Run **"Full Pipeline"** and observe the job graph:

```
build  ->  deploy-dev  ->  deploy-staging
```

4. Verify each job receives its correct inputs and that `deploy-staging` runs in dry-run mode.

### Nesting limits

- A reusable workflow **can** call another reusable workflow (nesting).
- Maximum nesting depth: **4 levels**.
- A single workflow file can call a maximum of **20 reusable workflows**.
- Reusable workflows called from the same caller run **sequentially by default** unless they have no `needs:` dependency.

---

## Exercise 5: Cross-Repository Reusable Workflows (5 minutes)

### Reference workflows from a shared organization repository (conceptual)

In an enterprise setting, a central team typically maintains a "shared workflows" repository that all other repositories call.

**Repository structure** (e.g., `my-org/shared-workflows`):

```
.github/
  workflows/
    reusable-lint.yml
    reusable-security-scan.yml
    reusable-deploy.yml
```

**Caller in another repo** (e.g., `my-org/my-app`):

```yaml
name: CI

on:
  push:
    branches: [main]

jobs:
  lint:
    uses: my-org/shared-workflows/.github/workflows/reusable-lint.yml@v1
    # ↑ Cross-repo reference format:
    #   <owner>/<repo>/.github/workflows/<file>@<ref>
    #   @v1 is a tag — pin to a tag or SHA for stability

  security:
    uses: my-org/shared-workflows/.github/workflows/reusable-security-scan.yml@v1
    secrets: inherit

  deploy:
    needs: [lint, security]
    uses: my-org/shared-workflows/.github/workflows/reusable-deploy.yml@v1
    with:
      environment: production
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
```

### Visibility requirements

| Calling repo visibility | Reusable workflow repo must be |
|------------------------|-------------------------------|
| Internal | Internal or public (same org) |
| Private | Same repo, or internal/public in same org |
| Public | Public |

For enterprise organizations, **internal** repositories are the standard choice for shared workflows — visible to all org members but hidden from the outside.

### Version pinning strategies

| Ref style | Example | Trade-off |
|-----------|---------|-----------|
| Branch | `@main` | Always latest — convenient but unstable |
| Tag | `@v1` | Stable — consumers opt into updates |
| Full SHA | `@a1b2c3d...` | Most secure — immune to tag rewriting |

---

## Summary

| Concept | Syntax | Purpose |
|---------|--------|---------|
| Reusable trigger | `on: workflow_call:` | Makes a workflow callable |
| Calling a reusable workflow | `jobs.<id>.uses: path@ref` | Invokes the reusable workflow |
| Passing inputs | `with: {key: value}` | Send parameters to the called workflow |
| Passing secrets | `secrets: {name: value}` or `secrets: inherit` | Send sensitive data |
| Returning outputs | `on.workflow_call.outputs:` | Expose data back to the caller |
| Cross-repo call | `uses: org/repo/.github/workflows/file.yml@ref` | Shared workflows across repos |
| Nesting limit | 4 levels deep, 20 calls per file | Platform constraint |

Reusable workflows are the primary mechanism for standardizing CI/CD across an organization. In the next labs you will learn how to build custom **actions** (JavaScript, Docker, composite) for more granular, step-level reuse.
