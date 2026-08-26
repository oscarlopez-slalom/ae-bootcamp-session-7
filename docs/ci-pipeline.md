# CI/CD Pipeline — Golden Path

This document explains the reusable CI/CD workflow that backs the todo-service golden path, and how a new service team adopts it.

## Overview

Two files make up the pipeline:

```text
.github/workflows/
├── golden-path-ci.yml        ← Reusable workflow (on: workflow_call)
└── todo-service-ci.yml       ← Caller workflow for todo-service
```

`golden-path-ci.yml` is owned by the platform team. It encodes every required check for a service on this golden path — lint, test, security scan, Terraform plan/apply, and image build/push. Individual services never duplicate this logic; they call it with a handful of inputs, the same way `infra/stacks/dev/main.tf` calls the Terraform module instead of hand-rolling infrastructure.

## Jobs in `golden-path-ci.yml`

| Job | Runs when | What it validates / does |
|---|---|---|
| `lint` | always | ESLint on `packages/backend` and `packages/frontend`. Catches style and correctness issues before tests even run. |
| `test` | always | Jest with coverage on the backend (coverage threshold — 80% lines/branches — is enforced by `packages/backend/package.json`, so the job fails the build if coverage regresses). Publishes a coverage summary to the job's `$GITHUB_STEP_SUMMARY`. |
| `security-scan` | `run_terraform_plan == true` | Runs `checkov` against `infra/` and hard-fails on HIGH severity findings. Keeps insecure infrastructure changes out of `main`. |
| `terraform-plan` | `run_terraform_plan == true` | Authenticates to AWS via OIDC, runs `terraform init -backend=false`-equivalent init against the real S3 backend, and produces a plan. On pull requests (no real networking IDs yet) it substitutes mock `vpc_id`/subnet values so the plan can still run without a deployed VPC. Uploads the plan as an artifact for the apply job to consume later. |
| `docker-build` | pull requests only, after `lint`+`test` | Builds both `packages/backend/Dockerfile` and `packages/frontend/Dockerfile` to catch broken Dockerfiles early — no push, no AWS credentials needed. |
| `terraform-apply` | `run_terraform_apply == true` (pushes to `main` only) | Removes the mock-credential lines from the provider block, re-inits against the real S3 backend, downloads the plan artifact, and applies it. Publishes the resulting load balancer URL to the job summary. |
| `build-and-push` | `build_and_push == true` (pushes to `main` only) | Resolves the ECR repository URLs from Terraform outputs, builds and pushes both images (tagged with the commit SHA and `latest`), then forces a new ECS deployment. |

## Why this shape?

- **Cheap by default, thorough when it matters.** PRs get fast feedback (lint, test, a Docker build sanity check, and a Terraform plan) without needing write access to AWS. Only pushes to `main` trigger `terraform-apply` and `build-and-push`, which actually change infrastructure or ship images.
- **One source of truth for "what does a compliant pipeline look like."** If the platform team changes a required check (e.g. adds SAST), every service adopting `golden-path-ci.yml` gets it automatically the next time they run CI — no copy-pasting YAML across repos.
- **Least-privilege permissions.** The top-level `permissions:` block grants only `contents: read` and `pull-requests: write`. Jobs that need OIDC federation (`terraform-plan`, `terraform-apply`, `build-and-push`) declare `id-token: write` at the job level, not the workflow level, so only those specific jobs can mint a short-lived AWS credential.

## Adopting the golden path (minimum caller workflow)

A new service only needs a caller workflow like `todo-service-ci.yml`:

```yaml
name: Todo Service CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write   # required at the CALLER level for OIDC to work

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

> **Important:** `id-token: write` must be declared in the *caller's* top-level `permissions:` block. GitHub only grants the OIDC token to workflows that explicitly request it — declaring it only inside `golden-path-ci.yml` is not enough.

## Configuring the `terraform-plan`/`terraform-apply` secret

The `aws_role_arn` secret input is how the reusable workflow receives the AWS IAM role to assume via OIDC — it is never hardcoded in the workflow file.

1. In AWS, create (or reuse) an IAM role with a trust policy that allows GitHub's OIDC provider (`token.actions.githubusercontent.com`) to assume it, scoped to this repository.
2. In the repository's **Settings → Secrets and variables → Actions**, add a repository secret named `AWS_ROLE_ARN` with that role's ARN.
3. The caller workflow forwards it as `secrets.aws_role_arn` to the reusable workflow. Inside `golden-path-ci.yml`, jobs reference `${{ secrets.aws_role_arn }}` (the `workflow_call` secret input) — never `secrets.AWS_ROLE_ARN` directly, since that name only exists in the caller's scope.

## Required checks this pipeline satisfies

- `lint`, `test`, `security-scan`, `terraform-plan` — the four required status checks for the golden path (per `.github/copilot-instructions.md`)
- Coverage gate — enforced by Jest's `coverageThreshold`, not by custom CI scripting
- OIDC-only AWS access — no long-lived AWS credentials stored as secrets
