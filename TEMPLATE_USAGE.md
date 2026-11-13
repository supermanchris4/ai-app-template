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
| The template owner | `.github/CODEOWNERS` | The GitHub users or teams who review each path |
| Example roles | [specs/50-auth.md](specs/50-auth.md) | The application's roles |
| Roles and synthetic people | `infra/local/mock-idp.json` | The application's roles, and made-up people to sign in as locally |

## 4. Take the platform packages

The platform code is installed from the packages released by `ai-app-platform`, never generated or copied ([specs/10-architecture.md](specs/10-architecture.md#platform-packages), which names the versions). None of them needs a GitHub or AWS secret.

**API platform.** In `apps/api/pyproject.toml`, depend on `ai-app-platform` and install it from its release tag. `uv.lock` records the tag's commit. uv fetches git sources with `git`, so the image's build stage installs it ([specs/20-api-scaffold.md](specs/20-api-scaffold.md#container-image)). Do not install the wheel attached to the release instead: Dependabot cannot check its download link, and then updates none of the API's dependencies.

```toml
[project]
dependencies = ["ai-app-platform=={version}"]

[tool.uv.sources]
ai-app-platform = { git = "https://github.com/supermanchris4/ai-app-platform", tag = "api-v{version}", subdirectory = "packages/api" }
```

**Web platform.** It is published to GitHub Packages, whose npm registry needs a token to read even a public package.

1. Create `apps/web/.npmrc`, which names the registry only and never holds a token:

   ```ini
   @supermanchris4:registry=https://npm.pkg.github.com
   min-release-age=7
   ```

2. On each developer's machine, sign in once with a personal access token (classic) that has only the `read:packages` scope and an expiry date. npm keeps it in the developer's own `~/.npmrc`:

   ```bash
   npm login --scope=@supermanchris4 --auth-type=legacy --registry=https://npm.pkg.github.com
   ```

3. Install it at an exact version: `npm --prefix apps/web install --save-exact @supermanchris4/ai-app-platform-web@{version}`.
4. Give every workflow job that runs `npm ci` the `packages: read` permission, and `NODE_AUTH_TOKEN: ${{ github.token }}` with `actions/setup-node`'s `registry-url: https://npm.pkg.github.com` and `scope: "@supermanchris4"`. The run's own token reads the package.
5. Add the package's build to the Tailwind `content` setting ([specs/40-ui.md](specs/40-ui.md#template-conventions)).

`min-release-age=7` makes npm wait seven days before using a newly published version, as uv's `exclude-newer` does for the API. It applies to the platform's own releases too; for an urgent platform fix, install that one version with `--min-release-age=0`.

**Mock identity provider.** In `infra/local/docker-compose.yml`, the `mock-idp` service uses the image at its version, pinned by the digest its release notes give, with the application's roles and people mounted ([specs/11-local-development.md](specs/11-local-development.md#local-mock-identity-provider)):

```yaml
mock-idp:
  image: ghcr.io/supermanchris4/ai-app-platform-mock-idp:{version}@sha256:{digest}
  environment:
    MOCK_IDP_CONFIG: /config/mock-idp.json
  volumes:
    - ./mock-idp.json:/config/mock-idp.json:ro
```

Each package's README in `ai-app-platform` has the full detail, and its `CHANGELOG.md` says what each version changed. Upgrade the platform packages on purpose, not through Dependabot: change the version, read the changelog, and run the tests.

## 5. Turn on Dependabot for the application

The template's `.github/dependabot.yml` covers only what the template itself holds, because Dependabot fails on a folder with nothing to update. Once `apps/` and `infra/` hold code, replace it with the application's settings:

```bash
git mv -f .github/dependabot.example.yml .github/dependabot.yml
```

Remove the entries for anything the application does not have, such as `/apps/web` if it has no web interface.

## 6. Fill in the application stubs

Each specification marked **APPLICATION STUB** is filled in before any domain code is generated, starting with [specs/00-overview.md](specs/00-overview.md), [specs/01-glossary.md](specs/01-glossary.md) and [specs/02-requirements.md](specs/02-requirements.md). Then write the delivery plan in [docs/delivery/](docs/delivery/README.md).

Delete what the application does not need, with an ADR where a template specification says one is required: the external issuer if it has no external users, the directory lookup if it needs none, and `apps/web/` if it has no web interface.

## 7. Set up the identity provider, GitHub and AWS

Follow [docs/runbook.md](docs/runbook.md) for the identity provider registrations, the GitHub Environments and variables, and the AWS account bootstrap. AWS is needed only when the application is first deployed; local development needs none of it.
