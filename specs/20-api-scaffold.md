# API scaffold

The structure, settings and cross-cutting behaviour of the API in `apps/api/`. Domain endpoints are added on this scaffold as [21-api.md](21-api.md) defines them.

## Directory structure

The platform code is the API platform package, `ai_app_platform`, installed at a released version ([10-architecture.md](10-architecture.md#platform-packages)). The application's repository holds only its own code:

```
apps/api/
├── main.py                     FastAPI app: configures the platform, adds its middleware and error handlers, includes the routers
├── settings.py                 The application's settings, extending the platform's PlatformSettings (see Settings)
├── database.py                 The engine and session factory, from the platform's create_database_engine
├── dependencies.py             The application's role dependencies, on the platform's authenticate
├── models.py                   SQLAlchemy models on the platform's Base; attaches the application's audit entity types
├── domain/
│   ├── models.py               Domain models (dataclasses)
│   ├── enums.py                Domain enums, including the application's roles
│   └── exceptions.py           Domain exception hierarchy
├── repositories/
├── services/
├── routes/                     The application's versioned routes
│   └── directory.py            Directory lookup (optional; not yet in the platform)
├── schemas/                    The application's request and response models
├── jobs/                       Entry points for scheduled ECS tasks
├── alembic/                    Migrations; the first creates the template tables (30-data-model.md)
├── tests/
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── alembic.ini
├── Dockerfile
├── pyproject.toml
├── uv.lock
└── README.md
```

What the package provides, by module of `ai_app_platform`:

| Module | Provides |
|---|---|
| `config` | `configure(settings=..., session_factory=..., after_authentication=...)`: what the application gives the platform once, at start-up |
| `settings` | `PlatformSettings`, with the settings and deployed-environment checks below |
| `database` | The declarative `Base`, `create_database_engine` and the Secrets Manager credential provider |
| `dependencies` | `get_db`, `DbSession`, `authenticate` and `Auth` |
| `auth` | Providers from discovery, web sign-in, sessions, cross-site request forgery checks, bearer token validation, `AuthContext`, `ActorType` and `require_role` |
| `audit` | The `audit_records` model and `AuditService` ([31-audit.md](31-audit.md)) |
| `middleware` | Correlation and request IDs, request logging, and the unexpected-error handler |
| `errors` | The error response shape, `error_body` and `install_error_handlers` |
| `routes` | `GET /api/health` and the `/api/auth/` sign-in endpoints |
| `schemas` | The error and pagination schemas |
| `migrations` | `create_platform_tables`, and `grant_runtime` and `revoke_runtime` for the runtime database user |
| `logging_config` | The JSON log formatter |
| `testing` | `TestIssuer`, the test key pair the application's tests sign tokens with |

## Settings

Every setting is declared once, as a Pydantic Settings model. The platform's `PlatformSettings` declares those below, apart from `DIRECTORY_ENABLED`; the application's `Settings`, in `settings.py`, extends it with that and its own settings, and gives the platform its settings through `configure`. Each declaration states its type, its default (or that it is required) and whether it is secret. Settings are validated when the API starts, and the API refuses to start if validation fails. Secret settings are typed `SecretStr`, so they are never printed or logged.

| Setting | Type | Default | Secret | Notes |
|---|---|---|---|---|
| `APP_ENVIRONMENT` | `local`, `dev`, `test` or `prod` | Required | No | Controls the deployed-environment checks below |
| `APP_VERSION` | string | `unknown` | No | Set by the pipeline to the commit SHA |
| `APP_PUBLIC_URL` | URL | Required | No | The address people use, for example `https://inspections.example`; used for redirect URIs and the `Origin` check |
| `API_BASE_PATH` | string | `/api/v1` | No | |
| `LOG_LEVEL` | `DEBUG`, `INFO`, `WARNING`, `ERROR` | `INFO` | No | |
| `APP_BUSINESS_TIME_ZONE` | IANA time zone name | `Australia/Perth` | No | Sets the local day for business rules that depend on it. Display uses the browser's time zone. |
| `AUTH_INTERNAL_METADATA_URL` | URL | Required | No | OIDC discovery document of the internal issuer. For Entra ID: `https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration` |
| `AUTH_INTERNAL_CLIENT_ID` | string | Required | No | The application's registration with the internal issuer; also the bearer token audience |
| `AUTH_EXTERNAL_METADATA_URL` | URL | None | No | OIDC discovery document of the external issuer. For Entra External ID: `https://{tenant-name}.ciamlogin.com/{tenant-id}/v2.0/.well-known/openid-configuration`. External sign-in and external tokens are refused when unset. |
| `AUTH_EXTERNAL_CLIENT_ID` | string | None | No | The application's registration with the external issuer; required with the metadata URL |
| `AUTH_ALLOWED_ISSUER_HOSTS` | comma-separated host patterns | `login.microsoftonline.com,*.ciamlogin.com` | No | The hosts a metadata URL may use in deployed environments. Change it when using another provider or a custom sign-in domain. |
| `AUTH_SUBJECT_CLAIM` | string | `oid` | No | The claim that identifies a person. `oid` for Entra ID; `sub` for most other providers. |
| `AUTH_CLIENT_CERTIFICATE_SECRET_ARN` | string | None | No | Secrets Manager secret holding the client certificate and private key. Required in deployed environments. |
| `AUTH_USER_SCOPE` | string | `access_as_user` | No | Delegated scope required in delegated bearer tokens |
| `SESSION_IDLE_MINUTES` | integer | `30` | No | |
| `SESSION_MAX_HOURS` | integer | `10` | No | |
| `DATABASE_HOST` | string | Required | No | |
| `DATABASE_PORT` | integer | `5432` | No | |
| `DATABASE_NAME` | string | Required | No | |
| `DATABASE_SSL_MODE` | `verify-full` or `disable` | `verify-full` | No | `disable` is permitted only when `APP_ENVIRONMENT` is `local` |
| `DATABASE_CREDENTIALS_SECRET_ARN` | string | None | No | The ARN is an identifier. Required in deployed environments. |
| `DATABASE_USER` | string | None | No | Local only |
| `DATABASE_PASSWORD` | `SecretStr` | None | Yes | Local only, holding the throwaway local container password |
| `DATABASE_CREDENTIALS_CACHE_SECONDS` | integer | `300` | No | How long fetched credentials are reused |
| `DIRECTORY_ENABLED` | boolean | `false` | No | Enables the directory lookup endpoint |

**Checks in deployed environments.** When `APP_ENVIRONMENT` is `dev`, `test` or `prod`, the API refuses to start unless:

- each configured metadata URL uses HTTPS on a host matching `AUTH_ALLOWED_ISSUER_HOSTS`, and `APP_PUBLIC_URL` uses HTTPS
- `AUTH_CLIENT_CERTIFICATE_SECRET_ARN` is set
- `DATABASE_SSL_MODE` is `verify-full`
- `DATABASE_CREDENTIALS_SECRET_ARN` is set, and `DATABASE_USER` and `DATABASE_PASSWORD` are not

There is no setting that turns authentication off, and no cross-origin (CORS) setting: CloudFront serves the web interface and the API on one address, and consumers call from their servers.

## Database access

- SQLAlchemy 2.x with `pool_pre_ping`, a pool of 5 and an overflow of 10.
- The API connects as the application's runtime database user, which can read and write data but cannot change the schema. Only the migration step connects as the migration user ([12-aws-deployment.md](12-aws-deployment.md#database)).
- In deployed environments, `database.py` fetches the runtime user's credentials from Secrets Manager (`DATABASE_CREDENTIALS_SECRET_ARN`) when it opens a connection, and reuses them for `DATABASE_CREDENTIALS_CACHE_SECONDS`. If a connection fails authentication, it fetches the credentials again once before failing. A rotated password is therefore picked up without a redeployment (Standard ADR-0013).
- Connections use TLS with `sslmode=verify-full`, verifying the server against the Amazon RDS certificate bundle copied into the image.
- The session dependency yields a session, commits on success, rolls back on exception and always closes the session. It is declared with `Depends(get_db, scope="function")`, so the commit happens before the response is sent. With FastAPI's default scope, the code after `yield` runs once the response has gone, and a client could act on a write that is not yet committed, or be told a write succeeded when its commit then fails.

## Authentication and authorisation

**Requirements:** [NFR-SEC-01](02-requirements.md#nfr-sec-01-authenticated-and-authorised-access)

The model is in [50-auth.md](50-auth.md). The API platform package implements it as follows; the modules named are within `ai_app_platform`. Version 0.1 implements the internal issuer only: the external issuer's settings and discovery are in place, but its sign-in, sessions and tokens are not, so an application with external users writes them until the standard's work plan item F03.5 provides them.

**Providers.** `auth.providers` reads each provider's discovery document at start-up and caches it, including the issuer, the endpoints and the signing keys. Keys are refreshed when an unknown key ID appears. Tokens are validated against the issuer the discovery document gives, and the API refuses to start if the discovery document cannot be read.

**Sign-in endpoints** (`routes.auth`, unversioned because they are part of the platform rather than the business API):

| Endpoint | Purpose |
|---|---|
| `GET /api/auth/login?provider={internal or external}&return_to={path}` | Starts the authorisation code flow with PKCE. `return_to` must be a path on the application's own address. |
| `GET /api/auth/callback` | Exchanges the code using the client certificate, validates the ID token (signature, issuer, audience, expiry, nonce), creates the session and sets the cookie, then redirects to `return_to` |
| `GET /api/auth/session` | Returns the signed-in person's display name, actor type, roles and the session's cross-site request forgery token, or 401 |
| `POST /api/auth/logout` | Deletes the session, clears the cookie and returns the provider's sign-out address |

**Sessions.** `auth.sessions` stores sessions in the `auth_sessions` table ([30-data-model.md](30-data-model.md#template-tables)). The cookie value is 32 random bytes; the table holds its SHA-256 hash. Each request with the cookie updates the session's last-used time, at most once a minute. Expired sessions are removed by a scheduled job.

**Request authentication.** `dependencies.authenticate` resolves every request except `/api/health` and the sign-in endpoints in this order:

1. An `Authorization: Bearer` header is validated by `auth.tokens`: signature, issuer (the internal or external provider), audience (that provider's client ID), expiry, not-before and algorithm (`RS256`). A delegated token must include `AUTH_USER_SCOPE`; an app-only token must carry at least one role.
2. Otherwise, a valid, unexpired `__Host-session` cookie is used. For state-changing methods, `auth.csrf` also checks the `X-CSRF-Token` header and that the `Origin` header matches `APP_PUBLIC_URL`.
3. Otherwise the request is refused with 401.

After a caller is authenticated, the platform calls the application's `after_authentication` hook, if `configure` was given one, for the application's own per-request work; it may refuse the request.

`AuthContext` holds the subject (the claim named by `AUTH_SUBJECT_CLAIM`), the actor type, the provider, the display name where present and the roles. `require_role(authenticate, *roles)` is a dependency factory, which the application's `dependencies.py` uses to declare its role dependencies. It returns the `AuthContext` when the caller holds at least one of the listed roles, and raises an authorisation error otherwise. There is no role hierarchy.

A missing, expired or invalid token or session returns 401, and a failed cross-site request forgery check or missing role returns 403. No response says which check failed. The reason is logged without the token or cookie.

## Actor types

`ActorType` is defined once in the platform's `auth.context`, separate from roles:

| Value | Who |
|---|---|
| `internal` | A person signed in through the internal provider |
| `external` | A person signed in through the external provider, directly or through a consumer acting for them. Not yet in the package (F03.5) |
| `system` | Another system using an app-only token |
| `job` | A scheduled task of this application, with the job name as the subject |

Audit records and logs use it to say what kind of actor acted.

## Domain exceptions

Defined in `domain/exceptions.py`. Each accepts a message and an optional list of details.

| Exception | HTTP status | Error code |
|---|---|---|
| `DomainError` | (base class) | |
| `EntityNotFoundError` | 404 | `NOT_FOUND` |
| `ValidationError` | 400 | `VALIDATION_ERROR` |
| `ConflictError` | 409 | `CONFLICT` |
| `InvalidStateError` | 409 | `CONFLICT` |
| `AuthorisationError` | 403 | `FORBIDDEN` |

`main.py` registers handlers that turn the application's domain exceptions into the error response, and calls the platform's `install_error_handlers`, which does the same for authentication and access failures, request validation failures and unexpected errors, in the error response in [21-api.md](21-api.md#error-response). Unexpected errors return `INTERNAL_ERROR` without internal detail and are logged with the stack trace.

## Middleware

The platform's `middleware` module provides both, and the application adds them in `main.py`.

**Correlation.** Reads `X-Correlation-ID` from the request or generates a UUID, generates an `X-Request-ID` for every request, stores both on the request state and in a context variable for logging, and returns both in the response headers.

**Request logging.** Logs every completed request with the method, the path template, the status code, the duration in milliseconds, the correlation and request IDs and the actor type. It does not log the subject or any other personal information; the audit trail records who acted, linked by the correlation ID.

There is no rate limiting in the API. Rate-based rules are applied by AWS WAF at CloudFront ([12-aws-deployment.md](12-aws-deployment.md#edge)).

## Logging

**Requirements:** [NFR-OBS-01](02-requirements.md#nfr-obs-01-logging-and-alarms)

The platform's `logging_config` configures one JSON formatter for all loggers, writing one JSON object per line to standard output. Every line includes the timestamp (UTC, ISO 8601), level, logger, message, correlation ID and request ID, plus every field passed through `extra`. Personal information (including names, email addresses and subject identifiers), tokens, cookies, passwords and connection strings are never logged, and the formatter drops any `extra` field named in its deny list (`authorization`, `cookie`, `token`, `password`, `email`, `name`, `oid`, `subject`). Path templates are logged rather than full paths, so identifiers in URLs are not.

## Health endpoint

`GET /api/health`, from the platform's `routes.health`, needs no authentication. It runs `SELECT 1` against the database and returns:

- 200 with `{"status": "healthy", "version": "...", "timestamp": "..."}` when the database responds
- 503 with `{"status": "unhealthy", "checks": {"database": "unavailable"}}` when it does not

The container health check, the load balancer health check and the pipeline's check through CloudFront all use this path.

## Directory lookup

`GET /api/v1/directory/users?search={text}` finds internal users in the internal issuer's directory (Microsoft Graph for Entra ID) for pickers in the web interface. It is optional, and the only path to the directory; it is not yet in the platform package, so an application that needs it writes it (the standard's work plan item F03.4); the web interface never calls the directory directly (Standard ADR-0010).

- Available only when `DIRECTORY_ENABLED` is true, and to the roles listed for it in [21-api.md](21-api.md).
- Requires at least three characters of search text and returns at most 25 people.
- Returns only the ID, display name, email address and job title.
- Calls the directory with the narrowest application permission it offers, granted to the application's registration, and caches results for five minutes.

## Container image

`apps/api/Dockerfile` is a multi-stage build:

1. A build stage, from a Python base image pinned by digest, installs `git`, which uv needs to fetch the platform package from its tag, then installs dependencies from `uv.lock` with `uv sync --frozen --no-dev` into a virtual environment.
2. A runtime stage, from the same base image, copies only the virtual environment, the application code and the RDS certificate bundle. It runs as a non-root user and starts `uvicorn` on `0.0.0.0:8000`.

The image has a container health check calling `/api/health`. The same image runs the API, the migration task (`alembic upgrade head`) and scheduled jobs (`python -m apps.api.jobs.{job}`).

**Network certificate.** Some networks inspect encrypted traffic and re-sign it with their own root certificate. Each build step that downloads packages mounts an optional build secret, `corporate_ca`, with `RUN --mount=type=secret,id=corporate_ca,required=false`. When the secret is present, the step sets `SSL_CERT_FILE` to it for that step only. The secret is a full certificate bundle, because `SSL_CERT_FILE` replaces uv's trusted certificates rather than adding to them. On a developer's machine, Compose supplies it ([11-local-development.md](11-local-development.md#image-builds)). Pipeline builds supply no secret. A secret mount is not stored in any image layer, so the image is the same wherever it is built.

## Dependencies

| Purpose | Package |
|---|---|
| The platform | `ai-app-platform`, from its release tag ([10-architecture.md](10-architecture.md#platform-packages)) |
| Web framework and server | `fastapi`, `uvicorn[standard]` |
| Database | `sqlalchemy`, `psycopg[binary]`, `alembic` |
| Settings and schemas | `pydantic`, `pydantic-settings` |
| Token validation | `pyjwt[crypto]`, which the platform uses for sign-in, client authentication and bearer tokens |
| AWS | `boto3` |
| HTTP client (discovery, directory lookup) | `httpx` |
| Development | `pytest`, `pytest-cov`, `ruff`, `pyright[nodejs]` |

The platform package depends on the same libraries; the application lists those it imports itself.

Versions are pinned in `uv.lock`, which is committed. The Python version is set in `pyproject.toml` (`requires-python`) and matches the Dockerfile base image.

## Tests

`tests/conftest.py` provides:

- `db_engine`: connected to PostgreSQL, the same database engine as production
- `db_session`: each test in a transaction with a savepoint, rolled back afterwards
- `client`: a `TestClient` with the database session dependency overridden
- `issue_token(roles, delegated)`: signs bearer tokens with the platform's `testing.TEST_ISSUER`, whose key pair the API validates through its normal code path, once the tests set it as the internal provider
- `signed_in(roles)`: creates a sign-in session with the platform's `auth.sessions.create_session` and returns the cookie and cross-site request forgery token

The platform's own behaviour is tested in `ai-app-platform`; the application's tests cover its configuration of the platform and its own code.

## Acceptance criteria

- The API starts in Docker Compose and `GET /api/health` returns 200 when the database is available and 503 when it is not.
- The API image builds in the pipeline without the `corporate_ca` secret, and on a developer's machine with it. No file in the built image comes from the secret.
- The API refuses to start in a deployed environment with an issuer that is not on HTTPS at an allowed host, without a client certificate, without TLS verification or with local database credentials.
- Signing in through `/api/auth/login` sets a `__Host-session` cookie that is `HttpOnly`, `Secure` and `SameSite=Lax`, and no token reaches the browser.
- Every endpoint except `/api/health` and `/api/auth/` returns 401 without a valid token or session, and 403 without a permitted role.
- A state-changing request with the session cookie but without the cross-site request forgery token, or from another origin, returns 403.
- An external user requesting a record they are not allowed to see receives 404.
- Correlation and request IDs are returned on every response and appear in every log line.
- Logs are valid JSON, one object per line.
- Domain exceptions produce the shared error response.
- `pytest` passes with the coverage thresholds in [99-generation-rules.md](99-generation-rules.md#tests).
