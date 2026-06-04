# Centralized GitHub Actions Pipelines

This repository serves as the centralized orchestration hub for DevOps, security, and build workflows executed across our organization's GitHub repositories. 

It provides reusable, multi-job structural skeletons (**GitHub Reusable Workflows**) that render native dependency graphs (DAGs), isolated environments, and granular job-level retry capabilities inside the GitHub Actions UI.

⚠️ **SECURITY NOTICE:** This repository contains **zero proprietary logic, scripts, or business code**. It contains only structural framework files. The actual processing code remains securely locked behind our private GitLab instance and is fetched dynamically into volatile runner environments at runtime.

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
    uses: k-int/github-actions/.github/workflows/hello-world.yml@main
    
    # Explicitly pass down the GITLAB_DEPLOY secrets to cross the organization boundary
    secrets:
      GITLAB_DEPLOY_USER: ${{ secrets.GITLAB_DEPLOY_USER }}
      GITLAB_DEPLOY_TOKEN: ${{ secrets.GITLAB_DEPLOY_TOKEN }}

```

---

## Expected UI Layout

By referencing these reusable blueprints, your repository inherits a native, interactive visual matrix directly within the GitHub Actions console, allowing you to debug and re-run failed sub-tasks independently without resetting the entire pipeline.

## 📋 Centralized Pipeline & Utility Catalog

This repository acts as the visual layout orchestrator. Below is the structural index of available reusable blueprints and internal tooling.

### Reusable Workflows (End-User Blueprints)

| Workflow Blueprint File         | Canonical Path (`uses:`)                                                | Strategy Pattern                | Status        |
|---------------------------------|-------------------------------------------------------------------------|---------------------------------|---------------|
| **`sbom-gradle.yml`**           | `k-int/github-actions/.github/workflows/sbom-gradle.yml@main`           | **Pattern A** (Multi-Job Graph) | ● Active      |
| **`hello-world.yml`**           | `k-int/github-actions/.github/workflows/hello-world.yml@main`           | **Pattern A** (Demo Layout)     | ● Active      |
| **`build-grails-4-gradle.yml`** | `k-int/github-actions/.github/workflows/build-grails-4-gradle.yml@main` | Native Legacy Build             | ⚠️ Deprecated |

---

### Architecture Demo: `hello-world.yml`

Our standard multi-job baseline orchestration. It splits tasks across three independent runner platforms (Hello, World, and Combine) to demonstrate visual dependency tracking and multi-stage artifact passing across ephemeral boundaries.

#### Implementation

```yaml
jobs:
   core-pipeline:
      uses: k-int/github-actions/.github/workflows/hello-world.yml@main
      secrets:
         GITLAB_DEPLOY_USER: ${{ secrets.GITLAB_DEPLOY_USER }}
         GITLAB_DEPLOY_TOKEN: ${{ secrets.GITLAB_DEPLOY_TOKEN }}

```

---

### Security Compliance: `sbom-gradle.yml`

Generates an accurate Software Bill of Materials (SBOM) across Gradle projects. It uses an *in situ* dynamic lockfile injector before scanning via `anchore/syft`.

#### Inputs

* `gradle_project_path` *(String, Optional, Default: `.`)*: Relative directory path containing the target `gradlew` wrapper script (useful for monorepos like `mod-agreements/service`).
* `syft_args` *(String, Optional, Default: `''`)*: Direct passthrough arguments for Syft. If blank, defaults to `-o cyclonedx-json=cdx-bom.json`.
* `tracked_files` *(String, Optional, Default: `cdx-bom.json`)*: Space-separated list of generated compliance files for the Git engine to track and pull.
* `commit_lockfiles` *(String, Optional, Default: `'false'`)*: Set to `'true'` to commit the dynamically compiled `gradle.lockfile` elements to your repository history alongside the BOM.

#### Implementation

```yaml
jobs:
   generate-sbom:
      permissions:
         contents: write
      uses: k-int/github-actions/.github/workflows/sbom-gradle.yml@main
      secrets:
         GITLAB_DEPLOY_USER: ${{ secrets.GITLAB_DEPLOY_USER }}
         GITLAB_DEPLOY_TOKEN: ${{ secrets.GITLAB_DEPLOY_TOKEN }}

```

---


### Legacy Automation: `build-grails-4-gradle.yml`

> 🛑 **DEPRECATION NOTICE:** This pipeline has been functional for an extended lifecycle period. It relies on legacy action versions (`actions/checkout@v3`, `setup-java@v3`) and does not utilize our secure automated GitLab cloning framework. **Do not use this blueprint for new repository onboarding initiatives.**

Standard Java 11 / Grails 4 workspace execution compiling packages natively using raw Gradle wrapper build arguments.

---

###  Shared Helpers (Internal Tooling)

#### `clone-action-scripts` (Composite Action)

* **Location:** `.github/actions/clone-action-scripts/action.yml`
* **Access Level:** Internal Infrastructure Helper.

This internal utility abstracts our vault synchronization logic. It maps corporate credentials to run a shallow clone (`--depth 1`) of our private business logic into the volatile runner footprint under `./central-scripts`.

> 💡 **DevOps Note:** This action must be declared at the initiation sequence of **every independent job block** within Pattern A configurations, as separate runner instances do not share on-disk storage spaces.
> ```yaml
> - name: Initialize action scripts
>   uses: k-int/github-actions/.github/actions/clone-action-scripts@main
>   with:
>     deploy_user: ${{ secrets.GITLAB_DEPLOY_USER }}
>     deploy_token: ${{ secrets.GITLAB_DEPLOY_TOKEN }}
> 
> ```
>
>