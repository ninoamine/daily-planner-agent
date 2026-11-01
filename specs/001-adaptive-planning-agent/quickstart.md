# Quickstart and Validation Guide

This guide defines the end-to-end checks for the planned CLI. Command names and
arguments follow [the CLI contract](./contracts/cli.md); records follow the
[data model](./data-model.md); advisor payloads follow the
[LLM contract](./contracts/llm.md).

## Prerequisites

- Python 3.12 or newer supported by the project
- uv for development setup
- No API key is required to run tests or exercise local-only capture, browsing, and
  storage behavior
- A user-selected OpenAI-compatible Chat Completions endpoint is required only to
  validate live LLM-assisted operations

## Install and Run Quality Checks

From the repository root:

```sh
uv sync --group dev
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy src
```

Install the CLI locally with either `uv tool install .` or `pipx install .`. The
installed command is `daily-planner`.

## Scenario: Capture Work Without Losing the Source

```sh
daily-planner init
daily-planner capture "Architecture asked me for a review by Thursday"
daily-planner items list --all
```

Expected outcomes:

- The original natural-language entry is retained.
- The extracted request appears with its requester and interpreted date, or an explicit
  unknown/clarification state if the date cannot be safely interpreted.
- Listing the work does not require an LLM connection; LLM failure is reported without
  losing the captured entry.

## Scenario: Produce and Confirm a Daily Briefing

Configure `base_url` and `model` in the user's TOML file and set
`DAILY_PLANNER_LLM_API_KEY` in the environment. Do not put a real secret in shell
history, an example file, or source control.

```sh
daily-planner objectives add
daily-planner capture "Prepare the next program increment objective"
daily-planner brief --date 2026-11-02 --available 360 --protect 60
```

Copy the ID printed by `brief`, then run:

```sh
PLAN_ID="paste-proposal-id-here"
daily-planner plan show "$PLAN_ID"
daily-planner plan accept "$PLAN_ID"
```

Expected outcomes:

- The proposal fits the supplied time budget and reports any infeasible constraint.
- Every recommended item has a concise rationale, and stale objectives and at-risk
  commitments are surfaced.
- The proposal remains unaccepted until `plan accept` or an explicit interactive
  confirmation.
- Reordering/editing requires a new explicit confirmation; rejection leaves the last
  accepted plan unchanged.

## Scenario: Re-plan an Interruption Safely

```sh
daily-planner capture "Production incident: investigate service Y"
daily-planner items list
```

Copy the incident item ID from `items list` and the proposal ID from `replan`:

```sh
ITEM_ID="paste-incident-item-id-here"
daily-planner replan --item "$ITEM_ID"
PLAN_ID="paste-proposal-id-here"
daily-planner plan show "$PLAN_ID"
daily-planner plan reject "$PLAN_ID"
ACCEPTED_PLAN_ID="paste-previously-accepted-plan-id-here"
daily-planner plan show "$ACCEPTED_PLAN_ID"
```

Expected outcomes:

- The re-plan shows a diff, including postponed work and consequences for objectives or
  commitments.
- Rejecting it retains the previous accepted plan and the new incident.
- Accepting the same kind of proposal in a separate test run updates the accepted plan
  only after confirmation and increments deferral counts only for confirmed deferrals.

## Scenario: Offline Data Safety and Portability

With the LLM endpoint unavailable or unset:

```sh
daily-planner capture "Follow up with architecture next week"
daily-planner items list
daily-planner export --format json --output ./planning-export.json
daily-planner export --format markdown --output ./planning-export.md
daily-planner backup --output ./planning-backup.sqlite
```

Expected outcomes:

- Capture and local review remain available without a network connection.
- JSON and Markdown exports are readable and do not contain the API key or prompts.
- The database backup can be opened as a consistent SQLite snapshot.
- Completed and archived items remain searchable unless explicitly deleted.

## Automated Test Coverage Expectations

- Unit tests cover deadline interpretation boundaries, objective inactivity, at-risk
  working-day thresholds, dependency ordering/cycle rejection, capacity, protected
  time, and deferral counts.
- SQLite integration tests start from a temporary database, apply each migration, and
  verify export/restore or backup behavior without touching user data.
- CLI tests assert command arguments, output/exit codes, confirmation requirements, and
  non-TTY behavior using Typer's test runner.
- Mocked LLM contract tests cover extraction, valid/invalid JSON, unknown IDs, timeout,
  retryable failures, non-retryable failures, and preservation of the accepted plan.
- Briefing snapshots use fixed dates, stable ordering, and color-free output; focused
  behavior assertions remain the source of truth.
- With 1,000 active captured items and no LLM network call, p95 latency for capture
  persistence, item listing, and deterministic schedule generation remains below
  500 ms.
- v1 exposes no connector/plugin command or registration interface, sends no telemetry,
  and makes no external-system requests other than the user-configured LLM request.
- CI runs without live credentials and checks the supported Python versions.
