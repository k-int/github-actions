# DevOps Developer's Guide: Writing Orchestrations & Actions

Welcome to the engineering architecture guide for our organization's GitHub automation. This document outlines how to build, extend, and implement pipelines inside this repository while maintaining tight platform security.

As a DevOps engineer working here, you are composing **GitHub Reusable Workflows** and structural orchestration helpers. Because this entire repository is automatically mirrored to GitHub (`k-int/github-actions`), you have three architectural design patterns available depending on data sensitivity and complexity.

---

## The 3 Architectural Patterns (Choose Your Model)

When introducing a new workflow or automation requirement, map your strategy against our three core design patterns:

| Pattern | Where Logic Lives | Best Used For | Visual UI Graph? | Individual Retries? |
| :--- | :--- | :--- | :--- | :--- |
| **A. Privatized Multi-Job Graph** | `external-github-actions-scripts` (GitLab) | Proprietary, business-sensitive, multi-stage build/test/deploy logic. | **Yes** (Full DAG) | **Yes** (Per Job) |
| **B. Native Public Actions** | Directly in this repository under `.github/actions/` | Generic utilities, lint rules, open-source wrappers, non-sensitive tasks. | No (Single block) | No (All-or-nothing) |
| **C. Inlined Script Orchestration** | Raw execution steps within a single master job | Lightweight pipelines, execution sequences that don't need granular UI tracking. | No (Single card) | No (All-or-nothing) |

---

## Pattern A: Writing a Privatized Multi-Job Graph

Use this pattern if your execution code contains proprietary information *and* you want the full **visual dashboard experience** with step-level failure retries on GitHub.

### 1. The Core Logic (On GitLab Vault Repo)
Write your isolated composite steps inside `external-github-actions-scripts` under an intuitive domain path (e.g., `actions/domain/my-step/action.yml`).

### 2. The Blueprint Orchestrator (In this Repo)
Create your reusable workflow under `.github/workflows/your-pipeline.yml`. You **must** utilize the `clone-action-scripts` helper step at the beginning of *every independent job* to pull the private logic into that runner's memory safely. Global secrets are inherited natively by the helper.

```yaml
name: Shared Reusable Pipeline

on:
  workflow_call:
    secrets:
      GITLAB_DEPLOY_USER:
        required: true
      GITLAB_DEPLOY_TOKEN:
        required: true

jobs:
  job-stage-one:
    runs-on: ubuntu-latest
    steps:
      - name: Initialize Vault Scripts
        uses: ./.github/actions/clone-action-scripts # <-- Injects secret context automatically

        # The clone-action-scripts creates a directory called central-scripts
        # within which the individual scripts can be accessed
      - name: Execute Secret Task
        uses: ./central-scripts/actions/domain/my-step

  job-stage-two:
    runs-on: ubuntu-latest
    needs: job-stage-one # <-- Connects the visual dependency line
    steps:
      - name: Initialize Vault Scripts
        uses: ./.github/actions/clone-action-scripts
        
      - name: Execute Next Secret Task
        uses: ./central-scripts/actions/domain/next-step

```

---

## Pattern B: Housing Non-Sensitive / Native Actions Directly

If you are creating an action that contains **zero proprietary logic** (e.g., a standardized wrapper around standard Node linting, running yarn install configurations, or public style-checks), **you do not need to over-engineer it by splitting it into the private GitLab action-scripts repo.** You can house the entire Composite Action right here. It will mirror seamlessly to GitHub and can be consumed instantly by our repositories.

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
      uses: actions/setup-node@v4
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

This model is significantly cleaner for small sequential workflows because you only run the `clone-action-scripts` step **exactly once**, saving execution time.

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
        uses: ./.github/actions/clone-action-scripts # <-- Clones once here

        # The clone-action-scripts creates a directory called central-scripts
        # within which the individual scripts can be accessed
      - name: Run Sequential Step A
        uses: ./central-scripts/actions/hello-world/echo-hello

      - name: Run Sequential Step B
        uses: ./central-scripts/actions/hello-world/echo-world

      - name: Inline Bash Fallback Execution
        shell: bash
        run: |
          echo "Running an ad-hoc DevOps internal script straight from the runner..."
          bash ./central-scripts/scripts/custom-cleanup.sh

```

---

## Crucial Engineering Constraints

### 1. Mirroring Overwrite Warning

This repository uses an automated git mirror strategy managed via our `.gitlab-ci.yml` file. All changes **MUST** be committed on GitLab. Any direct code edits, PRs, or branching patterns executed on the GitHub web interface will be permanently wiped out on the next sync event.

### 2. The Cross-Org Secret Mapping Rule

Because downstream repositories (e.g., living in the `folio-org` organization namespace) consume these workflows out of our `k-int` space, **`secrets: inherit` will silently fail**. When writing user documentation for your new workflow, explicitly instruct developers that they must pass the context variables directly via the `secrets:` block on their implementation call.