# Workflow Catalog

This directory houses reusable workflows for K-Int internal projects.

## Standard Pipelines

### [Centralized Gradle SBOM Generation](./sbom-gradle.yml)
The standardized pipeline for dependency tracking, dynamic lockfile injections, and repository synchronization.

### [Centralized Git PR Delivery](./commit-via-pr.yml)
The reusable automation delivery engine that tracks workspace updates, creates isolated upstream tracking branches, opens PRs, and triggers automated rebase fast-forwards.

#### ⚠️ Branch Protection & Merge Strategy
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
