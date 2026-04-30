# Lab 10: CI/CD Pipelines

## Overview

**Duration:** 60-90 minutes  
**Difficulty:** Intermediate

Build a complete, production-grade CI/CD pipeline from scratch. You'll create a real Node.js application, add automated testing, build artifacts, deploy through staged environments with approvals, add caching for performance, and automate releases — all using real tools and real operations.

### Learning Objectives

By the end of this lab, you will:
- Build a multi-stage CI/CD pipeline with real build and test steps
- Manage build artifacts with upload and download between jobs
- Deploy through staged environments (staging → production) with protection rules
- Optimize pipelines with dependency and build caching
- Automate semantic versioning and GitHub Releases


---

## Exercise 1: Build and Test Pipeline (15 minutes)

### Create a real Node.js application with automated CI

1. In your repository, go to **Code** → **Add file** → **Create new file**.

2. Name the file `package.json` and add:

```json
{
  "name": "cicd-lab",
  "version": "1.0.0",
  "description": "GitHub Actions CI/CD training lab",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "test": "node test/run.js",
    "lint": "node lint/check.js",
    "build": "node scripts/build.js"
  },
  "license": "MIT"
}
```

3. Click **Commit changes...** → **Commit changes** (commit directly to main).

4. Go back to **Code** tab → **Add file** → **Create new file**.

5. Name the file `src/index.js` (GitHub will create the `src` folder automatically) and add:

```javascript
function add(a, b) {
  return a + b;
}

function multiply(a, b) {
  return a * b;
}

function greet(name) {
  if (!name || typeof name !== 'string') {
    throw new Error('Name must be a non-empty string');
  }
  return `Hello, ${name}!`;
}

module.exports = { add, multiply, greet };

if (require.main === module) {
  console.log('CI/CD Lab Application v1.0.0');
  console.log(greet('World'));
  console.log(`2 + 3 = ${add(2, 3)}`);
  console.log(`4 × 5 = ${multiply(4, 5)}`);
}
```

6. Click **Commit changes...** → **Commit changes**.

7. Go back to **Code** tab → **Add file** → **Create new file**.

8. Name the file `test/run.js` and add:

```javascript
const { add, multiply, greet } = require('../src/index');

let passed = 0;
let failed = 0;

function assert(description, actual, expected) {
  if (actual === expected) {
    console.log(`  PASS: ${description}`);
    passed++;
  } else {
    console.error(`  FAIL: ${description}: expected ${expected}, got ${actual}`);
    failed++;
  }
}

function assertThrows(description, fn) {
  try {
    fn();
    console.error(`  FAIL: ${description}: expected an error but none was thrown`);
    failed++;
  } catch (e) {
    console.log(`  PASS: ${description}`);
    passed++;
  }
}

console.log('\nRunning tests...\n');

console.log('add():');
assert('adds positive numbers', add(2, 3), 5);
assert('adds negative numbers', add(-1, -2), -3);
assert('adds zero', add(0, 5), 5);

console.log('\nmultiply():');
assert('multiplies positive numbers', multiply(4, 5), 20);
assert('multiplies by zero', multiply(7, 0), 0);
assert('multiplies negative numbers', multiply(-3, -4), 12);

console.log('\ngreet():');
assert('greets by name', greet('World'), 'Hello, World!');
assert('greets with different name', greet('Actions'), 'Hello, Actions!');
assertThrows('throws on empty string', () => greet(''));
assertThrows('throws on null', () => greet(null));

console.log(`\nResults: ${passed} passed, ${failed} failed out of ${passed + failed} tests\n`);

if (failed > 0) {
  process.exit(1);
}
```

9. Click **Commit changes...** → **Commit changes**.

10. Go back to **Code** tab → **Add file** → **Create new file**.

11. Name the file `lint/check.js` and add:

