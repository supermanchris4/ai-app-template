# Domain logic

> **APPLICATION STUB.** Replace this callout and the guidance below with the application's business rules before any domain code is generated. Keep the template rules section.

The behaviour, rules and calculations the API enforces. [21-api.md](21-api.md) describes the shape of the API; this file describes what it means.

## Template rules

These come from [99-generation-rules.md](99-generation-rules.md) and always apply:

1. All business logic is in the API services. The web interface and other clients never apply a business rule themselves.
2. Outcomes are calculated by the API and stored with the record, with the rule configuration version that produced them.
3. Thresholds and rule parameters are versioned configuration, never literals in code.
4. Services raise domain exceptions, do not commit transactions and record an audit event for every auditable operation.

## Sections to complete

1. **Entities and their lifecycle.** States, permitted transitions and what triggers each, and invariants.
2. **Rules and calculations.** For each rule: its inputs, the calculation, its configurable parameters, the outcome stored and the requirement it implements.
3. **Rule configuration.** Where each rule's parameters are held, who may change them (which role) and how versions are kept.
4. **Validation beyond the request schema.** Rules spanning fields or records.
5. **Concurrency.** How simultaneous changes are handled (for example optimistic locking with a version column).
6. **Scheduled and background work.** What runs on a schedule, what triggers it, what it guarantees if it fails part-way, and its job name ([10-architecture.md](10-architecture.md#background-processing)).
7. **Reason codes.** The application's reason codes, extending the starter set in [31-audit.md](31-audit.md#reason-codes).

## Entity block

```
### {Entity}

**States:** {list}

**Transitions:**
{from} -> {to}: {trigger}, roles {role values}

**Invariants:**
- {rule}

**Audited actions:** created, {domain actions}

**Service:** {entity}_service.py
```

## Rule block

```
### {Rule name}

**Inputs:** {fields}
**Calculation:** {plain description or formula}
**Parameters:** {name}: {meaning}, current value {value}, range {min to max}
**Outcome stored:** {field}, with the configuration version
**Requirement:** {FR ID}
```
