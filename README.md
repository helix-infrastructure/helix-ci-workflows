# helix-ci-workflows

Shared, reusable GitHub Actions workflows (`workflow_call`) for every Helix repo — nine independently-callable stages, not one monolith with skip-flags. Each consuming repo composes only what it needs in its own thin `.github/workflows/ci.yml`.

## Versioning

Versioned exactly like a Terraform module (see `helix-ai-context/teams/devops/conventions/modules.md`): tag `v0.1.0` first, same semver rules — a backwards-compatible new input is a minor bump, a required-input change or removed input is a major bump, a behavior fix with no interface change is a patch. **Always pin `@vX.Y.Z` in your `uses:` line. Never pin `@main`.** Bump pins deliberately, the same way you'd bump a module `ref`.

## Workflow files

| File | Purpose |
|---|---|
| `lint-tf.yml` | `terraform fmt`/`init -backend=false`/`validate`/`tflint` — Terraform-only, no AWS credentials |
| `build-app.yml` | Docker build+push to ECR, or Lambda zip packaging |
| `test-app.yml` | Runs the app's own test suite (`setup_command` + `test_command`) |
| `security.yml` | gitleaks always; tfsec/checkov if `scan_type` includes `terraform`; `npm audit` if it includes `app`; trivy if it includes `container` |
| `plan-tf.yml` | `terraform plan`, uploads the plan as an artifact — `tf-control` only |
| `apply-tf.yml` | Downloads that artifact, `terraform apply`s it, moves the `deployed-{environment}` tag — `tf-control` only |
| `test-endpoint.yml` | Post-deploy health check, direct URL or via `terraform output` | 
| `notify.yml` | Google Chat / Teams webhook post |
| `rollback.yml` | Opens a revert PR against `tf-control` — never applies directly |

`lint-tf.yml` and `security.yml` are shared across all three repo types below; the rest are type-specific.

## Consumption pattern per repo type

| Repo type | Workflows called | Template |
|---|---|---|
| Module repos (`terraform-aws-*`) | `lint-tf` + `security-tf` | [`examples/module-repo-ci.yml`](examples/module-repo-ci.yml) |
| App repos (`helix-status`, `helix-api-*`) | `lint-tf` (if `infra/` exists) + `build-app` + `test-app` + `security-app`, plus a bump-PR step into `tf-control` on push to `main` | [`examples/app-repo-ci.yml`](examples/app-repo-ci.yml) |
| `tf-control` | all 9 — `lint-tf`/`security-tf` pre-merge; `plan-tf`→`apply-tf`→`test-endpoint`→`notify` post-merge per environment; `rollback` via `workflow_dispatch` | [`examples/tf-control-ci.yml`](examples/tf-control-ci.yml) |

`tf-control` is the only repo with Terraform backend state, so it's the only repo that actually plans, applies, tests the deployed endpoint, or rolls back — both infra changes and app deploys (image-tag bumps) are controlled from there.

`plan-tf` and `apply-tf` stay separate jobs with an artifact handoff between them, deliberately — not collapsed into one job. That split is what makes the saved plan reviewable, even without a human approval gate in front of `apply-tf` today (see below) — collapsing them would lose that regardless.

**No job in `tf-control`'s `ci.yml` sets a job-level `environment:` key, and that's deliberate.** Two independent reasons converge on the same answer: (1) GitHub Environment protection rules (the required-reviewer gate) aren't available on private repos on Free-tier orgs, and (2) confirmed by direct testing, setting `environment:` on a job that also has `uses:` breaks GitHub's ability to recognize it as a reusable-workflow-call job at all — it demands `runs-on`/`steps:` instead and the whole file fails at startup with zero jobs created, regardless of whether `with:` holds a literal or computed value. There's no known workaround short of making `tf-control` public or upgrading the org's plan — both rejected for now since `tf-control` holds real Terraform (IAM policies, security groups, account structure). **Practical upshot: there is currently no human-approval gate before `apply-tf` runs.** `test` applies automatically on merge/dispatch, same as it effectively did before. Revisit before ever standing up `prod/`, which Helix's own guardrails already say not to do yet regardless. `environment:` is still passed as a plain string in `with:` — `plan-tf.yml`/`apply-tf.yml` use it only to pick the right `helix-{env}-bootstrap` AWS profile, unrelated to GitHub's Environment feature.

