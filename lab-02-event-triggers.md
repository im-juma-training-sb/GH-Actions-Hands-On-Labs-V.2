# Lab 2: Event Triggers

**Duration:** 35-45 minutes  
**Level:** Beginner  
**Prerequisites:** Lab 1 completed, repository `actions-fundamentals-lab` available

---

## Overview

GitHub Actions workflows are event-driven. In this lab you will configure workflows to react to different repository events—pushes filtered by branch and path, pull request activity, scheduled cron jobs, and manual triggers with user-supplied inputs. By the end you will know how to control *when* your automation runs.

All exercises are performed directly in your GitHub repository.

---

## Exercise 1: Branch and Path Filtering (10 minutes)

### Control which pushes trigger a workflow

1. In your `actions-fundamentals-lab` repository, click **Add file** → **Create new file**.

2. Name the file `.github/workflows/filtered-push.yml`.

3. Paste the following:

```yaml
name: Filtered Push

on:
  push:
    branches:
      - main
      - develop
      - 'release/**'
    # ↑ Only trigger on pushes to main, develop, or any branch under release/
    # Glob pattern release/** matches release/v1, release/2.0, etc.
    paths:
      - 'src/**'
      - '*.js'
    # ↑ Further restrict: only trigger if changed files match these patterns
    # src/** = any file inside the src/ directory (recursive)
    # *.js   = any .js file at the repository root

jobs:
  report:
    runs-on: ubuntu-latest

    steps:
      - name: Show trigger context
        run: |
          echo "Branch  : ${{ github.ref_name }}"
          echo "SHA     : ${{ github.sha }}"
          echo "Actor   : ${{ github.actor }}"
          echo "Event   : ${{ github.event_name }}"
```

4. Commit to **main**.

5. **Test 1 — Edit README.md on main**:
   - Edit `README.md` (any change) and commit.
   - Go to **Actions** → The "Filtered Push" workflow does **NOT** run because `README.md` does not match `src/**` or `*.js`.

6. **Test 2 — Create a matching file**:
   - Click **Add file** → **Create new file**.
   - Name it `src/index.js`, paste `console.log("hello");`, commit to main.
   - Go to **Actions** → "Filtered Push" **runs** because `src/index.js` matches `src/**`.

### How filtering works

```
push event
  │
  ├── Does the branch match? ─── No ──→ Skip
  │         │
  │        Yes
  │         │
  └── Do changed paths match? ── No ──→ Skip
              │
             Yes
              │
         Trigger workflow
```

> **Note:** You cannot combine `paths:` and `paths-ignore:` in the same trigger—use one or the other.

---

## Exercise 2: Pull Request Events (10 minutes)

### React to pull request lifecycle activity

1. Create `.github/workflows/pr-events.yml`:

```yaml
name: PR Checks

on:
  pull_request:
    types: [opened, synchronize, reopened]
    # ↑ Activity types that fire this trigger:
    #   opened       — PR is first created
    #   synchronize  — New commits pushed to the PR's head branch
    #   reopened     — A previously closed PR is reopened
    branches:
      - main
    # ↑ Only fire when the PR targets the main branch

jobs:
  info:
    runs-on: ubuntu-latest

    steps:
      - name: Display PR metadata
        run: |
          echo "PR Number : #${{ github.event.pull_request.number }}"
          echo "Title     : ${{ github.event.pull_request.title }}"
          echo "Author    : ${{ github.event.pull_request.user.login }}"
          echo "Base      : ${{ github.event.pull_request.base.ref }}"
          echo "Head      : ${{ github.event.pull_request.head.ref }}"
          echo "Action    : ${{ github.event.action }}"

      - uses: actions/checkout@v4
        # ↑ Official action: checks out the merge commit of the PR
        #   (the result of merging head into base)

      - name: Count files in PR branch
        run: |
          echo "Total files in repository:"
          find . -type f | wc -l
          # find . -type f → list all files (-type f = regular files only)
          # wc -l          → count lines (one file per line)
```

2. Commit to **main**.

3. Create a branch and open a pull request:
   - From **Code** tab, click the branch dropdown → type `feature/test-pr` → **Create branch: feature/test-pr from main**.
   - Switch to that branch, edit `README.md`, commit.
   - Go to **Pull requests** → **New pull request** → base: `main`, compare: `feature/test-pr`.
   - Add a title and description → **Create pull request**.

4. Go to **Actions** → Observe "PR Checks" ran with activity type `opened`.

5. Push another commit to `feature/test-pr` (edit any file on that branch).

6. Go to **Actions** → "PR Checks" runs again with activity type `synchronize`.

### Common PR activity types

| Type | When |
|------|------|
| `opened` | PR created |
| `synchronize` | New commits pushed to head branch |
| `reopened` | Closed PR reopened |
| `closed` | PR closed or merged |
| `labeled` | Label added |
| `review_requested` | Review requested from a user or team |

---

## Exercise 3: Scheduled Workflows (Cron) (10 minutes)

