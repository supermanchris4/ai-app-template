# Application rules

The rules every application built from the template follows, whatever its domain. They state what must be true rather than how it is built; the specification named in a rule, or the template conventions, say how. Each rule's source is in the AI application standard's design (`docs/design.md`, "Application rules"). An application that departs from a rule records an ADR first.

[22-domain-logic.md](22-domain-logic.md) holds each application's own rules and refers to these by ID rather than restating them.

## Where rules run and how they are configured

| ID | Rule |
|---|---|
| <a id="ar-01"></a>AR-01 | The system enforces a business rule on every path that writes the data, never only in a screen. |
| <a id="ar-02"></a>AR-02 | A value the business may change is reference data or versioned configuration, never hardcoded in code, a view or a screen. |
| <a id="ar-03"></a>AR-03 | Each rule, definition and calculation is implemented once and used by every path, screen, report, export, notification, job and domain that needs it, so the same data always gives the same answer. Where domains or variants genuinely differ, the difference is stated as configuration or as its own rule, not as a copy. |
| <a id="ar-04"></a>AR-04 | Every configuration value has a declared type and, where one applies, a stated default. A missing, unrecognised or wrongly typed value is refused when loaded or stops the work that needs it, and is never replaced silently by an empty value, a zero or a fallback. |
| <a id="ar-05"></a>AR-05 | A query, validation, preview, report or export never changes stored data or the thing being checked. Data changes only through an explicit operation. |
| <a id="ar-06"></a>AR-06 | An operation changes only the fields it names. An operation that changes nothing writes nothing, and adds no audit record or modification stamp. |
| <a id="ar-07"></a>AR-07 | An operation's result depends only on its own inputs and stored data. Nothing carries over from an earlier request, session, item or row, and reprocessed data is handled with the context it arrived with. |
| <a id="ar-08"></a>AR-08 | No function is built that nothing reaches. A capability a legacy system holds only as unreachable, disabled or never-working code is specified as a requirement in its own right where the business needs it, or retired. |

## Identity and access

