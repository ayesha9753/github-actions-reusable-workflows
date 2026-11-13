# GitHub Actions — Reusable Workflows Library

A small, production-style library of **reusable GitHub Actions workflows** and a
**composite action** that any repository can call with a few lines of YAML.
Instead of copy-pasting the same pipeline into every project, each pipeline is
written once, versioned here, and consumed via `workflow_call`.

> Portfolio project by Ayesha Imran (DevOps & Cloud Engineer). Built to
> demonstrate senior-level CI/CD patterns — DRY pipelines, keyless AWS auth,
> image scanning, and deploys that fail loudly when the app is unhealthy.
> No client work is claimed; the examples use placeholder hosts and ARNs.

## What's included

| Name | File | Purpose | Key inputs | Secrets |
|---|---|---|---|---|
| Docker Build, Scan & Push | `.github/workflows/reusable-docker-build.yml` | Build a Docker image with Buildx + layer caching, scan it with Trivy (fails on CRITICAL/HIGH), optionally push to GHCR | `image-name` (required), `context`, `dockerfile`, `push`, `registry`, `tag` | None — GHCR uses the automatic `GITHUB_TOKEN`; callers pass `secrets: inherit` |
| Terraform Plan / Apply | `.github/workflows/reusable-terraform.yml` | `fmt -check` → `init` → `validate` → `plan` (artefact uploaded), `apply` only on request **and** on `main` | `working-directory`, `terraform-version`, `aws-region` (required), `role-to-assume` (required), `command` (`plan`/`apply`) | None — AWS auth is via OIDC role assumption |
| Deploy over SSH | `.github/workflows/reusable-deploy-ssh.yml` | rsync files to a server, run a restart command, then poll an HTTP health URL until 200 — job fails if it never succeeds | `host`, `user`, `deploy-path`, `restart-command`, `health-url` (all required), `health-retries` | `ssh-key` (declared explicitly, see below) |
| setup-python-app (composite action) | `.github/actions/composite-setup/action.yml` | Set up Python with pip caching; install `requirements.txt` only when the file exists | `python-version`, `requirements-file` | None |

## Usage

### 1. Docker build & push

```yaml
permissions:
  contents: read
  packages: write # needed only when pushing

jobs:
  docker:
    uses: ayesha9753/github-actions-reusable-workflows/.github/workflows/reusable-docker-build.yml@main
    with:
      image-name: my-app
      push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets: inherit
```

Full example: [`examples/caller-docker.yml`](examples/caller-docker.yml)

### 2. Terraform with OIDC (no AWS keys)

```yaml
permissions:
  contents: read
  id-token: write # required for OIDC

jobs:
  plan:
    uses: ayesha9753/github-actions-reusable-workflows/.github/workflows/reusable-terraform.yml@main
    with:
      working-directory: infra
      aws-region: eu-west-1
      role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
      command: plan
```

Full example (plan on PRs, apply from main): [`examples/caller-terraform.yml`](examples/caller-terraform.yml)

### 3. Deploy over SSH with health check

```yaml
permissions:
  contents: read

jobs:
  deploy:
    uses: ayesha9753/github-actions-reusable-workflows/.github/workflows/reusable-deploy-ssh.yml@main
    with:
      host: example.com
      user: deploy
      deploy-path: /srv/my-app
      restart-command: "docker compose up -d"
      health-url: https://example.com/health
    secrets:
      ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

Full example: [`examples/caller-deploy.yml`](examples/caller-deploy.yml)

### 4. Composite action (inside a normal job)

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ayesha9753/github-actions-reusable-workflows/.github/actions/composite-setup@main
    with:
      python-version: "3.12"
  - run: pytest
```

## Security notes

- **OIDC instead of long-lived keys.** The Terraform workflow authenticates to
  AWS with `aws-actions/configure-aws-credentials` and a role ARN. GitHub mints
  a short-lived token per run; there are no AWS access keys stored as secrets
  to leak or rotate.
- **Least-privilege permissions.** Every caller example declares an explicit
  top-level `permissions:` block (`contents: read`, plus only what that job
  needs — `packages: write` for pushes, `id-token: write` for OIDC) instead of
  relying on the default permissive token.
- **Scan gate before push.** Images are built locally, scanned with Trivy, and
  the job exits non-zero on CRITICAL/HIGH findings *before* anything is pushed.
- **Secrets are declared, not inherited blindly.** `workflow_call` secrets must
  be declared in the reusable workflow. The SSH workflow declares exactly one
  (`ssh-key`); the key is written with `600` permissions and deleted in an
  `if: always()` cleanup step.
- **Apply is double-guarded.** Terraform `apply` runs only when the caller
  passes `command: apply` **and** the ref is `refs/heads/main`.
- **Pin actions.** Third-party actions are pinned to release tags (and the
  Trivy action to a full version), reducing supply-chain surprise. For stricter
  environments, pin to full commit SHAs.

## Skills demonstrated

- Reusable workflows (`workflow_call`) with typed inputs, outputs and declared secrets
- Composite actions
- GitHub OIDC → AWS IAM role assumption (keyless cloud auth)
- Docker Buildx, GitHub Actions cache (`type=gha`), GHCR publishing
- Container vulnerability scanning with Trivy as a pipeline gate
- Terraform CI: fmt/validate/plan artefacts, guarded apply
- SSH/rsync deployment with post-deploy HTTP health verification
- Least-privilege `GITHUB_TOKEN` permission design