```javascript
const fs = require('fs');
const path = require('path');

const srcDir = path.join(__dirname, '..', 'src');
let issues = 0;
let filesChecked = 0;

console.log('\nRunning lint checks...\n');

function lintFile(filePath) {
  const content = fs.readFileSync(filePath, 'utf8');
  const lines = content.split('\n');
  const relativePath = path.relative(process.cwd(), filePath);
  filesChecked++;

  lines.forEach((line, index) => {
    const lineNum = index + 1;

    // Check for console.error without a message
    if (line.includes('console.error()')) {
      console.log(`  WARNING: ${relativePath}:${lineNum} - Empty console.error()`);
      issues++;
    }

    // Check for lines longer than 120 characters
    if (line.length > 120) {
      console.log(`  WARNING: ${relativePath}:${lineNum} - Line exceeds 120 characters (${line.length})`);
      issues++;
    }

    // Check for trailing whitespace
    if (line !== line.trimEnd() && line.trim().length > 0) {
      console.log(`  WARNING: ${relativePath}:${lineNum} - Trailing whitespace`);
      issues++;
    }

    // Check for var usage (prefer const/let)
    if (/\bvar\b/.test(line) && !line.trim().startsWith('//')) {
      console.log(`  WARNING: ${relativePath}:${lineNum} - Use 'const' or 'let' instead of 'var'`);
      issues++;
    }
  });
}

if (fs.existsSync(srcDir)) {
  const files = fs.readdirSync(srcDir).filter(f => f.endsWith('.js'));
  files.forEach(file => lintFile(path.join(srcDir, file)));
}

console.log(`\nLint: ${filesChecked} files checked, ${issues} issues found\n`);

if (issues > 0) {
  process.exit(1);
}

console.log('All lint checks passed!\n');
```

12. Click **Commit changes...** → **Commit changes**.

13. Go back to **Code** tab → **Add file** → **Create new file**.

14. Name the file `scripts/build.js` and add:

```javascript
const fs = require('fs');
const path = require('path');

const distDir = path.join(__dirname, '..', 'dist');
const srcDir = path.join(__dirname, '..', 'src');

console.log('\nBuilding application...\n');

// Create dist directory
if (!fs.existsSync(distDir)) {
  fs.mkdirSync(distDir, { recursive: true });
}

// Copy source files to dist
const files = fs.readdirSync(srcDir).filter(f => f.endsWith('.js'));
files.forEach(file => {
  const src = path.join(srcDir, file);
  const dest = path.join(distDir, file);
  fs.copyFileSync(src, dest);
  console.log(`  ${file} -> dist/${file}`);
});

// Generate build manifest
const manifest = {
  name: 'cicd-lab',
  version: require('../package.json').version,
  buildTime: new Date().toISOString(),
  files: files,
  nodeVersion: process.version,
  platform: process.platform
};

fs.writeFileSync(
  path.join(distDir, 'manifest.json'),
  JSON.stringify(manifest, null, 2)
);
console.log('  manifest.json generated');

console.log(`\nBuild complete! ${files.length} files written to dist/\n`);
```

15. Click **Commit changes...** → **Commit changes**.

16. Go back to **Code** tab → **Add file** → **Create new file**.

17. Name the file `.github/workflows/ci.yml` (GitHub will create the `.github/workflows` folders automatically) and paste this CI workflow:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Run linter
        run: npm run lint

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: lint

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Run tests
        run: npm test

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Build application
        run: npm run build

      - name: Verify build output
        run: |
          echo "Build output:"
          ls -la dist/
          echo ""
          echo "Build manifest:"
          cat dist/manifest.json
