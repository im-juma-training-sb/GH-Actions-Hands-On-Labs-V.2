# Lab 11: Matrix Strategies

**Duration:** 45-55 minutes  
**Level:** Intermediate  
**Prerequisites:** Labs 1-10 completed, repository `actions-fundamentals-lab` available

---

## Overview

Matrix strategies allow a single job definition to spawn multiple parallel job instances, each with a different combination of variables. This is how you test across multiple Node.js versions, operating systems, or configurations without duplicating YAML. In this lab you will build matrices from simple one-dimensional arrays through multi-dimensional combinations, learn to include and exclude specific entries, use `fail-fast` to control failure behavior, and dynamically generate matrices from JSON.

All exercises are performed directly in your GitHub repository.

---

## Exercise 1: Single-Dimension Matrix (10 minutes)

### Test across multiple Node.js versions

A basic matrix iterates over one variable. GitHub Actions creates a separate job for each value.

1. Create `.github/workflows/matrix-basic.yml`:

```yaml
name: Matrix Basic

on:
  workflow_dispatch:
  # Manual trigger for experimentation

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: ['18', '20', '22']
        # strategy.matrix defines the matrix dimensions.
        # Each entry in the array becomes a separate job instance.
        # This produces 3 parallel jobs: one for Node 18, one for 20, one for 22.

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          # matrix.<key> references the current value for this job instance.
          # Job 1 gets '18', Job 2 gets '20', Job 3 gets '22'.

      - name: Display version
        run: |
          echo "Node.js version: $(node --version)"
          echo "npm version: $(npm --version)"
          # Each parallel job runs independently with its own Node.js version.

      - name: Run tests
        run: npm test
```

2. Commit to **main**.

3. Go to **Actions** -> **"Matrix Basic"** -> **Run workflow**.

4. Observe: three jobs appear, each labeled with the matrix value (e.g., `test (18)`, `test (20)`, `test (22)`). All three run in parallel.

### How the matrix label works

GitHub automatically appends the matrix values to the job name. For a job named `test` with `node-version: '20'`, the display name becomes `test (20)`. This is how you identify which combination each job represents.

---

## Exercise 2: Multi-Dimension Matrix (10 minutes)

### Test across operating systems AND Node.js versions simultaneously

When you define multiple keys in `strategy.matrix`, GitHub creates every possible combination (the Cartesian product).

1. Create `.github/workflows/matrix-multi.yml`:

```yaml
name: Matrix Multi-Dimension

on:
  workflow_dispatch:

jobs:
  test:
    runs-on: ${{ matrix.os }}
    # The runner OS itself comes from the matrix.
    # Each combination gets a different operating system.

    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: ['18', '20']
        # Two dimensions: 3 OS values x 2 Node versions = 6 parallel jobs.
        # Combinations: ubuntu/18, ubuntu/20, windows/18, windows/20, macos/18, macos/20

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }} on ${{ matrix.os }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Show environment
        shell: bash
        # 'shell: bash' is explicit here because Windows defaults to PowerShell.
        # Using bash ensures consistent behavior across all three OS platforms.
        run: |
          echo "OS: ${{ matrix.os }}"
          echo "Node: $(node --version)"
          echo "Architecture: $(uname -m)"

      - name: Run tests
        run: npm test
```

2. Commit to **main** and run the workflow.

3. Observe: **6 jobs** appear (3 OS x 2 Node versions). Each is labeled like `test (ubuntu-latest, 18)`.

### Matrix math

| os (3 values) | node-version (2 values) | Total jobs |
|----------------|--------------------------|------------|
| ubuntu-latest | 18, 20 | 2 |
| windows-latest | 18, 20 | 2 |
| macos-latest | 18, 20 | 2 |
| **Total** | | **6** |

Adding a third dimension (e.g., `database: [postgres, mysql]`) would multiply by 2 again = 12 jobs. Be mindful of runner costs.

---

## Exercise 3: Include and Exclude (10 minutes)

### Fine-tune the matrix with specific additions and removals

Not every combination is valid or needed. Use `exclude` to remove specific combinations and `include` to add entries with extra variables.

1. Create `.github/workflows/matrix-include-exclude.yml`:

