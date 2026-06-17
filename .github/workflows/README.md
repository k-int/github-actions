# Workflow Catalog

This directory houses reusable workflows for K-Int internal projects.

## Standard Pipelines

### [Centralized Gradle SBOM Generation](./sbom-gradle.yml)
The standardized pipeline for dependency tracking, automated compliance reporting, and repository synchronization.

#### ⚠️ Branch Protection & Merge Strategy
This workflow is designed to be "protection-aware." To function correctly, ensure the following requirements are met:

* **Permissions**: The calling job **must** define `pull-requests: write` permissions.
* **Auto-Merge**:
    * If [Auto-merge](https://docs.github.com/en/pulls/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/automatically-merging-a-pull-request) is enabled in repository settings, the workflow will automatically attempt to merge the generated SBOM PR.
    * If [Merge Queues](https://docs.github.com/en/pulls/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/merging-a-pull-request-with-a-merge-queue) are enabled on the target branch, the workflow will automatically queue the update, bypassing direct push restrictions.
* **Fallback**: If auto-merge is disabled, the workflow will create the PR and leave a comment for a human maintainer to perform the final review and merge.

---

## Examples & Templates
* **[Hello World Example](./hello-world.yml)**: A structural reference for multi-job orchestration and artifact passing.

---

## Deprecated
> **Note**: These workflows are no longer actively maintained.
* `build-grails-4-gradle.yml`: Use standard `gradle/gradle-build-action` patterns.
