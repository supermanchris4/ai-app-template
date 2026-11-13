# AI application template

The starter repository for applications built with AI coding assistants, such as Claude Code. An application's repository is created from this template and then owns its copy. Applications have a web interface, an API running in a container and a PostgreSQL database, hosted on AWS with source code and pipelines in GitHub.

The template implements the AI application standard, held in the separate `ai-app-standard` repository. The standard holds the architecture and the decisions behind it. Set up your workstation by following its developer setup guide (`docs/developer-setup.md` in `ai-app-standard`) before you clone a repository made from this template. This repository holds only what an application needs in its own repository, and refers to the standard for the reasons behind it.

To create an application from the template, follow [TEMPLATE_USAGE.md](TEMPLATE_USAGE.md).

## Structure

| Folder | Contents |
|---|---|
| [apps/](apps/) | The API (`api/`) and the web interface (`web/`) |
| [infra/](infra/) | The AWS bootstrap, the Terraform for the deployed application and the local development containers |
| [specs/](specs/) | The specifications the AI coding assistant builds from |
| [adrs/](adrs/) | This application's own decision records, including any deviation from the standard |
| [docs/](docs/) | The delivery plan and the runbook this application needs in its own repository |
| `.github/` | The pipelines, the pull request template and code owners |

`CLAUDE.md` and `.claude/settings.json` at the root tell the AI coding assistant how to work in the repository.

## What an application writes and what it takes

The platform code the template specifies is released as packages from `ai-app-platform`, and an application installs them at a version rather than writing them ([specs/10-architecture.md](specs/10-architecture.md#platform-packages)):

| From the platform | Written by the application |
|---|---|
| The API platform: settings, logging, database access, middleware, error handling, health, sign-in and access, audit recording, the template tables and test helpers | Its settings, roles, domain code, routes, migrations and tests |
| The web platform: the API client, the session provider, the route guard, sign-in navigation, the error notice, the environment banner, local dates and navigation links | `src/lib/api.ts`, its pages, its component kit and its theme |
| The mock identity provider image | Its roles and synthetic people file |
| The Terraform modules, one per concern of the AWS infrastructure and the account bootstrap | Its two Terraform roots, which compose them, and its database initialisation job |
| The reusable workflows: the pull request checks, the Terraform checks, plan and apply, and the build and deployment | The four workflows that call them |

Not yet in the packages, so written by the application from the specifications: external users and the directory lookup. The standard's work plan (`work/README.md`) schedules each.

## Status

The template holds specifications and repository settings only; an application generates its own code from them, taking the platform code from the packages above. The platform code was first built inside the first application built to the standard, and then extracted into the packages (the standard's work plan, Feature F03.2).

## Licence

Apache License 2.0: see [LICENSE](LICENSE).
