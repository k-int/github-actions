# Folio Ecosystem Specific Workflows

This directory contains targeted orchestrator workflows designed specifically around the configurations, requirements, and compliance standards of FOLIO modules.

## Workflows

### [Folio OpenAPI Pipeline](./folio-api-pipeline.yml)

A multi-stage delivery pipeline that wraps together local project dependencies, Redocly tools, and upstream Python validation utilities seamlessly.

#### Pipeline Topology
1. **Validation Stage**: Executes Redocly specification lints alongside FOLIO's custom schema and API testing engines in parallel paths. (Triggers on both Pushes and Pull Requests).
2. **Build & Release Stage**: Runs bundlers, exports generated documentation sites, and safely deploys static assets out to centralized AWS S3 tracking buckets. (Triggers strictly on `main`/`master` pushes or semantic version release tags).

#### Inputs & Secrets Integration
To utilize this workflow wrapper, invoke it inside your workflow file via `uses:` blocks and supply required target secrets:

```yaml
jobs:
  run-pipeline:
  - uses: k-int/github-actions/.github/workflowws/folio-specific/folio-api-pipeline@main
    secrets:
      S3_ACCESS_KEY_ID: ${{ secrets.YOUR_S3_KEY_ID }}
      S3_SECRET_ACCESS_KEY: ${{ secrets.YOUR_S3_SECRET }}
```