```

18. Click **Commit changes...** → **Commit changes**.

19. Go to the **Actions** tab at the top of your repository.

20. In the left sidebar, click **CI Pipeline**.

21. Click the **Run workflow** dropdown (right side) → select the **main** branch → click the green **Run workflow** button.

22. Wait for the workflow to start. Click on the running workflow to see its progress.

23. **Observe**: Three sequential jobs run — **Lint → Test → Build** — each executing real code against your actual source files. Click on any job name to expand its steps and see the real output.

### What just happened?

You created a complete CI pipeline with real operations:
- **Lint job** scans your `src/` files for style issues (long lines, trailing whitespace, `var` usage)
- **Test job** runs 10 actual unit tests against your `add()`, `multiply()`, and `greet()` functions
- **Build job** compiles source files into `dist/` and generates a `manifest.json` with build metadata
- Jobs are chained with `needs:` — test only runs after lint passes, build only runs after tests pass
- If any step fails (non-zero exit code), the pipeline stops — just like a real CI system
- `permissions: contents: read` follows least-privilege security

---

## Exercise 2: Artifact Management (10 minutes)

### Upload build output and share it between jobs

1. In your repository, go to **Code** tab → navigate to `.github/workflows/` folder → click on `ci.yml`.

2. Click the **pencil icon** in the top-right of the file to edit it.

3. **Select all** the existing content (Ctrl+A / Cmd+A) and **delete it**.

4. Paste this updated pipeline that manages artifacts:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Run linter
        run: npm run lint

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: lint

    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Run tests
        run: npm test

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Build application
        run: npm run build

      - name: Verify build output
        run: |
          echo "Build output:"
          ls -la dist/
          echo ""
          echo "Build manifest:"
          cat dist/manifest.json

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7

  verify-artifact:
    name: Verify Artifact
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: downloaded-build/

      - name: Inspect artifact contents
        run: |
          echo "Downloaded artifact contents:"
          ls -la downloaded-build/
          echo ""
          echo "Build manifest from artifact:"
          cat downloaded-build/manifest.json
          echo ""
          echo "Verifying all expected files exist..."
          test -f downloaded-build/index.js && echo "  index.js present"
          test -f downloaded-build/manifest.json && echo "  manifest.json present"
          echo ""
          echo "Artifact verification complete!"
```

5. Click **Commit changes...** → **Commit changes**.

6. Go to the **Actions** tab → click **CI Pipeline** in the left sidebar.

7. Click the **Run workflow** dropdown → select **main** branch → click the green **Run workflow** button.

8. Click on the running workflow to watch its progress.

9. After the run completes, click on the completed run name at the top. Scroll down to the **Artifacts** section at the bottom of the summary page to see `build-output`. You can click on it to download the artifact.

10. **Observe**: The test job now runs across Node 18, 20, and 22 in parallel (3 jobs at once). The build artifact is produced once and verified in a separate job.

### What just happened?

Artifacts let you persist and share data between jobs:
- `actions/upload-artifact@v4` saves files from a job's workspace
- `actions/download-artifact@v4` retrieves those files in a different job
- `retention-days: 7` controls how long artifacts are stored (default 90 days)
- Artifacts appear on the workflow run summary page and can be downloaded
- The `verify-artifact` job proves the artifact is complete and intact
- Tests now run across 3 Node.js versions via matrix — a real compatibility check
- This is the foundation for deployment: build once, deploy the same artifact everywhere

---

## Exercise 3: Multi-Stage Deployment Pipeline (15 minutes)

### Deploy through staging and production with environment protection

1. First, configure environments. In your repository, click **Settings** (top menu bar).

2. In the left sidebar, scroll down and click **Environments**.

3. Click the **New environment** button. Type `staging` as the name and click **Configure environment**. No extra rules needed — staging deploys automatically. Click **Save protection rules** (even with no rules checked).

4. Click **Environments** in the breadcrumb (top of page) to go back. Click **New environment** again, type `production`, and click **Configure environment**.

5. Under **Deployment protection rules**, check the box for **Required reviewers**. In the search box that appears, type your GitHub username and select yourself. Click **Save protection rules**.

6. Go back to the **Code** tab of your repository.

7. Click **Add file** → **Create new file**.

8. Name the file `.github/workflows/deploy.yml` and paste this deployment workflow:

