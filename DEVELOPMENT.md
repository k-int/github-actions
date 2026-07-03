# DevOps Developer's Guide: Writing Orchestrations & Actions

Welcome to the engineering architecture guide for our organization's GitHub automation. This document outlines how to build, extend, and implement pipelines inside this repository while maintaining tight platform security.

As a DevOps engineer working here, you are composing **GitHub Reusable Workflows** and structural orchestration helpers. Because this entire repository is automatically mirrored to GitHub (`k-int/github-actions`), you have three architectural design patterns available depending on data sensitivity and complexity.

---

## The 3 Architectural Patterns (Choose Your Model)

When introducing a new workflow or automation requirement, map your strategy against our three core design patterns:

| Pattern | Where Logic Lives | Best Used For | Visual UI Graph? | Individual Retries? |
| :--- | :--- | :--- | :--- | :--- |
| **A. Privatized Multi-Job Graph** | `shared-pipeline-scripts` (GitLab) | Proprietary, business-sensitive, multi-stage build/test/compliance logic. | **Yes** (Full DAG) | **Yes** (Per Job) |
| **B. Native Public Actions** | Directly in this repository under `.github/actions/` | Generic utilities, lint rules, open-source wrappers, non-sensitive tasks. | No (Single block) | No (All-or-nothing) |
| **C. Inlined Script Orchestration** | Raw execution steps within a single master job | Lightweight pipelines, execution sequences that don't need granular UI tracking. | No (Single card) | No (All-or-nothing) |

---

## Pattern A: Writing a Privatized Multi-Job Graph

Use this pattern if your execution code contains proprietary information *and* you want a clear visual dashboard experience with step-level failure retries on GitHub.

### 1. The Core Logic (On GitLab Vault Repo)
Write your standalone shell scripts or isolated composite actions inside `shared-pipeline-scripts` under an intuitive domain path (e.g., `scripts/sbom-generation/gradle/generate-locks.sh`).

### 2. The Blueprint Orchestrator (In this Repo)
Create your reusable workflow under `.github/workflows/your-pipeline.yml`. You **must** utilize our DRY `clone-pipeline-scripts` helper step at the beginning of *every independent job* to pull the private logic into that runner's memory safely.

> 🚨 **CRITICAL DEVELOPMENT RULE:** You **cannot** reference the clone helper using a relative path like `uses: ./.github/actions/clone-pipeline-scripts` inside a reusable workflow. It will evaluate relative to the *caller application workspace* and break. You **must** use its full canonical path:

```yaml
jobs:
  job-stage-one:
    runs-on: ubuntu-latest
    steps:
      - name: Initialize Pipeline Scripts
        uses: k-int/github-actions/.github/actions/clone-pipeline-scripts@main # <-- MUST use absolute path
        with:
          deploy_user: ${{ secrets.GITLAB_DEPLOY_USER }}
          deploy_token: ${{ secrets.GITLAB_DEPLOY_TOKEN }}

      - name: Execute Shell Script Core
        shell: bash
        run: |
          chmod +x ./pipeline-scripts/scripts/domain/my-script.sh
          ./pipeline-scripts/scripts/domain/my-script.sh --arg value

```

---

## Pattern B: Housing Non-Sensitive / Native Actions Directly

If you are creating an action that contains **zero proprietary logic** (e.g., a standardized wrapper around standard Node linting, running yarn install configurations, or public style-checks), **do not over-engineer it by splitting it into GitLab.** You can house the entire Composite Action right here. It will mirror seamlessly to GitHub and can be consumed instantly by our repositories.

### Implementation Setup

1. Create a folder structure under `.github/actions/your-utility-name/`.
2. Author a standard native `action.yml` file.

```yaml
# Location: .github/actions/yarn-lint-helper/action.yml
name: 'Standard Yarn Linter'
description: 'Runs standard linting safely for non-sensitive apps'

runs:
  using: "composite"
  steps:
    - name: Setup Node
      uses: actions/setup-node@v5
      with:
        node-version: '22.x'
    - name: Install and Run
      shell: bash
      run: |
        yarn install --frozen-lockfile
        yarn lint

```

---

## Pattern C: Single-Runner Inline Script Orchestration

Sometimes, a pipeline needs to pull proprietary logic from GitLab, but you **don't** want or need a massive multi-job grid on the GitHub UI dashboard. You just want one worker machine to turn on, run a sequence of scripts, and shut down.

This model is significantly cleaner for small sequential workflows because you only run the `clone-pipeline-scripts` helper step **exactly once**, saving execution time and completely bypassing the multi-cloning constraint.

### Implementation Setup

Define a standard single-job reusable workflow structure. Run the vault initialization step at the very top of the script array, and then execute your shell tasks or target actions sequentially within that exact same runner footprint:

```yaml
# Location: .github/workflows/lightweight-sync.yml
name: Lightweight Monolithic Sync

on:
  workflow_call:
    secrets:
      GITLAB_DEPLOY_USER:
        required: true
      GITLAB_DEPLOY_TOKEN:
        required: true

jobs:
  execute-all-in-one:
    runs-on: ubuntu-latest
    steps:
      - name: Pull Private Logic Layer
        uses: k-int/github-actions/.github/actions/clone-pipeline-scripts@main # <-- Clones once here
        with:
          deploy_user: ${{ secrets.GITLAB_DEPLOY_USER }}
          deploy_token: ${{ secrets.GITLAB_DEPLOY_TOKEN }}

      - name: Run Sequential Step A
        shell: bash
        run: ./pipeline-scripts/hello-world/echo-hello.sh

      - name: Run Sequential Step B
        shell: bash
        run: ./pipeline-scripts/hello-world/echo-world.sh

```

---

## Crucial Engineering Constraints

### 1. Mirroring Overwrite Warning

This repository uses an automated git mirror strategy managed via our `.gitlab-ci.yml` file. All changes **MUST** be committed on GitLab. Any direct code edits, PRs, or branching patterns executed on the GitHub web interface will be permanently wiped out on the next sync event.

### 2. The Cross-Org Secret Mapping Rule

Because downstream repositories (e.g., living in the `folio-org` organization namespace) consume these workflows out of our `k-int` space, **`secrets: inherit` will silently fail**. When writing user documentation for your new workflow, explicitly instruct developers that they must pass the context variables directly via the `secrets:` block on their implementation call.

## 🧪 Testing & Branch Management Workflow

Because this repository acts as a core infrastructure dependency for the entire organization, changes must be validated safely before hitting production.

To facilitate this, both the `main` and `test` branches are automatically mirrored from GitLab to GitHub (`k-int/github-actions`). **The `test` branch must always exist on both platforms; if it is deleted or missing, downstream pipelines targeting it will instantly break.**

When modifying workflows, developing new features, or fixing bugs, you must follow this strict development lifecycle:

### The 5-Step Pipeline Lifecycle

| Step | Phase | Action | Target Branch |
| :--- | :--- | :--- | :--- |
| **1** | **Isolate** | Commit and push your workflow updates to the upstream **GitLab** `test` branch. | `test` |
| **2** | **Target** | Point a consumer test application to the `@test` tag of the reusable blueprint. | `test` |
| **3** | **Verify** | Execute the consumer pipeline, check the visual UI layout, and verify script execution. | `test` |
| **4** | **Promote** | Open a Merge Request (MR) from `test` to `main` inside GitLab and merge it. | `main` |
| **5** | **Sync** | Reset/Fast-forward the `test` branch to the new **TIP** of `main` to prepare for the next change. | `test` |

---

### Step-by-Step Implementation Guide

#### 1. Push to GitLab Test
Make your structural changes or update your script definitions in the upstream GitLab repository on the `test` branch. Wait a moment for the mirror engine to push the updates to GitHub.

#### 2. Configure a Consumer App for Testing
In your target application repository (or a scratch testing repo), modify your workflow file to point explicitly to the `@test` branch version of the reusable orchestrator:

```yaml
name: Application CI - Sandbox Testing

on:
  workflow_dispatch: # Allowed for easy manual triggering during tests

jobs:
  core-pipeline:
    # 💥 CRITICAL: Target the @test branch instead of @main
    uses: k-int/github-actions/.github/workflows/hello-world.yml@test
    
    secrets:
      GITLAB_DEPLOY_USER: ${{ secrets.GITLAB_DEPLOY_USER }}
      GITLAB_DEPLOY_TOKEN: ${{ secrets.GITLAB_DEPLOY_TOKEN }}
```

Trigger this workflow manually via the GitHub Actions UI to verify that your new logic behaves exactly as expected.

#### 3. Promote to Production (`main`)
Once your validation passes completely:
1. Navigate to **GitLab**.
2. Open a Merge Request from `test` ➔ `main`.
3. Complete the code review process and **Merge**.
4. The mirror engine will immediately push `main` to GitHub, making your changes live for all production consumers.

#### 4. Refresh the Test Branch
To prevent code drift and ensure the next developer starts from a clean slate, you **must** fast-forward the `test` branch to match the production tip immediately after merging.

Run the following commands in your local terminal (configured to your GitLab upstream remote):

```bash
# Fetch the latest changes from upstream
git fetch origin

# Switch to main and pull the merged changes
git checkout main
git pull origin main

# Force-reset test to the exact state of main
git checkout test
git reset --hard main

# Push the synchronized test branch back upstream
git push origin test --force
```

> ⚠️ **CRITICAL WARNING:** Always coordinate with your team before executing a `--force` push to the `test` branch to ensure you do not overwrite another engineer's active pipeline testing session.