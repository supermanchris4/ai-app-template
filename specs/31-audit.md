# Audit

What is audited, what each audit record holds and how corrections are recorded.

## What is audited

- **Every state change to a business record:** creation, change, soft deletion, restoration and each domain transition. The application team lists the audited actions per entity in [22-domain-logic.md](22-domain-logic.md).
- **Every correction to recorded data** by a person, with a mandatory reason code and reason text.
- **Every change to rule configuration and reference data**, with the before and after state.
- **Every action by another system or a scheduled job**, with its actor type.

Each audit record is written by the service in the same transaction as the change it records, so a change and its audit record succeed or fail together. The API platform package provides the `audit_records` model and `AuditService`, whose `record` method the services call ([10-architecture.md](10-architecture.md#platform-packages)); the application lists the entity types and actions it audits.

## Audit record fields

| Field | Description | Required |
|---|---|---|
| `id` | Audit record ID | Yes |
| `entity_type` | Type of record changed | Yes |
| `entity_id` | ID of the record changed | Yes |
| `action` | Action performed | Yes |
| `actor_id` | Subject of the actor (the person's subject claim from their issuer, the system's client ID, or the job name) | Yes |
| `actor_type` | `internal`, `external`, `system` or `job`. Version 0.1 of the package records `internal`, `system` and `job` (`external` comes with the standard's work plan item F03.5) | Yes |
| `occurred_at` | When the change was committed (UTC) | Yes |
| `correlation_id` | Correlation ID of the request or job run | Yes |
| `reason_code` | Reason code | For corrections |
| `reason_text` | Free-text reason | For corrections |
| `before_state` | Record state before the change (JSON) | For changes and corrections |
| `after_state` | Record state after the change (JSON) | For creations, changes and corrections |
| `metadata` | Other context (JSON) | No |

`before_state` and `after_state` hold business fields only, never tokens or secrets.

## Personal information in audit records

Audit records may hold personal information: the actor's subject identifier, and personal information in `before_state` and `after_state` where the audited record holds it. This is permitted because `audit_records` is in the same database as the records it describes, and so has the same access control, encryption and backups. The same controls apply to any personal information it holds:

- Audit records are read only through the API, by the roles granted in [21-api.md](21-api.md).
- Audit records are not copied to logs, a BI tool or any other store unless that store has the same controls and the data owner approves.
- When personal information is disposed of under `privacy_disposal`, the application's retention schedule sets how the audit records holding it are handled.

## Reason codes

The starter set below applies to every application. The application team adds its own in [22-domain-logic.md](22-domain-logic.md).

| Code | Use |
|---|---|
| `data_entry_error` | Correcting a value entered wrongly |
| `duplicate_record` | Removing a record that duplicates another |
| `source_system_correction` | Applying a correction made in the source system |
| `privacy_correction` | Correcting personal information at the request of the person it is about, under the privacy legislation the application team identifies |
| `privacy_disposal` | Removing personal information that is no longer needed, under that law and the retention schedule |
| `other` | Any other reason; the reason text must explain it |

## Immutability

Audit records are append-only. The runtime database user can insert and read `audit_records` but cannot update or delete them ([30-data-model.md](30-data-model.md#conventions)).

## Queries

Audit records can be queried by entity type and ID, actor, action, date range and correlation ID. The application team defines the audit endpoints and the roles that may read them in [21-api.md](21-api.md).

## Retention

Audit records are kept for the period the application's retention schedule sets for the records they describe, recorded in [30-data-model.md](30-data-model.md#data-classification).