```yaml
name: Deploy Pipeline

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy'
        required: true
        default: '1.0.0'
        type: string

permissions:
  contents: read

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Build application
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: deploy-package
          path: dist/

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: deploy-package
          path: deploy/

      - name: Verify deployment package
        run: |
          echo "Deployment package contents:"
          ls -la deploy/
          echo ""
          echo "Manifest:"
          cat deploy/manifest.json

      - name: Deploy to staging
        run: |
          echo "Deploying version ${{ inputs.version }} to STAGING..."
          echo ""
          echo "Deployment steps:"
          echo "  1. Validating package integrity..."
          test -f deploy/manifest.json && echo "     Manifest verified"
          test -f deploy/index.js && echo "     Application code verified"
          echo "  2. Running pre-deployment health check..."
          node deploy/index.js
          echo "  3. Deployment complete!"
          echo ""
          echo "Staging deployment summary:"
          echo "   Version:     ${{ inputs.version }}"
          echo "   Environment: staging"
          echo "   Timestamp:   $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
          echo "   Status:      SUCCESS"

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: deploy-staging

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Run integration tests against staging
        run: |
          echo "Running integration tests against staging..."
          echo ""
          npm test
          echo ""
          echo "All integration tests passed against staging!"

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: integration-tests
    environment:
      name: production
      url: https://production.example.com

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: deploy-package
          path: deploy/

      - name: Deploy to production
        run: |
          echo "Deploying version ${{ inputs.version }} to PRODUCTION..."
          echo ""
          echo "Deployment steps:"
          echo "  1. Validating package integrity..."
          test -f deploy/manifest.json && echo "     Manifest verified"
          test -f deploy/index.js && echo "     Application code verified"
          echo "  2. Running pre-deployment health check..."
          node deploy/index.js
          echo "  3. Deployment complete!"
          echo ""
          echo "Production deployment summary:"
          echo "   Version:     ${{ inputs.version }}"
          echo "   Environment: production"
          echo "   Timestamp:   $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
          echo "   Status:      SUCCESS"

  post-deploy-verification:
    name: Post-Deploy Verification
    runs-on: ubuntu-latest
    needs: deploy-production

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Smoke tests
        run: |
          echo "Running post-deployment smoke tests..."
          echo ""
          npm test
          echo ""
          node src/index.js
          echo ""
          echo "Post-deployment verification passed!"
```

9. Click **Commit changes...** → **Commit changes**.

10. Go to the **Actions** tab → click **Deploy Pipeline** in the left sidebar.

11. Click the **Run workflow** dropdown → select **main** branch → in the **Version to deploy** field type `1.0.0` → click the green **Run workflow** button.

12. Click on the running workflow to watch its progress.

13. **Observe the pipeline stages**: Build → Staging (deploys automatically) → Integration Tests → Production (**pauses here** waiting for your approval) → Post-Deploy Verification.

14. When the pipeline pauses at the **Deploy to Production** step, you'll see a yellow banner saying "Waiting for review". Click **Review deployments**.

15. In the dialog, check the box next to **production**, optionally add a comment, and click **Approve and deploy**.

16. The pipeline continues: the Production deploy runs, followed by the Post-Deploy Verification job.

### What just happened?

You built a real multi-stage deployment pipeline:
- **Build** compiles once and uploads the artifact — the same package is deployed everywhere
- **Staging** deploys automatically, validating the package and running the actual application
- **Integration tests** run the full test suite against the staging deployment
- **Production** requires manual approval via environment protection rules
- **Post-deploy verification** runs smoke tests after production deployment
- `environment:` links the job to a GitHub Environment, enabling protection rules and deployment history
- `url:` in the environment sets the deployment URL shown in the GitHub UI
- The same artifact flows through every stage — what you test is what you ship
- This pattern (Build → Stage → Test → Approve → Prod → Verify) is an industry standard

---

## Exercise 4: Caching for Performance (10 minutes)

### Speed up your pipeline with dependency caching

The `cache: 'npm'` option in `actions/setup-node@v4` requires a `package-lock.json` file to exist in your repository. Without it, the setup step will fail with a "Dependencies lock file is not found" error. You'll create this file first.

