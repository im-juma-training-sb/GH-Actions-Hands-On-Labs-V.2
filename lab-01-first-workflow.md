# Lab 1: Your First Workflow

**Duration:** 30-40 minutes  
**Level:** Beginner  
**Prerequisites:** GitHub account with access to create repositories

---

## Overview

In this lab you will create a GitHub repository, write your first workflow file from scratch, trigger it automatically, and explore the Actions UI to read logs. By the end you will understand the anatomy of a workflow file and be comfortable navigating run output.

All exercises are performed directly in your GitHub repository—no local tooling required.

---

## Exercise 1: Create Your Training Repository (5 minutes)

### Set up a dedicated sandbox for all Actions labs

1. Sign in to **github.com**.

2. Click the **+** icon (top-right) → **New repository**.

3. Configure the repository:
   - **Repository name**: `actions-fundamentals-lab`
   - **Description**: `Hands-on GitHub Actions fundamentals training`
   - **Visibility**: Select **Internal**
     - *Internal repos are visible to all members of your GitHub Enterprise organization while remaining hidden from the public*
   - Check **Add a README file**
   - Click **Create repository**

4. Verify you see `README.md` listed in the repository root.

### What just happened?

You now have a repository with a default `main` branch that includes a single `README.md` commit. Every workflow you create in the `.github/workflows/` directory will be picked up automatically by GitHub Actions.

---

## Exercise 2: Create a Hello World Workflow (10 minutes)

### Write and commit your first workflow file

1. In your repository, click the **Actions** tab.

2. GitHub presents starter templates. Click **"set up a workflow yourself"** (link near the top).

3. GitHub opens an in-browser editor pre-filled with a template. **Delete all content** and paste the following:

```yaml
name: Hello World
# ↑ Display name shown in the Actions tab sidebar

on:
  push:
    branches: [main]
  # ↑ Trigger this workflow every time code is pushed to the 'main' branch

jobs:
  greet:
    # ↑ Job identifier — must be unique within this workflow
    runs-on: ubuntu-latest
    # ↑ Use the latest GitHub-hosted Ubuntu runner

    steps:
      - name: Say hello
        # ↑ Human-readable label shown in logs
        run: echo "Hello, GitHub Actions!"
        # ↑ Execute a shell command on the runner

      - name: Print runner details
        run: |
          echo "Runner OS   : $RUNNER_OS"
          echo "Runner Arch : $RUNNER_ARCH"
          echo "Workspace   : $GITHUB_WORKSPACE"
        # ↑ The pipe (|) starts a multi-line YAML scalar — each line runs sequentially in the same shell

      - name: Show date and time
        run: date -u
        # ↑ -u flag outputs time in UTC to avoid timezone ambiguity
```

4. In the file path above the editor, confirm the name is `.github/workflows/hello-world.yml`.

5. Click **Commit changes…** → Keep "Commit directly to the `main` branch" → Click **Commit changes**.

### What just happened?

- GitHub created the directory `.github/workflows/` and the file `hello-world.yml` inside it.
- Because the commit targets `main`, the `on: push` trigger fires immediately.
- The workflow is now executing on a cloud-hosted runner.

---

## Exercise 3: Explore the Actions UI (10 minutes)

### Read logs and understand the execution flow

1. Click the **Actions** tab.

2. You should see a workflow run named after your commit message (e.g., *"Create hello-world.yml"*). Status indicators:
   - Yellow dot = In progress
   - Green check = Success
   - Red X = Failed

3. Click on the run to open its **summary page**. Note:
   - **Event**: `push`
   - **Branch**: `main`
   - **Actor**: your username
   - A visual job graph showing `greet`

4. Click the **greet** job box.

5. Expand each step to see its log output:
   - **Set up job** — GitHub automatically provisions the runner
   - **Say hello** — Displays `Hello, GitHub Actions!`
   - **Print runner details** — Shows OS, architecture, workspace path
   - **Show date and time** — UTC timestamp
   - **Complete job** — Cleanup (automatic)

6. Click the **⋯** menu on the run and select **"View workflow file"** to jump directly to the YAML source.

### Key Takeaway

Every step's output is logged in real time. If a step exits with a non-zero code, the job is marked failed and subsequent steps are skipped (by default).

---

## Exercise 4: Edit the Workflow and Re-trigger (10 minutes)

### Iterate on your workflow by adding a new step

Every commit to `main` triggers the workflow again. Use this to verify changes.

1. Go to the **Code** tab.

2. Navigate to `.github/workflows/hello-world.yml` and click the **pencil icon** (✏️) to edit.

3. Add the following step **below** the existing `Show date and time` step:

```yaml
      - name: List environment variables
        run: env | sort | head -20
        # env        → print all environment variables
        # sort       → alphabetical order
        # head -20   → show only the first 20 lines to keep output concise
```

4. Click **Commit changes…** → commit directly to `main`.

5. Go to the **Actions** tab. A **new run** appears because you just pushed to `main`.

6. Click into the run → open the **greet** job → expand **List environment variables**.

7. Scan the output and look for variables GitHub injects automatically, such as:
   - `GITHUB_ACTOR` — the username that triggered the run
   - `GITHUB_REPOSITORY` — `<owner>/<repo>`
   - `GITHUB_REF` — the branch ref (`refs/heads/main`)
   - `RUNNER_OS` — the runner operating system

### Key Takeaway

Editing a workflow file in the default branch immediately triggers a new run. This is the fastest feedback loop while developing workflows—commit a change, watch it execute, read the logs, repeat.

---

## Summary

| Concept | What you learned |
|---------|-----------------|
| Workflow file location | `.github/workflows/*.yml` |
| Trigger events | `push` (on branch) |
| Job & step structure | `jobs.<id>.steps[].run` |
| Runner selection | `runs-on: ubuntu-latest` |
| Actions UI navigation | Runs → Jobs → Steps → Logs |

You now have a working workflow and understand how to create, trigger, and inspect it. In the next lab you will explore additional event triggers and filtering.
