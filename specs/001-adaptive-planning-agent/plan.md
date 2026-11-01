# Implementation Plan: Adaptive Personal Planning Agent

**Branch**: `001-adaptive-planning-agent` | **Date**: 2026-11-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-adaptive-planning-agent/spec.md`

## Summary

Build a local-first, single-user CLI planning agent that captures conversational work,
tracks objectives and commitments, and proposes explainable plans for explicit human
approval. Use a small layered Python application: Typer/Rich CLI, application
orchestration, deterministic domain and planning logic, SQLite persistence, and one
OpenAI-compatible LLM advisor using the official OpenAI Python SDK and Pydantic response
validation. See [research.md](./research.md) for choices and tradeoffs.

## Technical Context

**Language/Version**: Python 3.12+

**Primary Dependencies**: Typer, Rich, OpenAI Python SDK, Pydantic, platformdirs

**Storage**: Local SQLite in the per-user data directory; TOML configuration in the
per-user config directory; API key from environment

**Testing**: pytest; Typer CLI tests; mocked LLM contract tests; SQLite migration and
export/import tests; stable briefing snapshots; Ruff and mypy

**Target Platform**: Local Python installation on Linux, macOS, and Windows

**Project Type**: Packaged CLI application with an interface-independent application
and domain core

**Performance Goals**: With 1,000 active captured items in the local database, p95
latency for capture persistence, item listing, and deterministic schedule generation
MUST be under 500 ms, excluding LLM network time. LLM calls have a 30-second timeout
and bounded retries; the CLI reports failure clearly when the service is unavailable.

**Constraints**: All planning data stays local except minimum request context sent to
the user-configured LLM. No telemetry, external-system actions, connectors, plugins,
or connector extension points in v1. The accepted plan changes only after confirmation.
The core remains usable for local operations without an LLM connection.

**Scale/Scope**: One local user and one SQLite database; personal task volumes; one
active local process; no authentication, hosted service, web UI, or external data
imports in v1.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Gate | Result |
|---|---|---|
| Advisor only | No external actions; recommendations include rationale | PASS |
| Human validation loop | Proposed plan changes are diffs and require confirmation | PASS |
| LLM-agnostic | User-supplied endpoint, key, and model; no model-specific behavior | PASS |
| Privacy by default | Local data, no telemetry, minimum LLM context, secrets outside Git | PASS |
| Deterministic core | State, capacity, deadlines, dependencies, and plan validity use tested code; validate LLM output | PASS |
| Manual input only | v1 accepts user-entered conversational input only | PASS |
| Testability | Unit tests for core; LLM is mockable | PASS |
| Simplicity | One installable CLI package; avoid premature infrastructure | PASS |
| Open-source hygiene | README, contribution guide, and MIT license | PASS |

The requested source-connector interface is intentionally excluded: even an unimplemented
connector port is a v1 extension point prohibited by Principle VI. Revisit integrations
only after a concrete need exists and the governance requirements are met. The CLI
depends on application services rather than domain or interface code, preserving a path
to a future UI without introducing a speculative connector mechanism. Task prerequisites
are persisted as item-to-item dependency links; the scheduler rejects cycles and
requires each prerequisite to be completed before scheduling its dependent.

## Project Structure

### Documentation (this feature)

```text
specs/001-adaptive-planning-agent/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── cli.md
│   └── llm.md
└── tasks.md
```

### Source Code (repository root)

```text
pyproject.toml
uv.lock
README.md
CONTRIBUTING.md
LICENSE
.pre-commit-config.yaml
.github/workflows/ci.yml
src/
└── daily_planner/
    ├── __init__.py
    ├── __main__.py
    ├── cli.py
    ├── config.py
    ├── application/
    │   ├── capture.py
    │   ├── objectives.py
    │   ├── planning.py
    │   └── export.py
    ├── domain/
    │   ├── models.py
    │   ├── rules.py
    │   └── errors.py
    ├── planning/
    │   ├── scheduler.py
    │   └── prioritization.py
    ├── advisor/
    │   ├── client.py
    │   ├── prompts.py
    │   └── schemas.py
    └── storage/
        ├── sqlite.py
        └── migrations/
tests/
├── unit/
├── integration/
├── contract/
├── snapshots/
└── performance/
```

**Structure Decision**: Use one `daily_planner` package with narrowly separated domain,
application, deterministic planning, advisor, storage, and CLI modules. Keep the
application use cases independent of Typer/Rich so a later UI can call them. Use direct,
small SQLite access and numbered SQL migrations rather than adding an ORM or repository
framework before the app needs one. Do not define connector protocols or plugin
registration.

## Complexity Tracking

No constitution violations or additional projects are proposed.

## Design Artifacts

- [Data model](./data-model.md)
- [CLI command contracts](./contracts/cli.md)
- [LLM prompt and response contracts](./contracts/llm.md)
- [Quickstart validation guide](./quickstart.md)
