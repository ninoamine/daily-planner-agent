---
description: "Implementation tasks for Adaptive Personal Planning Agent"
---

# Tasks: Adaptive Personal Planning Agent

**Input**: Design documents from `/specs/001-adaptive-planning-agent/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Organization**: Tasks are grouped by user story so each story can be implemented and
validated as an increment. Tests are included because the project constitution requires
tested deterministic core logic and mockable LLM calls.

## Phase 1: Setup

**Purpose**: Establish the installable package and shared development quality gates.

- [ ] T001 Create the `daily_planner` package skeleton and `tests/unit`, `tests/integration`, `tests/contract`, `tests/snapshots`, and `tests/performance` directories according to `specs/001-adaptive-planning-agent/plan.md`
- [ ] T002 Configure Python `>=3.12`, runtime/dev dependencies, console script `daily-planner`, pytest, Ruff, and mypy in `pyproject.toml`
- [ ] T003 [P] Add pinned development and application dependency resolution in `uv.lock`
- [ ] T004 [P] Add Ruff, mypy, and test quality checks for Python 3.12+ in `.github/workflows/ci.yml`
- [ ] T005 [P] Configure the same Ruff, mypy, and pytest checks as local hooks in `.pre-commit-config.yaml`
- [ ] T006 [P] Add the MIT license text to `LICENSE`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Implement shared domain types, local configuration, SQLite lifecycle, and error handling before story work.

- [ ] T007 Define shared domain enums, typed IDs, timestamp helpers, and application errors in `src/daily_planner/domain/models.py` and `src/daily_planner/domain/errors.py`
- [ ] T008 Implement TOML/environment/CLI configuration precedence, platform-specific config and data paths, and secret masking in `src/daily_planner/config.py`
- [ ] T009 Implement SQLite connection setup, transaction helpers, `PRAGMA user_version` migration runner, and foreign-key enforcement in `src/daily_planner/storage/sqlite.py`
- [ ] T010 Create the source-capture-only table in `src/daily_planner/storage/migrations/001_source_captures.sql`, with stable unique ID, required immutable original text, and required creation timestamp; do not create captured-item or dependency tables in this migration
- [ ] T011 Add foundational unit tests for configuration precedence, secret masking, migration ordering, rollback, and SQLite foreign keys in `tests/unit/test_config.py` and `tests/integration/test_storage.py`
- [ ] T012 Create CLI application entry point, Typer root app, Rich/plain-output setup, and shared error-to-exit-code mapping in `src/daily_planner/__main__.py` and `src/daily_planner/cli.py`

**Checkpoint**: Package, configuration, local DB initialization, migration runner, and CLI shell are available; user stories can proceed.

---

## Phase 3: User Story 1 - Capture and recover work (Priority: P1) — MVP

**Goal**: Capture manual conversational input without loss, extract useful details, clarify only when necessary, and review/search all retained work.

**Independent Test**: Capture an entry with clear details and one with ambiguous details; verify original text persists, extraction is reviewable, unanswered items remain visible, and completed/archived records remain searchable until explicit deletion.

### Tests for User Story 1

- [ ] T013 [P] [US1] Test source-capture required text and timestamp, captured-item required/optional fields, enum values, positive estimates, status transitions, self-dependency rejection, and cycle rejection in `tests/unit/test_capture_models.py`
- [ ] T014 [P] [US1] Test capture CLI invocation, item listing/search, status filters, dependency edit options, and non-interactive output in `tests/integration/test_capture_cli.py`
- [ ] T015 [P] [US1] Test capture extraction request/response validation, ambiguous dates, unknown objective IDs, and missing clarification fields with a mocked LLM in `tests/contract/test_capture_advisor.py`

### Implementation for User Story 1

- [ ] T016 [US1] Add the captured-items table with required `source_capture_id` foreign key and unique directional item-dependency table in `src/daily_planner/storage/migrations/002_captured_items.sql`; enforce `kind` in `task, request, incident, note`, `status` in `inbox, actionable, in_progress, waiting, done, archived`, nullable requester/deadline/estimate, nonnegative `defer_count`, distinct existing dependency endpoints, unique dependency pairs, and no cascading dependency deletion
- [ ] T017 [P] [US1] Define Source Capture and Captured Item validation/state transitions in `src/daily_planner/domain/models.py`, preserving required `original_text` and `source_capture_id`, requiring `title` once identified, allowing optional description/requester/deadline/estimate/clarification, requiring positive `estimate_minutes` when present, and rejecting self-links or dependency cycles
- [ ] T018 [US1] Implement local capture repository operations for inserting an original source before extraction, listing/searching all retained states, editing and validating dependency links, completing, archiving, restoring, and explicitly deleting in `src/daily_planner/storage/captured_items.py`; when deleting a prerequisite, return dependent items for confirmation rather than silently cascading links
- [ ] T019 [P] [US1] Define strict Pydantic capture-extraction request/response schemas and extraction prompt in `src/daily_planner/advisor/schemas.py` and `src/daily_planner/advisor/prompts.py`, with `kind` limited to `task, request, incident, note`, estimates positive or null, supplied objective IDs only, and unknown facts null
- [ ] T020 [US1] Implement the configurable OpenAI-compatible client boundary, finite 30-second timeout, at most two retries for transient errors, and explicit non-retryable error mapping in `src/daily_planner/advisor/client.py`
- [ ] T021 [US1] Implement capture use case that saves source text first, optionally extracts details, retains the item on LLM failure, and asks only essential clarification questions in `src/daily_planner/application/capture.py`
- [ ] T022 [US1] Add `capture`, `items list`, `items show`, `items edit --blocked-by/--remove-blocker`, `items complete`, `items archive`, `items restore`, and confirmation-required `items delete` commands in `src/daily_planner/cli.py`
- [ ] T023 [US1] Add stable unit and integration coverage for capture persistence on LLM failure, multi-item source retention, search over `done` and `archived`, explicit deletion, unresolved clarification visibility, dependency persistence, and confirmed dependent-link removal in `tests/unit/test_capture.py` and `tests/integration/test_capture_storage.py`

**Checkpoint**: The user can capture, inspect, correct, search, complete, archive, restore, and explicitly delete work without losing its original entry.

---

## Phase 4: User Story 2 - Get a balanced daily briefing (Priority: P1)

**Goal**: Produce a capacity-valid, explainable daily proposal that protects objective time, surfaces risks, and changes accepted plans only after confirmation.

**Independent Test**: With local sample work, deadlines, estimates, dependencies, and a stated time budget, generate and accept/reject a proposal; verify rationales, capacity, protected time, objective alerts, and preference retention.

### Tests for User Story 2

- [ ] T024 [P] [US2] Test deterministic eligibility, prerequisite completion ordering, cycle rejection, estimate/capacity fit, protected minutes, conflict reporting, and objective inactivity in `tests/unit/test_scheduler.py`
- [ ] T025 [P] [US2] Test plan proposal/accept/reject/edit CLI flows and non-TTY confirmation semantics in `tests/integration/test_brief_cli.py`
- [ ] T026 [P] [US2] Add fixed-date, color-free briefing output snapshots and assertions that every recommended item has a rationale in `tests/snapshots/test_briefing.py`
- [ ] T027 [P] [US2] Test plan-judgment request minimization, Pydantic response validation, unique ranks, allowed IDs, resolved dependency IDs, timeouts, retry limits, absence of unsolicited network/telemetry requests, and error preservation with a mocked SDK in `tests/contract/test_plan_advisor.py`

### Implementation for User Story 2

- [ ] T028 [US2] Add the transactional planning migration for objectives, objective progress, captured-item/objective links, plans, plan entries, and preferences in `src/daily_planner/storage/migrations/003_planning.sql`
- [ ] T029 [P] [US2] Define Objective, Objective Progress, Plan, Plan Entry, and Planning Preference validation in `src/daily_planner/domain/models.py`, with objective horizon `short, mid, long`, objective status `active, paused, completed`, plan status `proposed, accepted, rejected, superseded`, nonnegative capacity, `protected_minutes <= available_minutes`, unique zero-based entry positions, and positive estimates when present
- [ ] T030 [US2] Implement SQLite persistence for proposed plans, entries, plan history, and preferences in `src/daily_planner/storage/plans.py` and `src/daily_planner/storage/preferences.py`, preserving proposals separately from the accepted plan
- [ ] T031 [US2] Implement deterministic candidate eligibility, persisted dependency resolution, topological prerequisite ordering, blocking until every prerequisite is `done`, capacity checks, protected-time reservation, and infeasibility reporting in `src/daily_planner/planning/scheduler.py`
- [ ] T032 [P] [US2] Implement explainable value/urgency/cost-of-delay ranking and plan-judgment Pydantic schemas in `src/daily_planner/planning/prioritization.py` and `src/daily_planner/advisor/schemas.py`
- [ ] T033 [US2] Implement daily brief use case to gather minimum LLM context, call the advisor, validate output, schedule deterministically, and persist a `proposed` plan with a rationale for each recommended item in `src/daily_planner/application/planning.py`
- [ ] T034 [US2] Implement objective inactivity and at-risk commitment/waiting alerts using 14 calendar days and two working days by default, respectively, in `src/daily_planner/domain/rules.py`
- [ ] T035 [US2] Add `brief`, `plan show`, `plan accept`, `plan reject`, and proposal-edit/reorder commands; ensure only an explicit accept updates the active plan in `src/daily_planner/cli.py`
- [ ] T036 [US2] Persist confirmed plan corrections as user preferences and leave rejected proposals, accepted plan, and deferral counts unchanged in `src/daily_planner/application/planning.py` and `src/daily_planner/storage/preferences.py`
- [ ] T037 [US2] Add deterministic scheduling, time-conflict, stale-objective, at-risk alert, rationale, acceptance, rejection, and preference-retention tests in `tests/unit/test_planning.py` and `tests/integration/test_plan_acceptance.py`

**Checkpoint**: Briefings fit available time or explain conflicts, protect objective time, include rationale, and never silently replace the accepted plan.

---

## Phase 5: User Story 3 - Re-plan around interruptions (Priority: P1)

**Goal**: Propose an intraday plan diff for a new interruption, explain consequences, and preserve the accepted plan until explicit confirmation.

**Independent Test**: Start with an accepted plan, capture an incident, request a re-plan, then reject and accept proposals in separate runs; verify exact diffs and deferral counter updates only on acceptance.

### Tests for User Story 3

- [ ] T038 [P] [US3] Test diff generation for additions, moves, postponements, removals, consequences, rejection, and accepted-plan immutability in `tests/unit/test_plan_diff.py`
- [ ] T039 [P] [US3] Test `replan`, plan display, accept, and reject commands against an existing accepted plan in `tests/integration/test_replan_cli.py`

### Implementation for User Story 3

- [ ] T040 [US3] Implement deterministic old-plan/new-proposal diff generation and consequence records in `src/daily_planner/application/plan_diff.py`
- [ ] T041 [US3] Implement intraday re-plan use case using the accepted plan, the new captured item, validated advisor judgment, and deterministic scheduling in `src/daily_planner/application/planning.py`
- [ ] T042 [US3] Add `replan --item ITEM_ID` and ensure proposals show added/moved/deferred work and affected objectives/commitments without updating accepted-plan state in `src/daily_planner/cli.py`
- [ ] T043 [US3] Apply confirmed plan revision atomically, supersede the previous accepted plan, and increment `defer_count` only for confirmed deferrals in `src/daily_planner/storage/plans.py`
- [ ] T044 [US3] Test rejected-plan immutability, atomic accepted revisions, superseded proposal history, and two-consecutive-deferral tracking in `tests/integration/test_replan_state.py`

**Checkpoint**: Intraday changes are visible as a diff, with postponement consequences, and remain proposals until confirmed.

---

## Phase 6: User Story 4 - Maintain objectives, commitments, and waiting work (Priority: P2)

**Goal**: Manage short-, mid-, and long-term objectives, progress history, promises, waiting items, follow-up alerts, and conflict challenges.

**Independent Test**: Create linked objectives and commitments, record progress and follow-ups, advance a fixed test clock, and verify stale, overdue, at-risk, and repeated-deferral alerts.

### Tests for User Story 4

- [ ] T045 [P] [US4] Test objective, progress, commitment, and waiting-item validation, relationships, and status transitions in `tests/unit/test_objective_commitment_models.py`
- [ ] T046 [P] [US4] Test objective/commitment/waiting CLI commands and date-based briefing alerts in `tests/integration/test_objective_commitment_cli.py`

### Implementation for User Story 4

- [ ] T047 [US4] Define Objective Progress, Commitment, and Waiting Item models in `src/daily_planner/domain/models.py`, requiring objective `title` and `outcome`, person `person_label` and description for commitments/waiting items, optional due/follow-up dates, objective horizons `short, mid, long`, and statuses `open, fulfilled, cancelled` or `open, resolved, cancelled`
- [ ] T048 [US4] Implement persistence and non-cascading links for objectives, progress records, commitments, and waiting items in `src/daily_planner/storage/objectives.py` and `src/daily_planner/storage/commitments.py`
- [ ] T049 [US4] Implement objective CRUD, many-to-many item links, progress recording, and last-progress updates in `src/daily_planner/application/objectives.py`
- [ ] T050 [US4] Implement commitment and waiting-item capture, status updates, due-date rules, and user-configurable two-working-day at-risk window in `src/daily_planner/application/commitments.py`
- [ ] T051 [US4] Add `objectives add/list/progress`, `commitments add`, and `waiting add` commands with date and ID validation in `src/daily_planner/cli.py`
- [ ] T052 [US4] Add briefing annotations for stale objectives, overdue/at-risk commitments, objective conflicts, and repeated postponement with concise rationales in `src/daily_planner/application/planning.py`
- [ ] T053 [US4] Test 14-day objective inactivity, configurable working-day risk thresholds, non-cascading item links, repeated deferrals, and commitment/objective conflicts in `tests/unit/test_objective_commitment_rules.py` and `tests/integration/test_briefing_alerts.py`

**Checkpoint**: Objectives and people-related commitments remain linked to captured work and surface timely progress/follow-up alerts.

---

## Phase 7: User Story 5 - Receive delegation advice and configure an LLM (Priority: P3)

**Goal**: Configure the user's compatible endpoint and receive delegation advice without external actions.

**Independent Test**: Configure a mock OpenAI-compatible endpoint, request delegation advice, and verify configuration is safe and the response suggests only a role/team and reason.

### Tests for User Story 5

- [ ] T054 [P] [US5] Test TOML/environment/CLI precedence, missing configuration, secret masking, and `config validate` behavior without exposing the API key in `tests/unit/test_llm_config.py`
- [ ] T055 [P] [US5] Test delegation advice schema, allowed item references, rationale, and absence of external-action fields with a mocked client in `tests/contract/test_delegation_advice.py`
- [ ] T056 [P] [US5] Test delegated advice output and configurable endpoint/model behavior through CLI commands in `tests/integration/test_advisor_cli.py`

### Implementation for User Story 5

- [ ] T057 [US5] Complete non-secret TOML settings and `DAILY_PLANNER_LLM_API_KEY` resolution with per-user config paths in `src/daily_planner/config.py`
- [ ] T058 [US5] Implement a minimal redacted endpoint validation request and masked effective-settings output in `src/daily_planner/advisor/client.py` and `src/daily_planner/cli.py`
- [ ] T059 [US5] Add strict delegation-advice response schema and prompt requiring a suggested role/team and rationale without claiming assignment or contact in `src/daily_planner/advisor/schemas.py` and `src/daily_planner/advisor/prompts.py`
- [ ] T060 [US5] Implement `advise delegate ITEM_ID` using only the selected item's minimum context and return clear offline/schema errors in `src/daily_planner/application/advice.py`
- [ ] T061 [US5] Add `config show` and `config validate` commands with explicit secret redaction and no persistence of credentials in `src/daily_planner/cli.py`

**Checkpoint**: LLM configuration is user-controlled and redacted; delegation remains advice only.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Deliver portability, documentation, reproducible checks, and end-to-end verification across stories.

- [ ] T062 [P] Implement versioned JSON/Markdown exports that exclude credentials, prompts, and hidden request logs in `src/daily_planner/application/export.py`
- [ ] T063 [P] Implement consistent SQLite backup using SQLite's backup API in `src/daily_planner/storage/sqlite.py`
- [ ] T064 [P] Add export/backup commands and path/stdout behavior from the CLI contract in `src/daily_planner/cli.py`
- [ ] T065 Add migration-upgrade, JSON export round-trip, Markdown export, backup consistency, and secret-exclusion tests in `tests/integration/test_export_backup.py`
- [ ] T066 [P] Document install, quickstart, configuration precedence, local paths, privacy, LLM providers verified by contract tests, and CLI examples in `README.md`
- [ ] T067 [P] Document contribution workflow, tests, Ruff/mypy, pre-commit, and no-live-credentials CI policy in `CONTRIBUTING.md`
- [ ] T068 Run every scenario in `specs/001-adaptive-planning-agent/quickstart.md` and correct mismatches in the owning source or test files
- [ ] T069 [P] Add a performance test that seeds 1,000 active items, warms up each operation, samples 100 capture-persistence, item-list, and deterministic schedule-generation runs, and asserts p95 below 500 ms excluding LLM time in `tests/performance/test_local_operations.py`
- [ ] T070 [P] Add a v1 privacy/scope contract test confirming no telemetry or external-system requests and no connector/plugin command or registration interface in `tests/contract/test_v1_privacy_scope.py`
- [ ] T071 Run `uv run pytest`, `uv run ruff check .`, `uv run ruff format --check .`, and `uv run mypy src`; resolve failures in affected files under `src/` and `tests/`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies; package metadata, CI, hooks, and license can be prepared immediately.
- **Foundational (Phase 2)**: Depends on Setup; blocks all user story implementation.
- **User Story 1 (Phase 3)**: Depends on the foundation; delivers the MVP capture/review loop.
- **User Story 2 (Phase 4)**: Depends on US1 captured items and shared persistence.
- **User Story 3 (Phase 5)**: Depends on US2 plans and accepted-plan persistence.
- **User Story 4 (Phase 6)**: Depends on US1 item links and US2 briefing/alert integration.
- **User Story 5 (Phase 7)**: Depends on the shared advisor/config boundary; item-specific delegation advice also depends on US1.
- **Polish (Phase 8)**: Depends on the stories selected for release; T062-T065 and T070 can be developed after storage/foundation work, T069 depends on capture/list/scheduler implementations, and T071 runs after all implementation and validation tasks.

### User Story Dependencies

- **US1 (P1)**: Foundation only; first independently testable increment and MVP.
- **US2 (P1)**: Requires captured work and shared storage from US1.
- **US3 (P1)**: Requires the proposed/accepted plan lifecycle from US2.
- **US4 (P2)**: Requires captured-item links from US1 and briefing integration from US2; domain records can be developed independently once migrations are established.
- **US5 (P3)**: Requires the advisor client from US1; delegation requires US1 item records. Configuration and delegation are independently testable with a mocked client.
- **Dependency support**: Defined in US1 item metadata and persistence; US2 consumes those links in deterministic scheduler eligibility and ordering.

### Parallel Opportunities

- Setup T003-T006 can be done in parallel after T002 establishes project metadata ownership.
- Foundational T008 and T010 can proceed in parallel; T009 must exist before migration tests in T011.
- Within US1, T013-T015 are separate test files; model/schema/prompt tasks can proceed independently, while repository/use-case/CLI tasks build on them.
- Within US2, T024-T027 are separate test files. Objective models (T029), plan persistence (T030), ranking schemas (T032), and alert rules (T034) touch separate files; scheduler/use case/CLI depend on shared types and persistence.
- Within US3, T038 and T039 can be authored independently; diff logic and atomic persistence touch separate modules.
- Within US4, T045-T046 can be authored in parallel; objective and commitment storage/application modules are separate after migration/model contracts are fixed.
- Within US5, T054-T056 are independent test files; configuration, delegation schema/prompt, and advice use case are separate once the client is stable.
- Export/backup implementation tasks T062-T064 touch separate application, storage, and CLI modules after foundational SQLite exists.
- Privacy/scope contract T070 can be implemented independently after the CLI and advisor seams exist; performance test T069 follows the capture, listing, and scheduler implementations it measures.

### Parallel Example: User Story 1

```text
After T012:
  T013 Capture model tests
  T014 Capture CLI tests
  T015 Capture advisor contract tests
After tests define expected behavior:
  T017 Domain validation/state transitions
  T019 Extraction schema and prompt
Then:
  T018 Storage -> T021 Capture use case -> T022 CLI integration
  T020 Advisor client can proceed alongside storage/use-case work
Finally:
  T023 End-to-end capture persistence checks
```

## Implementation Strategy

1. Complete Setup and Foundation before story development.
2. Deliver **US1 first as the MVP**: reliable local capture and review, including graceful LLM failure.
3. Implement US2 to make captured work actionable through a deterministic, confirmed daily plan.
4. Implement US3 to add safe intraday plan diffs and explicit revision.
5. Add US4 for objective progress, commitments, and waiting-item visibility.
6. Add US5 delegation and configuration UX, then export, backups, documentation, and full validation.
7. Keep each story usable at its checkpoint; an unfinished later story must not disable earlier local capture or accepted-plan safety.
