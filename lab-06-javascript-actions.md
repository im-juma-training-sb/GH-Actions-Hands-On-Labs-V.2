# Lab 6: JavaScript Actions

**Duration:** 55-70 minutes  
**Level:** Intermediate  
**Prerequisites:** Labs 1-5 completed, basic JavaScript/Node.js familiarity, repository `actions-fundamentals-lab` available

---

## Overview

Custom JavaScript actions let you encapsulate logic into reusable steps that run directly on the runner's Node.js runtime. They are fast (no container startup), cross-platform (Linux, macOS, Windows), and integrate deeply with the Actions toolkit for logging, inputs/outputs, and API access. In this lab you will build real-world JavaScript actions from scratch, covering the full lifecycle: metadata, implementation, dependency bundling, and consumption in a workflow.

All exercises are performed directly in your GitHub repository.

---

## Exercise 1: Action Metadata and Project Structure (10 minutes)

### Understand the anatomy of a JavaScript action

Every action lives in its own directory and requires an `action.yml` metadata file.

1. Create `.github/actions/issue-greeter/action.yml`:

```yaml
name: 'Issue Greeter'
description: 'Automatically posts a welcome comment on newly opened issues'
# ↑ 'name' and 'description' are required — they appear in logs and the Marketplace

inputs:
  github-token:
    description: 'Token to authenticate GitHub API calls'
    required: true
    # ↑ The caller must provide this input. Without it the action fails immediately.
  welcome-message:
    description: 'Custom message to post as the comment body'
    required: false
    default: 'Thanks for opening this issue! The team will review it shortly.'
    # ↑ 'default' provides a fallback when the caller does not supply a value.

outputs:
  comment-id:
    description: 'The ID of the created comment'
    # ↑ Declared outputs can be consumed by subsequent steps in the caller workflow.

runs:
  using: 'node20'
  # ↑ Tells GitHub to execute the action using Node.js 20.
  #   Supported values: node16, node20
  main: 'index.js'
  # ↑ Entry point file — GitHub runs: node index.js
```

2. Commit to **main**.

### File layout for a JavaScript action

```
.github/actions/issue-greeter/
  action.yml       <- metadata (required)
  index.js         <- entry point (specified in action.yml)
  package.json     <- declares npm dependencies
  node_modules/    <- bundled dependencies (must be committed)
```

> **Why commit `node_modules`?** GitHub does not run `npm install` before executing your action. All dependencies must be present in the repository. Alternatively you can bundle with `ncc` (covered in Exercise 5).

---

## Exercise 2: Implement the Issue Greeter Action (15 minutes)

### Write the JavaScript logic using the Actions toolkit

1. Create `.github/actions/issue-greeter/package.json`:

```json
{
  "name": "issue-greeter",
  "version": "1.0.0",
  "description": "Posts a welcome comment on new issues",
  "main": "index.js",
  "dependencies": {
    "@actions/core": "^1.10.0",
    "@actions/github": "^6.0.0"
  }
}
```

**Dependency explanation:**
- `@actions/core` — provides `getInput()`, `setOutput()`, `setFailed()`, logging functions
- `@actions/github` — provides an authenticated Octokit REST client and context about the current event

2. Commit to **main**.

3. Create `.github/actions/issue-greeter/index.js`:

```javascript
const core = require('@actions/core');
// ↑ Import the core toolkit — used for inputs, outputs, logging, and failure signaling

const github = require('@actions/github');
// ↑ Import the github toolkit — provides Octokit client and event context

async function run() {
  try {
    // --- Read inputs declared in action.yml ---
    const token = core.getInput('github-token', { required: true });
    // ↑ getInput(name, options)
    //   - Reads the input value passed by the caller
    //   - { required: true } throws if the value is empty

    const message = core.getInput('welcome-message');
    // ↑ If not supplied by the caller, returns the 'default' from action.yml

    // --- Access event context ---
    const context = github.context;
    // ↑ github.context contains:
    //   .eventName   — e.g., 'issues', 'pull_request'
    //   .payload     — full webhook payload
    //   .repo        — { owner, repo }
    //   .issue       — { owner, repo, number } (shortcut for issue/PR events)

    if (context.eventName !== 'issues' || context.payload.action !== 'opened') {
      core.info('Not a newly opened issue — skipping.');
      // ↑ core.info() writes an info-level log line visible in the Actions UI
      return;
    }

    const issueNumber = context.payload.issue.number;
    const issueAuthor = context.payload.issue.user.login;

    core.info(`Issue #${issueNumber} opened by ${issueAuthor}`);

    // --- Create authenticated API client ---
    const octokit = github.getOctokit(token);
    // ↑ getOctokit(token) returns an Octokit instance authenticated with the provided token.
    //   All REST API calls made through this client are authenticated.

    // --- Post the comment ---
    const personalizedMessage = message.replace('{author}', issueAuthor);
    // ↑ Allow callers to use {author} placeholder in their custom message

    const { data: comment } = await octokit.rest.issues.createComment({
      owner: context.repo.owner,
      repo: context.repo.repo,
      issue_number: issueNumber,
      body: personalizedMessage
    });
    // ↑ REST API call: POST /repos/{owner}/{repo}/issues/{issue_number}/comments
    //   Returns the created comment object

    core.info(`Comment posted: ${comment.html_url}`);

    // --- Set outputs for downstream steps ---
    core.setOutput('comment-id', comment.id.toString());
    // ↑ setOutput(name, value) makes the value available to subsequent steps
    //   via: steps.<step-id>.outputs.comment-id

  } catch (error) {
    core.setFailed(`Action failed: ${error.message}`);
    // ↑ setFailed(message) sets the step status to failed and logs the error.
    //   The workflow will proceed according to normal failure rules (skip subsequent steps).
  }
}

run();
```

4. Commit to **main**.

### Toolkit function reference

| Function | Purpose |
|----------|---------|
| `core.getInput(name)` | Read an input declared in action.yml |
| `core.setOutput(name, value)` | Set an output for downstream steps |
| `core.setFailed(message)` | Mark the step as failed |
| `core.info(message)` | Info-level log |
| `core.warning(message)` | Warning annotation (yellow) |
| `core.error(message)` | Error annotation (red) |
| `core.debug(message)` | Debug log (only visible when debug logging enabled) |
| `core.setSecret(value)` | Register a value for log redaction |
| `github.getOctokit(token)` | Create authenticated API client |
| `github.context` | Current event payload and repo info |

---

## Exercise 3: Install Dependencies and Test the Action (10 minutes)

### Bundle node_modules so the action can execute

GitHub does not run `npm install` before an action. You must commit the dependencies.

1. Create `.github/workflows/setup-action-deps.yml`:

```yaml
name: Setup Action Dependencies

on:
  workflow_dispatch:

jobs:
  install:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
        # ↑ Official action: configures the specified Node.js version on the runner

      - name: Install issue-greeter dependencies
        run: |
          cd .github/actions/issue-greeter
          npm install --production
          # npm install --production
          #   --production → skip devDependencies, install only runtime deps
          # This creates the node_modules/ directory

      - name: Commit node_modules
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          # ↑ Configure git identity for the commit (required in CI)

          git add .github/actions/issue-greeter/node_modules
          # ↑ Stage the entire node_modules directory

          git diff --staged --quiet && echo "No changes to commit" || \
            (git commit -m "chore: bundle issue-greeter dependencies" && git push)
          # git diff --staged --quiet → exits 0 if nothing staged (no changes)
          # || → if there ARE changes, commit and push
          # This avoids failing when node_modules is already up to date
```

2. Commit to **main**.

3. Go to **Actions** -> **"Setup Action Dependencies"** -> **Run workflow**.

4. Wait for the workflow to complete. It commits `node_modules/` back to the repository.

5. Verify: navigate to `.github/actions/issue-greeter/node_modules/` in the Code tab and confirm packages are present.

---

## Exercise 4: Use the Action in a Workflow (10 minutes)

### Wire up the issue greeter to respond to new issues

1. Create `.github/workflows/issue-welcome.yml`:

```yaml
name: Issue Welcome

on:
  issues:
    types: [opened]
    # ↑ Trigger only when a new issue is opened (not edited, closed, etc.)

