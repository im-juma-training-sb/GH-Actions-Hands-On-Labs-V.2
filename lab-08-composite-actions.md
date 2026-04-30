# Lab 8: Composite Actions

**Duration:** 45-55 minutes  
**Level:** Intermediate  
**Prerequisites:** Labs 1-7 completed, repository `actions-fundamentals-lab` available

---

## Overview

Composite actions bundle multiple steps into a single reusable unit — like a macro you can call from any workflow. Unlike JavaScript actions (which require Node.js code) or Docker actions (which require a container), composite actions are written purely in YAML using the same `run:` and `uses:` syntax you already know. They are the simplest way to package a sequence of steps for reuse across workflows and repositories.

All exercises are performed directly in your GitHub repository.

---

## Exercise 1: Your First Composite Action (10 minutes)

### Bundle multiple steps behind a single `uses:` reference

1. Create `.github/actions/setup-project/action.yml`:

```yaml
name: 'Setup Project'
description: 'Checks out code, installs Node.js, and installs dependencies in one step'

inputs:
  node-version:
    description: 'Node.js version to install'
    required: false
    default: '20'

runs:
  using: 'composite'
  # ↑ Declares this as a composite action.
  #   No 'main' or 'image' key — instead, you define 'steps' directly.

  steps:
    - name: Checkout repository
      uses: actions/checkout@v4
      # ↑ Composite actions can use other actions via 'uses:'

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        # ↑ Access the composite action's declared inputs via inputs.<name>

    - name: Install dependencies
      shell: bash
      # ↑ REQUIRED in composite actions — every 'run:' step must specify 'shell:'
      #   This is different from regular workflows where 'bash' is the default.
      run: |
        if [ -f package-lock.json ]; then
          npm ci
          # npm ci → clean install from lock file (faster, deterministic)
        elif [ -f package.json ]; then
          npm install
        else
          echo "No package.json found — skipping npm install"
        fi
```

2. Commit to **main**.

3. Create `.github/workflows/test-setup.yml`:

```yaml
name: Test Setup Action

on:
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Setup project
        uses: ./.github/actions/setup-project
        # ↑ All three internal steps (checkout, setup-node, npm install) run as one unit
        with:
          node-version: '20'

      - name: Verify setup
        run: |
          echo "Node version: $(node --version)"
          echo "npm version: $(npm --version)"
          echo "Working directory contents:"
          ls -la
```

4. Commit to **main** and run the workflow.

5. In the run log, expand the "Setup project" step. Notice all three internal steps appear as sub-steps under the composite action.

### Key rule: `shell:` is mandatory

In composite action steps, every `run:` block **must** have an explicit `shell:` key. Valid values:

| Shell | Platform |
|-------|----------|
| `bash` | Linux, macOS |
| `pwsh` | Linux, macOS, Windows (PowerShell Core) |
| `python` | Any (runs as a Python script) |
| `sh` | Linux, macOS |
| `cmd` | Windows only |
| `powershell` | Windows only (Windows PowerShell) |

---

## Exercise 2: Composite Action with Outputs (10 minutes)

### Return computed values to the calling workflow

1. Create `.github/actions/git-info/action.yml`:

```yaml
name: 'Git Info'
description: 'Extracts useful git metadata from the current checkout'

outputs:
  short-sha:
    description: 'Short commit SHA (7 characters)'
    value: ${{ steps.extract.outputs.short-sha }}
    # ↑ Composite action outputs reference a specific step's output by ID
  branch:
    description: 'Current branch name'
    value: ${{ steps.extract.outputs.branch }}
  commit-message:
    description: 'Subject line of the current commit'
    value: ${{ steps.extract.outputs.message }}
  commit-timestamp:
    description: 'ISO timestamp of the current commit'
    value: ${{ steps.extract.outputs.timestamp }}

runs:
  using: 'composite'
  steps:
    - name: Extract git info
      id: extract
      # ↑ 'id' is required so outputs can be referenced as steps.<id>.outputs.<name>
      shell: bash
      run: |
        SHORT_SHA=$(git rev-parse --short=7 HEAD)
        # git rev-parse --short=7 HEAD
        #   rev-parse    → translates a git reference to its SHA
        #   --short=7    → abbreviate to 7 characters
        #   HEAD         → the current commit

        BRANCH=$(git rev-parse --abbrev-ref HEAD)
        # --abbrev-ref HEAD → returns the branch name (e.g., 'main') instead of the SHA
        #   If detached, returns 'HEAD'

        MESSAGE=$(git log -1 --format='%s')
        # git log -1 --format='%s'
        #   -1         → only the most recent commit
        #   --format='%s' → output only the subject line (first line of commit message)

        TIMESTAMP=$(git log -1 --format='%aI')
        # --format='%aI' → author date in strict ISO 8601 format

        echo "short-sha=$SHORT_SHA" >> "$GITHUB_OUTPUT"
        echo "branch=$BRANCH" >> "$GITHUB_OUTPUT"
        echo "message=$MESSAGE" >> "$GITHUB_OUTPUT"
        echo "timestamp=$TIMESTAMP" >> "$GITHUB_OUTPUT"

    - name: Display summary
      shell: bash
      run: |
        echo "SHA     : ${{ steps.extract.outputs.short-sha }}"
        echo "Branch  : ${{ steps.extract.outputs.branch }}"
        echo "Message : ${{ steps.extract.outputs.message }}"
        echo "Time    : ${{ steps.extract.outputs.timestamp }}"
```

