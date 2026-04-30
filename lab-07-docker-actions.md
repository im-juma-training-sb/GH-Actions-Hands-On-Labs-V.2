# Lab 7: Docker Actions

**Duration:** 50-60 minutes  
**Level:** Intermediate  
**Prerequisites:** Labs 1-6 completed, basic Docker/containerization knowledge, repository `actions-fundamentals-lab` available

---

## Overview

Docker actions run your code inside a container that you define. This gives you full control over the operating system, installed tools, and runtime environment. Use Docker actions when you need tools not available on the runner, require a specific OS configuration, or want to write actions in languages other than JavaScript (Python, Go, Bash, Ruby, etc.). The trade-off is a slower startup (container image must be built or pulled) and Linux-only execution.

All exercises are performed directly in your GitHub repository.

---

## Exercise 1: Anatomy of a Docker Action (10 minutes)

### Understand the required files and how GitHub executes them

A Docker action consists of three files:

```
.github/actions/my-docker-action/
  action.yml       <- metadata: declares inputs, outputs, and points to the Dockerfile
  Dockerfile       <- defines the container image
  entrypoint.sh    <- (or any executable) runs inside the container
```

**Execution flow:**

```
1. GitHub reads action.yml
2. Builds the Docker image from the Dockerfile (or pulls a pre-built image)
3. Runs the container with:
   - Inputs passed as environment variables (INPUT_<NAME> in uppercase)
   - The workspace mounted at /github/workspace
   - GITHUB_OUTPUT file for setting outputs
4. Container exits — exit code 0 = success, non-zero = failure
```

1. Create `.github/actions/repo-stats/action.yml`:

```yaml
name: 'Repository Stats'
description: 'Analyzes the repository and reports file counts, sizes, and language breakdown'

inputs:
  scan-path:
    description: 'Directory to scan (relative to repository root)'
    required: false
    default: '.'
  include-hidden:
    description: 'Include hidden files/directories in the scan'
    required: false
    default: 'false'

outputs:
  total-files:
    description: 'Total number of files found'
  total-size:
    description: 'Total size of all files (human-readable)'
  largest-file:
    description: 'Path to the largest file'

runs:
  using: 'docker'
  # ↑ Tells GitHub this is a Docker action (not node16/node20/composite)
  image: 'Dockerfile'
  # ↑ Path to the Dockerfile relative to this action.yml.
  #   GitHub will build this image at runtime.
  #   Alternatively, use a pre-built image: image: 'docker://alpine:3.19'
```

2. Commit to **main**.

### `image:` options

| Value | Behavior |
|-------|----------|
| `'Dockerfile'` | Build from local Dockerfile each run (slower, flexible) |
| `'docker://alpine:3.19'` | Pull pre-built image from Docker Hub (faster, fixed environment) |
| `'docker://ghcr.io/org/image:tag'` | Pull from GitHub Container Registry |

---

## Exercise 2: Build a Shell-Based Docker Action (15 minutes)

### Implement the repo-stats action using Bash inside Alpine Linux

1. Create `.github/actions/repo-stats/Dockerfile`:

```dockerfile
FROM alpine:3.19
# ↑ Base image: Alpine Linux 3.19 — minimal (5 MB), fast to pull
#   Using a specific version tag for reproducibility (not 'latest')

RUN apk add --no-cache bash coreutils findutils
# ↑ apk add          → Alpine's package manager (like apt-get on Debian)
#   --no-cache       → don't store the package index locally (keeps image small)
#   bash             → Bash shell (Alpine uses ash by default)
#   coreutils        → GNU core utilities (du, sort, head with full flag support)
#   findutils        → GNU find (supports -printf and other extensions)

COPY entrypoint.sh /entrypoint.sh
# ↑ Copy the script from the build context into the image at /entrypoint.sh

RUN chmod +x /entrypoint.sh
# ↑ Make the script executable
#   chmod +x → add execute permission for all users

ENTRYPOINT ["/entrypoint.sh"]
# ↑ Define the command that runs when the container starts.
#   GitHub passes action inputs as environment variables to this process.
```

2. Create `.github/actions/repo-stats/entrypoint.sh`:

```bash
#!/bin/bash
set -euo pipefail
# set -e          → exit immediately if any command returns non-zero
# set -u          → treat unset variables as errors
# set -o pipefail → a pipeline fails if any command in it fails (not just the last)

# --- Read inputs ---
# GitHub converts inputs to env vars: INPUT_<NAME> (uppercase, hyphens become underscores)
SCAN_PATH="${INPUT_SCAN-PATH:-/github/workspace}"
# Fallback in case the variable name with hyphen doesn't resolve
SCAN_PATH="${INPUT_SCAN_PATH:-${SCAN_PATH}}"
INCLUDE_HIDDEN="${INPUT_INCLUDE-HIDDEN:-false}"
INCLUDE_HIDDEN="${INPUT_INCLUDE_HIDDEN:-${INCLUDE_HIDDEN}}"

# Resolve path relative to workspace
if [ "$SCAN_PATH" != "." ] && [ "$SCAN_PATH" != "/github/workspace" ]; then
  SCAN_PATH="/github/workspace/$SCAN_PATH"
else
  SCAN_PATH="/github/workspace"
fi

echo "=== Repository Stats ==="
echo "Scanning: $SCAN_PATH"
echo "Include hidden: $INCLUDE_HIDDEN"
echo ""

# --- Count files ---
FIND_OPTS="-type f"
if [ "$INCLUDE_HIDDEN" = "false" ]; then
  FIND_OPTS="$FIND_OPTS -not -path '*/.*'"
  # -not -path '*/.*' → exclude files whose path contains /. (hidden files/dirs)
fi

TOTAL_FILES=$(eval find "$SCAN_PATH" $FIND_OPTS | wc -l)
# wc -l → count lines (one file per line from find output)

echo "Total files: $TOTAL_FILES"

# --- Calculate total size ---
TOTAL_SIZE=$(du -sh "$SCAN_PATH" | cut -f1)
# du -sh <path>
#   -s → summarize (single total, not per-file)
#   -h → human-readable (KB, MB, GB)
# cut -f1 → extract the first field (the size, tab-separated from the path)

echo "Total size: $TOTAL_SIZE"

# --- Find largest file ---
LARGEST=$(find "$SCAN_PATH" -type f -exec du -b {} + | sort -rn | head -1)
# find ... -exec du -b {} + → get byte size of each file
#   -b → apparent size in bytes
#   {} → placeholder for each found file
#   +  → pass multiple files to du at once (efficient)
# sort -rn → sort numerically (-n) in reverse (-r) — largest first
# head -1  → take only the first line (largest file)

LARGEST_SIZE=$(echo "$LARGEST" | cut -f1)
LARGEST_PATH=$(echo "$LARGEST" | cut -f2)

echo "Largest file: $LARGEST_PATH ($LARGEST_SIZE bytes)"

# --- Set outputs ---
# Write outputs to the file specified by GITHUB_OUTPUT
echo "total-files=$TOTAL_FILES" >> "$GITHUB_OUTPUT"
echo "total-size=$TOTAL_SIZE" >> "$GITHUB_OUTPUT"
echo "largest-file=$LARGEST_PATH" >> "$GITHUB_OUTPUT"
# ↑ Same mechanism as regular workflow steps: append name=value lines to $GITHUB_OUTPUT

echo ""
echo "=== Scan Complete ==="
```

3. Commit both files to **main**.

### How inputs reach the container

GitHub maps each input to an environment variable:

| Input name in action.yml | Environment variable inside container |
|--------------------------|---------------------------------------|
| `scan-path` | `INPUT_SCAN-PATH` (also `INPUT_SCAN_PATH`) |
| `include-hidden` | `INPUT_INCLUDE-HIDDEN` (also `INPUT_INCLUDE_HIDDEN`) |

The workspace is mounted at `/github/workspace` inside the container.

---

## Exercise 3: Test the Docker Action (10 minutes)

### Run the action in a workflow and inspect output

1. Create `.github/workflows/repo-stats.yml`:

```yaml
name: Repository Stats

on:
  workflow_dispatch:

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        # ↑ Required: checks out the repo so the Docker action can scan it.
        #   The checked-out code is what gets mounted into the container.

      - name: Run repo stats
        id: stats
        uses: ./.github/actions/repo-stats
        with:
          scan-path: '.'
          include-hidden: 'false'

      - name: Display results
        run: |
          echo "Files  : ${{ steps.stats.outputs.total-files }}"
          echo "Size   : ${{ steps.stats.outputs.total-size }}"
          echo "Largest: ${{ steps.stats.outputs.largest-file }}"
```

2. Commit to **main**.

3. Go to **Actions** -> **"Repository Stats"** -> **Run workflow**.

4. Open the run and expand the "Run repo stats" step. You will see:
   - Docker image being built (FROM alpine, RUN apk add, etc.)
   - Container output with file counts and sizes
   - Outputs captured in the "Display results" step

### Observing the Docker build

