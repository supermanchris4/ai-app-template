# Requirements

> **APPLICATION STUB.** Replace this callout and the guidance below with the application's requirements before any domain code is generated. Keep the requirements under [Met by the template](#met-by-the-template), and replace the [Default targets](#default-targets) with agreed targets.

The single source of functional and non-functional requirements. Other specifications refer to requirements by ID rather than restating them.

## Sections

1. Scope and out of scope
2. Actors: the people (by role) and systems that use the application
3. User journeys
4. Functional requirements (`FR-*`), with one subsection per functional area
5. Data requirements, including the data classification
6. Integration requirements
7. Reporting and data supply requirements, placing each need as Standard ADR-0017 sets out: operational screens in the application, defined exports or integrations for data supplied to other organisations and systems, and any separate BI tool only through an ADR
8. Regulatory and compliance requirements (`CMP-*`)
9. Non-functional requirements (`NFR-*`)

## Requirement IDs

| Prefix | Category |
|---|---|
| `FR-{AREA}-{NN}` | Functional, for example `FR-INSP-01` |
| `CMP-{NN}` | Regulatory and compliance |
| `NFR-PERF-{NN}` | Performance |
| `NFR-AVL-{NN}` | Availability and recovery |
| `NFR-SCL-{NN}` | Scale |
| `NFR-SEC-{NN}` | Security |
| `NFR-AUD-{NN}` | Auditability |
| `NFR-ACC-{NN}` | Accessibility |
| `NFR-OBS-{NN}` | Observability |

Every requirement has a unique, stable ID, can be tested on its own, and names the specifications that implement it once they are written.

## Functional areas

Functional requirements are grouped by functional area: a business capability the application provides, such as recording inspections, managing schedules or administering reference data. Each area has a short code, used as `{AREA}` in its requirement IDs, and its own heading under Functional requirements.

List the areas in a table before writing their requirements:

| Code | Area | Covers | Main actors |
|---|---|---|---|
| `INSP` | {for example Inspections} | {what is in and out of this area} | {roles} |

Rules for areas:

- **Group by what the business does, not by screen, release or team.** Screens and delivery plans change; capabilities rarely do, so IDs stay stable.
- **Each requirement belongs to exactly one area.** A requirement that seems to span two usually needs splitting, or belongs to the area that owns the data it changes.
- **Aim for areas of roughly 5 to 25 requirements.** Split a larger area into narrower capabilities; merge areas with only one or two requirements.
- **Order requirements within an area by the user journey,** from first to last step, then exceptions and administration.
- **IDs are never reused or renumbered.** A withdrawn requirement keeps its ID and is marked withdrawn. A new requirement takes the next number in its area, wherever it sits in the order.
- **Administration and reference data form their own area** (for example `ADM`), so the core business areas stay focused.

When this file holds more than about 50 functional requirements, move each area to its own file, `specs/requirements/{code}.md`, and keep the area table and the other sections here. Requirement IDs do not change when they move.

## Regulatory and compliance requirements

The template does not determine which legislation, regulations or organisational policies apply to an application. The application team identifies them, with the relevant business owner and specialists such as privacy, records management and legal, and records each obligation as a `CMP-*` requirement that is implemented and tested like any other.

Typical areas to check include privacy and personal information, records retention and disposal, industry and environmental regulation, financial controls, and information security classification. For each obligation, record the source, what the application must do, and how compliance is shown (for example an audit record, a report or a retention job).

## Entry format

Every requirement, whatever its ID prefix, uses this format:

```
### {ID}: {short title}

**Statement:** {one sentence, present tense}

**Source:** {CMP only: the legislation, regulation or policy, and the clause}

**Priority:** {Must, Should, Could or Won't (this release)}

**Rationale:** {the business or technical reason}

**Acceptance:** {observable behaviour or measurable outcome}

**Status:** {Proposed, Agreed or Validated, with who and when for Agreed and Validated}

**Implemented by:** {specification references, added as work progresses}
```

For example `### FR-INSP-01: Record an inspection`, `### NFR-PERF-01: API response time` or `### CMP-03: Retain inspection records`. Non-functional acceptance states a measurable target, such as "95% of API requests complete within 500 ms". Compliance acceptance states how compliance is shown, such as an audit record, a report or a retention job.

The status records how far the business has confirmed the requirement. Code does not wait for it.

| Status | Meaning |
|---|---|
| `Proposed` | Written, and not yet agreed by the business |
| `Agreed` | The business has agreed the requirement before using what implements it |
| `Validated` | The business has confirmed the requirement after using what implements it, at a demonstration or in acceptance testing |

