# Delivery pipeline

How changes move from a pull request to production, in GitHub Actions workflows in `.github/workflows/`. This is where the SDLC gates bind to the tooling (Standard D-07 and D-15).

The work is done by the platform's reusable workflows, released from `ai-app-platform` ([Platform workflows](#platform-workflows)). The application writes only the four workflows that call them.

## Environments and gates

| GitHub Environment | Deploys | Gate |
|---|---|---|
| `dev` | Automatically on every merge to `main` | Required pull request review and passing checks before merge |
| `test` | The image already running in `dev`, on request | Required reviewers; deployment from `main` only |
| `prod` | The image already running in `test`, on request | Required reviewers; deployment from `main` only |

### Private repositories and the GitHub plan

The gates above need features that, on a private repository, depend on the GitHub plan (Standard assumption A-02, checked 2026-11-13). A public repository has them all on every plan.

| Feature on a private repository | GitHub Free | GitHub Pro | GitHub Team | GitHub Enterprise |
|---|---|---|---|---|
| `main` protected, by a ruleset or branch protection | No | Yes | Yes | Yes |
| GitHub Environments, with deployment restricted to `main` | No | Yes | Yes | Yes |
| Required reviewers on an environment | No | No | No | Yes |
| Secret scanning and push protection | No | No | If GitHub Secret Protection is bought | If GitHub Secret Protection is bought |

Where a feature is missing, the application uses the fallback, and records its plan and the fallbacks it uses in an ADR:

