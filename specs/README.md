# Specifications

The source of truth for what is built. The AI coding assistant reads these before writing code, and every change in behaviour updates the relevant specification in the same change.

In these specifications, **the application** is what is built from this template, together with its repository, and **the application team** is the people who build and run it.

There are two kinds of specification:

- **Template specifications** set the architecture and conventions every application shares. An application extends them but does not remove their rules. An application that needs to depart from one records an ADR in [adrs/](../adrs/) first.
- **Application stubs** are marked **APPLICATION STUB**. The application team fills them in with the application's domain before any domain code is generated.

References in the form "Standard D-04" point to decision D-04 in the AI application standard (the `ai-app-standard` repository), which gives the reasons. Once a decision is confirmed it is recorded there as the ADR with the same number (ADR-0004).

## Numbering

The first digit of a specification's number is its group, and the second is its order within the group. A new specification takes the next free number in the group it belongs to. Groups 6 to 8 are free for areas the template does not cover, such as integrations. [99-generation-rules.md](99-generation-rules.md) is always last.

| Group | Covers |
|---|---|
| 0 | The application: overview, terms and requirements |
| 1 | Architecture and platform |
| 2 | API and business logic |
| 3 | Data and audit |
| 4 | Web interface |
| 5 | Security |
| 99 | Generation rules |

## Specifications

| Specification | Kind | Contents |
|---|---|---|
| [00-overview.md](00-overview.md) | Application stub | What the application is, who uses it and what it does not do |
| [01-glossary.md](01-glossary.md) | Application stub | Domain terms |
| [02-requirements.md](02-requirements.md) | Application stub, with requirements met by the template | Functional and non-functional requirements, by ID |
| [10-architecture.md](10-architecture.md) | Template | Technology baseline, layering, API governance and observability |
| [11-local-development.md](11-local-development.md) | Template | Docker Compose, the local mock token issuer and local settings |
| [12-aws-deployment.md](12-aws-deployment.md) | Template | AWS infrastructure in Terraform |
| [13-delivery-pipeline.md](13-delivery-pipeline.md) | Template | GitHub Actions workflows, environments and deployment |
| [20-api-scaffold.md](20-api-scaffold.md) | Template | API code structure, settings, authentication, middleware and tests |
| [21-api.md](21-api.md) | Application stub with template conventions | API endpoints, on the shared conventions |
| [22-domain-logic.md](22-domain-logic.md) | Application stub | Business rules, state, calculations and reason codes |
| [30-data-model.md](30-data-model.md) | Application stub with template conventions | Tables, conventions and data classification |
| [31-audit.md](31-audit.md) | Template | Audit trail rules |
| [40-ui.md](40-ui.md) | Application stub with template conventions | Web interface pages, on the shared conventions |
| [50-auth.md](50-auth.md) | Template, with application roles | Internal and external sign-in, app roles and authorisation |
| [99-generation-rules.md](99-generation-rules.md) | Template | Rules for all code generation (authoritative) |