### Automate tasks on a time-based schedule

1. Create `.github/workflows/scheduled.yml`:

```yaml
name: Nightly Report

on:
  schedule:
    - cron: '30 2 * * 1-5'
    # ↑ Cron expression — runs at 02:30 UTC, Monday through Friday
    #
    # ┌───────────── minute        (0-59)
    # │  ┌────────── hour          (0-23)
    # │  │  ┌─────── day of month  (1-31)
    # │  │  │  ┌──── month         (1-12)
    # │  │  │  │  ┌─ day of week   (0=Sun, 1=Mon … 6=Sat)
    # │  │  │  │  │
    # 30 2  *  *  1-5
    #
    # 1-5 means Monday through Friday (weekdays only)

  workflow_dispatch:
  # ↑ Always add this so you can test the workflow without waiting for the schedule

jobs:
  report:
    runs-on: ubuntu-latest

    steps:
      - name: Generate report timestamp
        run: |
          echo "=== Nightly Report ==="
          echo "Generated at: $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
          # date -u         → output in UTC
          # '+%Y-%m-%d ...' → custom format: 2026-04-29 02:30:00 UTC

      - name: Simulate report tasks
        run: |
          echo "• Checking repository health..."
          echo "• Counting open issues..."
          echo "• Summarizing recent activity..."
          echo "Report complete."
```

2. Commit to **main**.

3. Since you cannot wait for the schedule, **test manually**:
   - Go to **Actions** → select "Nightly Report" in the sidebar → **Run workflow** → branch `main` → **Run workflow**.

4. Click into the run and verify both steps produce output.

### Cron quick-reference

| Expression | Meaning |
|-----------|---------|
| `0 0 * * *` | Every day at midnight UTC |
| `*/15 * * * *` | Every 15 minutes |
| `0 9 * * 1` | Every Monday at 09:00 UTC |
| `0 0 1 * *` | First day of each month at midnight |
| `30 2 * * 1-5` | Weekdays at 02:30 UTC |

> **Important:** Scheduled workflows only run on the repository's default branch. Delays of up to 15 minutes are possible during periods of high Actions demand.

---

## Exercise 4: Manual Trigger with Inputs (10 minutes)

### Accept user-provided parameters at run time

1. Create `.github/workflows/greet-user.yml`:

```yaml
name: Greet User

on:
  workflow_dispatch:
    inputs:
      username:
        description: 'Name to greet'
        required: true
        # ↑ The user must provide a value before the workflow can start
        type: string
        default: 'World'
        # ↑ Pre-filled value shown in the UI (user can override)

      greeting-style:
        description: 'How formal should the greeting be?'
        required: true
        type: choice
        # ↑ Presents a dropdown with predefined options
        options:
          - casual
          - formal
          - enthusiastic

      include-timestamp:
        description: 'Append current UTC time to the greeting'
        required: false
        type: boolean
        # ↑ Renders as a checkbox in the UI
        default: true

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: Build greeting
        run: |
          USERNAME="${{ inputs.username }}"
          STYLE="${{ inputs.greeting-style }}"

          if [ "$STYLE" = "casual" ]; then
            MSG="Hey $USERNAME, what's up!"
          elif [ "$STYLE" = "formal" ]; then
            MSG="Good day, $USERNAME. Welcome."
          else
            MSG="HELLO $USERNAME!!! Great to see you!"
          fi

          echo "$MSG"

      - name: Show timestamp
        if: inputs.include-timestamp == 'true'
        # ↑ Boolean inputs are passed as strings — compare with 'true'/'false'
        run: |
          echo "Timestamp: $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
```

2. Commit to **main**.

3. Go to **Actions** → select **"Greet User"** → click **Run workflow**.

4. Fill in the inputs:
   - **Name to greet**: Enter your name
   - **How formal…**: Select `enthusiastic`
   - **Append current UTC time**: checked

5. Click **Run workflow**, then click into the run to see output.

6. Run again with different values—try `formal` with the timestamp unchecked.

### Input types reference

| Type | UI Control | Value in expressions |
|------|-----------|---------------------|
| `string` | Text field | The entered text |
| `choice` | Dropdown | The selected option string |
| `boolean` | Checkbox | `'true'` or `'false'` (string) |
| `number` | Text field (validated) | Numeric string |
| `environment` | Environment dropdown | Environment name string |

---

## Summary

| Trigger | Key Syntax | Use Case |
|---------|-----------|----------|
| Push (filtered) | `on: push: branches: / paths:` | Run only for relevant code changes |
| Pull Request | `on: pull_request: types: / branches:` | Validate PRs before merge |
| Schedule | `on: schedule: - cron: '...'` | Nightly builds, reports, cleanup |
| Manual with inputs | `on: workflow_dispatch: inputs:` | On-demand tasks with parameters |

You now know how to control *when* workflows execute. In the next lab you will learn how to control *whether* individual jobs and steps run using conditional logic and job dependencies.