Agreement comes before validation because a requirement can read correctly and still turn out, once it is used, to mean something other than what the business intended. A requirement that use shows to be wrong is rewritten, and it is then agreed or validated again. The requirements under [Met by the template](#met-by-the-template) are set by the technology standards rather than by the business, so they have no status.

## Non-functional requirements

### Default targets

Replace each default with a target agreed with the business owner before deployment to `test`. The defaults are placeholders for sizing, not commitments.

### NFR-PERF-01: API response time

**Statement:** API requests complete quickly enough for interactive use.

**Priority:** Must

**Rationale:** Template default.

**Acceptance:** 95% of API requests complete within 500 ms, measured at the load balancer in `prod`.

**Status:** Proposed

**Implemented by:** [12-aws-deployment.md](12-aws-deployment.md#monitoring) (alarm on target response time)

### NFR-AVL-01: Service availability

**Statement:** The service is available during operating hours.

**Priority:** Must

**Rationale:** Template default.

**Acceptance:** 99.5% availability during operating hours, measured monthly in `prod`.

**Status:** Proposed

**Implemented by:** [12-aws-deployment.md](12-aws-deployment.md#inputs) (`api_desired_count` and `db_multi_az`), [12-aws-deployment.md](12-aws-deployment.md#compute)

### NFR-AVL-02: Recovery point

**Statement:** Data lost after a failure is limited to a set period.

**Priority:** Must

**Rationale:** Template default.

**Acceptance:** At most 1 hour of data is lost in `prod`, and at most 24 hours in `dev`.

**Status:** Proposed

**Implemented by:** [12-aws-deployment.md](12-aws-deployment.md#database)

### NFR-AVL-03: Recovery time

**Statement:** Service is restored within a set time after a failure.

**Priority:** Must

**Rationale:** Template default.

**Acceptance:** Service is restored within 4 hours in `prod`, and within 1 business day in `dev`.

**Status:** Proposed

**Implemented by:** [12-aws-deployment.md](12-aws-deployment.md#database)

### NFR-SCL-01: Users and record volumes

**Statement:** The application supports its expected number of concurrent users and records.

**Priority:** Must

**Rationale:** Template default. It sets the default sizing.

**Acceptance:** Tens to low hundreds of concurrent users. The application team sets the record volumes.

**Status:** Proposed

**Implemented by:** [12-aws-deployment.md](12-aws-deployment.md#inputs) (`api_cpu`, `api_memory`, `db_instance_class` and `db_allocated_storage`)

### Met by the template

Keep these requirements. The template meets them, and an application that changes how one is met records an ADR first. Application requirements in the same categories continue the numbering.

### NFR-SEC-01: Authenticated and authorised access

**Statement:** Every endpoint except `/api/health` and `/api/auth/` requires a valid sign-in session or access token from a configured issuer, and a permitted app role.

**Priority:** Must

**Rationale:** Only identified callers with a role that allows it reach the application's data.

**Acceptance:** A request without a valid session or token returns 401, and one without a permitted role returns 403.

**Implemented by:** [50-auth.md](50-auth.md), [20-api-scaffold.md](20-api-scaffold.md#authentication-and-authorisation)

### NFR-SEC-02: No tokens in the browser

**Statement:** No access, ID or refresh token reaches the browser.

**Priority:** Must

**Rationale:** A token held in the browser can be taken by a script running in the page.

**Acceptance:** The browser holds only the HttpOnly session cookie.

**Implemented by:** [50-auth.md](50-auth.md#web-sign-in-the-api-as-backend-for-frontend)

### NFR-SEC-03: Encryption in transit

**Statement:** Traffic is encrypted with TLS 1.2 or later from the browser to CloudFront, from CloudFront to the load balancer, and from the API to the database.

**Priority:** Must

**Rationale:** Protects data and credentials on every network path.

**Acceptance:** Each connection refuses unencrypted traffic and TLS versions earlier than 1.2.

**Implemented by:** [12-aws-deployment.md](12-aws-deployment.md#edge), [12-aws-deployment.md](12-aws-deployment.md#database)

### NFR-SEC-04: Encryption at rest

**Statement:** Data is encrypted at rest in RDS, S3 and Secrets Manager.

**Priority:** Must

**Rationale:** Protects data held in storage.

**Acceptance:** Encryption is on for the database, the buckets and the secrets in every environment.

**Implemented by:** [12-aws-deployment.md](12-aws-deployment.md#database), [12-aws-deployment.md](12-aws-deployment.md#secrets)

### NFR-AUD-01: Audit trail

**Statement:** Audit records are append-only and can be queried by record, actor, action, date and correlation ID.

**Priority:** Must

**Rationale:** Shows who changed what, and when.

**Acceptance:** The runtime database user cannot update or delete audit records, and the audit queries return records by each of these fields.

**Implemented by:** [31-audit.md](31-audit.md#immutability), [31-audit.md](31-audit.md#queries)

### NFR-OBS-01: Logging and alarms

**Statement:** The API writes structured JSON logs with correlation IDs, and alarms cover AWS's recommended ECS metrics.

**Priority:** Must

**Rationale:** Support staff can trace a request and learn of a failure before users report it.

**Acceptance:** Every log line is a JSON object with a correlation ID, and each alarm in the monitoring set sends to the alarm topic.

**Implemented by:** [20-api-scaffold.md](20-api-scaffold.md#logging), [12-aws-deployment.md](12-aws-deployment.md#monitoring)

### NFR-AVL-04: Backups and restore

**Statement:** The database is backed up in every environment, and the restore is tested before go-live.

**Priority:** Must

**Rationale:** Makes the recovery point and recovery time achievable.

**Acceptance:** Backups are kept for `db_backup_retention_days`, and a restore has been completed in `test` before go-live.

**Implemented by:** [12-aws-deployment.md](12-aws-deployment.md#database)

### NFR-ACC-01: Accessibility

**Statement:** The web interface meets the accessibility level in the standard's configuration defaults (WCAG 2.2 level AA unless the application records another).

**Priority:** Must

**Rationale:** Set by the standard's configuration defaults.

**Acceptance:** Every page passes the automated accessibility checks in the pull request checks.

**Implemented by:** [40-ui.md](40-ui.md#template-conventions), [13-delivery-pipeline.md](13-delivery-pipeline.md#pull-request-checks)
