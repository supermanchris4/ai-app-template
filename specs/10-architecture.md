# Architecture

The technology baseline and architectural rules every application built from the template follows. The reasons are in the AI application standard.

## Shape

- The application has a web interface, an API running in a container and a PostgreSQL database, hosted on AWS.
- The API holds all business logic, validation and access control. The web interface is one client of the API among others; other systems use the same versioned API.
- Nothing reads or writes the database except the API application's own code: the API service, its scheduled tasks and the migration step, all from the same image. Only the migration step changes the schema. Anything outside the API application, including other systems and the web interface, goes through the API.

```mermaid
---
config:
  layout: elk
---
flowchart LR
  user[People in browsers] --> cf[CloudFront with AWS WAF]
  cx[Consumers] -->|user or system token| cf
  cf -->|static files| s3[(S3 web bucket)]
  cf -->|/api over HTTPS, VPC origin| alb[Internal load balancer]
  subgraph app[API application: one image on ECS Fargate]
    api[API service]
    job[Scheduled task]
  end
  alb --> api
  sched[EventBridge Scheduler] -->|starts| job
  app --> db[(RDS PostgreSQL)]
  app --> sm[Secrets Manager]
  api -. signs people in .-> idp[OIDC issuers]
```

## Technology baseline

| Component | Choice | Standard |
|---|---|---|
| API language and framework | Python with FastAPI | ADR-0002 |
| ORM and migrations | SQLAlchemy 2.x and Alembic | ADR-0002 |
| Python packaging | uv, with `uv.lock` committed | ADR-0002 |
| Web interface | React single-page app built with Vite, in TypeScript | ADR-0003 |
| Web routing | React Router | ADR-0003 |
| API client in the web interface | Generated from the API's OpenAPI document | ADR-0002 |
| Identity | Any OpenID Connect issuer, configured by discovery URL: Entra ID by default for internal users, and Entra External ID by default for external users where the application has them, with app roles | ADR-0004 |
| Web sign-in | The API as backend-for-frontend: it signs people in and holds a server-side session; the browser holds only a session cookie | ADR-0003 |
| Database | PostgreSQL on Amazon RDS | |
| Compute | ECS on AWS Fargate | |
| Web hosting | S3 behind CloudFront | ADR-0003, ADR-0006 |
| Infrastructure as code | Terraform | |
| Pipelines | GitHub Actions, authenticating to AWS through OIDC | ADR-0007 |
| Monitoring | CloudWatch Logs, Container Insights and alarms | ADR-0016 |
| Local development | Docker Compose in WSL with Docker Engine | ADR-0011 |

### Versions

| Component | Version | Pinned in |
|---|---|---|
| Python | 3.13 | `apps/api/pyproject.toml` (`requires-python`) and the `apps/api/Dockerfile` base image |
| Node | 24 (LTS) | `apps/web/.nvmrc` and `engines` in `apps/web/package.json` |
| PostgreSQL | 17 | The Terraform RDS engine version and the `postgres` image in `infra/local/docker-compose.yml` |

These are the template's working versions (Standard configuration default C-02). Each is pinned in the places listed and changed in all of them in the same change. Dependabot proposes patch and minor updates; a major version change is a deliberate change across the listed files.

## Platform packages

The platform code every application needs is released as packages from `ai-app-platform`, a public repository. An application installs each at a released version and keeps no copy of its code. It writes only what the last column lists. A fix to platform code is made in `ai-app-platform` and reaches an application as a new version. Each package's README says how to use it, and its `CHANGELOG.md` says what each version changed.

`{platform-owner}`, here and in the other specifications, stands for the GitHub account or organisation that owns the `ai-app-platform` repository the application takes its packages from. The packages' names, the image and the repository address all use it, since GitHub Packages and GitHub Container Registry name a package after its owner. The application's code uses the actual owner, in lower case; an unreplaced `{platform-owner}` is an error.

