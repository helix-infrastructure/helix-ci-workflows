# helix-ci-workflows

Shared, reusable GitHub Actions workflows (`workflow_call`) for every Helix repo — nine independently-callable stages, not one monolith with skip-flags. Each consuming repo composes only what it needs in its own thin `.github/workflows/ci.yml`.

## Versioning

Versioned exactly like a Terraform module (see `helix-ai-context/teams/devops/conventions/modules.md`): tag `v0.1.0` first, same semver rules — a backwards-compatible new input is a minor bump, a required-input change or removed input is a major bump, a behavior fix with no interface change is a patch. **Always pin `@vX.Y.Z` in your `uses:` line. Never pin `@main`.** Bump pins deliberately, the same way you'd bump a module `ref`.

## Consumption pattern per repo type

| Repo type | Workflows called | Template |
|---|---|---|
| Module repos (`terraform-aws-*`) | `lint` + `security` | [`examples/module-repo-ci.yml`](examples/module-repo-ci.yml) |
| App repos (`helix-status`, `helix-api-*`) | `lint` (if `infra/` exists) + `build` + `test` + `security`, plus a bump-PR step into `tf-control` on push to `main` | [`examples/app-repo-ci.yml`](examples/app-repo-ci.yml) |
| `tf-control` | all 9 — `lint`/`security` pre-merge; `plan`→`deploy`→`verify`→`notify` post-merge per environment; `rollback` via `workflow_dispatch` | [`examples/tf-control-ci.yml`](examples/tf-control-ci.yml) |

`tf-control` is the only repo with Terraform backend state, so it's the only repo that actually plans, deploys, verifies, or rolls back — both infra changes and app deploys (image-tag bumps) are controlled from there.

## Secrets and GitHub Environments

| Secret | Where | Used by |
|---|---|---|
| `AWS_ROLE_ARN` | app repo, repo-level secret | `build.yml` (ECR push only) |
| `AWS_STATE_ROLE_ARN` | `tf-control`, per-Environment secret | `plan.yml`, `deploy.yml`, `verify.yml` (fallback path) |
| `AWS_DEPLOY_ROLE_ARN` | `tf-control`, per-Environment secret | `plan.yml`, `deploy.yml` |
| `GOOGLE_CHAT_WEBHOOK_URL` / `TEAMS_WEBHOOK_URL` | `tf-control`, repo or Environment secret | `notify.yml` (either or both — at least one) |
| `HELIX_BOT_PAT` | org secret, fine-grained PAT, `pull-requests:write` on `tf-control` only | the bump-PR step in app repos' `ci.yml`, and `rollback.yml` |

GitHub Environment names must match the `environment:` input passed to `plan.yml`/`deploy.yml` exactly (`sandbox`, `dev`, `test`, `prod`, `management`) — the IAM role trust policy's `sub` claim is scoped to `repo:helix-infrastructure/tf-control:environment:<name>`, and the Environment's required-reviewer protection rule is what actually gates `deploy.yml` (no custom approval logic in the workflow itself).

## Pin upgrade procedure

1. Tag a new version in this repo following the semver rules above.
2. In each consuming repo, bump the `@ref` in `ci.yml` deliberately — open a normal `chore/` PR, same as bumping any other module pin.
3. Never let a consumer float to `@main`.
