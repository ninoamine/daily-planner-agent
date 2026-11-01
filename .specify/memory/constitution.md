<!--
Sync Impact Report
Version change: none -> 1.0.0
Modified principles: none
Added principles:
- Advisor Only
- Human Validation Loop
- LLM-Agnostic
- Privacy by Default
- Deterministic Core, LLM for Judgment
- Manual Input Only
- Testability
- Simplicity
- Open-Source Hygiene
Added sections: Product Scope; Development Workflow
Removed sections: none
Follow-up TODOs: Confirm the original ratification date.
-->
# Daily Planner Agent Constitution

## Core Principles

### I. Advisor Only
The agent MUST NOT perform actions in external systems. It may propose and explain actions,
but the human decides whether to act. Every priority recommendation MUST include a short
rationale that the user can accept, edit, or reject.

### II. Human Validation Loop
The agent MUST NOT silently reprioritize or change a plan. It MUST present every proposed
plan change as a diff and wait for the user's confirmation before applying it.

### III. LLM-Agnostic
Users MUST be able to provide the LLM endpoint URL, API key, and model of their choice.
The project MUST NOT hardcode a provider or rely on behavior specific to one model.

### IV. Privacy by Default
All planning data MUST be stored locally, and the application MUST NOT send telemetry.
Secrets MUST be supplied through environment variables or a local configuration file
excluded from version control. The agent MUST send only the minimum context necessary
for an LLM request.

### V. Deterministic Core, LLM for Judgment
Plain code MUST compute task state, deadlines, deferral counts, time budgets, and plan
validity; these computations MUST be covered by tests. The LLM MAY interpret free text,
reason about priorities, explain recommendations, and challenge the user's plan. LLM
output MUST be validated against a schema before it is used.

### VI. Manual Input Only
Version 1 MUST accept conversational input from the user and no other data source. It
MUST NOT include external-system integrations, plugins, or connectors. The project MUST
NOT build extension points until a concrete need exists.

### VII. Testability
Core logic MUST have unit tests, and LLM calls MUST be mockable so tests can run without
calling an external model.

### VIII. Simplicity
The project MUST deliver the smallest solution that works, avoid premature infrastructure
and abstractions, and remain installable with one command.

### IX. Open-Source Hygiene
The repository MUST include a clear README, a license, and a contribution guide. It MUST
NOT contain company-specific content.

## Product Scope

The project is an open-source, single-user personal planning agent. The CLI is the
initial interface; a UI may be added later without changing the single-user, advisor-only
role or the validation requirements in this constitution.

## Development Workflow

Changes MUST preserve the deterministic and human-confirmed planning behavior defined
above. Tests MUST cover core calculations, and tests involving LLM requests MUST be able
to use mocks. Changes that consume LLM output MUST validate its schema before use.
Contributors and reviewers MUST check proposed changes against this constitution.

## Governance

An amendment MUST document its rationale and impact, including any change to existing
requirements. Project maintainers MUST review and approve amendments before adoption.
An adopted amendment MUST update this document's version and last-amended date.

Constitution versions follow Semantic Versioning: MAJOR for incompatible changes to
existing principles or governance, MINOR for new principles or materially expanded
requirements, and PATCH for clarifications and non-semantic corrections. The ratification
date records the original adoption and does not change on amendment.

Contributors and reviewers MUST verify compliance with this constitution during review.
Any exception MUST be explicitly documented and approved as an amendment before the
affected change is adopted.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): Original adoption date is unknown. | **Last Amended**: 2026-11-01
