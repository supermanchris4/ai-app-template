# Glossary

> **APPLICATION STUB.** Add the application's domain terms below the template entries, and delete this callout.

Every domain term used in the specifications, code and screens. One shared vocabulary prevents ambiguity in what is built.

## Rules

1. One definition per term. Other specifications link here rather than redefining it.
2. Entries are nouns in the singular.
3. A term that means something different outside the domain has a clause saying which meaning applies.
4. Abbreviations have their own entry, linked to the full term.

## Entry format

```
### {Term}

**Definition:** {one to three sentences}

**Used by:** {specifications or modules}

**Related:** {other entries}
```

## Template entries

### Actor

**Definition:** Anyone or anything that performs an operation through the API. Every actor has a subject (its identity), an actor type and, for authorisation, the app roles it holds.

**Used by:** [50-auth.md](50-auth.md), [31-audit.md](31-audit.md)

**Related:** Actor type, App role

### Actor type

**Definition:** The kind of actor: `internal` (a person signed in through the internal issuer), `external` (a person signed in through the external issuer, such as a customer or someone from another company, directly or through a consumer acting for them), `system` (another application using the client credentials flow) or `job` (a scheduled or background task run by this application). Actor types are defined once and are separate from roles.

**Used by:** [50-auth.md](50-auth.md), [31-audit.md](31-audit.md)

### App role

**Definition:** A permission defined in the application's registration with the identity provider and assigned to people or systems. It arrives in the access token's `roles` claim and is checked on every route.

**Used by:** [50-auth.md](50-auth.md), [21-api.md](21-api.md)

### Audit record

**Definition:** An append-only record of a state change, written in the same transaction as the change.

**Used by:** [31-audit.md](31-audit.md), [30-data-model.md](30-data-model.md)

### Consumer

**Definition:** Another system, or a separate front end, that calls the application's API from its own servers with its own registration. A consumer acting for an external user passes that user's own token.

**Used by:** [10-architecture.md](10-architecture.md#api-consumers), [50-auth.md](50-auth.md)

### Correlation ID

**Definition:** A UUID carried in the `X-Correlation-ID` header from the browser through the API into the logs and audit records, tying together everything done for one user action.

**Used by:** [20-api-scaffold.md](20-api-scaffold.md), [31-audit.md](31-audit.md)

### Issuer

**Definition:** An OpenID Connect identity provider the API trusts, identified by its discovery document. The application has an internal issuer (Entra ID by default) and, if it has external users, an external issuer (Entra External ID by default).

**Used by:** [50-auth.md](50-auth.md), [20-api-scaffold.md](20-api-scaffold.md#authentication-and-authorisation)

### Sign-in session

**Definition:** The record the API keeps after signing a person in to the web interface, identified by the session cookie. The browser holds only the cookie; the API holds the identity and roles.

**Used by:** [50-auth.md](50-auth.md), [20-api-scaffold.md](20-api-scaffold.md), [30-data-model.md](30-data-model.md#template-tables)

### Rule configuration version

**Definition:** A numbered, immutable version of the thresholds and parameters a business rule uses. Each stored outcome records the version that produced it.

**Used by:** [22-domain-logic.md](22-domain-logic.md), [99-generation-rules.md](99-generation-rules.md)