1. In your repository, go to **Code** tab → **Add file** → **Create new file**.

2. Name the file `package-lock.json` and add:

```json
{
  "name": "cicd-lab",
  "version": "1.0.0",
  "lockfileVersion": 3,
  "requires": true,
  "packages": {
    "": {
      "name": "cicd-lab",
      "version": "1.0.0",
      "license": "MIT"
    }
  }
}
```

3. Click **Commit changes...** → **Commit changes**.

4. Go to **Code** tab → navigate to `.github/workflows/` folder → click on `ci.yml`.

5. Click the **pencil icon** in the top-right of the file to edit it.

6. **Select all** the existing content (Ctrl+A / Cmd+A) and **delete it**.

7. Paste this updated pipeline that adds caching:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js with cache
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm install --ignore-scripts

      - name: Run linter
        run: npm run lint

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: lint

    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }} with cache
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm install --ignore-scripts

      - name: Run tests
        run: npm test

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test

    outputs:
      build-id: ${{ steps.build-meta.outputs.build-id }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js with cache
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm install --ignore-scripts

      - name: Build application
        run: npm run build

      - name: Generate build metadata
        id: build-meta
        run: |
          BUILD_ID="build-$(date +%Y%m%d-%H%M%S)-${GITHUB_SHA::8}"
          echo "build-id=$BUILD_ID" >> $GITHUB_OUTPUT
          echo "Build ID: $BUILD_ID"

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7

  verify-artifact:
    name: Verify Artifact
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: downloaded-build/

      - name: Inspect artifact contents
        run: |
          echo "Build ID: ${{ needs.build.outputs.build-id }}"
          echo ""
          echo "Downloaded artifact contents:"
          ls -la downloaded-build/
          echo ""
          echo "Build manifest:"
          cat downloaded-build/manifest.json
          echo ""
          echo "Artifact verification complete!"
```

8. Click **Commit changes...** → **Commit changes**.

9. Go to the **Actions** tab → click **CI Pipeline** in the left sidebar.

10. Click the **Run workflow** dropdown → select **main** branch → click the green **Run workflow** button. Wait for it to complete.

11. **Run the workflow a second time**: Click the **Run workflow** dropdown again → select **main** branch → click the green **Run workflow** button.

12. **Compare the two runs**: Click on each run in the list to see its duration. The second run should be noticeably faster because `actions/setup-node@v4` restores cached npm packages from the first run.

13. Click on any job in the second run and expand the **Setup Node.js with cache** step — look for a line that says `Cache hit` or `Cache restored` (first run will show `Cache miss` or `Cache not found`).

### What just happened?

Caching dramatically improves pipeline performance:
- `cache: 'npm'` in `actions/setup-node@v4` automatically caches the npm global cache directory
- **A `package-lock.json` file must exist in the repo** — without it, the `cache: 'npm'` option fails with a "Dependencies lock file is not found" error
- On first run: cache miss — dependencies are downloaded fresh
- On subsequent runs: cache hit — dependencies are restored instantly from cache
- The cache key is derived from the `package-lock.json` content — when dependencies change, the cache is automatically invalidated and rebuilt
- Build metadata (`build-id`) is passed as a job output for traceability
- In production, caching saves minutes per run — significant at scale with hundreds of daily runs
- `npm install --ignore-scripts` is a security best practice that prevents running arbitrary scripts during installation

---

## Exercise 5: Release Automation (15 minutes)

### Automate semantic versioning and GitHub Releases

1. In your repository, go to **Code** tab → **Add file** → **Create new file**.

2. Name the file `scripts/changelog.js` and add:

```javascript
const { execSync } = require('child_process');
const fs = require('fs');

const version = process.argv[2] || '0.0.0';

console.log(`\nGenerating changelog for v${version}...\n`);

// Get recent commits
let commits;
try {
  commits = execSync('git log --oneline -20 --no-decorate', { encoding: 'utf8' })
    .trim()
    .split('\n')
    .filter(line => line.trim().length > 0);
} catch (e) {
  commits = ['(no git history available)'];
}

// Categorize commits by conventional commit prefixes
const features = [];
const fixes = [];
const other = [];

commits.forEach(commit => {
  const msg = commit.replace(/^[a-f0-9]+ /, '');
  if (/^feat[:(]/i.test(msg)) features.push(msg);
  else if (/^fix[:(]/i.test(msg)) fixes.push(msg);
  else other.push(msg);
});

// Build changelog content
let changelog = `# Changelog — v${version}\n\n`;
changelog += `**Release Date:** ${new Date().toISOString().split('T')[0]}\n\n`;

if (features.length > 0) {
  changelog += `## Features\n\n`;
  features.forEach(f => { changelog += `- ${f}\n`; });
  changelog += '\n';
}

if (fixes.length > 0) {
  changelog += `## Bug Fixes\n\n`;
  fixes.forEach(f => { changelog += `- ${f}\n`; });
  changelog += '\n';
}

if (other.length > 0) {
  changelog += `## Other Changes\n\n`;
  other.forEach(o => { changelog += `- ${o}\n`; });
  changelog += '\n';
}

changelog += `---\n\n`;
changelog += `**Full Diff:** Compare with previous release on GitHub\n`;
changelog += `**Commit Count:** ${commits.length}\n`;

// Write to file
fs.writeFileSync('CHANGELOG.md', changelog);
console.log('Changelog written to CHANGELOG.md');
console.log(`  Features: ${features.length}`);
console.log(`  Fixes:    ${fixes.length}`);
console.log(`  Other:    ${other.length}`);
console.log('');
```

3. Click **Commit changes...** → **Commit changes**.

4. Go back to **Code** tab → **Add file** → **Create new file**.

5. Name the file `.github/workflows/release.yml` and paste this release workflow:

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Release version (e.g., 1.2.0)'
        required: true
        type: string
      release-type:
        description: 'Type of release'
        required: true
        type: choice
        options:
          - patch
          - minor
          - major

permissions:
  contents: write

jobs:
  validate:
    name: Validate Release
    runs-on: ubuntu-latest

    outputs:
      version: ${{ steps.validate.outputs.version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Validate version format
        id: validate
        run: |
          VERSION="${{ inputs.version }}"

          # Check semantic version format
          if [[ ! "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "::error::Invalid version format '$VERSION'. Use semantic versioning: MAJOR.MINOR.PATCH"
            exit 1
          fi

          # Check if tag already exists
          if git rev-parse "v$VERSION" >/dev/null 2>&1; then
            echo "::error::Tag v$VERSION already exists. Choose a different version."
            exit 1
          fi

          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Version v$VERSION is valid and available"

  test:
    name: Test Before Release
    runs-on: ubuntu-latest
    needs: validate

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Run full test suite
        run: |
          echo "Running all checks before release..."
          echo ""
          npm run lint
          echo ""
          npm test

  build-release:
    name: Build Release
    runs-on: ubuntu-latest
    needs: [validate, test]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Build application
        run: npm run build

      - name: Generate changelog
        run: node scripts/changelog.js "${{ needs.validate.outputs.version }}"

      - name: Display changelog
        run: cat CHANGELOG.md

      - name: Upload release artifacts
        uses: actions/upload-artifact@v4
        with:
          name: release-v${{ needs.validate.outputs.version }}
          path: |
            dist/
            CHANGELOG.md

  publish-release:
    name: Publish GitHub Release
    runs-on: ubuntu-latest
    needs: [validate, build-release]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download release artifacts
        uses: actions/download-artifact@v4
        with:
          name: release-v${{ needs.validate.outputs.version }}
          path: release/

      - name: Create Git tag
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git tag -a "v${{ needs.validate.outputs.version }}" -m "Release v${{ needs.validate.outputs.version }}"
          git push origin "v${{ needs.validate.outputs.version }}"

      - name: Create GitHub Release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          VERSION="${{ needs.validate.outputs.version }}"

          gh release create "v$VERSION" \
            --title "v$VERSION" \
            --notes-file release/CHANGELOG.md \
            release/dist/* \
            release/CHANGELOG.md

          echo ""
          echo "Release v$VERSION published!"
          echo "   View at: ${{ github.server_url }}/${{ github.repository }}/releases/tag/v$VERSION"
```

6. Click **Commit changes...** → **Commit changes**.

7. Go to the **Actions** tab → click **Release** in the left sidebar.

8. Click the **Run workflow** dropdown → select **main** branch.

9. In the **Release version** field, type `1.0.0`. In the **Type of release** dropdown, select `minor`.

10. Click the green **Run workflow** button.

11. Click on the running workflow to watch its progress.

12. **Observe the pipeline stages**: Validate → Test Before Release → Build Release → Publish GitHub Release. Each stage does real work.

13. After the workflow completes successfully, go to the **Code** tab. In the right sidebar, click **Releases** (or look for the "1 Release" link). You'll see your published release `v1.0.0` with the generated changelog and downloadable build artifacts.

### What just happened?

You created a complete release automation pipeline:
- **Validate** checks that the version follows semantic versioning and the tag doesn't already exist
- **Test** runs the full lint + test suite to ensure quality before release
- **Build Release** compiles the application and generates a real changelog from your git history
- **Publish Release** creates a Git tag, pushes it, and creates a GitHub Release with artifacts
- `gh release create` uses the GitHub CLI (pre-installed on runners) to publish releases
- `fetch-depth: 0` clones full git history so changelog generation and tag checks work correctly
- `permissions: contents: write` allows creating tags and releases
- The changelog categorizes commits into Features, Bug Fixes, and Other using conventional commit prefixes
- Release artifacts are downloadable directly from the GitHub Releases page

---

## What You've Learned

### CI Pipeline Fundamentals

Build automated quality gates:
- **Lint → Test → Build** is the standard CI flow
- Each stage validates a different aspect of code quality
- Jobs chain with `needs:` to enforce ordering
- Real linting catches real style issues; real tests catch real bugs
- `permissions: contents: read` follows least-privilege principle

### Artifact Management

Share build outputs across jobs and workflows:
- `actions/upload-artifact@v4` persists files from builds
- `actions/download-artifact@v4` retrieves them in other jobs
- `retention-days` controls storage duration
- Build once, deploy the same artifact everywhere — ensures consistency
- Artifacts are downloadable from the workflow run summary

### Multi-Stage Deployments

Deploy safely through environments:
- GitHub Environments (staging, production) provide deployment boundaries
- Protection rules enforce approval gates before production
- The same tested artifact flows through all stages
- Integration tests validate after staging, smoke tests verify after production
- `environment:` in a job links it to a GitHub Environment for audit and control

### Caching for Performance

Speed up repeated operations:
- `cache: 'npm'` in `actions/setup-node@v4` handles npm caching automatically
- Cache keys are derived from lockfile contents
- Cache hit = instant restore; cache miss = fresh install
- Invalidation is automatic when dependencies change
- At enterprise scale, caching saves hours of compute daily

### Release Automation

Automate the release lifecycle:
- Validate version format and tag uniqueness before starting
- Run the full test suite as a release gate
- Generate changelogs from real git history
- Create Git tags and GitHub Releases with artifacts
- `gh release create` publishes releases via GitHub CLI
- `fetch-depth: 0` ensures full git history is available

### Enterprise CI/CD Patterns

Production-grade practices demonstrated in this lab:
- **Build once, deploy everywhere** — identical artifact through all stages
- **Shift left** — lint and test early, fail fast before expensive steps
- **Approval gates** — human review before production deployments
- **Semantic versioning** — structured version numbering for predictable releases
- **Automated changelogs** — generated from commit history, not manual documentation
- **Caching** — reduce cost and time at scale
- **Least privilege** — minimal permissions on every workflow
- **Traceability** — build IDs, manifests, and artifacts for audit
