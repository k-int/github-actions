# Shared OpenAPI & FOLIO Composite Actions

This directory provides highly modular, reusable composite building blocks for handling OpenAPI specifications and FOLIO utility scripts. You can mix and match these components inside any custom pipeline workspace.

## Available Actions

### 1. Redocly Build & Bundle (`./redocly-build`)
Installs Node environments, sets up workspace caching, runs OpenAPI lints, and bundles split spec configurations into an optimized delivery layout.
#### Usage
```yaml
- uses: k-int/github-actions/.github/actions/openapi/redocly-build@main
```

### 2. Folio API Lint (`./folio-api-lint`)

Clones the upstream `folio-org/folio-tools` suite and runs the validation script `api_lint.py` to assert API compliance rules across target folders.

#### Usage
```yaml
- uses: k-int/github-actions/.github/actions/openapi/folio-api-lint@main
  with:
    api_directories: 'openapi'
```



### 3. Folio Schema Lint (`./folio-api-schema-lint`)

Clones the `folio-org/folio-tools` suite and targets JSON Schema assets using `api_schema_lint.py`.

#### Usage
```yaml
- uses: k-int/github-actions/.github/actions/openapi/folio-api-schema-lint@main

```



### 4. Folio API Doc & S3 Deploy (`./folio-api-doc-deploy`)

Compiles HTML/Markdown API distribution layouts and syncs tracking documentation assets into central AWS S3 buckets. Handles automated version extractions when triggered via releases.

#### Usage
```yaml
- uses: k-int/github-actions/.github/actions/openapi/folio-api-doc-deploy@main
  with:
    s3_access_key_id: ${{ secrets.S3_ACCESS_KEY_ID }}
    s3_secret_access_key: ${{ secrets.S3_SECRET_ACCESS_KEY }}
```