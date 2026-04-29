# Lab 3: Conditional Execution

**Duration:** 35-45 minutes  
**Level:** Beginner-to-Intermediate  
**Prerequisites:** Labs 1-2 completed, repository `actions-fundamentals-lab` available

---

## Overview

Not every job or step should run every time. In this lab you will use `needs:` to create job dependencies, `if:` expressions to conditionally skip or execute jobs and steps, and status check functions to handle failures gracefully. These are the building blocks for controlling workflow logic—without jumping into full CI/CD pipelines.

All exercises are performed directly in your GitHub repository.

---

## Exercise 1: Job Dependencies with `needs` (10 minutes)

### Define execution order between jobs

By default all jobs in a workflow run **in parallel**. Use `needs:` to make one job wait for another.

1. Create `.github/workflows/job-dependencies.yml`:

```yaml
name: Job Dependencies

on:
  workflow_dispatch:
  # ↑ Manual trigger only — no automatic runs while experimenting

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - name: Initialize environment
        run: |
          echo "Setting up environment..."
          echo "Done."

  lint:
    needs: setup
    # ↑ 'lint' will not start until 'setup' completes successfully
    runs-on: ubuntu-latest
    steps:
      - name: Run linter
        run: |
          echo "Linting code..."
          echo "No issues found."

  test:
    needs: setup
    # ↑ 'test' also depends on 'setup' but is independent of 'lint'
    #    → lint and test run in PARALLEL after setup finishes
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        run: |
          echo "Running test suite..."
          echo "All 42 tests passed."

  report:
    needs: [lint, test]
    # ↑ Array syntax — 'report' waits for BOTH lint AND test to complete
    runs-on: ubuntu-latest
    steps:
      - name: Generate summary
        run: |
          echo "All upstream jobs succeeded."
          echo "   - setup  [passed]"
          echo "   - lint   [passed]"
          echo "   - test   [passed]"
          echo "Generating final report..."
```

2. Commit to **main**.

3. Go to **Actions** → select **"Job Dependencies"** → **Run workflow**.

4. Click into the run and observe the **job graph**:

```
  setup
   / \
 lint  test    ← run in parallel
   \ /
  report       ← waits for both
```

5. Verify timing: `lint` and `test` start at approximately the same time, after `setup` finishes. `report` starts only after both complete.

### What happens if an upstream job fails?

If `lint` fails, `report` is **automatically skipped** (its dependency was not satisfied). `test` still runs because it only depends on `setup`.

---

## Exercise 2: Conditional Steps with `if` (10 minutes)

### Skip or run individual steps based on context

1. Create `.github/workflows/conditional-steps.yml`:

```yaml
name: Conditional Steps

on:
  workflow_dispatch:
    inputs:
      log-level:
        description: 'Logging verbosity'
        type: choice
        options:
          - normal
          - verbose
        default: 'normal'

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Start build
        run: echo "Building project..."

      - name: Verbose diagnostics
        if: inputs.log-level == 'verbose'
        # ↑ This step only runs when the user selects 'verbose'
        #   Condition is a GitHub Actions expression (no ${{ }} needed at the job/step if: level)
        run: |
          echo "=== Verbose Mode ==="
          echo "Runner name : ${{ runner.name }}"
          echo "Runner OS   : ${{ runner.os }}"
          echo "Runner arch : ${{ runner.arch }}"
          echo "Run ID      : ${{ github.run_id }}"
          echo "Run number  : ${{ github.run_number }}"
          echo "Workflow     : ${{ github.workflow }}"
          uname -a
          # uname -a → print all system information (kernel, hostname, version, etc.)

      - name: Show actor info
        if: github.actor != 'dependabot[bot]'
        # ↑ Skip this step when Dependabot triggers the workflow
        #   Useful for suppressing noisy notifications from bots
        run: |
          echo "Triggered by: ${{ github.actor }}"
          echo "Actor ID    : ${{ github.actor_id }}"

      - name: Build complete
        run: echo "Build finished successfully."
```

2. Commit to **main**.

3. Run the workflow twice:
   - First with **log-level = normal** -> "Verbose diagnostics" step is skipped (shown greyed out).
   - Second with **log-level = verbose** → All steps execute.

4. In the run log, notice skipped steps display the reason: *"Conditional was not met"*.

### Expression syntax notes

| Expression | Meaning |
|-----------|---------|
| `if: github.ref == 'refs/heads/main'` | Equality check |
| `if: contains(github.event.head_commit.message, '[skip ci]')` | Substring search |
| `if: startsWith(github.ref, 'refs/tags/')` | Prefix match |
| `if: github.event_name == 'pull_request'` | Event type check |
| `if: inputs.dry-run == 'true'` | Input comparison (always strings) |

---

## Exercise 3: Conditional Jobs Based on Event Context (10 minutes)

### Run entire jobs only when conditions are met

1. Create `.github/workflows/conditional-jobs.yml`:

```yaml
name: Conditional Jobs

on:
  push:
    branches: [main, develop, 'feature/**']
  workflow_dispatch:

jobs:
  always-runs:
    runs-on: ubuntu-latest
    steps:
      - name: Common step
        run: |
          echo "This job runs on every qualifying push."
          echo "Branch: ${{ github.ref_name }}"

  main-only:
    if: github.ref == 'refs/heads/main'
    # ↑ Job-level condition — entire job is skipped if false
    runs-on: ubuntu-latest
    steps:
      - name: Main branch action
        run: echo "This only executes on the main branch."

  feature-only:
    if: startsWith(github.ref, 'refs/heads/feature/')
    # ↑ startsWith() matches any branch whose full ref starts with refs/heads/feature/
    runs-on: ubuntu-latest
    steps:
      - name: Feature branch notice
        run: |
          echo "Feature branch detected: ${{ github.ref_name }}"
          echo "Remember to open a PR when ready."

  skip-bot-commits:
    if: github.actor != 'github-actions[bot]' && github.actor != 'dependabot[bot]'
    # ↑ Logical AND (&&) — both conditions must be true
    #   Skips the job when automated bots push commits
    runs-on: ubuntu-latest
    steps:
      - name: Human-triggered work
        run: echo "Commit by a human: ${{ github.actor }}"
```

2. Commit to **main**.

3. Go to **Actions** → observe which jobs ran:
   - `always-runs` — ran
   - `main-only` — ran (because you committed to main)
   - `feature-only` — skipped
   - `skip-bot-commits` — ran (you are not a bot)

4. Create a branch `feature/lab3-test`, make any commit, and verify:
   - `always-runs` — ran
   - `main-only` — skipped
   - `feature-only` — ran
   - `skip-bot-commits` — ran

### Job-level `if` vs step-level `if`

| Scope | Effect when false | Dependent jobs |
|-------|-------------------|----------------|
| **Job** `if:` | Entire job is skipped, no runner is provisioned | Jobs that `needs:` this job are also skipped |
| **Step** `if:` | Single step is skipped, remaining steps continue | N/A — steps are within a job |

---

## Exercise 4: Handling Failures with Status Functions (10 minutes)

### Use `always()`, `failure()`, and `success()` to control behavior after failures

1. Create `.github/workflows/failure-handling.yml`:

```yaml
name: Failure Handling

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Simulate build
        run: echo "Building..."

      - name: Simulate failing test
        id: tests
        # ↑ 'id' assigns a unique identifier to this step within the job.
        #   Unlike 'name' (which is just a display label), 'id' is a machine-readable
        #   key you can reference later in expressions, for example:
        #     steps.tests.outcome   → 'success', 'failure', 'cancelled', or 'skipped'
        #     steps.tests.outputs.* → any outputs this step sets
        #   Without an 'id', the step's result cannot be referenced by other steps.
        run: |
          echo "Running tests..."
          exit 1
          # ↑ Non-zero exit code signals failure — this step FAILS

      - name: This step is skipped
        run: echo "You will never see this."
        # ↑ After a failure, subsequent steps are skipped by default

      - name: Cleanup (runs even on failure)
        if: always()
        # ↑ always() overrides default behavior — step runs regardless of prior failures
        run: |
          echo "Cleaning up temporary files..."
          echo "Cleanup complete (even though tests failed)."

      - name: Failure notification
        if: failure()
        # ↑ failure() → runs ONLY if a previous step in this job failed
        run: |
          echo "ERROR: Something went wrong!"
          echo "Failed step: tests"

  post-build:
    needs: build
    if: always()
    # ↑ Without always(), this job would be skipped because 'build' failed.
    #   With always(), it runs regardless of the outcome of 'build'.
    runs-on: ubuntu-latest
    steps:
      - name: Check upstream result
        run: |
          echo "Build job result: ${{ needs.build.result }}"
          # ↑ needs.<job_id>.result → 'success', 'failure', 'cancelled', or 'skipped'

      - name: Conditional on upstream failure
        if: needs.build.result == 'failure'
        # ↑ Access the specific result of an upstream job
        run: echo "WARNING: Build failed — notifying team."

      - name: Conditional on upstream success
        if: needs.build.result == 'success'
        run: echo "Build passed — proceeding."
```

2. Commit to **main**.

3. Run the workflow manually and observe:
   - `build` job: step "Simulate failing test" **fails** (red X).
   - "This step is skipped" shows ⊘ skipped.
   - "Cleanup" **runs** (because of `always()`).
   - "Failure notification" **runs** (because of `failure()`).
   - `post-build` job **runs** (because of `always()` on the job).
   - "Check upstream result" prints `failure`.
   - "Conditional on upstream failure" **runs**.
   - "Conditional on upstream success" is ⊘ skipped.

### Status check functions

| Function | When it evaluates to `true` |
|----------|----------------------------|
| `success()` | All previous steps/jobs succeeded (this is the **default** implicit condition) |
| `failure()` | At least one previous step/job failed |
| `always()` | Always — runs even after failure or cancellation |
| `cancelled()` | Workflow run was cancelled |

---

## Summary

| Concept | Syntax | Purpose |
|---------|--------|---------|
| Job dependency | `needs: [job-a, job-b]` | Control execution order |
| Step condition | `if: <expression>` on a step | Skip/run individual steps |
| Job condition | `if: <expression>` on a job | Skip/run entire jobs |
| Status functions | `always()`, `failure()`, `success()`, `cancelled()` | Override default skip-on-failure behavior |
| Upstream result | `needs.<job>.result` | Inspect outcome of dependency |

You now have full control over *when* jobs and steps execute within a workflow. These patterns form the foundation for building sophisticated automation without yet needing CI/CD-specific tooling.
