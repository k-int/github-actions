# Workflow Catalog

This repository houses the standardized, reusable GitHub Actions workflows for K-Int internal projects. 

> ⚠️ **Engine Restriction**: All reusable workflows in this repository must remain at the root level. GitHub Actions does not support invoking reusable workflows located within subdirectories.

---

## Standard Pipelines

| Workflow                                                               | Purpose | Integration Type |
|:-----------------------------------------------------------------------| :--- | :--- |
| **[Folio OpenAPI Pipeline](./folio-api-build.yml)**                    | High-level core orchestrator for complete FOLIO module API lifecycles. Chains together compilation and validation sub-pipelines using secure artifact handshakes. | Orchestrator (`uses`) |
| **[Redocly Specification Compiler](./redocly-build.yml)**              | Handles isolated validation, linting, and bundling of split OpenAPI specification templates. Includes an auto-commit engine for PR branches. | Sub-pipeline / Direct |
| **[Folio API Tools Validation Suite](./folio-api-tools-pipeline.yml)** | Runs `api-lint`, `api-schema-lint`, and `api-doc` concurrently. Deploys verified assets directly to AWS S3. **Non-destructive**—processes directory blocks safely. | Sub-pipeline / Direct |
| **[Centralized Gradle SBOM Generation](./sbom-gradle.yml)**            | Standardized engine for dependency tracking, dynamic lockfile injections, and security bill-of-materials sync. | Independent |
| **[Centralized Git PR Delivery](./commit-via-pr.yml)**                 | Automation engine that tracks workspace updates, creates isolated upstream tracking branches, and automates fast-forward rollouts. | Independent |

---

## Required Repository Layout Conventions

To use the **Folio OpenAPI Pipeline** safely without running into pipeline purges or path-clashing errors, consumer repositories **must** isolate development configurations from active specification assets. 

Set up your repository tree using the following unified structure:

```text
your-repository/
└── docs/
    └── API/
        ├── config/
        │   ├── package.json         # Contains Redocly CLI deps
        │   ├── package-lock.json
        │   └── redocly.yaml         # Redocly routing rules
        └── yamls/
            ├── agreements.yaml       # Output target for the compiler
            └── ad-hoc-spec.yaml     # Optional: Any extra manually managed specs

```

> 💡 **The Directory Advantage**: Because the validation suite targets the entire `yamls/` folder natively, any additional hand-crafted or ad-hoc OpenAPI specifications placed in that directory will automatically be tracked, linted, and deployed to S3 alongside your main generated specification.

---

## Architecture & Lifecycle Topology

When a repository triggers the **Folio OpenAPI Pipeline**, the end-to-end execution lifecycle passes through three decoupled phases across isolated runner VMs:

```text
 [ Push/PR Event ]
         │
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ 1. Compilation Stage (redocly-build.yml)               │
 │    - Installs node tooling from /config                │
 │    - Bundles templates into target file within /yamls  │
 │    - Auto-commits changes back if running inside a PR  │
 └───────────────────────┬────────────────────────────────┘
                         │ (Pushes Entire /yamls Folder via Artifact)
                         ▼
 ┌────────────────────────────────────────────────────────┐
 │ 2. Parallel Validation Stage (folio-api-tools-suite)   │
 │    ├─► Folio API Lint                                  │
 │    └─► Folio Schema Lint                               │
 └───────────────────────┬────────────────────────────────┘
                         │ (Runs Only on Production Track Branches)
                         ▼
 ┌────────────────────────────────────────────────────────┐
 │ 3. Release & Deploy Stage (folio-api-tools-suite)      │
 │    - Compiles static HTML documentation matrices       │
 │    - Deploys full workspace specifications to AWS S3   │
 └────────────────────────────────────────────────────────┘

```

---

## Implementation Reference Matrix

To activate the orchestration suite within an application repository, create `.github/workflows/openapi-pipeline.yml` using this standardized block:

```yaml
name: OpenAPI Pipeline

on:
  push:
    branches: [ main, master ]
  pull_request:

jobs:
  run-openapi-pipeline:
    uses: k-int/github-actions/.github/workflows/folio-api-build.yml@main
    permissions:
      contents: write
      pull-requests: write
    with:
      redocly-dir: docs/API/config
      spec-dir: docs/API/yamls
      openapi-filename: agreements.yaml
      # allowed-deploy-branch: "optional-feature-test-branch" # Defaults to 'main master'
    secrets:
      S3_ACCESS_KEY_ID: ${{ secrets.S3_ACCESS_KEY_ID }}
      S3_SECRET_ACCESS_KEY: ${{ secrets.S3_SECRET_ACCESS_KEY }}
      GITLAB_DEPLOY_USER: ${{ secrets.GITLAB_DEPLOY_USER }}
      GITLAB_DEPLOY_TOKEN: ${{ secrets.GITLAB_DEPLOY_TOKEN }}

```

### ⚠️ Branch Protection & Merge Strategy (for PR Delivery workflows)

This delivery framework is designed to be fully protection-aware. To function correctly across enterprise repositories with active branch rules, ensure the following requirements are met:

* **Permissions**: The calling job block **must** define explicit `pull-requests: write` and `contents: write` permissions.
* **Auto-Merge Automation**:
* If **Auto-merge** is enabled in repository settings, the workflow will automatically enqueue the generated compliance updates.
* If **Merge Queues** are enabled on your destination branch, the workflow will queue the transaction automatically, bypassing direct push locks.


* **Fallback Rules**: If auto-merge features are completely turned off in the repo, the workflow will safely create the PR and leave a tracking timeline comment for a human maintainer to execute manual merge actions.

---

## Sandbox & Reference Templates

* **[Hello World Structural Blueprint](https://www.google.com/search?q=./hello-world.yml)**: A structural reference layout demonstrating parallel multi-job orchestration, runtime environments, and cross-VM artifact passing.

---

## Deprecated Workflows

> ⛔ **Archived**: These files are no longer actively maintained and should not be implemented in new projects.

* `build-grails-4-gradle.yml`: Superseded. Implement official open-source `gradle/actions/setup-gradle` ecosystems directly.
* `folio-api-pipeline.yml`: Broken down. Refactored into separate modular files under the decoupled `folio-api-build.yml` orchestrator model.
