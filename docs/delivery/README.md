# Delivery plan

> **APPLICATION STUB.** Replace this callout and the guidance below with the application's delivery plan before the first release is specified. Keep the [standard tasks](#standard-tasks), extending them if the application needs more.

The order in which the application is built and what each release covers. The specifications in [specs/](../../specs/) say what is built; this plan says when.

## Structure

The plan follows a work-breakdown hierarchy of Outcome, Epic, Feature, Story and Task:

| Level | What it is | Duration | Where it is described |
|---|---|---|---|
| Outcome | The end state the business seeks: why, the hypothesis, benefits and measurable results | Years | This file |
| Epic (business or enabler) | A high-level requirement: pitch, benefits, measurable results, acceptance criteria and dependencies | 2 to 3 quarters | One folder per Epic, with its `README.md` |
| Feature (business or enabler) | A business or functional requirement: benefits, acceptance criteria and dependencies | Under a quarter | A file in its Epic's folder once it is near; a summary in the Epic file until then |
| User Story or Enabler | A low-level requirement and its acceptance criteria | Under a sprint | In its Feature file |
| Task | One step of a story, such as one of the [standard tasks](#standard-tasks) | Hours to days | In its Feature file |

Detail follows distance:

| Horizon | Work | Detail held |
|---|---|---|
| Now | The release in progress and the next | A Feature file with Stories and Tasks |
| Next | The release after that | A Feature summary with outline acceptance criteria, in the Epic file |
| Later | Everything beyond | One line per Feature, in the Epic file |

A Feature moves up a horizon when the release before it is reviewed.

**IDs and files.** Plan IDs are Epic `E01`, Feature `F01.1` and Story `S01.1.1`. This directory is the record of the plan. Items may also be raised as GitHub issues, with the issue number recorded in the tracker. An Epic's folder is `eNN-{slug}/`, holding `README.md` and one `fNN-N-{slug}.md` per detailed Feature.

**Status.** `tracker.md` in this directory holds one row for every Epic, Feature and Story, with its status and its GitHub issue where it has one. It is the only place status is recorded; the plan files describe the work and are not updated as it progresses. Tasks are tracked with their Story, which is Done when all its Tasks are.

## Contents of this file

1. **Context.** What the application is, and where its requirements come from.
2. **Outcome.** The Outcome the application serves, with the fields in the hierarchy table.
3. **Epics.** Each with its type, the functional areas it builds and a one-line pitch, linked to its folder.
4. **Releases and implementation sequence.** Which Features each release groups, and a Mermaid diagram of the order, without dates or durations.
5. **Standard tasks.** The table below.
6. **Dependencies.** What the application needs from other teams, and which Feature needs it.
7. **Decisions needed.** Each with its question, decider and the Feature or release that needs it. An architecture decision, once made, is recorded as an ADR in [adrs/](../../adrs/).
8. **Risks.** Each with its mitigation.

Update the plan when a release is planned or when feedback on a release changes what follows. Update the tracker whenever an item's status changes.

## Releases

- **Each release takes Features from several Epics.** A capability is built out over several releases, so feedback on an early release can still change it.
- **Each release runs and is accepted locally.** AWS environments join when their dependencies are in place and do not hold up a release.
- **A release is accepted** when its pull request checks pass and its business reviewers have used it. Once `dev` exists, the release is also deployed there and passes its health check. The reviewers confirm or change each requirement the release implements: a confirmed requirement becomes `Validated` ([requirement status](../../specs/02-requirements.md#entry-format)), and feedback updates the requirements and specifications, then the plan for the next release.

## Standard tasks

Every business Story is delivered through these Tasks, in order. A Feature file lists only what is specific to each Story.

| Task | Layer | Delivers | Done when |
|---|---|---|---|
| T1 | Specifications | Requirements, rules, entities, tables, endpoints, pages and roles in the application stubs | Reviewed before any code |
| T2 | Database | Schema migrations for the tables, constraints and indexes in [30-data-model.md](../../specs/30-data-model.md), the models, seed data and synthetic data | Schema migrations apply cleanly from empty and from the previous release; constraint tests pass against PostgreSQL |
| T3 | Domain and services | Domain models, enums and exceptions; repositories; services holding the rules, recording audit and storing outcomes with their configuration version | Unit tests per rule, with the coverage thresholds in [99-generation-rules.md](../../specs/99-generation-rules.md#tests) |
| T4 | API | Request and response schemas with ranges, routes with roles, and the regenerated OpenAPI document | Integration tests per endpoint, including 401 and 403; the contract check passes |
| T5 | Web interface | Pages built on the regenerated client, as [40-ui.md](../../specs/40-ui.md) documents them | Component tests, end-to-end tests of the Story's journey and the accessibility checks pass |
| T6 | Scheduled jobs | Job entry points, where the Story has any | Each job runs on demand locally as the `job` actor |

Each test names the requirement it verifies.

## Epic file

```
# E{NN} {Epic name}

Part of the [delivery plan](../README.md). {Business or enabler} Epic: {one sentence}.

## Pitch

{What the Epic delivers in the application first, then how any system it replaces does it today}

## Benefits
## Measurable results
## Acceptance criteria
## Dependencies
## Features

| Feature | Type | Release | Detail |
|---|---|---|---|
| F{NN}.1 {name} | {Business or Enabler} | {release} | {link to the Feature file, or a summary} |
```

## Feature file

```
# F{NN}.{N} {Feature name}

Part of [E{NN} {Epic name}](README.md) in the [delivery plan](../README.md). {Business or enabler} Feature: {one sentence}.

## Benefits
## Acceptance criteria
## Dependencies
## Stories

### S{NN}.{N}.{N} {Story name}

As a {role}, I want {capability}, so that {benefit}.

**Acceptance criteria:** {observable behaviour}

Tasks: {standard tasks that apply, and anything specific to this Story}
```

## Tracker file

```
# Tracker

The status of every Epic, Feature and Story in the [delivery plan](README.md), and each item's GitHub issue where it has one.

| Status | Meaning |
|---|---|
| Not started | No work has begun |
| In progress | Work has begun |
| Blocked | Work cannot continue until what the note names is in place |
| Done | The acceptance criteria are met |

| ID | Level | Title | Parent | Release | Status | Issue | Note |
|---|---|---|---|---|---|---|---|
| [E01](e01-{slug}/README.md) | Epic | {Epic name} | | | Not started | | |
| [F01.1](e01-{slug}/f01-1-{slug}.md) | Feature | {Feature name} | E01 | {release} | Not started | | |
| [S01.1.1](e01-{slug}/f01-1-{slug}.md#s0111-{story-name}) | Story | {Story name} | F01.1 | {release} | Not started | | |
```

Each ID links to where its item is described. A Feature without its own file links to its summary in the Epic file, or to the Epic's Features table.