| Missing | Fallback |
|---|---|
| Protection of `main` | The ruleset is still kept in `.github/rulesets/`, so it can be applied unchanged when the plan allows. Until then, a pull request whose checks fail is not merged, by agreement. |
| GitHub Environments | The deploy roles trust the `main` branch instead of an environment ([Authentication to AWS](#authentication-to-aws)). Deploying to `test` or `prod` is a run started by hand from `main`, by someone agreed to approve releases. The platform workflows' `github-environment` input is left empty. |
| Required reviewers | Deployment to `test` and `prod` is restricted to `main`, and started by hand only by someone agreed to approve releases; the run records who started it. |
| Secret scanning and push protection | The pull request checks' secret scan and the developers' pre-commit secret scan, which are run on every plan anyway. |

Each merge to `main` produces one build: one API image, identified by its digest, and one web build. The same build is deployed to `dev`, then `test`, then `prod`, with only the configuration differing ([10-architecture.md](10-architecture.md#environments)). Nothing is rebuilt for `test` or `prod`.

## Workflows

| Workflow | Trigger | What it does | Platform workflows it calls |
|---|---|---|---|
| `ci.yml` | Every pull request, and pushes to `main` | The checks below. Required to pass before merge. | `python-checks`, `web-checks`, `contract-check`, `terraform-checks`, `image-checks`, `security-checks`, `workflow-checks` |
| `infra-plan.yml` | Pull requests that change `infra/` | `terraform plan` for each environment with the read-only role, posted to the pull request | `terraform-plan`, once per environment |
| `infra-apply.yml` | Merge to `main` changing `infra/`, or on request | `terraform apply` for `dev`, then `test` and `prod` through their approved environments | `terraform-apply`, once per environment |
| `deploy.yml` | Merge to `main`, or on request with an environment | On merge, builds once and deploys to `dev`. On request, deploys an existing build to the chosen environment | `build-application`, then `deploy-application` per environment |
| `bootstrap.yml` | Not used | The bootstrap runs from an operator's machine with their own credentials ([12-aws-deployment.md](12-aws-deployment.md#account-bootstrap)) | |

## Platform workflows

Each calling job names the platform workflow at the commit its release tag names, with the tag in a comment, as for every action, and every call uses the same release:

```yaml
jobs:
  api:
    name: API
    permissions:
      contents: read
    uses: {platform-owner}/ai-app-platform/.github/workflows/python-checks.yml@{commit SHA} # workflows-v{version}
    with:
      working-directory: apps/api
```

The commit is taken only from the release's notes or from the tag in `ai-app-platform`, never from a link, a pull request or a fork, since a public repository's address also serves commits pushed to its forks. The calling workflow holds the trigger, the GitHub Environment, the inputs and the permissions: a reusable workflow's jobs have no more permission than the calling job grants, so each calling job grants what the workflow's README lists, and no more. The README in `ai-app-platform`'s `.github/workflows` lists each workflow's inputs, and the Terraform outputs the build and deployment read ([12-aws-deployment.md](12-aws-deployment.md#outputs)). The workflows are upgraded on purpose: change the commit and comment in every call, read the changelog, and run the pipelines.

## Pull request checks

| Area | Checks |
|---|---|
| API | Ruff lint and format check, Pyright in strict mode, unit tests, integration tests against a PostgreSQL service container, coverage thresholds |
| Web interface | ESLint, Prettier check, TypeScript type check, Vitest, build, Playwright and accessibility checks against the built app |
| Contract | The OpenAPI document generated from the API matches the committed one used to generate the web client |
| Infrastructure | `terraform fmt -check`, `terraform validate`, `terraform test`, infrastructure security scanning |
| Images | Build the API image; scan it for vulnerabilities |
| Security | Code scanning, dependency review and secret scanning (GitHub's own features where the plan provides them, otherwise open-source equivalents such as gitleaks) |
| Workflows | actionlint, with shellcheck, over `.github/workflows` |

## Application deployment

On merge to `main`:

1. Build the API image once, tagged with the commit SHA, and push it to ECR in `dev`. Record its digest.
2. Build the web interface once and keep it as a workflow artefact. The bundle holds no environment-specific values; the environment name is supplied at deployment in a generated `config.json`.
3. Deploy to `dev` as below.

To deploy to an environment:

1. If the environment is in a different AWS account from `dev`, copy the image unchanged into that account's ECR repository. The digest stays the same, which shows it is the same build.
2. Run the migration task once, with the migration user's credentials, and stop if it fails.
3. Register a new task definition revision with the image digest and the environment's API settings (from GitHub variables), and update the ECS service. Wait for it to become stable; the circuit breaker rolls back a failed deployment.
4. Upload the web build to the environment's bucket (hashed assets first, then `index.html` and `config.json`) and invalidate the CloudFront cache for HTML and `config.json`.
5. Call `/api/health` through CloudFront and fail the deployment if it does not return 200.

## Authentication to AWS

- Workflows authenticate through OIDC. No AWS access keys are stored in GitHub.
- Deploy roles trust only the repository's immutable ID and the named GitHub Environment, so a renamed or transferred repository cannot assume them; the calling workflow names the environment in the platform workflow's `github-environment` input. Without GitHub Environments, they trust the repository's immutable ID and its `main` branch, and the input is left empty.
- Plans on pull requests use a read-only role.

## Variables and secrets

- Role ARNs, the region, network IDs, the state bucket and other identifiers are **GitHub variables**, not secrets. They are repository variables, prefixed with the environment's name where they differ by environment, such as `DEV_DEPLOY_ROLE_ARN`, `TEST_DEPLOY_ROLE_ARN` and `PROD_DEPLOY_ROLE_ARN`: a calling workflow passes them to the platform workflows as inputs, and a job that calls a reusable workflow cannot read an environment's variables.
- Any sensitive value a workflow needs is read from Secrets Manager at run time through OIDC. Where a GitHub secret is unavoidable, it is scoped to an environment behind required reviewers; without them, no GitHub secret is used.
- Push protection and secret scanning are on for the repository where the GitHub plan provides them, the pull request checks run a secret scan, and developers run the pre-commit secret scan (`.pre-commit-config.yaml`).

## Workflow hardening

- The ruleset protecting `main` is kept in `.github/rulesets/main.json`, so a change to it is reviewed like any other, and an administrator applies it once merged.
- Every third-party action and reusable workflow is pinned to a full commit SHA, with the version in a comment. Dependabot proposes updates to actions; the platform workflows are upgraded on purpose ([Platform workflows](#platform-workflows)).
- The repository requires actions to be pinned to a full commit SHA (Settings → Actions → General), and its default workflow token is read-only.
- A public repository requires approval before workflows run on pull requests from all external contributors.
- `permissions: {}` at the top of each workflow; each job is granted only the permissions it needs (for example `id-token: write` and `contents: read` for a deploy job).
- Jobs run on GitHub-hosted runners.
- `pull_request_target` is not used.
