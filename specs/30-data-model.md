# Data model

The tables, columns, constraints and indexes. The SQLAlchemy models and Alembic migrations must match this file.

## Data classification

> **APPLICATION STUB.** Complete this section before any table holding business data is created.

| Item | Application value |
|---|---|
| Highest classification of data held | {for example Public, Internal, Confidential} |
| Personal information held | {categories, or "none"} |
| Data owner | {business role} |
| Retention and disposal | {period and method, by record type} |

Per table, record the classification and whether the table holds personal information in the table block below.

**Rules, for every application:**

- **Only synthetic data enters the repository.** This includes tests, fixtures, seed scripts, schema migrations, examples, documentation and screenshots. Synthetic data is invented, not derived from, anonymised from or sampled from real records.
- Real data is held only in deployed databases, and copied between environments only as the data owner approves.
- Personal information is collected only where a requirement needs it.
- Personal information is never written to logs ([99-generation-rules.md](99-generation-rules.md#secrets-and-data)). It may be held in audit records, which sit in the same database with the same controls ([31-audit.md](31-audit.md#personal-information-in-audit-records)).
- Corrections and disposal of personal information follow the legislation and policies the application team identifies in [02-requirements.md](02-requirements.md#regulatory-and-compliance-requirements), using the reason codes in [31-audit.md](31-audit.md#reason-codes).

## Conventions

| Concern | Convention |
|---|---|
| Primary keys | `UUID`, default `gen_random_uuid()` |
| Timestamps | `TIMESTAMPTZ`, stored in UTC. Every table has `created_at` and `updated_at`, both set by the server when the row is created, default `now()`. `updated_at` is set again on every change, so a row never changed has `updated_at` equal to `created_at` ([NFR-DAT-01](02-requirements.md#nfr-dat-01-change-stamps)). |
| Calendar dates | `DATE` for a date with no time of day, such as a due date. Never stored as a timestamp at midnight. |
| Soft delete | `deleted_at TIMESTAMPTZ NULL`. Audited records are not hard-deleted. |
| Documents | `JSONB` only |
| Enumerations | `VARCHAR` with a `CHECK` constraint matching the Python `StrEnum` values |
| Concurrency | A `version INTEGER` column for optimistic locking where records are edited concurrently |
| History | Records whose full history is needed also keep a revision table (`{table}_revisions`) holding each prior state |
| Rule configuration | Versioned rows; an outcome column is stored with a reference to the configuration version |
| Naming | Tables plural `snake_case`; columns `snake_case`; constraints `ck_{table}_{rule}`, `uq_{table}_{columns}`, `fk_{table}_{column}`; indexes `ix_{table}_{columns}` |
| Migrations | One Alembic revision per logical change. Shipped migrations are never edited or squashed. |
| Privileges | Migrations run as the migrator user and grant the runtime user `SELECT`, `INSERT`, `UPDATE` and `DELETE` on application tables, and only `INSERT` and `SELECT` on `audit_records` |

## Template tables

The template provides two tables, defined by the API platform package. The application's first migration creates them by calling `create_platform_tables(entity_types=[...])` from `ai_app_platform.migrations`, with the entity types it audits, then `add_platform_change_stamps()`, which gives `audit_records` its `updated_at` ([NFR-DAT-01](02-requirements.md#nfr-dat-01-change-stamps)); an audit record is never changed, so the two stay equal. Its models attach the same entity-type check to the platform's model. Their core columns are not changed.

**`audit_records`** holds the audit trail, with the fields in [31-audit.md](31-audit.md#audit-record-fields). Applications extend the permitted `entity_type` and `action` values.

**`auth_sessions`** holds web sign-in sessions ([50-auth.md](50-auth.md#web-sign-in-the-api-as-backend-for-frontend)):

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | UUID | No | Primary key |
| `token_hash` | BYTEA | No | SHA-256 of the cookie value; unique |
| `csrf_token` | VARCHAR(64) | No | Returned by `GET /api/auth/session` |
| `provider` | VARCHAR | No | `internal` or `external`. Version 0.1 of the package permits `internal` only (the standard's work plan item F03.5) |
| `subject` | VARCHAR | No | The claim named by `AUTH_SUBJECT_CLAIM` |
| `actor_type` | VARCHAR | No | `internal` or `external` |
| `display_name` | VARCHAR | Yes | |
| `roles` | JSONB | No | Roles at sign-in |
| `created_at` | TIMESTAMPTZ | No | |
| `updated_at` | TIMESTAMPTZ | No | |
| `last_used_at` | TIMESTAMPTZ | No | |
| `expires_at` | TIMESTAMPTZ | No | Sign-in time plus `SESSION_MAX_HOURS` |

It holds personal information (the subject and display name), under the same controls as the rest of the database. A scheduled job deletes expired rows.

## Record access for external users

> **APPLICATION STUB.** If external users can call the API, define which records they can see and how the application decides, and record it as requirements in [02-requirements.md](02-requirements.md). Delete this section if the application has no external users.

App roles set which operations an external user may call. They do not set which records the user can see, so each application with external users defines that here. The API enforces it in the service layer ([50-auth.md](50-auth.md#authorisation)).

## Tables

> **APPLICATION STUB.** Document every table before it is created.

Document related tables together as a group, and break a group that is still too large to read into sub-groups. Each group or sub-group opens with its own Mermaid entity relationship diagram of its tables, laid out with ELK, followed by a table of its relationships to tables in other groups. There is never one diagram of the whole model, and a diagram shows only documented tables. Entity diagrams belong here rather than in the glossary, where each entry states its relationships in words.

```
### {table_name}

**Requirements:** {requirement IDs}
**Purpose:** {one line}
**Classification:** {classification}; personal information: {yes or no}

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| id | UUID | No | gen_random_uuid() | Primary key |
| created_at | TIMESTAMPTZ | No | now() | |
| updated_at | TIMESTAMPTZ | No | now() | |

**Constraints:** {list}
**Indexes:** {list}
**Relationships:** {column} references {table}.{column}
```

## Enumerations

Every `StrEnum` in `apps/api/domain/enums.py` is listed here with its values. A change to the values updates the `CHECK` constraint in the same migration.
