# Overview

> **APPLICATION STUB.** Replace this callout and the guidance below with the application's overview before any domain code is generated.

A one-to-two page narrative a new team member could repeat back. It reads like a product brief rather than a schema; detail belongs in [21-api.md](21-api.md), [22-domain-logic.md](22-domain-logic.md) and [30-data-model.md](30-data-model.md).

Answer these questions:

1. **What is this application?** One paragraph.
2. **Who uses it?** The people (by role) and the other systems that call its API, and what each needs from it.
3. **What problem does it solve?** The business need or the system it replaces.
4. **What are the core concepts?** Three to six domain terms that recur in the specifications, defined in [01-glossary.md](01-glossary.md).
5. **What does it not do?** Explicit non-goals, including any reporting or data supply that sits outside the application (Standard ADR-0017).
6. **How does it fit with other systems?** Upstream and downstream systems, and what data flows each way.
7. **What data does it hold?** The highest data classification it handles (see [30-data-model.md](30-data-model.md#data-classification)).

Update this file when the framing of the application changes, not when implementation details change.
