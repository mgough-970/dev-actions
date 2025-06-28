# GitHub Actions Workflows for Jupyter Notebook Projects

This repository provides a robust and flexible CI/CD solution for Jupyter Notebook projects, centered around automated testing, execution, security scanning, versioning of executed notebooks, and publishing high-quality JupyterBooks.

## 📋 Table of Contents

- [Overview](#overview)
  - [Core Architecture](#core-architecture)
  - [Key Features](#key-features)
- [Workflows Provided](#workflows-provided)
  - [1. Notebook CI Pipeline (`ci_pipeline.yml`) - Reusable](#1-notebook-ci-pipeline-ci_pipelineyml---reusable)
  - [2. JupyterBook Publisher (`jupyterbook_build.yml`) - Complete](#2-jupyterbook-publisher-jupyterbook_buildyml---complete)
  - [3. Scheduled Notebook Tests (`scheduled_notebook_tests.yml`) - Complete](#3-scheduled-notebook-tests-scheduled_notebook_testsyml---complete)
  - [4. Deprecation Manager (`ci_deprecation_manager.yml`) - Reusable](#4-deprecation-manager-ci_deprecation_manageryml---reusable)
- [Example Caller Workflows](#example-caller-workflows)
  - [Pull Request Checks](#pull-request-checks)
  - [On-Demand Operations](#on-demand-operations)
  - [Scheduled Health Checks](#scheduled-health-checks)
- [Quick Start: Local Testing](#quick-start-local-testing)
- [Prerequisites](#prerequisites)
- [Package Manager Strategy](#package-manager-strategy)
- [Versioning & Releases](#versioning--releases)
- [Contributing](#contributing)

## 🎯 Overview

This system streamlines the development and maintenance of notebook-based projects by automating critical quality and publishing tasks.

### Core Architecture

The architecture is designed for reliability and flexibility:

1.  **`ci_pipeline.yml` (Reusable Workflow)**:
    *   The workhorse for all notebook processing: validation (`pytest nbval`), execution (`jupyter nbconvert`), and security scanning (`bandit`).
    *   Handles various triggers: Pull Requests, on-demand manual runs, and scheduled checks.
    *   Intelligently selects notebooks based on changes in a PR (direct notebook changes, `requirements.txt` updates, or general code changes).
    *   Bypasses notebook processing for PRs with only documentation or static file changes.
    *   Crucially, for PRs and specific on-demand operations, it pushes successfully executed (and non-deprecated) notebooks to the `gh-storage` branch.

2.  **`gh-storage` Branch**:
    *   A dedicated data branch in your repository. It stores the executed versions of your notebooks, ensuring that your documentation is built from tested and up-to-date notebook outputs.
    *   Also stores static files (like `.md`, images) that are part of your documentation if a PR only contains those changes.

3.  **`jupyterbook_build.yml` (Complete Workflow)**:
    *   Automatically triggers on pushes to your main branch (e.g., after a PR merge).
    *   Checks out the necessary JupyterBook configuration (`_config.yml`, `_toc.yml`, etc.) from your main branch and the executed notebooks (and static files) from the `gh-storage` branch.
    *   Builds the JupyterBook.
    *   Supports an optional post-processing script for custom modifications before deployment.
    *   Deploys the final HTML site to your `gh-pages` branch (or as configured).

4.  **`scheduled_notebook_tests.yml` (Complete Workflow)**:
    *   Runs weekly (or on your configured schedule) to perform a full suite of tests (execute, validate, security) on all notebooks, ensuring ongoing project health without affecting `gh-storage` or published docs.

### Key Features

- **Comprehensive PR Checks**: Ensures notebooks are valid, executable, and secure before merging.
- **Versioned Executed Notebooks**: `gh-storage` acts as a reliable source of truth for executed notebook content.
- **Automated JupyterBook Publishing**: Keeps your documentation site effortlessly up-to-date.
- **Flexible On-Demand Operations**: Manually trigger tests, storage updates, or full documentation rebuilds for all or specific notebooks.
- **Scheduled Health Checks**: Proactively catch regressions or environment issues.
- **Customizable Environments**: Use directory-specific `requirements.txt`, a root `requirements.txt`, or a custom requirements file.
- **Deprecated Notebook Handling**: Avoids overwriting specially marked deprecated notebooks in `gh-storage` by using a user-provided check script.
- **Static File Handling**: PRs with only static file changes (e.g., markdown, images) will update these files in `gh-storage` for inclusion in the book.

## 🔧 Workflows Provided

This repository offers the following core workflow files, typically found in `.github/workflows/`:

### 1. Notebook CI Pipeline (`ci_pipeline.yml`) - Reusable

**Purpose**: The central engine for notebook processing. Called by other workflows.

**Trigger**: `workflow_call`

**Key Inputs**:
| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `python-version` | string | ❌ | `'3.11'` | Python version for the environment. |
| `operation-mode` | string | ✅ | - | Defines behavior: `pr-check`, `on-demand-test`, `on-demand-store`, `scheduled-test`. |
| `single-filename` | string | ❌ | - | Path to a single notebook for on-demand/test modes. |
| `custom-requirements-path` | string | ❌ | - | Path to a custom `requirements.txt`. |
| `notebook-sources-path` | string | ❌ | `'./'` | Root path to search for notebooks. |
| `force-store` | boolean | ❌ | `false` | For `on-demand-store`, set to `true` to store to `gh-storage`. |
| `is-deprecated-check-script` | string | ❌ | - | Path to a script that checks if a notebook is deprecated (outputs 'true' or 'false'). Used to prevent overwriting deprecated notebooks in `gh-storage`. |

**Key Secrets**:
| Secret | Required | Description |
|---|---|---|
| `CASJOBS_USERID` | ❌ | Optional CasJobs User ID. |
| `CASJOBS_PW` | ❌ | Optional CasJobs Password. |
| `GH_TOKEN_FOR_STORAGE` | ✅ | GitHub token with write access to `gh-storage` branch. Required by definition, but only used if an operation involves writing to `gh-storage`. |

**Core Logic**:
1. Analyzes changed files (for PRs) or inputs to select notebooks.
2. Sets up Python environment based on `requirements.txt` hierarchy.
3. Performs validation, execution, and security scans based on `operation-mode`.
4. If applicable (and notebook not deprecated), pushes executed notebooks to `gh-storage`.

### 2. JupyterBook Publisher (`jupyterbook_build.yml`) - Complete

**Purpose**: Builds and deploys your JupyterBook site after merges to the main branch.

**Trigger**: `on: push: branches: [main]` (or your default branch)

**Key Features**:
- Sources executed notebooks and static files from the `gh-storage` branch.
- Sources book configuration (`_config.yml`, `_toc.yml`) from the main branch.
- Optionally runs a post-processing script before deploying to `gh-pages`.

**Configuration**:
- `POST_PROCESSING_SCRIPT` (environment variable within the workflow): Set the path to your optional post-build script (e.g., `scripts/jdaviz_image_replacement.sh`).

### 3. Scheduled Notebook Tests (`scheduled_notebook_tests.yml`) - Complete

**Purpose**: Performs a weekly health check of all notebooks.

**Trigger**: `on: schedule: cron: '0 3 * * 0'` (configurable) and `workflow_dispatch`

**Action**: Calls `ci_pipeline.yml` with `operation-mode: scheduled-test`. Does not modify `gh-storage`.

### 4. Deprecation Manager (`ci_deprecation_manager.yml`) - Reusable
*(This workflow was part of the original set and is assumed to still be relevant for managing notebook deprecation states, which `ci_pipeline.yml` can now leverage via `is-deprecated-check-script`.)*
- **Purpose**: Manages notebook lifecycle, including marking for deprecation.
- Refer to its specific documentation if available.

## 📁 Example Caller Workflows

Copy these examples from `notebook-ci-actions/examples/workflows/` to your repository's `.github/workflows/` directory and adapt as needed. Remember to change `uses: ./.github/workflows/...` to `uses: spacetelescope/notebook-ci-actions/.github/workflows/...@main` (or a specific version tag).

### Pull Request Checks
(`.github/workflows/notebook_pr_checks.yml` - based on `examples/workflows/notebook-ci-pr.yml`)
- Calls `ci_pipeline.yml` with `operation-mode: 'pr-check'`.
- Includes a conditional job `handle-static-files-pr` that runs if `ci_pipeline.yml` indicates only static/doc files were changed, pushing them to `gh-storage`.

```yaml
# Example: .github/workflows/notebook_pr_checks.yml
name: Notebook CI - Pull Request

on:
  pull_request:
    branches: [main, develop]
    paths: # Define paths relevant to your project
      - 'notebooks/**'
      - 'src/**'
      - 'docs/**'
      - 'assets/**'
      - 'images/**'
      - '**/*.md'
      - '**/*.html'
      - '**/*requirements.txt'
      - 'requirements.txt'
      - '_config.yml'
      - '_toc.yml'

jobs:
  notebook-pr-check:
    name: Notebook Processing CI
    uses: spacetelescope/notebook-ci-actions/.github/workflows/ci_pipeline.yml@main # Adjust version
    outputs:
      bypass_notebook_processing: ${{ steps.initial_setup.outputs.bypass_notebook_processing }}
      is_docs_or_static_only_change: ${{ steps.initial_setup.outputs.is_docs_or_static_only_change }}
    with:
      operation-mode: 'pr-check'
      python-version: '3.11'
      notebook-sources-path: 'notebooks/' # Adjust if your notebooks are elsewhere
      is-deprecated-check-script: 'scripts/check_deprecated.sh' # Example path
    secrets:
      CASJOBS_USERID: ${{ secrets.CASJOBS_USERID }}
      CASJOBS_PW: ${{ secrets.CASJOBS_PW }}
      GH_TOKEN_FOR_STORAGE: ${{ secrets.YOUR_GH_TOKEN_FOR_GH_STORAGE }}

  handle-static-files-pr:
    name: Handle Static File Changes
    runs-on: ubuntu-latest
    needs: notebook-pr-check
    if: needs.notebook-pr-check.outputs.bypass_notebook_processing == 'true' && needs.notebook-pr-check.outputs.is_docs_or_static_only_change == 'true'
    permissions:
      contents: write
    # ... (steps to checkout PR code, identify changed static files, and push to gh-storage)
    # Refer to examples/workflows/notebook-ci-pr.yml for the full job definition.
    steps:
      - name: Checkout PR code
        uses: actions/checkout@v4
        with: { path: pr_code }
      - name: Identify Changed Static Files & Push to gh-storage
        id: static_changes
        working-directory: ./pr_code
        env: { GH_TOKEN: "${{ secrets.YOUR_GH_TOKEN_FOR_GH_STORAGE }}" }
        run: |
          # Simplified script content - see full example for details
          echo "Identifying and pushing static files to gh-storage..."
          # Actual script would diff, identify static files, clone gh-storage, copy, commit, push.
          # This is detailed in examples/workflows/notebook-ci-pr.yml
          # For brevity here, we acknowledge its existence.
          # If no static files changed or need update, this script should handle it gracefully.
          # Example:
          # CHANGED_FILES=$(git diff --name-only origin/${{ github.event.pull_request.base.ref }} HEAD)
          # ... logic to filter static files and push them ...
          echo "Static file handling complete (conceptual)."

```

### On-Demand Operations
(`.github/workflows/on_demand_ops.yml` - based on `examples/workflows/notebook-ci-on-demand.yml`)
- Provides `workflow_dispatch` inputs for various operations:
  - `test_notebooks`: Pure testing, no storage.
  - `store_notebooks`: Test and store to `gh-storage`.
  - `store_and_rebuild_html`: Test, store, then rebuild and deploy HTML from `gh-storage`.
  - `rebuild_html_only`: Rebuild and deploy HTML from `gh-storage` without prior testing.
- Calls `ci_pipeline.yml` and/or includes HTML building logic.

```yaml
# Example: .github/workflows/on_demand_ops.yml
name: Notebook CI & Publish - On Demand

on:
  workflow_dispatch:
    inputs:
      operation:
        description: 'Select operation'
        required: true
        default: 'test_notebooks'
        type: choice
        options: ['test_notebooks', 'store_notebooks', 'store_and_rebuild_html', 'rebuild_html_only']
      # ... other inputs like single_filename, python_version, etc.
      # Refer to examples/workflows/notebook-ci-on-demand.yml for full input list.

jobs:
  process_notebooks_on_demand:
    if: github.event.inputs.operation == 'test_notebooks' || github.event.inputs.operation == 'store_notebooks' || github.event.inputs.operation == 'store_and_rebuild_html'
    name: Process Notebooks (${{ github.event.inputs.operation }})
    uses: spacetelescope/notebook-ci-actions/.github/workflows/ci_pipeline.yml@main # Adjust version
    with:
      operation-mode: ${{ (github.event.inputs.operation == 'test_notebooks' && 'on-demand-test') || 'on-demand-store' }}
      force-store: ${{ github.event.inputs.operation == 'store_notebooks' || github.event.inputs.operation == 'store_and_rebuild_html' }}
      # ... map other inputs: single_filename, python_version, custom_requirements_path, etc.
      is-deprecated-check-script: ${{ github.event.inputs.is_deprecated_check_script_path }}
    secrets:
      CASJOBS_USERID: ${{ secrets.CASJOBS_USERID }}
      CASJOBS_PW: ${{ secrets.CASJOBS_PW }}
      GH_TOKEN_FOR_STORAGE: ${{ secrets.YOUR_GH_TOKEN_FOR_GH_STORAGE }}

  build_html_on_demand:
    if: github.event.inputs.operation == 'store_and_rebuild_html' || github.event.inputs.operation == 'rebuild_html_only'
    needs: [process_notebooks_on_demand]
    name: Build & Deploy HTML On Demand
    runs-on: ubuntu-latest
    permissions: { contents: write }
    # ... (steps to checkout main_repo, gh_storage_content, rsync, build, post-process, deploy)
    # Refer to examples/workflows/notebook-ci-on-demand.yml for the full job definition.
    steps:
      - name: Build and Deploy HTML from gh-storage
        env:
          POST_PROCESSING_SCRIPT: ${{ github.event.inputs.post_processing_script_path_for_html }}
        run: |
          # Simplified script content - see full example for details
          echo "Building and deploying HTML from gh-storage..."
          # Actual script would checkout files, run jupyter-book, post-process, deploy.
          # This is detailed in examples/workflows/notebook-ci-on-demand.yml's second job.
          echo "HTML build and deploy complete (conceptual)."
```

### Scheduled Health Checks
- Copy `notebook-ci-actions/.github/workflows/scheduled_notebook_tests.yml` to your repository.
- Adjust schedule and paths as needed.

### Publishing Documentation (Post-Merge)
- Copy `notebook-ci-actions/.github/workflows/jupyterbook_build.yml` to your repository.
- Customize `POST_PROCESSING_SCRIPT` environment variable within this file if needed.

## 🚀 Quick Start: Local Testing
*(This section needs review to align `test-local-ci.sh` or other local test scripts with the new `operation-mode` and inputs of `ci_pipeline.yml` if users want to simulate specific modes locally.)*
...

## 📋 Prerequisites
- **`gh-storage` branch**: Will be auto-created if it doesn't exist by `ci_pipeline.yml` during operations that store data.
- **JupyterBook Configuration**: Your main branch should contain `_config.yml`, `_toc.yml`, and any other files needed for JupyterBook structure (e.g., custom CSS, logo).
- **Secrets**:
    - `YOUR_GH_TOKEN_FOR_GH_STORAGE` (or a name of your choice): A GitHub Personal Access Token (PAT) or a token from a GitHub App with `contents: write` permission for your repository. This is essential for `ci_pipeline.yml` to push to `gh-storage` and for the static file handler job.
    - `CASJOBS_USERID`, `CASJOBS_PW` (optional, if notebooks require them).
- **Actions Permissions**:
    - Workflows calling `ci_pipeline.yml` (if using `secrets.GITHUB_TOKEN` as `GH_TOKEN_FOR_STORAGE`): Need `permissions: contents: write`.
    - `jupyterbook_build.yml` (and the on-demand HTML build job): Needs `permissions: contents: write` to deploy to the `gh-pages` branch.
- **`is-deprecated-check-script` (Optional)**: If you use this feature, provide a script in your repository. Example script (`scripts/check_deprecated.sh`):
  ```bash
  #!/bin/bash
  # Usage: scripts/check_deprecated.sh <notebook_path>
  # Outputs "true" if deprecated, "false" otherwise.
  notebook_path="$1"
  # Example: check for a tag in the notebook's JSON, or a marker file, or a list.
  if grep -q '"deprecated": true' "$notebook_path"; then
    echo "true"
  else
    echo "false"
  fi
  ```
  Make sure this script is executable.

## 🔧 Package Manager Strategy
*(This section can remain largely the same.)*
...

## 🏷️ Versioning & Releases
*(This section can remain the same.)*
...

## 🤝 Contributing
*(Review and update based on new testing procedures or workflow complexities.)*
...

---
*This README provides a comprehensive guide to the refactored notebook CI/CD system. Ensure all scripts and example workflows are tested with these new configurations.*
