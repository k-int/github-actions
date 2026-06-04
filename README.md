# Centralized GitHub Actions Pipelines

This repository serves as the centralized orchestration hub for DevOps, security, and build workflows executed across our organization's GitHub repositories. 

It provides reusable, multi-job structural skeletons (**GitHub Reusable Workflows**) that render job graphs, isolate environments, and grants granular job-level retries inside the GitHub Actions UI.

⚠️ **SECURITY NOTICE:** This repository contains **zero proprietary logic, scripts, or business code**. It contains only structural framework files. Most actual processing code remains securely locked behind our private GitLab instance and is fetched dynamically into volatile runner environments at runtime.

---

## 🛑 Developer Notice
> **CRITICAL FOR DEVOPS ENGINEERS:** If you are modifying, extending, or adding new pipelines/actions to this repository, you **must** read the developer specifications before committing any code. Failure to follow the established architectural design patterns will break downstream consumer pipelines.
>
> 👉 **[Read the Internal DevOps Engineering Guide (DEVELOPMENT.md)](./DEVELOPMENT.md)**

### Automated Synchronization
This repository is maintained on GitLab and automatically synchronized to GitHub (`k-int/github-actions`) using a mirror engine.

**Do not commit changes directly to GitHub;** all updates must be made in the primary upstream GitLab repository. Any updates made directly to GitHub will be silently overwritten and wiped out by the next sync process.

---

## Prerequisites & Authentication

Because the execution logic is stored in a private upstream vault, any host repository utilizing these workflows **must** configure access credentials, or the pipeline will instantly fail on step 1.

### 1. Retrieve the Runtime Credentials
The credentials for the required read-only deployment token are securely stored in Keeper:
* **Keeper Path:** `Libraries/Gitlab DevOps/deploy token READ_ACTIONS_SCRIPTS`

### 2. Configure Your Host Repository
Before implementing a workflow, add the token to your application's GitHub repository:
1. Navigate to your GitHub repository -> **Settings** -> **Secrets and variables** -> **Actions**.
2. Create the following two **Repository Secrets**:
   * `GITLAB_DEPLOY_USER` : *(The username string from Keeper)*
   * `GITLAB_DEPLOY_TOKEN` : *(The alphanumeric token string from Keeper)*

---

## How to Implement

To consume a centralized workflow, create a file in your application repository (e.g., `.github/workflows/ci.yml`) and reference the mirrored structural file using the `uses:` keyword.

> 🚨 **CROSS-ORGANIZATION SECURITY RULE:** Because these reusable workflows cross GitHub Organization boundaries (e.g., from `folio-org` to `k-int`), implicit inheritance (`secrets: inherit`) is blocked by GitHub's compiler. You **must** explicitly map the secrets into the call block as shown below.

### Example Implementation

```yaml
name: Application CI Pipeline

on:
  push:
    branches: [ "main" ]
  pull_request:
  workflow_dispatch:

jobs:
  # This single block calls the central structural blueprint
  core-pipeline:
    uses: k-int/github-actions/.github/workflows/hello-world-ui.yml@main
    
    # Explicitly pass down the GITLAB_DEPLOY secrets to cross the organization boundary
    secrets:
      GITLAB_DEPLOY_USER: ${{ secrets.GITLAB_DEPLOY_USER }}
      GITLAB_DEPLOY_TOKEN: ${{ secrets.GITLAB_DEPLOY_TOKEN }}

```

---

## Expected UI Layout

By referencing these reusable blueprints, your repository inherits a native, interactive visual matrix directly within the GitHub Actions console, allowing you to debug and re-run failed sub-tasks independently without resetting the entire pipeline.