| ID | Rule |
|---|---|
| <a id="ar-09"></a>AR-09 | Identity and credentials belong to the identity provider. The application holds no password and creates no database account per person, so it has no rule on password expiry, change, policy or reset, sign-in lockout or unlocking. Whether a person has access, and their app roles, are assigned in the identity provider ([50-auth.md](50-auth.md)). |
| <a id="ar-10"></a>AR-10 | Access is denied unless a check positively grants it. An unknown role, an unresolved scope or record origin, or a check that cannot be evaluated denies the action, and the denial is reported and logged. Every admission check passes before any session is issued. |
| <a id="ar-11"></a>AR-11 | A role is a named set of permissions and grants only what it names. Whether a person may act depends on whether their permissions include the action, never on a rank, the order of roles or the exclusion of one role, and a read permission never confers a write. |
| <a id="ar-12"></a>AR-12 | The server authorises every operation against the actor's permissions and their right over every record it touches, including both source and destination and every item of a bulk action. One decision applies per entity and action, the same for create, change and delete on every path. What a screen shows or enables is never the control, and records the system generates cannot be created by a person. |
| <a id="ar-13"></a>AR-13 | Every person, system and job acts under its own identity with only the privileges its work needs. Nothing is granted to everyone, ordinary work never uses a shared or privileged account, and no routine that runs with raised privileges can be reached by callers who lack them. |
| <a id="ar-14"></a>AR-14 | A secret is held only in the secrets store and read at run time. It is never returned to a client, written to a log or an application table, built into code, or derivable from another value, and a machine identity uses a managed credential ([12-aws-deployment.md](12-aws-deployment.md#secrets)). |
| <a id="ar-15"></a>AR-15 | Withdrawing a person's access ends it within a stated time, including sessions already open, and a change to a person's data scope applies from their next request. The time is set by Standard ADR-0004. |

## Records, keys and integrity

| ID | Rule |
|---|---|
| <a id="ar-16"></a>AR-16 | A record is identified by a stable key, never by its display text or a legacy code. |
| <a id="ar-17"></a>AR-17 | Every reference to another record is held as a key to that record, enforced by the database. |
| <a id="ar-18"></a>AR-18 | A uniqueness, required-value, permitted-value or no-overlap rule on stored data is declared as a database constraint as well as checked by the service, so a concurrent write cannot break it. A uniqueness test excludes the record being edited, empty values are treated as the rule states, and a breach is refused with a business reason rather than a database error. |
| <a id="ar-19"></a>AR-19 | A value, status, indicator, count or membership derived from other data is derived by one rule from the current source whenever that changes, or kept in step with it in the same operation. No screen sets it directly. |
| <a id="ar-20"></a>AR-20 | A business operation that writes several records succeeds or fails as a whole, in one transaction or with a stated compensating step where one transaction cannot cover the work. A failed operation undoes only its own work and leaves stored data as it was. |
| <a id="ar-21"></a>AR-21 | When two people change the same record, a change made from an out-of-date copy of it is refused and reported. It never silently overwrites the other person's change. |
| <a id="ar-22"></a>AR-22 | Deleting a business record is a soft delete. The record is marked deleted and kept with its history and its links to other records, and the deletion is audited. |
| <a id="ar-23"></a>AR-23 | A record that others depend on is never removed. Deleting it marks it inactive, a soft delete under AR-22: it stays for the records that refer to it and cannot be chosen for new ones. |
| <a id="ar-24"></a>AR-24 | A rule that picks one record (the latest, the earliest or the next) uses an explicit order with a tie-break, and any row limit applies after ordering. No behaviour depends on the order the database happens to return rows in or on a list position, and a lookup that can match several records handles each or treats the ambiguity as an error. |
| <a id="ar-25"></a>AR-25 | A hierarchy of records admits no cycle, and a parent is of the type the hierarchy allows. Both are checked on every path that sets a parent. |
| <a id="ar-26"></a>AR-26 | A fact owned by another corporate system of record is maintained only there. The application receives the facts its rules need from that system, read-only and with the dates they take effect, and holds no copy it maintains. Identity belongs to the identity provider (AR-09) and physical asset data to the corporate asset system. |
| <a id="ar-27"></a>AR-27 | The database schema is defined only by versioned migrations in source control, and only the migration step changes it ([30-data-model.md](30-data-model.md#conventions)). |

## Values and types

| ID | Rule |
|---|---|
| <a id="ar-28"></a>AR-28 | Each value is held in its own data type: a number as a number, a date or time as a date or time, a yes-or-no value as a boolean and a coded value as a key to reference data. No value is held as text that has to be parsed, and no fact is encoded in another value's text, such as a prefix or suffix. |
| <a id="ar-29"></a>AR-29 | A value's type, length and format are set by what the value is: its business definition, a standard, or the system or regulator that defines it. A legacy system's column and field widths are a guide and not the decision. |
| <a id="ar-30"></a>AR-30 | A query or command never contains input as text. Every value is passed as a bound parameter, so no value can change the statement and no character needs to be banned. |
| <a id="ar-31"></a>AR-31 | A value is stored as given or refused with the reason, and is never truncated, replaced or discarded without telling the user. Rounding and formatting are applied when a value is shown, compared or reported and never written back over it, and a display that shortens text says that it does. |
| <a id="ar-32"></a>AR-32 | Input is validated against what is permitted. The whole value is tested, a value is trimmed before a required-value test, formats such as email addresses, telephone numbers and dates are checked, and an unrecognised code or type is refused. Every failure is reported against its field in one response ([21-api.md](21-api.md#validation)). |
| <a id="ar-33"></a>AR-33 | Values are compared, sorted and calculated in their own type, never through their text, and no result depends on a session's language, date format or number format. A value becomes text only to be shown or exchanged, in a stated format such as ISO 8601. |
| <a id="ar-34"></a>AR-34 | A comparison, filter, join or change test treats an absent value as its rule states. A change to or from an absent value is a change, and an absent value never makes a test pass or fail, or drops a row, by accident. |
| <a id="ar-35"></a>AR-35 | A quantity is held and shown with its unit, and converted between units only through one maintained conversion for each pair of units. A calculation brings every input to one unit before combining or comparing them. |

## Time and periods

| ID | Rule |
|---|---|
| <a id="ar-36"></a>AR-36 | A value that changes over time is held with the date it takes effect, its periods do not overlap, and it changes by closing one period and opening the next rather than by overwriting. An outcome uses the value in force at its own date. The links that identify such a record are fixed once it exists, so changing them ends the record and starts a new one. |
| <a id="ar-37"></a>AR-37 | A period of calendar dates includes the whole of its first and last day, judged in the business time zone. A period ends on or after its start, and an open-ended period has no end date rather than a far-future one and is in force from its start onwards. One overlap test decides on create and on edit. |
| <a id="ar-38"></a>AR-38 | A period over date and time includes its start instant and excludes its end instant, so consecutive periods such as days, months and report windows partition time and each event falls in exactly one. A period stated as calendar dates follows AR-37. |
| <a id="ar-39"></a>AR-39 | Business days are counted against one maintained calendar of working days and public holidays in the business time zone, used by every rule and report that counts business days. The calendar is reference data, so its content is the business's to set. |

## Audit and evidence

| ID | Rule |
|---|---|
| <a id="ar-40"></a>AR-40 | The server records who acted and when on every record and audit entry, from the authenticated session and the server clock in UTC. These are never typed, taken from a workstation or written as fixed text. Acting for another person is recorded as such, a business date a person states is held apart from the record's own timestamps, and data captured by one person and loaded by another records both people ([31-audit.md](31-audit.md#audit-record-fields)). |
| <a id="ar-41"></a>AR-41 | Every change to business data, reference data and access is audited with its before and after state, including setting or clearing a value and changes to a record's notes. Audit records, approvals and other records kept as evidence are append-only, so a correction is a new record and no change or deletion loses the history ([31-audit.md](31-audit.md#immutability)). |
| <a id="ar-42"></a>AR-42 | An outcome records what decided it, such as the threshold applied and the configuration version, and later displays and notifications read that record rather than looking the inputs up again. |
| <a id="ar-43"></a>AR-43 | Each recorded value states how it arose (observed, entered, loaded from a named source or derived) in its own attribute, and a derived value identifies the rule version and inputs that produced it. |
| <a id="ar-44"></a>AR-44 | Every business record has a retention period set by the applicable records disposal authority. A record is disposed of only at the end of its period, by an explicit operation logged with what it removed, and none accumulates without a period ([30-data-model.md](30-data-model.md#data-classification)). |

## Calculations and figures

| ID | Rule |
|---|---|
| <a id="ar-45"></a>AR-45 | A failure is reported and recorded. It is never presented as success, an empty result, a zero or a placeholder value, and a value that is absent stays absent. An outcome that cannot be determined, because the data it needs is absent or invalid, is reported as not assessed and never as a pass. A recorded zero is a value, distinct from an absent one. An operation that fails part-way reports what it did and did not do. |
| <a id="ar-46"></a>AR-46 | Each calculation states its valid inputs and its result for every edge case, such as a zero divisor, an absent or non-numeric operand or a physically impossible result. An edge case gives that stated result or an exception, and never stops unrelated work or produces a figure presented as valid. |
| <a id="ar-47"></a>AR-47 | A formula a person defines is held in a defined expression language, with its operands as references to records. It is checked against the language and its references when saved, and evaluated only by an evaluator limited to that language, never by the database or a general interpreter. |
| <a id="ar-48"></a>AR-48 | Each calculated figure states its precision and rounding method, and rounding is applied once, to the final figure. The method and precision of each figure are configuration, confirmed with the business. |
| <a id="ar-49"></a>AR-49 | Values from different sources are matched by key, such as date and location, never by position or label. Every expected period appears in its own place and is shown as empty where nothing was recorded. |
| <a id="ar-50"></a>AR-50 | A count or total states its grain and counts each item exactly once at that grain. Combining data never multiplies it. |
| <a id="ar-51"></a>AR-51 | Each figure states whether it can be added across periods, places or records. Only an additive figure, such as a total or a count, is summed. An average, minimum, maximum, latest value, target or percentage is combined by its own rule from the underlying values and is never added. |
| <a id="ar-52"></a>AR-52 | Each summary statistic, such as a mean, median, minimum, maximum or percentile, is computed by its standard definition over the values its rule names, and the output names the statistic and the period it covers. An absent value, or one that is not a number, is never counted in a statistic and never stands in for a value. |

## Jobs and integration

| ID | Rule |
|---|---|
| <a id="ar-53"></a>AR-53 | Diagnostic output goes to the application log, never into business data or a business transaction, so it cannot fail or roll back a change. Detailed diagnostics are off by default and never hold personal information or secrets ([20-api-scaffold.md](20-api-scaffold.md#logging)). |
| <a id="ar-54"></a>AR-54 | An operation that may be repeated, by a retry, a re-run or a second request, has the effect of doing it once. It never adds a second record, action or notification for the same event, and within one operation each effect is recorded once. |
| <a id="ar-55"></a>AR-55 | A scheduled or background job, or any run over several items that a person starts, treats each item as its own unit of work. A failed item is recorded and reported and the others proceed, and a run finds outstanding work by its state and resumes from a checkpoint that advances only when the work is done, never by a fixed look-back window. |
| <a id="ar-56"></a>AR-56 | Every scheduled job and interface run keeps a run record: when it started and ended, what it took in, how many items it handled and each item's outcome. The people who support the process can query it, not only read it in a transient log. |
| <a id="ar-57"></a>AR-57 | Each external interface has a documented, versioned contract that states its format, fields, permitted values and how each record identifies what it refers to. Inbound data is checked against the contract, and each record is accepted, or rejected with its reason and held for correction. |
| <a id="ar-58"></a>AR-58 | A retry is limited in number and spaced out, and reaching the limit is reported as a failure. No operation is retried without limit. |
| <a id="ar-59"></a>AR-59 | Data exchanged with another system travels over an authenticated, encrypted channel under the system's own managed identity. |
| <a id="ar-60"></a>AR-60 | Outbound data is kept until its consumer acknowledges it, and a receiver handles a repeated delivery once. |
| <a id="ar-61"></a>AR-61 | Development, test and production are separate. No tool writes business data across environments, and configuration moves between them only through controlled deployment. |
| <a id="ar-62"></a>AR-62 | An inbound file is reconciled against its control totals, so the records received equal those accepted, rejected and failed. The reconciliation is recorded, and a file is recorded as received whether or not it is accepted. |
| <a id="ar-63"></a>AR-63 | An inbound interface changes only the data its contract says it supplies. Where its data disagrees with master data, the disagreement is raised for resolution rather than applied. |
| <a id="ar-64"></a>AR-64 | Every field that a publication path can send outside the organisation is classified as public or internal. Only fields classified public are sent, an unclassified field is treated as internal, and no rule that picks or combines values lets an internal value take the place of a public one. |

## Screens

| ID | Rule |
|---|---|
| <a id="ar-65"></a>AR-65 | Every option, filter and criterion offered takes effect, and every message states the rule actually applied. A result covers the whole scope and period requested or says what it leaves out, and an action is offered only where it would act. |
| <a id="ar-66"></a>AR-66 | An action that cannot be undone states what it will do, including the dependent records it affects, and asks for confirmation. Declining, or closing the confirmation by any other means, changes nothing. Leaving a screen with unsaved changes asks whether to discard them ([40-ui.md](40-ui.md#template-conventions)). |