**`terraform.tfvars` is gitignored repo-wide in `tf-control`** (it can carry account-scoped values — see `tf-control`'s `.gitignore`), so a bump step can never target it: git will never see the diff, and neither the bump-PR nor the `push`-triggered deploy can fire. Deploy-time parameters that CI needs to change (`image_tag` today) live in their own git-tracked `image_tag.auto.tfvars.json` per scaffold instead — Terraform loads `*.auto.tfvars.json` automatically alongside `terraform.tfvars`, no `.tf` code changes needed. The bump-PR step in `examples/app-repo-ci.yml` writes to this file, not `terraform.tfvars`.

## permissions: blocks are not optional

`plan-tf.yml`, `apply-tf.yml`, and `test-endpoint.yml` all request `id-token: write` in their own job (`apply-tf.yml` also `contents: write`, for the `deployed-{environment}` tag push) — needed for AWS OIDC role assumption. A reusable workflow can only receive permissions its caller already has; if the calling job in your `ci.yml` doesn't declare a matching `permissions:` block, this is **not** a runtime permission error. It's a hard schema validation failure that fails the *entire calling file* at startup with zero jobs created, regardless of which job would have hit the missing permission — this cost real debugging time to track down. Copy the `permissions:` blocks shown in `examples/tf-control-ci.yml` and `examples/app-repo-ci.yml` exactly; don't drop them to save a few lines.

## Secrets

All secrets below are **repo-level**, not Environment-scoped — see the `environment:` note above for why.

| Secret | Where | Used by |
|---|---|---|
| `AWS_ROLE_ARN` | app repo, repo-level secret | `build-app.yml` (ECR push only) |
| `AWS_STATE_ROLE_ARN` | `tf-control`, repo-level secret | `plan-tf.yml`, `apply-tf.yml`, `test-endpoint.yml` (fallback path) |
| `AWS_DEPLOY_ROLE_ARN` | `tf-control`, repo-level secret | `plan-tf.yml`, `apply-tf.yml` |
| `GOOGLE_CHAT_WEBHOOK_URL` / `TEAMS_WEBHOOK_URL` | `tf-control`, repo-level secret | `notify.yml` (either or both — at least one) |
| `HELIX_BOT_PAT` | org secret, fine-grained PAT, `pull-requests:write` on `tf-control` only | the bump-PR step in app repos' `ci.yml`, and `rollback.yml` |
| `TF_MODULE_READ_PAT` | repo-level secret on any repo whose Terraform sources private `terraform-aws-*` modules (currently `tf-control`, `helix-status`), fine-grained PAT, `contents:read` on the module repos | `lint-tf.yml`, `plan-tf.yml`, `apply-tf.yml` — `GITHUB_TOKEN` is scoped only to the repo the workflow runs in, so it can't clone sibling private module repos. Set at org level with `admin:org` scope if you want one secret shared across repos instead of copying it per repo — the token I used to set this up only had `repo` scope, so it went in per-repo. |

The AWS IAM role trust policies for `AWS_STATE_ROLE_ARN`/`AWS_DEPLOY_ROLE_ARN` are scoped to OIDC `sub` claim `repo:helix-infrastructure/tf-control:ref:refs/heads/main` — matching what GitHub actually presents for a job with no `environment:` key, restricted to `main` since `plan-tf`/`apply-tf` only ever run there (push or a `workflow_dispatch` also targeting `main`).

## Pin upgrade procedure

1. Tag a new version in this repo following the semver rules above.
2. In each consuming repo, bump the `@ref` in `ci.yml` deliberately — open a normal `chore/` PR, same as bumping any other module pin.
3. Never let a consumer float to `@main`.
