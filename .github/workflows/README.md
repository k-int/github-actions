# Workflow Catalog

This directory houses reusable workflows for K-Int internal projects.

> ⚠️ **Engine Restriction**: All reusable workflows in this directory must remain at the root level. GitHub Actions does not support execution from subdirectories.

## Standard Pipelines

### [Centralized Gradle SBOM Generation](./sbom-gradle.yml)
The standardized pipeline for dependency tracking, dynamic lockfile injections, and repository synchronization.

### [Centralized Git PR Delivery](./commit-via-pr.yml)
The reusable automation delivery engine that tracks workspace updates, creates isolated upstream tracking branches, opens PRs, and triggers automated rebase fast-forwards.

### [Folio OpenAPI Master Orchestrator](./folio-api-build.yml)
The high-level core orchestrator for complete FOLIO module API lifecycles. It chains together specification compilation and validation sub-pipelines using secure artifact handshakes.

### [Redocly Specification Compiler](./redocly-build.yml)
Handles the isolated validation, linting, and bundling of split OpenAPI specification templates into an optimized single asset. Features an integrated direct-commit engine that updates bundled specifications back onto feature PR branches automatically.

### [Folio API Tools Validation Suite](./folio-api-tools-pipeline.yml)
An independent validation and delivery engine wrapping FOLIO-specific scripts (`api-lint`, `api-schema-lint`, and `api-doc`). Runs validation jobs concurrently and pushes static documentation directly to AWS S3. Can be called downstream from an orchestrator or run directly on local specification directories.

---

## Architecture & Lifecycle Topology

When using the **Folio OpenAPI Master Orchestrator**, the end-to-end execution lifecycle passes through three decoupled phases across your independent sub-pipelines:

1. **Compilation & Delivery Stage (`redocly-build.yml`)**: Compiles raw templates into a single tracking specification file. If running within a `pull_request` context, it runs a direct push delivery script to commit the compiled file back to your branch window so it can be reviewed inside the code space.
2. **Parallel Validation Stage (`folio-api-tools-pipeline.yml`)**: Downloads the compiled specimen artifact and processes it concurrently through FOLIO's custom API validation and JSON Schema testing engines.
3. **Release & Deploy Stage (`folio-api-tools-pipeline.yml`)**: Generates final static HTML documentation from the verified specification and publishes tracking deployment assets to central AWS S3 buckets. (This stage triggers strictly on `main`/`master` pushes or semantic version release tags).

### ⚠️ Branch Protection & Merge Strategy (for PR Delivery workflows)
This delivery framework is designed to be fully "protection-aware." To function correctly across repositories with active branch rules, ensure the following requirements are met:

* **Permissions**: The calling job block **must** define explicit `pull-requests: write` and `contents: write` permissions.
* **Auto-Merge**:
  * If [Auto-merge](https://github.com/en/pulls/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/automatically-merging-a-pull-request) is enabled in repository settings, the workflow will automatically enqueue the generated compliance updates.
  * If [Merge Queues](https://github.com/en/pulls/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/merging-a-pull-request-with-a-merge-queue) are enabled on the destination branch, the workflow will queue the transaction automatically, bypassing direct push locks.
* **Fallback Rules**: If auto-merge features are completely turned off, the workflow will safely create the PR and leave a tracking timeline comment for a human maintainer to execute manual review actions.

---

## Examples & Templates
* **[Hello World Example](./hello-world.yml)**: A structural reference layout demonstrating parallel multi-job orchestration and cross-VM artifact passing.

---

## Deprecated Workflows
> **Note**: These workflows are no longer actively maintained or supported.
* `build-grails-4-gradle.yml`: Replaced. Implement standard open-source `gradle/gradle-build-action` modules instead.
* `folio-api-pipeline.yml`: Superceded. Refactored into separate modular workflows under the `folio-api-build.yml` orchestrator suite.