permissions:
  issues: write
  # ↑ Grant GITHUB_TOKEN permission to create comments on issues

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        # ↑ Required so that GitHub can find the action source in .github/actions/

      - name: Welcome the issue author
        id: greeter
        uses: ./.github/actions/issue-greeter
        # ↑ Local action reference — path relative to the repository root.
        #   GitHub reads action.yml from this directory and executes index.js.
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          welcome-message: |
            Hi @{author}, thanks for reporting this!
            A team member will triage this issue within 24 hours.

      - name: Log the comment ID
        run: echo "Created comment ID: ${{ steps.greeter.outputs.comment-id }}"
        # ↑ Access the output set by core.setOutput('comment-id', ...)
```

2. Commit to **main**.

3. **Test it**: Go to the **Issues** tab -> **New issue** -> add a title and body -> **Submit new issue**.

4. Within seconds, a comment from **github-actions[bot]** should appear on the issue with your custom message.

5. Go to **Actions** -> click the "Issue Welcome" run -> expand the steps to see the action's log output.

### Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Cannot find module @actions/core" | `node_modules` not committed | Re-run the setup workflow from Exercise 3 |
| "Resource not accessible by integration" | Missing `permissions:` | Add `issues: write` to the workflow |
| Action does not trigger | Wrong event type | Ensure `on: issues: types: [opened]` |

---

## Exercise 5: Bundle with ncc (Alternative to Committing node_modules) (10 minutes)

### Compile the action into a single file for cleaner distribution

`@vercel/ncc` compiles a Node.js project and all its dependencies into a single `dist/index.js` file. This eliminates the need to commit `node_modules/`.

1. Create `.github/workflows/bundle-action.yml`:

```yaml
name: Bundle Action

on:
  workflow_dispatch:

jobs:
  bundle:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install ncc globally
        run: npm install -g @vercel/ncc
        # ↑ @vercel/ncc is a compiler that bundles a Node.js module into a single file.
        #   -g installs it globally so the 'ncc' command is available on PATH.

      - name: Bundle the action
        run: |
          cd .github/actions/issue-greeter
          npm install
          ncc build index.js -o dist
          # ncc build <entry> -o <output-dir>
          #   <entry>       → the main JS file
          #   -o dist       → output the bundled file to ./dist/index.js
          # The result is a single file with all dependencies inlined.

      - name: Show bundle size
        run: |
          ls -lh .github/actions/issue-greeter/dist/index.js
          # ls -lh → list files in long format (-l) with human-readable sizes (-h)
          echo "Single file contains all dependencies"

      - name: Commit bundled output
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .github/actions/issue-greeter/dist/
          git diff --staged --quiet && echo "No changes" || \
            (git commit -m "chore: bundle issue-greeter with ncc" && git push)
```

2. Commit to **main** and run the workflow.

3. After it completes, update `.github/actions/issue-greeter/action.yml` to point at the bundle:

```yaml
runs:
  using: 'node20'
  main: 'dist/index.js'
  # ↑ Changed from 'index.js' to 'dist/index.js'
  #   Now GitHub runs the bundled file — node_modules/ is no longer needed at runtime.
```

4. Commit the change. The action now uses the compiled bundle.

### ncc vs committed node_modules

| Approach | Pros | Cons |
|----------|------|------|
| Commit `node_modules/` | Simple, no build step | Large diffs, many files in repo |
| Bundle with `ncc` | Single file, small footprint | Requires a build/bundle workflow |

Most published actions on the Marketplace use the `ncc` approach.

---

## Exercise 6: Real Use Case — PR Size Labeler (15 minutes)

### Build an action that labels pull requests by size

This action counts changed lines in a PR and applies a size label (small, medium, large).

1. Create `.github/actions/pr-size-labeler/action.yml`:

```yaml
name: 'PR Size Labeler'
description: 'Labels pull requests based on the number of changed lines'

inputs:
  github-token:
    description: 'GitHub token for API access'
    required: true
  small-threshold:
    description: 'Maximum lines for a small PR'
    required: false
    default: '50'
  medium-threshold:
    description: 'Maximum lines for a medium PR'
    required: false
    default: '200'

outputs:
  size-label:
    description: 'The label that was applied (size/S, size/M, size/L)'
  total-changes:
    description: 'Total number of changed lines'

runs:
  using: 'node20'
  main: 'index.js'
```

2. Create `.github/actions/pr-size-labeler/package.json`:

```json
{
  "name": "pr-size-labeler",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@actions/core": "^1.10.0",
    "@actions/github": "^6.0.0"
  }
}
```

3. Create `.github/actions/pr-size-labeler/index.js`:

```javascript
const core = require('@actions/core');
const github = require('@actions/github');