The first run builds the image from scratch. Subsequent runs on the same runner may use cached layers. In the logs you will see lines like:

```
Building docker image...
Step 1/5 : FROM alpine:3.19
Step 2/5 : RUN apk add --no-cache bash coreutils findutils
...
Successfully built abc123def456
```

---

## Exercise 4: Docker Action with a Pre-Built Image (10 minutes)

### Skip the build step by pulling an existing image

When your action uses only standard tools, you can reference a public image directly. This is faster because GitHub pulls instead of building.

1. Create `.github/actions/markdown-lint/action.yml`:

```yaml
name: 'Markdown Linter'
description: 'Checks markdown files for style and formatting issues'

inputs:
  files:
    description: 'Glob pattern for markdown files to check'
    required: false
    default: '**/*.md'
  config:
    description: 'Path to markdownlint config file'
    required: false
    default: ''

outputs:
  issues-found:
    description: 'Number of linting issues found'

runs:
  using: 'docker'
  image: 'docker://davidanson/markdownlint-cli2:v0.12.1'
  # ↑ Pre-built image from Docker Hub.
  #   Format: docker://<registry>/<image>:<tag>
  #   No Dockerfile needed — GitHub pulls this image directly.
  args:
    - ${{ inputs.files }}
  # ↑ 'args' are passed as command-line arguments to the container's ENTRYPOINT.
  #   The markdownlint-cli2 image expects file patterns as positional arguments.
```

2. Commit to **main**.

3. Create `.github/workflows/lint-markdown.yml`:

```yaml
name: Lint Markdown

on:
  workflow_dispatch:
  pull_request:
    paths:
      - '**.md'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run markdown linter
        uses: ./.github/actions/markdown-lint
        with:
          files: '**/*.md'
```

4. Commit to **main** and run the workflow manually.

5. If your markdown files have linting issues, the step will fail (non-zero exit from the container). Fix the issues or add a `.markdownlint.yaml` config to customize rules.

### Pre-built image vs Dockerfile

| Approach | When to use |
|----------|-------------|
| `image: 'Dockerfile'` | Custom tools, proprietary code, complex setup |
| `image: 'docker://...'` | Standard tools already available as public images |

---

## Exercise 5: Docker Action in Python (10 minutes)

### Use any language — write the action logic in Python

1. Create `.github/actions/pr-complexity/action.yml`:

```yaml
name: 'PR Complexity Analyzer'
description: 'Estimates PR review complexity based on file types and change patterns'

inputs:
  max-complexity:
    description: 'Maximum allowed complexity score before warning'
    required: false
    default: '50'

outputs:
  complexity-score:
    description: 'Calculated complexity score (0-100)'
  recommendation:
    description: 'Review recommendation based on complexity'

runs:
  using: 'docker'
  image: 'Dockerfile'
```

2. Create `.github/actions/pr-complexity/Dockerfile`:

```dockerfile
FROM python:3.12-slim
# ↑ Official Python 3.12 slim image — smaller than the full image but includes pip

COPY analyze.py /analyze.py
# ↑ Copy the Python script into the image

ENTRYPOINT ["python", "/analyze.py"]
# ↑ Run the Python script as the container's entrypoint
#   GitHub passes inputs as INPUT_* environment variables automatically
```

3. Create `.github/actions/pr-complexity/analyze.py`:

```python
import os
import sys

def main():
    # Read inputs from environment variables
    # GitHub sets INPUT_<name> (uppercase, hyphens become underscores)
    max_complexity = int(os.environ.get('INPUT_MAX-COMPLEXITY', '50'))

    # Read workspace path
    workspace = os.environ.get('GITHUB_WORKSPACE', '/github/workspace')

    print("=== PR Complexity Analyzer ===")
    print(f"Workspace: {workspace}")
    print(f"Max complexity threshold: {max_complexity}")
    print("")

    # Analyze files in the workspace
    complexity = 0
    file_count = 0
    extension_weights = {
        '.py': 3,    # Python — moderate complexity per file
        '.js': 3,    # JavaScript
        '.yml': 2,   # YAML config
        '.yaml': 2,
        '.md': 1,    # Documentation — low complexity
        '.json': 1,
        '.sh': 2,    # Shell scripts
    }

    for root, dirs, files in os.walk(workspace):
        # os.walk() recursively traverses the directory tree
        # root  = current directory path
        # dirs  = subdirectories in current directory
        # files = files in current directory

        # Skip hidden directories and node_modules
        dirs[:] = [d for d in dirs if not d.startswith('.') and d != 'node_modules']
        # ↑ Modifying dirs[:] in-place prunes the walk — os.walk won't descend into excluded dirs

        for filename in files:
            ext = os.path.splitext(filename)[1].lower()
            # os.path.splitext('file.py') → ('file', '.py')
            # [1] takes the extension, .lower() normalizes case

            weight = extension_weights.get(ext, 1)
            # .get(key, default) → returns the weight for this extension, or 1 if unknown

            complexity += weight
            file_count += 1

    print(f"Files analyzed: {file_count}")
    print(f"Raw complexity: {complexity}")

    # Normalize to 0-100 scale
    normalized = min(100, int((complexity / max(file_count * 3, 1)) * 100))
    # max(file_count * 3, 1) → avoid division by zero
    # min(100, ...) → cap at 100

    print(f"Normalized score: {normalized}")

    # Determine recommendation
    if normalized <= 25:
        recommendation = "Low complexity - quick review expected"
    elif normalized <= 50:
        recommendation = "Moderate complexity - standard review"
    elif normalized <= 75:
        recommendation = "High complexity - thorough review recommended"
    else:
        recommendation = "Very high complexity - consider splitting the PR"

    print(f"Recommendation: {recommendation}")

    # Write outputs to GITHUB_OUTPUT
    output_file = os.environ.get('GITHUB_OUTPUT', '')
    if output_file:
        with open(output_file, 'a') as f:
            f.write(f"complexity-score={normalized}\n")
            f.write(f"recommendation={recommendation}\n")
            # ↑ Same format as shell: name=value, one per line, appended to the file

    # Fail if over threshold
    if normalized > max_complexity:
        print(f"\nWARNING: Complexity {normalized} exceeds threshold {max_complexity}")
        sys.exit(1)
        # ↑ Non-zero exit = step failure. GitHub marks the step with a red X.

    print("\nAnalysis complete - within acceptable complexity.")
    sys.exit(0)


if __name__ == '__main__':
    main()
```

4. Commit all three files to **main**.

5. Create `.github/workflows/complexity-check.yml`:

```yaml
name: Complexity Check

on:
  workflow_dispatch:

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Analyze complexity
        id: complexity
        uses: ./.github/actions/pr-complexity
        with:
          max-complexity: '80'

      - name: Show result
        run: |
          echo "Score: ${{ steps.complexity.outputs.complexity-score }}"
          echo "Advice: ${{ steps.complexity.outputs.recommendation }}"
```

6. Commit to **main**, run the workflow, and inspect the output.

---

## Exercise 6: Passing Environment Variables and Volumes (5 minutes)

### Understand how GitHub connects the runner to the container

This is a conceptual exercise — review the mechanics below.

**What GitHub mounts/injects automatically:**

| Resource | Inside the container | Purpose |
|----------|---------------------|---------|
| Workspace | `/github/workspace` | Repository source code |
| `GITHUB_OUTPUT` | File path (e.g., `/github/file_commands/set_output_...`) | Step outputs |
| `GITHUB_ENV` | File path | Set env vars for subsequent steps |
| `GITHUB_STEP_SUMMARY` | File path | Write job summary markdown |
| All `INPUT_*` vars | Environment variables | Action inputs |
| All `GITHUB_*` vars | Environment variables | Context (repo, actor, ref, etc.) |

**You can also pass custom env vars in the workflow:**

```yaml
- name: Run action with extra vars
  uses: ./.github/actions/repo-stats
  with:
    scan-path: 'src'
  env:
    CUSTOM_VAR: some-value
    # ↑ This becomes available inside the container as $CUSTOM_VAR
    #   Useful for passing non-input configuration
```

**Container networking:**
- The container runs on the host network — it can reach the internet and any services on localhost.
- Services started in prior steps (e.g., a database container via `services:`) are accessible.

---

## Summary

| Concept | Key detail |
|---------|-----------|
| Action type | `runs.using: 'docker'` in action.yml |
| Image source | `image: 'Dockerfile'` (build) or `image: 'docker://...'` (pull) |
| Entry point | `ENTRYPOINT` in Dockerfile — receives inputs as `INPUT_*` env vars |
| Outputs | Write `name=value` lines to the file at `$GITHUB_OUTPUT` |
| Language | Any — Bash, Python, Go, Ruby, etc. |
| Platform | Linux only (Docker actions cannot run on Windows/macOS runners) |
| Trade-off | Full environment control vs. slower startup (image build/pull) |
| Workspace mount | `/github/workspace` inside the container |

Docker actions give you maximum flexibility over the execution environment. When you need something in between (multiple steps, mixed shells, no container overhead), use **composite actions** — covered in the next lab.