2. Commit to **main**.

3. Create `.github/workflows/test-git-info.yml`:

```yaml
name: Test Git Info

on:
  workflow_dispatch:

jobs:
  info:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          # ↑ fetch-depth: 0 → fetch full history (not just the latest commit)
          #   Needed for accurate branch detection in some edge cases

      - name: Get git info
        id: git
        uses: ./.github/actions/git-info

      - name: Use outputs
        run: |
          echo "Deploying commit ${{ steps.git.outputs.short-sha }}"
          echo "From branch: ${{ steps.git.outputs.branch }}"
          echo "Change: ${{ steps.git.outputs.commit-message }}"
          echo "Authored: ${{ steps.git.outputs.commit-timestamp }}"
```

4. Commit to **main** and run the workflow. Verify outputs are populated in the "Use outputs" step.

---

## Exercise 3: Composite Action Calling Other Actions (10 minutes)

### Nest actions within a composite action

Composite actions can use `uses:` to call other actions — including other composite actions, JavaScript actions, or Docker actions.

1. Create `.github/actions/ci-quality-gate/action.yml`:

```yaml
name: 'CI Quality Gate'
description: 'Runs checkout, linting, and testing as a single reusable quality gate'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
  run-lint:
    description: 'Whether to run the lint step'
    required: false
    default: 'true'
  test-command:
    description: 'Command to run tests'
    required: false
    default: 'npm test'

outputs:
  lint-status:
    description: 'Did linting pass (passed/skipped)'
    value: ${{ steps.lint-result.outputs.status }}
  test-status:
    description: 'Did tests pass (passed/failed)'
    value: ${{ steps.test-result.outputs.status }}

runs:
  using: 'composite'
  steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Install dependencies
      shell: bash
      run: |
        if [ -f package-lock.json ]; then
          npm ci
        elif [ -f package.json ]; then
          npm install
        fi

    - name: Run linter
      id: lint-result
      shell: bash
      run: |
        if [ "${{ inputs.run-lint }}" = "true" ]; then
          if [ -f package.json ] && grep -q '"lint"' package.json; then
            # grep -q '"lint"' → quietly check if a "lint" script exists in package.json
            #   -q → quiet mode (no output, just exit code)
            npm run lint && echo "status=passed" >> "$GITHUB_OUTPUT" \
                         || echo "status=failed" >> "$GITHUB_OUTPUT"
          else
            echo "No lint script found in package.json"
            echo "status=skipped" >> "$GITHUB_OUTPUT"
          fi
        else
          echo "Linting disabled via input"
          echo "status=skipped" >> "$GITHUB_OUTPUT"
        fi

    - name: Run tests
      id: test-result
      shell: bash
      run: |
        if [ -f package.json ] && grep -q '"test"' package.json; then
          ${{ inputs.test-command }} && echo "status=passed" >> "$GITHUB_OUTPUT" \
                                     || echo "status=failed" >> "$GITHUB_OUTPUT"
        else
          echo "No test script found — marking as passed"
          echo "status=passed" >> "$GITHUB_OUTPUT"
        fi
```

2. Commit to **main**.

3. Create `.github/workflows/quality-gate.yml`:

```yaml
name: Quality Gate

on:
  workflow_dispatch:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - name: Run quality gate
        id: gate
        uses: ./.github/actions/ci-quality-gate
        with:
          node-version: '20'
          run-lint: 'true'

      - name: Report
        shell: bash
        run: |
          echo "Lint: ${{ steps.gate.outputs.lint-status }}"
          echo "Test: ${{ steps.gate.outputs.test-status }}"
```

4. Commit to **main** and run the workflow.

### Nesting depth

Composite actions can call other composite actions, but there is a limit of **10 levels** of nesting. In practice, 2-3 levels is typical.

---

## Exercise 4: Conditional Logic Inside Composite Actions (10 minutes)

### Use `if:` on individual steps within the composite

1. Create `.github/actions/deploy-preview/action.yml`:

```yaml
name: 'Deploy Preview'
description: 'Deploys a preview environment for PRs with optional cleanup'

inputs:
  action:
    description: 'What to do: deploy or cleanup'
    required: true
    # ↑ No default — caller must explicitly choose
  pr-number:
    description: 'Pull request number'
    required: true
  base-url:
    description: 'Base URL for preview environments'
    required: false
    default: 'https://preview.internal.example.com'

outputs:
  preview-url:
    description: 'URL of the deployed preview (empty on cleanup)'
    value: ${{ steps.deploy.outputs.url }}

runs:
  using: 'composite'
  steps:
    - name: Validate inputs
      shell: bash
      run: |
        if [ "${{ inputs.action }}" != "deploy" ] && [ "${{ inputs.action }}" != "cleanup" ]; then
          echo "ERROR: 'action' must be 'deploy' or 'cleanup', got '${{ inputs.action }}'"
          exit 1
        fi
        echo "Action: ${{ inputs.action }} for PR #${{ inputs.pr-number }}"

    - name: Deploy preview
      id: deploy
      if: inputs.action == 'deploy'
      # ↑ 'if:' works on composite steps just like regular workflow steps
      shell: bash
      run: |
        PREVIEW_URL="${{ inputs.base-url }}/pr-${{ inputs.pr-number }}"
        echo "Deploying preview to: $PREVIEW_URL"
        echo "Building assets..."
        echo "Uploading to preview server..."
        echo "Preview is live."
        echo "url=$PREVIEW_URL" >> "$GITHUB_OUTPUT"

    - name: Cleanup preview
      if: inputs.action == 'cleanup'
      shell: bash
      run: |
        PREVIEW_URL="${{ inputs.base-url }}/pr-${{ inputs.pr-number }}"
        echo "Removing preview at: $PREVIEW_URL"
        echo "Deleting deployed assets..."
        echo "Preview environment removed."

    - name: Post-action summary
      shell: bash
      run: |
        echo "Completed '${{ inputs.action }}' for PR #${{ inputs.pr-number }}"
```

2. Commit to **main**.

3. Create `.github/workflows/preview-deploy.yml`:

```yaml
name: Preview Deploy

on:
  workflow_dispatch:
    inputs:
      action:
        type: choice
        options:
          - deploy
          - cleanup
      pr-number:
        type: string
        default: '42'

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run deploy preview
        id: preview
        uses: ./.github/actions/deploy-preview
        with:
          action: ${{ inputs.action }}
          pr-number: ${{ inputs.pr-number }}

      - name: Show preview URL
        if: inputs.action == 'deploy'
        run: echo "Preview available at: ${{ steps.preview.outputs.preview-url }}"
```

4. Commit to **main** and run the workflow twice — once with `deploy`, once with `cleanup`.

5. Verify that the deploy run shows the URL output and the cleanup run does not.

---

## Exercise 5: Comparison — When to Use Which Action Type (5 minutes)

### Decision guide (conceptual review)

| Criteria | Composite | JavaScript | Docker |
|----------|-----------|-----------|--------|
| **Language** | YAML (shell commands) | JavaScript/TypeScript | Any (Python, Go, Bash, etc.) |
| **Startup speed** | Fastest (no build) | Fast (Node.js on runner) | Slowest (image build/pull) |
| **Platform support** | Linux, macOS, Windows | Linux, macOS, Windows | Linux only |
| **Complexity** | Low (YAML only) | Medium (requires Node.js code) | Medium-High (Dockerfile + code) |
| **Can call other actions** | Yes (`uses:` in steps) | No (must use toolkit APIs) | No |
| **Environment control** | Uses runner's environment | Uses runner's Node.js | Full (own OS, packages) |
| **Best for** | Bundling common step sequences | API integrations, complex logic | Custom runtimes, non-JS tools |

### Decision flowchart

```
Do you need a specific OS or tool not on the runner?
  YES → Docker action
  NO  ↓

Do you need complex logic, API calls, or heavy computation?
  YES → JavaScript action
  NO  ↓

Are you bundling existing steps/actions into a reusable unit?
  YES → Composite action
```

### Composite vs Reusable Workflow

| Aspect | Composite action | Reusable workflow |
|--------|-----------------|-------------------|
| Scope | Step-level (`uses:` in a step) | Job-level (`uses:` in a job) |
| Can define jobs | No (only steps) | Yes |
| Can use `services:` | No (inherits from caller job) | Yes |
| Runner selection | Inherits from caller job | Defines its own `runs-on:` |
| Secrets access | Inherits from caller job | Must be passed explicitly |
| Max nesting | 10 levels | 4 levels |
| Visibility in logs | Steps appear inline in the caller job | Separate job in the graph |

---

## Summary

| Concept | Key syntax | Purpose |
|---------|-----------|---------|
| Composite declaration | `runs.using: 'composite'` | Declares a composite action in action.yml |
| Steps with shell | `steps[].shell: bash` (required) | Every `run:` must specify the shell |
| Using other actions | `steps[].uses: actions/checkout@v4` | Call actions from within composite |
| Inputs | `inputs.<name>` in expressions | Parameters from the caller |
| Outputs | `steps.<id>.outputs.<name>` in `outputs:` value | Return data to the caller |
| Conditionals | `if:` on individual steps | Skip/run steps inside the composite |
| Local reference | `uses: ./.github/actions/<dir>` | Call from same repo (needs checkout) |
| Cross-repo | `uses: org/action-repo@v1` | Published in its own repository |

Composite actions are the fastest path to reusable automation when you do not need a custom runtime. Combined with JavaScript and Docker actions (Labs 6-7), you now have the full toolkit for building custom automation at any level of complexity.