| Package | Provides | Installed in | Version | The application writes |
|---|---|---|---|---|
| API platform, `ai_app_platform` | Settings, logging, database access, middleware, error handling, the health and sign-in endpoints, sign-in and access, audit recording, the template tables, and test helpers ([20-api-scaffold.md](20-api-scaffold.md#directory-structure)) | `apps/api/pyproject.toml`, from its `api-v{version}` git tag | 0.3.0 | Its settings, its engine and session factory, the call that configures the platform, its roles, its domain code and its migrations |
| Web platform, `@{platform-owner}/ai-app-platform-web` | The API client, the session provider and route guard, sign-in navigation, the error notice, the environment banner, `config.json`, local dates and navigation links ([40-ui.md](40-ui.md#template-conventions)) | `apps/web/package.json`, from GitHub Packages | 0.1.0 | `src/lib/api.ts`, its pages, its component kit and its Tailwind theme |
| Mock identity provider, `ghcr.io/{platform-owner}/ai-app-platform-mock-idp` | The local OpenID Connect provider ([11-local-development.md](11-local-development.md#local-mock-identity-provider)) | `infra/local/docker-compose.yml`, pinned by digest | 0.1.0 | Its roles and synthetic people, in `infra/local/mock-idp.json` |
| Terraform modules, `packages/terraform/modules` | One module per concern of the AWS infrastructure and the account bootstrap ([12-aws-deployment.md](12-aws-deployment.md#platform-modules)) | Each module's `source` in `infra/aws/` and `infra/bootstrap/`, at the commit its `terraform-v{version}` tag names | 0.1.0 | Its two Terraform roots: their inputs, names, outputs and the modules they compose, its dashboard, and any resource only it needs; the bootstrap script; the database initialisation job |
| Reusable workflows, `.github/workflows` | The pull request checks, the Terraform checks, plan and apply, and the build and deployment ([13-delivery-pipeline.md](13-delivery-pipeline.md#workflows)) | Each call in the application's workflows, at the commit its `workflows-v{version}` tag names | 0.3.2 | Its four calling workflows, `ci.yml`, `infra-plan.yml`, `infra-apply.yml` and `deploy.yml`: their triggers, environments, permissions and inputs |

The versions above are the ones the template was last checked against; an application records its own in the files listed. An application upgrades the platform packages on purpose, not through Dependabot, after reading the changelog.

**Not yet in the packages.** The application writes these itself, from the specifications, until the standard's work plan item named provides them:

| What | Specified in | Work plan item |
|---|---|---|
| External users: the external issuer's sign-in, sessions and tokens, with the `external` actor type | [50-auth.md](50-auth.md) | F03.5 |
| The directory lookup endpoint | [20-api-scaffold.md](20-api-scaffold.md#directory-lookup) | F03.4 |

## API layers

| Layer | Directory | Responsibility |
|---|---|---|
| Routes | `apps/api/routes/` | HTTP handling, role checks, request and response serialisation, transaction ownership |
| Services | `apps/api/services/` | Business logic, rule evaluation, audit recording |
| Domain | `apps/api/domain/` | Domain models, enums, exceptions |
| Repositories | `apps/api/repositories/` | All database access |
| Schemas | `apps/api/schemas/` | Pydantic request and response models, including value-range validation |

Services are stateless and created per request. Routes own the transaction. Services raise domain exceptions and routes translate them to HTTP responses. The detailed rules are in [99-generation-rules.md](99-generation-rules.md#layers).

## API governance

- Versioned by path: `/api/v1`. `/api/health` and the sign-in endpoints under `/api/auth/` are the only unversioned endpoints.
- Paths name the resource, not the caller: `/api/v1/customers/{id}` and `/api/v1/accounts/{id}` serve internal users, external users and systems alike, and who may call each operation is set by the caller's identity and roles (Standard ADR-0018).
- The OpenAPI document generated by FastAPI is the contract. The web interface's client is generated from it.
- Backward compatibility is kept within a major version. A breaking change needs a new major version.
- One error format for every response ([21-api.md](21-api.md#error-response)).
- Correct HTTP status codes: 4xx for client errors and 5xx for server errors.

## API consumers

Every interface is a client of the API (Standard ADR-0018). Other systems, and any separate front end built for external users, call the API from their own servers with their own registrations. Features are therefore always built as API operations first, so an interface can be added or replaced without changing business logic.

Consumers reach the API through the application's CloudFront distribution and AWS WAF (Standard ADR-0019). The API validates every token itself and publishes its OpenAPI document, so a gateway can later be placed in front without changing the application.

## Background processing

| Need | Pattern | Standard |
|---|---|---|
| Scheduled work (data loads, defined exports, threshold checks) | EventBridge Scheduler starts an ECS task from the API image with a different command | ADR-0012 |
| Asynchronous work triggered by an API action | An SQS queue and a worker ECS service from the API image | ADR-0012 |
| Multi-step processing with retries and compensation | Step Functions | ADR-0012 |

Only the scheduled pattern is scaffolded. An application that needs the other two records an ADR and follows the standard's description of them. Background tasks run as the `job` actor type and use the same services, repositories and audit recording as the API.

## Reporting and data supply

Operational screens are built in the application (Standard ADR-0017). A defined supply of data to another organisation or system is built as a defined export or integration that records each run. The application provides no generic data export.

An application that needs reporting in a separate BI tool records an ADR first. The tool then reads a published set of reporting views through a read-only database role, never the application's tables directly, and never audit records or personal information without the data owner's approval.

## Observability

- Structured JSON logs to standard output, collected by CloudWatch Logs.
- The correlation ID is carried from the browser through the API into every log line and audit record.
- Container Insights, alarms and a dashboard per application, as set out in [12-aws-deployment.md](12-aws-deployment.md#monitoring).
- Distributed tracing is not in the baseline. An application that calls several downstream services records an ADR before adding it.

## Environments

Each merge to `main` produces one build: one API image and one web build. That same build is deployed to each environment in turn, unchanged. Only the configuration differs between environments: the API's settings, the values in Secrets Manager and the web interface's `config.json`, which holds only the environment name. Nothing is rebuilt for `test` or `prod`.

| Environment | Purpose | What is deployed, and when |
|---|---|---|
| Local | Development on a developer's machine | The developer's working copy, run with Docker Compose |
| `dev` | Shared development and integration | Each new build, automatically on merge to `main` |
| `test` | Testing and user acceptance | A build already deployed to `dev`, once approved |
| `prod` | Live service | A build already deployed to `test`, once approved |

This works because the application is stateless and reads all environment-specific values from configuration at run time.
