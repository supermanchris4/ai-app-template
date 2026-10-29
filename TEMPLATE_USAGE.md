# Using the template

How to create an application's repository from this template. Delete this file from the application's repository once these steps are done.

## 1. Create the repository

1. On GitHub, choose **Use this template** on `ai-app-template`, give the repository the application's short name and make it private. Or run `gh repo create {owner}/{app} --private --template {owner}/ai-app-template`.
2. Clone it into the WSL Linux filesystem, for example `~/src/{app}` (see the standard's developer setup guide).
3. Run `pre-commit install`.

## 2. Record where it came from

Add a line to the top of the new `README.md` naming the template commit it was created from, so later template changes can be compared:

```
Created from ai-app-template at {commit SHA} on {date}.
```

## 3. Replace the template placeholders

| Placeholder | Where | Replace with |
|---|---|---|
| `app` prefix | [specs/11-local-development.md](specs/11-local-development.md), `infra/local/` | The application's short name, used for container names, the database name and the database users |
| Template title and description | `README.md`, `CLAUDE.md` | The application's name and purpose |
| `@supermanchris4` | `.github/CODEOWNERS` | The GitHub users or teams who review each path, if not the template owner |
| Example roles | [specs/50-auth.md](specs/50-auth.md) | The application's roles |

## 4. Fill in the application stubs

Each specification marked **APPLICATION STUB** is filled in before any domain code is generated, starting with [specs/00-overview.md](specs/00-overview.md), [specs/01-glossary.md](specs/01-glossary.md) and [specs/02-requirements.md](specs/02-requirements.md). Then write the delivery plan in [docs/delivery/](docs/delivery/README.md).

Delete what the application does not need, with an ADR where a template specification says one is required: the external issuer if it has no external users, the directory lookup if it needs none, and `apps/web/` if it has no web interface.

## 5. Set up the identity provider, GitHub and AWS

Follow [docs/runbook.md](docs/runbook.md) for the identity provider registrations, the GitHub Environments and variables, and the AWS account bootstrap. AWS is needed only when the application is first deployed; local development needs none of it.