```yaml
name: Matrix Include/Exclude

on:
  workflow_dispatch:

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: ['18', '20', '22']
        # Base: 3 x 3 = 9 combinations

        exclude:
          - os: windows-latest
            node-version: '22'
          # Remove windows + Node 22 (not a supported combination in this project).
          # After exclude: 9 - 1 = 8 combinations remain.

          - os: macos-latest
            node-version: '18'
          # Remove macOS + Node 18 (deprecated platform target).
          # After this exclude: 8 - 1 = 7 combinations remain.

        include:
          - os: ubuntu-latest
            node-version: '20'
            experimental: true
          # 'include' adds extra variables to a MATCHING combination.
          # The ubuntu/20 job already exists in the matrix.
          # This adds an 'experimental' variable only to that specific job.
          # Other jobs do NOT get the 'experimental' variable.

          - os: ubuntu-latest
            node-version: '21'
            experimental: true
          # 'include' can also ADD entirely new combinations.
          # Node 21 is not in the original matrix — this creates a new job.
          # After include: 7 + 1 = 8 total combinations.

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Show matrix values
        shell: bash
        run: |
          echo "OS: ${{ matrix.os }}"
          echo "Node: ${{ matrix.node-version }}"
          echo "Experimental: ${{ matrix.experimental }}"
          # 'matrix.experimental' is empty string for jobs where it was not defined.

      - name: Run tests
        run: npm test

      - name: Experimental warning
        if: matrix.experimental == true
        # 'if:' can reference matrix variables to conditionally run steps.
        # Only the ubuntu/20 and ubuntu/21 jobs have experimental=true.
        shell: bash
        run: echo "WARNING - This is an experimental configuration"
```

2. Commit to **main** and run.

3. Count the jobs: 8 total (9 base - 2 excluded + 1 new included).

### Include vs Exclude rules

| Keyword | Behavior |
|---------|----------|
| `exclude` | Removes matching combinations from the Cartesian product |
| `include` (matching existing) | Adds extra variables to an already-existing combination |
| `include` (new combination) | Adds an entirely new job to the matrix |

---

## Exercise 4: Fail-Fast and Max-Parallel (8 minutes)

### Control failure behavior and concurrency

By default, if one matrix job fails, GitHub cancels all other running jobs. Use `fail-fast: false` to let all jobs complete regardless. Use `max-parallel` to limit how many jobs run simultaneously.

1. Create `.github/workflows/matrix-fail-fast.yml`:

```yaml
name: Matrix Fail-Fast

on:
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      # false → all jobs run to completion even if one fails.
      # true (default) → first failure cancels all remaining jobs immediately.
      # Use false when you want a FULL picture of what is broken across all versions.

      max-parallel: 2
      # Limit to 2 concurrent jobs at a time.
      # Without this, all 4 jobs would start simultaneously.
      # Useful for: rate-limited APIs, shared test databases, runner cost control.

      matrix:
        node-version: ['18', '19', '20', '22']

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Simulate version-specific behavior
        shell: bash
        run: |
          echo "Testing on Node.js ${{ matrix.node-version }}"
          if [ "${{ matrix.node-version }}" = "19" ]; then
            echo "Node 19 is EOL — simulating failure"
            exit 1
          fi
          echo "Tests passed."
          # Node 19 intentionally fails to demonstrate fail-fast behavior.
          # With fail-fast: false, the other 3 jobs still complete.
          # With fail-fast: true (default), failure here would cancel the rest.
```

2. Commit to **main** and run.

3. Observe:
   - Only 2 jobs start initially (`max-parallel: 2`).
   - When one finishes, the next starts.
   - The Node 19 job fails, but the others continue (`fail-fast: false`).

4. **Experiment:** Change `fail-fast` to `true` and run again. Now when Node 19 fails, remaining jobs are cancelled immediately.

### When to use each setting

| Scenario | `fail-fast` | Why |
|----------|-------------|-----|
| PR checks (fast feedback) | `true` (default) | Fail early, save runner minutes |
| Release validation (full report) | `false` | Need to know ALL failures before shipping |
| Nightly test suite | `false` | Complete picture of platform compatibility |
| Costly matrix (many runners) | `true` | Cancel early to reduce costs |

---

## Exercise 5: Dynamic Matrix from JSON (10 minutes)

### Generate matrix values at runtime

Sometimes the matrix values are not known ahead of time. You can generate them dynamically from a previous job's output using `fromJSON()`.

1. Create `.github/workflows/matrix-dynamic.yml`:

```yaml
name: Matrix Dynamic

on:
  workflow_dispatch:
    inputs:
      environments:
        description: 'Comma-separated environments to deploy (e.g., dev,staging,prod)'
        required: true
        default: 'dev,staging'
        type: string

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.generate.outputs.matrix }}
      # Job outputs make data available to downstream jobs.
      # The 'test' job reads this to build its matrix.

    steps:
      - name: Generate matrix from input
        id: generate
        run: |
          # Convert comma-separated input to JSON array
          INPUT="${{ inputs.environments }}"
          JSON_ARRAY=$(echo "$INPUT" | tr ',' '\n' | jq -R . | jq -s -c .)
          # tr ',' '\n'   → split on commas into separate lines
          # jq -R .       → read each line as a raw JSON string
          # jq -s -c .    → collect into a compact JSON array
          #   -s (slurp)  → read all inputs into a single array
          #   -c (compact) → output on one line (required for GITHUB_OUTPUT)

          echo "Generated matrix: $JSON_ARRAY"
          echo "matrix={\"environment\":$JSON_ARRAY}" >> "$GITHUB_OUTPUT"
          # Output format must be valid JSON object for fromJSON() to parse.
          # Example: {"environment":["dev","staging"]}

  deploy:
    needs: prepare
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJSON(needs.prepare.outputs.matrix) }}
      # fromJSON() parses the string output into a matrix object.
      # The result is equivalent to writing:
      #   matrix:
      #     environment: [dev, staging]
      # But the values come from runtime input instead of hardcoded YAML.

    steps:
      - name: Deploy to ${{ matrix.environment }}
        run: |
          echo "=== Deploying to ${{ matrix.environment }} ==="
          echo "Timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
          echo "Run ID: ${{ github.run_id }}"
          echo "Deployment complete."
```

