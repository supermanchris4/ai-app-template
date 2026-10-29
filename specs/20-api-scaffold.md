# API scaffold

The structure, settings and cross-cutting behaviour of the API in `apps/api/`. Domain endpoints are added on this scaffold as [21-api.md](21-api.md) defines them.

## Directory structure

```
apps/api/
├── main.py                     FastAPI app, middleware, exception handlers, routers
├── settings.py                 The settings declaration (see Settings)
├── database.py                 Engine, session factory, credential provider
├── dependencies.py             Database session and authentication dependencies
├── auth/
│   ├── providers.py            Internal and external identity providers, from OIDC discovery
│   ├── web_sign_in.py          Authorisation code flow, as a confidential client
│   ├── sessions.py             Server-side sign-in sessions and the session cookie
│   ├── csrf.py                 Cross-site request forgery checks
│   ├── tokens.py               Bearer access token validation
│   ├── context.py              AuthContext and ActorType
│   └── roles.py                The application's app roles and require_role
├── domain/
│   ├── models.py               Domain models (dataclasses)
│   ├── enums.py                Domain enums
│   └── exceptions.py           Domain exception hierarchy
├── repositories/
├── services/
│   └── audit_service.py        Audit recording
├── routes/
│   ├── health.py               GET /api/health
│   ├── auth.py                 /api/auth/ sign-in endpoints
│   └── directory.py            Directory lookup (optional)
├── schemas/
│   └── common.py               Error, pagination and shared schemas
├── middleware/
│   ├── correlation.py          Correlation and request IDs
│   └── request_logging.py      Request logging
├── logging_config.py           JSON log formatter
├── jobs/                       Entry points for scheduled ECS tasks
├── alembic/                    Migrations
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

## Settings

Every setting is declared once, in `settings.py`, as a Pydantic Settings model. Each declaration states its type, its default (or that it is required) and whether it is secret. Settings are validated when the API starts, and the API refuses to start if validation fails. Secret settings are typed `SecretStr`, so they are never printed or logged.

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
- In deployed environments, `database.py` fetches the runtime user's credentials from Secrets Manager (`DATABASE_CREDENTIALS_SECRET_ARN`) when it opens a connection, and reuses them for `DATABASE_CREDENTIALS_CACHE_SECONDS`. If a connection fails authentication, it fetches the credentials again once before failing. A rotated password is therefore picked up without a redeployment (Standard D-13).
- Connections use TLS with `sslmode=verify-full`, verifying the server against the Amazon RDS certificate bundle copied into the image.
- The session dependency yields a session, commits on success, rolls back on exception and always closes the session.

## Authentication and authorisation

The model is in [50-auth.md](50-auth.md). The scaffold implements it as follows.

**Providers.** `auth/providers.py` reads each provider's discovery document at start-up and caches it, including the issuer, the endpoints and the signing keys. Keys are refreshed when an unknown key ID appears. Tokens are validated against the issuer the discovery document gives, and the API refuses to start if the discovery document cannot be read.

**Sign-in endpoints** (`routes/auth.py`, unversioned because they are part of the platform rather than the business API):

| Endpoint | Purpose |
|---|---|
| `GET /api/auth/login?provider={internal or external}&return_to={path}` | Starts the authorisation code flow with PKCE. `return_to` must be a path on the application's own address. |
| `GET /api/auth/callback` | Exchanges the code using the client certificate, validates the ID token (signature, issuer, audience, expiry, nonce), creates the session and sets the cookie, then redirects to `return_to` |
| `GET /api/auth/session` | Returns the signed-in person's display name, actor type, roles and the session's cross-site request forgery token, or 401 |
| `POST /api/auth/logout` | Deletes the session, clears the cookie and returns the provider's sign-out address |

**Sessions.** `auth/sessions.py` stores sessions in the `auth_sessions` table ([30-data-model.md](30-data-model.md#template-tables)). The cookie value is 32 random bytes; the table holds its SHA-256 hash. Each request with the cookie updates the session's last-used time, at most once a minute. Expired sessions are removed by a scheduled job.

**Request authentication.** `dependencies.py` resolves every request except `/api/health` and the sign-in endpoints in this order:

1. An `Authorization: Bearer` header is validated by `auth/tokens.py`: signature, issuer (the internal or external provider), audience (that provider's client ID), expiry, not-before and algorithm (`RS256`). A delegated token must include `AUTH_USER_SCOPE`; an app-only token must carry at least one role.
2. Otherwise, a valid, unexpired `__Host-session` cookie is used. For state-changing methods, `auth/csrf.py` also checks the `X-CSRF-Token` header and that the `Origin` header matches `APP_PUBLIC_URL`.
3. Otherwise the request is refused with 401.

`AuthContext` holds the subject (the claim named by `AUTH_SUBJECT_CLAIM`), the actor type, the provider, the display name where present and the roles. `require_role(*roles)` is a dependency factory that returns the `AuthContext` when the caller holds at least one of the listed roles, and raises an authorisation error otherwise. There is no role hierarchy.

A missing, expired or invalid token or session returns 401, and a failed cross-site request forgery check or missing role returns 403. No response says which check failed. The reason is logged without the token or cookie.

## Actor types

`ActorType` is defined once in `auth/context.py`, separate from roles:

| Value | Who |
|---|---|
| `internal` | A person signed in through the internal provider |
| `external` | A person signed in through the external provider, directly or through a consumer acting for them |
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

`main.py` registers handlers that turn domain exceptions, request validation failures and unexpected errors into the error response in [21-api.md](21-api.md#error-response). Unexpected errors return `INTERNAL_ERROR` without internal detail and are logged with the stack trace.

## Middleware

**Correlation.** Reads `X-Correlation-ID` from the request or generates a UUID, generates an `X-Request-ID` for every request, stores both on the request state and in a context variable for logging, and returns both in the response headers.

**Request logging.** Logs every completed request with the method, the path template, the status code, the duration in milliseconds, the correlation and request IDs and the actor type. It does not log the subject or any other personal information; the audit trail records who acted, linked by the correlation ID.

There is no rate limiting in the API. Rate-based rules are applied by AWS WAF at CloudFront ([12-aws-deployment.md](12-aws-deployment.md#edge)).

## Logging

`logging_config.py` configures one JSON formatter for all loggers, writing one JSON object per line to standard output. Every line includes the timestamp (UTC, ISO 8601), level, logger, message, correlation ID and request ID, plus every field passed through `extra`. Personal information (including names, email addresses and subject identifiers), tokens, cookies, passwords and connection strings are never logged, and the formatter drops any `extra` field named in its deny list (`authorization`, `cookie`, `token`, `password`, `email`, `name`, `oid`, `subject`). Path templates are logged rather than full paths, so identifiers in URLs are not.

## Health endpoint

`GET /api/health` needs no authentication. It runs `SELECT 1` against the database and returns:

- 200 with `{"status": "healthy", "version": "...", "timestamp": "..."}` when the database responds
- 503 with `{"status": "unhealthy", "checks": {"database": "unavailable"}}` when it does not

The container health check, the load balancer health check and the pipeline's check through CloudFront all use this path.

## Directory lookup

`GET /api/v1/directory/users?search={text}` finds internal users in the internal issuer's directory (Microsoft Graph for Entra ID) for pickers in the web interface. It is optional, and the only path to the directory; the web interface never calls the directory directly (Standard D-10).

- Available only when `DIRECTORY_ENABLED` is true, and to the roles listed for it in [21-api.md](21-api.md).
- Requires at least three characters of search text and returns at most 25 people.
- Returns only the ID, display name, email address and job title.
- Calls the directory with the narrowest application permission it offers, granted to the application's registration, and caches results for five minutes.

## Container image

`apps/api/Dockerfile` is a multi-stage build:

1. A build stage, from a Python base image pinned by digest, installs dependencies from `uv.lock` with `uv sync --frozen --no-dev` into a virtual environment.
2. A runtime stage, from the same base image, copies only the virtual environment, the application code and the RDS certificate bundle. It runs as a non-root user and starts `uvicorn` on `0.0.0.0:8000`.

The image has a container health check calling `/api/health`. The same image runs the API, the migration task (`alembic upgrade head`) and scheduled jobs (`python -m apps.api.jobs.{job}`).

## Dependencies

| Purpose | Package |
|---|---|
| Web framework and server | `fastapi`, `uvicorn[standard]` |
| Database | `sqlalchemy`, `psycopg[binary]`, `alembic` |
| Settings and schemas | `pydantic`, `pydantic-settings` |
| Sign-in and token validation | `authlib` (OpenID Connect client: the authorisation code flow with PKCE and `private_key_jwt` client authentication), `pyjwt[crypto]` |
| AWS | `boto3` |
| HTTP client (discovery, directory lookup) | `httpx` |
| Development | `pytest`, `pytest-cov`, `ruff`, `pyright[nodejs]` |

Versions are pinned in `uv.lock`, which is committed. The Python version is set in `pyproject.toml` (`requires-python`) and matches the Dockerfile base image.

## Tests

`tests/conftest.py` provides:

- `db_engine`: connected to PostgreSQL, the same database engine as production
- `db_session`: each test in a transaction with a savepoint, rolled back afterwards
- `client`: a `TestClient` with the database session dependency overridden
- `issue_token(provider, roles, delegated)`: signs bearer tokens with a test key pair, which the API validates through its normal code path
- `signed_in(provider, roles)`: creates a sign-in session and returns the cookie and cross-site request forgery token

## Acceptance criteria

- The API starts in Docker Compose and `GET /api/health` returns 200 when the database is available and 503 when it is not.
- The API refuses to start in a deployed environment with an issuer that is not on HTTPS at an allowed host, without a client certificate, without TLS verification or with local database credentials.
- Signing in through `/api/auth/login` sets a `__Host-session` cookie that is `HttpOnly`, `Secure` and `SameSite=Lax`, and no token reaches the browser.
- Every endpoint except `/api/health` and `/api/auth/` returns 401 without a valid token or session, and 403 without a permitted role.
- A state-changing request with the session cookie but without the cross-site request forgery token, or from another origin, returns 403.
- An external user requesting a record they are not allowed to see receives 404.
- Correlation and request IDs are returned on every response and appear in every log line.
- Logs are valid JSON, one object per line.
- Domain exceptions produce the shared error response.
- `pytest` passes with the coverage thresholds in [99-generation-rules.md](99-generation-rules.md#tests).
