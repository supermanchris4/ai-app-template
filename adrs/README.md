# Decision records

This application's own architecture decision records (ADRs), including any deviation from the AI application standard. Decisions that apply to every application are recorded in the `ai-app-standard` repository, not here.

Write each record from [0000-adr-template.md](0000-adr-template.md). A departure from the standard or from a template specification is recorded here before the change is made.

## Statuses

Each ADR and each sub-decision has a status and, separately, the date it took that status.

| Status | Meaning |
|---|---|
| Draft | Still being written |
| Proposed | Ready for its decider to review. Nothing is built on it yet. |
| Interim | The position the work proceeds on until its decider confirms or changes it. If the decider changes it, the ADR is revised in place and its date becomes the day the changed position was adopted. |
| Decided | Confirmed by its decider. It is not edited except to mark it deprecated or superseded, and a changed decision gets a new ADR. |
| Rejected | Declined by its decider while it was Proposed or Interim, so it is not a valid position. It is kept as the record of what was considered. |
| Deprecated | A decided ADR that no longer applies, with nothing replacing it |
| Superseded | A decided ADR whose decision has changed. It names the ADR that replaces it, and the new ADR names it with "Supersedes". |

A decision depends on its sub-decisions, so it is Decided only once each of them is Decided, Rejected or Deprecated.
