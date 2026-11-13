# CLAUDE.md

Guidance for Claude Code when working in this repository.

This repository was created from `ai-app-template`, which implements the AI application standard held in the `ai-app-standard` repository. Follow the specifications here; the standard explains the reasons behind them.

## How to work

Claude is the implementation agent, not the designer. All behaviour comes from the specifications in [specs/](specs/), listed in [specs/README.md](specs/README.md).

- [specs/99-generation-rules.md](specs/99-generation-rules.md) is authoritative over all code generation. Read it before any non-trivial work.
- Read the specifications relevant to a task before writing code.
- Build only what the release in progress covers, as set out in the delivery plan in [docs/delivery/](docs/delivery/README.md).
- If a specification is missing, ambiguous or contradicts the request, ask. Do not infer.
- Every change to behaviour updates the specification that describes it, in the same change.
- Specifications marked **APPLICATION STUB** must be filled in before any domain code is generated.
- Do not write or copy code the platform packages provide ([specs/10-architecture.md](specs/10-architecture.md#platform-packages)). Install them; a fix belongs in `ai-app-platform`.
- `{platform-owner}` in the specifications stands for the owner of the `ai-app-platform` repository the application uses. Use the owner already in the application's files; if there is none, ask, and never guess or leave the placeholder in code.

## Stop rules

Stop and report to the user, without continuing the task, when any of these happen:

- **A credential is found** in a file, a log, command output or a commit. Report the file path and the type of credential (for example "database password" or "AWS access key"). Never repeat, copy, move or use the value.
- **A secret is about to be written to a file.** Secrets are never held in `.env` files, in source code, in specifications, in Terraform code or variables, in test fixtures or in pipeline variables. They live in AWS Secrets Manager. `.env.example` and `infra/local/.env` hold only non-secret settings and throwaway values for local containers, such as the local database password.
- **Real data is about to enter the repository.** Only synthetic data is held here, including in tests, fixtures, seed scripts and examples. See the data classification section of [specs/30-data-model.md](specs/30-data-model.md).
- **Code is about to be written outside the WSL Linux filesystem.** Before creating or changing code in `apps/` or `infra/`, check that the repository is in the WSL Linux filesystem rather than on a Windows drive (a path such as `C:\` or `/mnt/c/`). If it is not, stop and say so ([specs/11-local-development.md](specs/11-local-development.md), Standard ADR-0011).
- **A change would deploy or alter cloud resources.** Claude does not run `terraform apply`, `terraform destroy`, AWS CLI commands or `git push`. Changes reach AWS only through the pipelines.

## Repository structure

```
apps/api/              Python FastAPI API: all business logic, validation and access control, on the ai_app_platform package
apps/web/              Vite React single-page app: no business rules, on the web platform package
infra/bootstrap/       One-off AWS account bootstrap (OIDC provider, deploy roles, state bucket, optional network), on the platform's Terraform modules
infra/aws/             Terraform for the deployed application, one file per concern, on the platform's Terraform modules
infra/local/           Docker Compose for local PostgreSQL, the platform's mock identity provider and the API
specs/                 Specifications: the source of truth for what is built
adrs/                  This application's own decision records, including deviations from the standard
docs/                  The delivery plan and runbooks for this application
.github/workflows/     Test, infrastructure and deployment pipelines, calling the platform's reusable workflows
```

## Local commands

Run from the repository root. Full detail is in [specs/11-local-development.md](specs/11-local-development.md).

```bash
# Start PostgreSQL, the mock identity provider and the API
docker compose -f infra/local/docker-compose.yml --env-file infra/local/.env up -d postgres mock-idp api

# Apply database migrations
docker compose -f infra/local/docker-compose.yml --env-file infra/local/.env --profile tools run --rm migrate

# Start the web interface on the Vite development server (proxies /api to the local API)
npm --prefix apps/web run dev

# Run the API tests
uv run --directory apps/api pytest

# Stop the containers
docker compose -f infra/local/docker-compose.yml --env-file infra/local/.env down
```
