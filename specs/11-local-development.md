# Local development

How the application runs on a developer's machine. Developers work in WSL with Docker Engine, with the repository cloned into the WSL Linux filesystem (Standard ADR-0011). Workstation setup is in the developer setup guide in the AI application standard.

Local development needs no real secret. Sign-in uses the local mock identity provider, and AWS access, where needed, uses the developer's own single sign-on session.

## Files

| File | Contents |
|---|---|
| `infra/local/docker-compose.yml` | The local services |
| `infra/local/.env.example` | Local settings, copied to `infra/local/.env` (which git ignores) |
| `infra/local/ca-certificates.crt` | A copy of the developer's certificate bundle for image builds (which git ignores) |
| `infra/local/postgres-init/` | SQL run when the database volume is first created |
| `infra/local/mock-idp.json` | The application's roles and synthetic people, for the local mock identity provider |
| `infra/local/README.md` | The commands below |

## Services

All services join one bridge network, `local_net`. The application prefix (`app` in the template) is replaced with the application's short name when a repository is created from the template, and is used for container names and the database name.

| Service | Purpose | Image or build | Host port | Starts after |
|---|---|---|---|---|
| `postgres` | Local database | PostgreSQL image at the version the RDS instance uses, pinned by digest | `5432` | |
| `mock-idp` | Stands in for the internal and external issuers | The platform's released mock identity provider image, pinned by version and digest ([10-architecture.md](10-architecture.md#platform-packages)) | `9000` | |
| `api` | The API, with source mounted and auto-reload | Built from `apps/api/Dockerfile` | `8000` | `postgres` and `mock-idp` healthy |
| `migrate` | Applies migrations (`alembic upgrade head`), then exits. In the `tools` profile, so it runs only on request. | Same as `api` | | `postgres` healthy |

The web interface is not containerised locally. It runs on the Vite development server (`npm --prefix apps/web run dev`) at `http://localhost:5173`, which proxies `/api` to `http://localhost:8000`. The browser therefore reaches the web interface and the API on one address, as it does through CloudFront, and the session cookie works the same way.

**Postgres.** Data is kept in the named volume `pg_data`. The health check uses `pg_isready`. The scripts in `postgres-init/` create the same two users as the deployed database: `app_migrator`, which owns the schema, and `app_runtime`, which can read and write data but cannot change the schema. Their passwords are throwaway values from `.env`.

**API.** Runs with `APP_ENVIRONMENT=local` and `APP_PUBLIC_URL=http://localhost:5173`, connecting as `app_runtime` with `DATABASE_SSL_MODE=disable`. The health check calls `/api/health`.

**Migrate.** Connects as `app_migrator`, the same split as the deployed migration task.

### Image builds

Where the developer's network inspects encrypted traffic, re-signing it with its own root certificate, the images built locally download packages through it. `docker-compose.yml` defines a secret, `corporate_ca`, from `./ca-certificates.crt`, and the `api` build uses it. The Dockerfile mounts it as [20-api-scaffold.md](20-api-scaffold.md#container-image) describes. The file is a copy of the machine's certificate bundle, `/etc/ssl/certs/ca-certificates.crt`, which holds the network's root as well as the public ones after the developer setup guide's certificate step. It must be a copy rather than a link, because Compose builds read only files inside the project unless granted wider access. Where the network does not inspect traffic, the copy holds only the public roots and changes nothing. Every build needs the file, so the script that starts the end-to-end tests' stack copies `/etc/ssl/certs/ca-certificates.crt` there when it is missing; a fresh machine, such as a CI runner, then needs no step.

## Local mock identity provider

