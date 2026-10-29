# Authentication and authorisation

How internal users, external users and other systems sign in and what they may do. The implementation is in [20-api-scaffold.md](20-api-scaffold.md#authentication-and-authorisation). The model follows Standard D-03, D-04, D-10 and D-18.

## Identity providers

The API trusts one or two OpenID Connect issuers, each configured by its discovery URL ([20-api-scaffold.md](20-api-scaffold.md#settings)). Entra ID is the default internal issuer and Entra External ID the default external issuer (Standard D-04). An application with no external users configures only the internal issuer.

| Who | Issuer | How they reach the API |
|---|---|---|
| Internal users | Internal | The web interface, through a session the API holds |
| External users on the application's own screens | External | The web interface, through a session the API holds |
| External users through a consumer | External | The consumer calls the API with the user's own access token |
| Other systems, including consumers for calls not about one user | Internal or external, as the application owner sets up | Bearer access tokens from the client credentials flow |

Developers' machines use the local mock identity provider in place of both ([11-local-development.md](11-local-development.md#local-mock-identity-provider)). The API signs in and validates through the same code.

## Web sign-in: the API as backend-for-frontend

The web interface never handles tokens. The API signs people in and holds the result in a server-side session, and the browser holds only a session cookie that scripts cannot read. This follows the backend-for-frontend pattern that RFC 10017, *OAuth 2.0 for Browser-Based Applications*, ranks most secure.

1. The web interface sends the person to `GET /api/auth/login`, naming the identity provider (`internal` or `external`) and the page to return to.
2. The API redirects to the provider with the authorisation code flow and PKCE, as a confidential client.
3. The provider redirects back to `GET /api/auth/callback`. The API exchanges the code, authenticating with its client certificate, and validates the ID token.
4. The API creates a session row holding the subject, actor type, provider, display name and roles, and sets the session cookie.
5. The web interface calls `GET /api/auth/session` to learn who is signed in and which roles they hold, then calls the API with the cookie.
6. `POST /api/auth/logout` deletes the session and redirects to the provider's sign-out.

**The session cookie** is named `__Host-session`, and is `HttpOnly`, `Secure`, `SameSite=Lax` and `Path=/`. It holds only a random session ID; the database stores a hash of it.

**Session lifetime.** A session ends after `SESSION_IDLE_MINUTES` without use (default 30) or `SESSION_MAX_HOURS` after sign-in (default 10), whichever comes first. Roles are read at sign-in, so a change to someone's roles takes effect at their next sign-in. Deleting a session row ends it at once.

**Cross-site request forgery.** Every state-changing request (`POST`, `PUT`, `PATCH`, `DELETE`) made with the session cookie must carry the `X-CSRF-Token` header with the session's token, returned by `GET /api/auth/session`, and an `Origin` header matching the application's address. The API refuses the request otherwise.

**Tokens.** The API keeps no access or refresh tokens after sign-in, because it calls no downstream API on the person's behalf. If an application later needs to (for example the on-behalf-of flow), it stores the refresh token encrypted in the session row, through an ADR.

## Bearer tokens for systems and consumers

Other systems and consumers call the API with `Authorization: Bearer` access tokens.

- The API accepts only access tokens issued for its own registration by the configured internal or external provider, validated for signature, issuer, audience and expiry on every request.
- A token with the `scp` claim is delegated. It must include `access_as_user`. From the internal provider, its actor type is `internal`; from the external provider, `external`.
- A token without `scp` is app-only, from the client credentials flow. It must carry at least one app role, and its actor type is `system`.
- A consumer calls the API from its own servers, never from users' browsers. Acting for an external user, it passes that user's own token for anything about them, and its own app-only token only for calls not about one user, such as reference data.

## App registrations

The application owner creates these per environment, following the [runbook](../docs/runbook.md#identity-provider-registrations).

| Registration | Tenant | Purpose |
|---|---|---|
| The application | Internal | Confidential web client with a certificate, redirect URI `{address}/api/auth/callback`. Exposes the `access_as_user` scope and defines the app roles. Its client ID is the token audience. With Entra ID, it issues v2.0 access tokens (`requestedAccessTokenVersion` set to `2` in the manifest), so their issuer matches the v2.0 discovery document. |
| The application | External, if it has external users | As above, with the external issuer and the external app roles |
| Each consumer | The issuer its users or service belong to | Its own registration, granted the application's scope or app roles |

The client certificate's private key is held in Secrets Manager.

## App roles

Permissions come from the app roles a person or system holds: from the `roles` claim at web sign-in, or in a bearer token. They are defined in the application's registrations and assigned to people (directly or through groups) and to systems by the application owner. The API reads no other source of roles.

The application team defines the application's roles in the table below, following these rules:

- **Internal, external and system roles are separate.** An internal role is defined in the internal registration for people (in Entra ID, the `User` member type). An external role is defined in the external registration. A system role is assignable to applications only (in Entra ID, the `Application` member type), and its value starts with `system.`.
- **Values follow `{area}.{action}`**, for example `inspection.read`, `account.view` or `system.inspection.write`.
- **There is no hierarchy.** A role grants only what it names. Each route lists every role it permits, so who can do what is visible on the route.
- **Every route except `/api/health` and the `/api/auth/` endpoints requires at least one role.** Signing in is not enough on its own.

### Application roles

> **APPLICATION STUB.** Replace the example rows with the application's roles. Keep the rules above.

| Role value | Holder | Grants |
|---|---|---|
| `example.read` | Internal | Read the example records |
| `example.write` | Internal | Create and change the example records |
| `account.view` | External | Read their own example records |
| `system.example.read` | System | Read the example records from another system |

The role values are declared once in `apps/api/auth/roles.py` and must match the registrations exactly.

## Authorisation

- The API enforces every permission. A route uses `require_role(...)` with the roles that may call it.
- **External users reach only the records the application allows them.** Each application defines which records those are in [30-data-model.md](30-data-model.md#record-access-for-external-users). Every operation an external user can call applies that rule in the service layer and raises `EntityNotFoundError` for any other record, so the response does not reveal that the record exists.
- Other checks that depend on the record (for example "only the record's owner may change it") are made in the service layer and raise an authorisation error.
- The web interface uses the roles returned by `GET /api/auth/session` only to decide what to show. Hiding a button is never the control.

## Directory access

Where an application needs to look people up, it does so only through the API's optional directory endpoint ([20-api-scaffold.md](20-api-scaffold.md#directory-lookup)). The web interface never calls the directory.
