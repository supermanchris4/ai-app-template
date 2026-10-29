# Generation rules

The rules Claude must follow when generating code in this repository. They override any other guidance. Claude refuses work that would break them and says which rule applies.

## Working rules

### Must

- Read the relevant specifications before generating code.
- Implement only the change asked for. Do not regenerate unaffected code or refactor unrelated areas.
- Update the specification that describes any behaviour changed, in the same change. If a change reveals a gap in a specification, raise it rather than filling it silently.
- Flag unknowns. If a specification is missing or ambiguous, ask rather than assume.
- List and justify any new library before adding it. Pin it in the lockfile.
- Add every new setting to the settings declaration ([20-api-scaffold.md](20-api-scaffold.md#settings)) and, if it is used locally, to `infra/local/.env.example`.

### Must not

- Generate domain code where the relevant specification is still a **APPLICATION STUB**. Ask for the specification to be written first.
- Resolve a decision the specifications leave open, or depart from a template specification, without an ADR in [adrs/](../adrs/).
- Create generic "update arbitrary fields" endpoints. Every change is an explicit operation.
- Add infrastructure the baseline does not include (caches, message brokers, tracing stacks, extra databases, reverse proxies) without an ADR.

## Business rules and validation

- **All business logic is in the API.** Rules, calculations, state changes and outcomes are worked out by the API services. The web interface and other clients only collect input and show results.
- **Outcomes are calculated and stored by the API.** When a rule produces a result (a status, a score, a compliance outcome, a flag), the API calculates it and stores it with the record, together with the version of the rule configuration used. Clients never calculate it themselves, and reports read the stored value.
- **Thresholds are configuration, not code.** Numeric limits, time windows and other rule parameters are held in versioned configuration (a configuration table or file under version control), never as literals in code. Changing a threshold creates a new version and leaves earlier outcomes traceable to the version that produced them.
- **Every request is validated, including value ranges.** Request schemas declare types, required fields, lengths, formats and the permitted minimum and maximum for every numeric and date value. Cross-field and cross-record rules are checked in the service layer and raise domain exceptions.
- **The web interface may repeat validation for usability, never instead of the API.**
- **Times are UTC inside the application.** The API, database, logs and audit records hold timestamps in UTC, and only the web interface converts them for display. A rule that depends on the local day, such as "by the end of the day", uses `APP_BUSINESS_TIME_ZONE`, never the server's or the browser's time zone.
- **External users reach only the records the application allows them.** Every operation an external user may call applies the rule in [30-data-model.md](30-data-model.md#record-access-for-external-users) in the service layer, and returns 404 for any other record.
- **Features are API operations first.** A screen for external users is only a client of operations other consumers can also call.

## Secrets and data

- Secrets are never written to `.env` files, source code, specifications, Terraform code or variables, test fixtures, logs or pipeline variables. They are held in AWS Secrets Manager and read at run time ([12-aws-deployment.md](12-aws-deployment.md#secrets)).
- `.env.example` holds only non-secret settings and throwaway values for local containers.
- Logs never include personal information, including names, email addresses and subject identifiers, or tokens, passwords or connection strings. A log line is tied to a request by its correlation and request IDs, and who acted is found through the audit trail.
- Error responses never include personal information other than what the caller submitted, or internal detail.
- Personal information may be held in audit records, because they sit in the same database as the records they describe, under the same access control, encryption and retention. It is never copied from there to logs, metrics, exports or any other store without the same controls.
- Only synthetic data enters the repository, including tests, fixtures, seed scripts, examples and screenshots. Synthetic data must not be derived from real records. See [30-data-model.md](30-data-model.md#data-classification).
- If a credential or real data is found in the repository, stop and report the file path and the type of finding, never the value.

## Python (API)

### Code style

| Rule | Value |
|---|---|
| Formatter and linter | Ruff, configured in `pyproject.toml` |
| Type checker | Pyright in strict mode, configured in `pyproject.toml` (Standard D-02) |
| Indentation | 4 spaces |
| Encoding and line endings | UTF-8, LF |
| Type hints | Required on every function signature |
| Docstrings | Required on modules (except `__init__.py`) and public functions, in Google style |

### Structure

| Rule | Description |
|---|---|
| Import order | Standard library, third-party, local, separated by blank lines |
| Constants | After imports, before definitions, in `UPPER_SNAKE_CASE` |
| Executable code | Guarded with `if __name__ == "__main__":` |
| Circular imports | Prohibited. Restructure, or use `TYPE_CHECKING` imports |
| Star imports | Prohibited except in `__init__.py` to expose a public API |
| Function length | Prefer under 50 lines; refactor beyond 100 |

### Naming

| Component | Convention | Example |
|---|---|---|
| Classes | `PascalCase` | `InspectionService` |
| Functions and methods | `snake_case` | `get_by_id` |
| Private members | Leading underscore | `_validate_window` |
| Service file | `{name}_service.py` | `inspection_service.py` |
| Repository file | `{entity}_repository.py` | `inspection_repository.py` |
| Route and schema files | `{resource}.py` | `inspections.py` |
| Domain enums and exceptions | `domain/enums.py`, `domain/exceptions.py` | |

### Layers

| Layer | Rules |
|---|---|
| Routes | Handle HTTP, check roles with `require_role`, own the transaction (commit on success, roll back on exception) |
| Services | Classes with repositories injected through the constructor. Receive the database session as the first parameter. Hold business logic. Do not import FastAPI, Pydantic or anything HTTP-specific. Raise domain exceptions. Record audit events for every auditable operation. Never commit. |
| Repositories | One per aggregate root. Session injected through the constructor. Hold all query logic and return ORM models. Never commit. |
| Domain | Plain Python classes, dataclasses, enums and exceptions. No FastAPI, Pydantic or HTTP concepts. Exceptions inherit from one base domain exception. |
| Schemas | Pydantic request and response models, including value-range validation |

### Tests

| Aspect | Rule |
|---|---|
| Framework | pytest, with FastAPI `TestClient` for API tests |
| Location | `apps/api/tests/unit/` and `apps/api/tests/integration/` |
| Database | PostgreSQL (never SQLite), each test in a transaction rolled back afterwards |
| Authentication | Tokens signed by the test issuer, validated by the same code as production |
| Fixtures | pytest fixtures and factory functions producing synthetic data |
| Coverage | At least 80% for services and 70% overall, enforced in CI |
| Naming | `test_{module}.py`, with one test function per behaviour |

## TypeScript (web interface)

| Rule | Value |
|---|---|
| Compiler | TypeScript in strict mode |
| Formatter and linter | Prettier and ESLint |
| API access | Only through the API module, using the client generated from the OpenAPI document. No direct `fetch` to the API elsewhere. |
| Sign-in | Through the API's `/api/auth/` endpoints only. No sign-in library, and no token handling or storage, in the browser. |
| Components | Extend the kit in `components/ui/`. Do not add a component library that duplicates it. |
| Role checks | Decide only what is shown. The API enforces access. |
| Tests | Vitest for components, Playwright for end-to-end journeys, automated accessibility checks on every page |

## Terraform

| Rule | Value |
|---|---|
| Layout | One file per concern in `infra/aws/` (for example `network.tf`, `cloudfront.tf`, `ecs.tf`, `rds.tf`) |
| Inputs | Every variable has a description, a type and validation. Network inputs have no defaults. |
| Tags | Applied through the provider's `default_tags`, with the tags in [12-aws-deployment.md](12-aws-deployment.md#principles) |
| Secrets | No secret value is generated by Terraform or held in its state |
| Checks | `terraform fmt` and `terraform validate` pass before commit |