A small OpenID Connect provider, released by the platform as an image ([10-architecture.md](10-architecture.md#platform-packages)), that stands in for both the internal issuer and the external issuer. Its README in `ai-app-platform` (`packages/mock-idp`) gives its settings. The API signs people in and validates tokens against it through the same code as in deployed environments, and the web interface has no local-only sign-in path.

- It serves two issuers: `http://mock-idp:9000/internal` for internal users and `http://mock-idp:9000/external` for external users. An application with no external users uses only the first. Each has its own discovery document, signing keys, authorisation endpoint and token endpoint.
- It generates new RSA key pairs each time it starts. No key is stored on disk or in the repository.
- **The application's roles and people.** The image holds none of an application's roles or people. It reads them at start-up from the JSON file named by `MOCK_IDP_CONFIG`, which Compose mounts read-only from `infra/local/mock-idp.json`, and refuses to start without it or if it does not make sense. The file holds `human_roles` (offered on the sign-in page), `system_roles` (for test tokens only) and `people`, each a synthetic person with an `oid`, a `name` and default `roles` drawn from `human_roles`. People are synthetic, never real.
- **Signing in.** Its authorisation page, reached by the browser at `http://localhost:9000`, shows a list of the application's roles for that issuer and a choice of synthetic people. Choosing one completes the authorisation code flow back to the API. The discovery documents give the browser-facing address for the authorisation page and the container address for the token endpoint.
- **Bearer tokens.** `POST /{issuer}/test-token` returns a signed access token for the subject, roles and token type in the request body, for calling the API directly as a system or as a consumer acting for an external user would. A delegated token includes the `scp` claim with `access_as_user`; an app-only token has no `scp`.
- It accepts any client authentication, so no certificate is needed locally.
- It is never deployed. The API refuses to start outside `local` with an issuer that is not on HTTPS at an allowed host ([20-api-scaffold.md](20-api-scaffold.md#settings)).

**Tests.** Automated tests use their own test key pair and session helper ([20-api-scaffold.md](20-api-scaffold.md#tests)).

## Local settings

`infra/local/.env.example` lists every local setting with a safe default. It holds only non-secret settings and throwaway values for the local containers. It never holds a real password, key or token, and nothing in it works outside the developer's machine.

| Setting | Default |
|---|---|
| `APP_PREFIX` | `app` |
| `POSTGRES_PORT` | `5432` |
| `POSTGRES_DB` | `app` |
| `POSTGRES_PASSWORD` | `local-only-postgres` |
| `APP_MIGRATOR_PASSWORD` | `local-only-migrator` |
| `APP_RUNTIME_PASSWORD` | `local-only-runtime` |
| `API_PORT` | `8000` |
| `MOCK_IDP_PORT` | `9000` |
| `AUTH_INTERNAL_CLIENT_ID` | `local-internal-app` |
| `AUTH_EXTERNAL_CLIENT_ID` | `local-external-app` |
| `LOG_LEVEL` | `INFO` |

The API's own settings in Compose (`AUTH_INTERNAL_METADATA_URL`, `AUTH_EXTERNAL_METADATA_URL`, `DATABASE_HOST` and the rest) are set in `docker-compose.yml` from these values. A new required setting is added here and to [20-api-scaffold.md](20-api-scaffold.md#settings) in the same change.

The web interface needs no local settings; `apps/web/public/config.json` names the environment as `local`.

## Commands

Run from the repository root. `infra/local/README.md` lists the same commands.

| Task | Command |
|---|---|
| Create local settings (once) | `cp infra/local/.env.example infra/local/.env` |
| Copy the certificate bundle (once, and again after the machine's certificates change) | `cp /etc/ssl/certs/ca-certificates.crt infra/local/ca-certificates.crt` |
| Start the database, identity provider and API | `docker compose -f infra/local/docker-compose.yml --env-file infra/local/.env up -d postgres mock-idp api` |
| Apply migrations | `docker compose -f infra/local/docker-compose.yml --env-file infra/local/.env --profile tools run --rm migrate` |
| Start the web interface | `npm --prefix apps/web run dev` |
| Show container health | `docker compose -f infra/local/docker-compose.yml ps` |
| Stop | `docker compose -f infra/local/docker-compose.yml --env-file infra/local/.env down` |
| Reset the database (deletes local data) | `docker compose -f infra/local/docker-compose.yml --env-file infra/local/.env down -v` |

## Acceptance criteria

- On a clean machine, including one on a network that inspects encrypted traffic, the start command builds the images and brings up `postgres`, `mock-idp` and `api`, and all report healthy.
- The API starts only after the database and identity provider are healthy.
- `GET http://localhost:8000/api/health` returns 200.
- The web interface at `http://localhost:5173` signs an internal user in through the mock provider, receives the session cookie and calls the API through the proxy. The same works for an external user.
- A bearer token from the mock provider is accepted by the API, and a token signed by any other key is rejected.
- Schema migrations run through `migrate` without installing tools on the host.
- Data survives a restart, and `down -v` removes it.

## Not in the local baseline

Caches, message brokers, tracing or metrics stacks, reverse proxies, TLS termination, extra databases and secrets tooling are not added locally without an ADR.