2. Commit to **main**.

3. Run the workflow with the default input (`dev,staging`). Two jobs appear.

4. Run again with `dev,staging,prod`. Three jobs appear.

5. Run with just `prod`. One job appears.

### When to use dynamic matrices

| Scenario | Example |
|----------|---------|
| Deploy to user-selected environments | Input: `dev,staging,prod` |
| Test only changed packages in a monorepo | Script detects changed dirs, outputs them as JSON |
| Target specific regions | API call returns active regions |
| Conditional platform builds | Config file lists supported platforms |

---

## Exercise 6: Matrix with Reusable Outputs (7 minutes)

### Collect results from all matrix jobs

Matrix jobs run independently. To aggregate results (e.g., "all deployments succeeded"), use a downstream job that waits for the entire matrix.

1. Create `.github/workflows/matrix-outputs.yml`:

```yaml
name: Matrix with Summary

on:
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        component: [api, web, worker]
    outputs:
      result-${{ matrix.component }}: ${{ steps.result.outputs.status }}
      # NOTE: Matrix job outputs are set per-instance, but only the LAST
      # instance to complete wins for each output key.
      # For unique per-instance outputs, include the matrix value in the key name.

    steps:
      - name: Test ${{ matrix.component }}
        id: result
        run: |
          echo "Running tests for ${{ matrix.component }}..."
          echo "status=passed" >> "$GITHUB_OUTPUT"

  summary:
    needs: test
    runs-on: ubuntu-latest
    # This job runs AFTER all matrix instances complete (not after just one).
    # 'needs: test' waits for ALL jobs created by the matrix strategy.

    steps:
      - name: Aggregate results
        run: |
          echo "=== Test Summary ==="
          echo "API:    ${{ needs.test.outputs.result-api }}"
          echo "Web:    ${{ needs.test.outputs.result-web }}"
          echo "Worker: ${{ needs.test.outputs.result-worker }}"

      - name: Check for failures
        run: |
          ALL_PASSED=true
          for COMPONENT in api web worker; do
            STATUS="${{ needs.test.outputs[format('result-{0}', matrix.component)] }}"
            if [ "$STATUS" != "passed" ]; then
              echo "FAILED: $COMPONENT"
              ALL_PASSED=false
            fi
          done

          if [ "$ALL_PASSED" = true ]; then
            echo "All components passed."
          else
            echo "Some components failed."
            exit 1
          fi

      - name: Create summary
        run: |
          echo "## Test Results" >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo "| Component | Status |" >> "$GITHUB_STEP_SUMMARY"
          echo "|-----------|--------|" >> "$GITHUB_STEP_SUMMARY"
          echo "| API | ${{ needs.test.outputs.result-api }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Web | ${{ needs.test.outputs.result-web }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Worker | ${{ needs.test.outputs.result-worker }} |" >> "$GITHUB_STEP_SUMMARY"
```

2. Commit to **main** and run.

3. Observe:
   - Three `test` jobs run in parallel (api, web, worker).
   - `summary` waits for all three to complete.
   - The run summary shows a table of results.

### Important: Matrix outputs limitation

Matrix job outputs have a key constraint: if multiple matrix instances write to the **same** output key, only the last one to finish wins. To get per-instance outputs, include the matrix value in the output key name (e.g., `result-${{ matrix.component }}`).

---

## Summary

| Concept | What it does |
|---------|--------------|
| `strategy.matrix` | Creates parallel jobs for each combination of values |
| Multi-dimension matrix | Cartesian product of all defined arrays |
| `exclude` | Removes specific combinations from the matrix |
| `include` (existing) | Adds extra variables to a matching combination |
| `include` (new) | Adds entirely new combinations to the matrix |
| `fail-fast: false` | All jobs complete regardless of individual failures |
| `max-parallel: N` | Limits concurrent matrix jobs to N at a time |
| `fromJSON()` | Parses a JSON string into a matrix object at runtime |
| Matrix outputs | Per-instance output keys for downstream aggregation |

---

## Key Takeaways

1. Matrix strategies eliminate YAML duplication -- define the job once, run it many times.
2. The Cartesian product grows multiplicatively -- a 3x3x2 matrix creates 18 jobs.
3. `exclude` removes invalid or unsupported combinations; `include` adds special cases.
4. `fail-fast: false` gives you a complete failure picture; `true` saves runner time.
5. Dynamic matrices with `fromJSON()` enable runtime-determined job configurations.
6. Downstream jobs that `needs:` a matrix job wait for ALL matrix instances to finish.