async function run() {
  try {
    const token = core.getInput('github-token', { required: true });
    const smallMax = parseInt(core.getInput('small-threshold'), 10);
    const mediumMax = parseInt(core.getInput('medium-threshold'), 10);
    // ↑ parseInt(string, 10)
    //   - Converts the string input to an integer
    //   - 10 is the radix (base-10 / decimal)

    const octokit = github.getOctokit(token);
    const context = github.context;

    if (context.eventName !== 'pull_request') {
      core.info('Not a pull_request event — skipping.');
      return;
    }

    const prNumber = context.payload.pull_request.number;

    // Fetch the list of files changed in this PR
    const { data: files } = await octokit.rest.pulls.listFiles({
      owner: context.repo.owner,
      repo: context.repo.repo,
      pull_number: prNumber,
      per_page: 100
      // ↑ per_page: 100 — fetch up to 100 files per API call (max allowed)
    });

    // Sum additions and deletions across all files
    const totalChanges = files.reduce((sum, file) => {
      return sum + file.additions + file.deletions;
    }, 0);
    // ↑ Array.reduce(callback, initialValue)
    //   Iterates over each file, accumulating total additions + deletions

    core.info(`PR #${prNumber}: ${totalChanges} lines changed across ${files.length} files`);

    // Determine size label
    let label;
    if (totalChanges <= smallMax) {
      label = 'size/S';
    } else if (totalChanges <= mediumMax) {
      label = 'size/M';
    } else {
      label = 'size/L';
    }

    core.info(`Applying label: ${label}`);

    // Remove any existing size labels first
    const existingLabels = context.payload.pull_request.labels || [];
    for (const existing of existingLabels) {
      if (existing.name.startsWith('size/')) {
        await octokit.rest.issues.removeLabel({
          owner: context.repo.owner,
          repo: context.repo.repo,
          issue_number: prNumber,
          name: existing.name
        });
        core.info(`Removed old label: ${existing.name}`);
      }
    }

    // Apply the new label
    await octokit.rest.issues.addLabels({
      owner: context.repo.owner,
      repo: context.repo.repo,
      issue_number: prNumber,
      labels: [label]
      // ↑ addLabels expects an array — multiple labels can be added at once
    });

    // Set outputs
    core.setOutput('size-label', label);
    core.setOutput('total-changes', totalChanges.toString());

  } catch (error) {
    core.setFailed(`Action failed: ${error.message}`);
  }
}

run();
```

4. Commit all three files to **main**.

5. Update the setup workflow to also install this action's dependencies (or run `npm install --production` in `.github/actions/pr-size-labeler/` and commit `node_modules`).

6. Create `.github/workflows/pr-size.yml`:

```yaml
name: PR Size

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  pull-requests: write
  issues: write
  # ↑ 'issues: write' is needed because the labels API uses the issues endpoint

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Apply size label
        id: size
        uses: ./.github/actions/pr-size-labeler
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          small-threshold: '30'
          medium-threshold: '150'

      - name: Report
        run: |
          echo "Label applied: ${{ steps.size.outputs.size-label }}"
          echo "Total changes: ${{ steps.size.outputs.total-changes }}"
```

7. Commit to **main**.

8. Open a pull request (create a branch, make changes, open PR). The action should automatically apply a `size/S`, `size/M`, or `size/L` label.

---

## Summary

| Concept | Key detail |
|---------|-----------|
| Action metadata | `action.yml` with `name`, `inputs`, `outputs`, `runs.using: node20` |
| Entry point | `runs.main: index.js` (or `dist/index.js` if bundled) |
| Toolkit: core | `getInput()`, `setOutput()`, `setFailed()`, logging |
| Toolkit: github | `getOctokit(token)`, `context.payload`, `context.repo` |
| Dependencies | Commit `node_modules/` or bundle with `ncc` |
| Local reference | `uses: ./.github/actions/<dir>` (requires `actions/checkout` first) |
| Cross-repo reference | `uses: org/repo@v1` (publish to a separate repo or Marketplace) |

JavaScript actions are best when you need fast startup, cross-platform support, and deep GitHub API integration. Next you will learn Docker actions for cases where you need a specific runtime environment or non-JavaScript tooling.
