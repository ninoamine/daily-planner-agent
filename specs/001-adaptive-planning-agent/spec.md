# Feature Specification: Adaptive Personal Planning Agent

**Feature Branch**: `not created (workspace is not a git repository)`

**Created**: 2026-11-01

**Status**: Draft

**Input**: User description: "Build a single-user personal planning agent that captures conversational work, balances daily interruptions with longer-term objectives, and proposes explainable plans for the user to validate."

## Clarifications

### Session 2026-11-01

- Q: Should the agent ask for approval before sending planning details to your configured LLM service each time? → A: Automatically send only the minimum context needed to the service the user configured, without a separate prompt each time.
- Q: How should the agent decide that a commitment or follow-up is at risk before its due date? → A: Use a user-set warning period, defaulting to two working days.
- Q: What should happen to completed or archived items in the user's work history? → A: Keep completed and archived items searchable until the user explicitly deletes them.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Capture and recover work (Priority: P1)

The user records a task, request, incident, or note in natural language. The agent
preserves the original entry, identifies the useful details, and asks a focused question
only when missing information is important to handling the item. The user can later find
every captured item, including one that still needs clarification.

**Why this priority**: Planning cannot be trusted if interruptions and commitments are
lost before they reach the plan.

**Independent Test**: Enter requests with and without explicit dates, requesters, effort,
or objective links. Verify each original entry remains available, extracted details are
shown accurately, and a question is asked only when an important missing detail prevents
the next planning decision.

**Acceptance Scenarios**:

1. **Given** a natural-language request containing a task, requester, and deadline,
   **When** the user records it, **Then** the agent saves the original entry and presents
   the extracted details for review.
2. **Given** a captured item with an unknown deadline that is not needed for its current
   handling, **When** it is recorded, **Then** the agent keeps the deadline unknown and
   does not interrupt the user with an unnecessary question.
3. **Given** an item whose missing information is necessary to make a requested
   commitment or planning decision, **When** the agent needs that information,
   **Then** it asks a focused clarifying question and preserves the item until answered.
4. **Given** multiple captured items, including an unanswered clarification,
   **When** the user asks to review their work, **Then** every item is available with its
   current status and any unresolved information clearly identified.
5. **Given** the v1 agent, **When** the user records work, **Then** the work comes from
   the user's manual conversational input and is not imported from an external system.
6. **Given** a completed or archived item, **When** the user reviews or searches their
   work history, **Then** the item remains available until the user explicitly deletes it.

---

### User Story 2 - Get a balanced daily briefing (Priority: P1)

At the start of a work period, the user requests a briefing for their available time.
The agent proposes an ordered list that accounts for commitments, urgency, value,
dependencies, effort, and recent progress on objectives. It protects the time the user
has designated for mid- and long-term objectives and explains every recommendation.

**Why this priority**: A brief, reasoned plan addresses the user's daily re-prioritization
burden while preventing urgent work from continually eclipsing strategic objectives.

**Independent Test**: Create work items with different deadlines, values, efforts,
dependencies, and objective links; set the day's available time and protected objective
time; request a briefing and verify fit, rationale, and surfaced risks.

**Acceptance Scenarios**:

1. **Given** known work items and a stated available time budget, **When** the user
   requests a briefing, **Then** the agent proposes an ordered plan whose estimated work
   does not exceed that budget.
2. **Given** the user has designated time for mid- or long-term objectives, **When** the
   agent proposes the day, **Then** it explicitly preserves that time or explains why
   commitments make the requested allocation infeasible.
3. **Given** an item is recommended, **When** the briefing is shown, **Then** that item
   includes a concise rationale tied to relevant value, urgency, cost of delay,
   dependencies, commitment, or objective.
4. **Given** an objective has had no progress for the configured inactivity period,
   **When** the user requests a briefing, **Then** the objective is surfaced with its
   last progress date and the agent identifies a reasonable next action or asks the user
   to choose one.
5. **Given** overdue or at-risk commitments and follow-ups, **When** the user requests a
   briefing, **Then** those items are called out separately from ordinary suggested work.
6. **Given** the user accepts, reorders, edits, or rejects a proposed plan, **When** the
   user confirms the choice, **Then** the plan reflects that choice and the correction is
   retained as a planning preference for later briefings.
7. **Given** urgent commitments exceed available time or conflict with protected
   objective time, **When** the agent proposes a briefing, **Then** it identifies the
   conflict and asks the user to choose without silently exceeding the time budget or
   hiding displaced work.
8. **Given** a task is blocked by another captured item, **When** the user records or
   edits its dependencies, **Then** the prerequisite is linked and the briefing does
   not schedule the blocked task before its prerequisite is completed.

---

### User Story 3 - Re-plan around interruptions (Priority: P1)

When new work arrives during the day, the user records it and asks for an updated plan.
The agent explains the proposed changes, including what moves or is postponed and the
consequences for objectives and commitments. The accepted plan remains unchanged until
the user confirms the proposed update.

**Why this priority**: Interruptions are the main reason the user's existing static list
becomes stale and silently loses important work.

**Independent Test**: Accept a daily plan, capture a time-sensitive interruption, request
re-planning, and verify a change proposal is shown without modifying the accepted plan.
Confirm and reject the proposal in separate runs and verify the corresponding outcomes.

**Acceptance Scenarios**:

1. **Given** an accepted plan and a new incident, **When** the user requests re-planning,
   **Then** the agent proposes a revised ordered plan and shows each addition, move,
   postponement, or removal relative to the accepted plan.
2. **Given** a proposed change affects a commitment or objective, **When** the proposal is
   presented, **Then** the agent states the expected consequence and provides a reason
   for the change.
3. **Given** the user rejects the proposed update, **When** the rejection is confirmed,
   **Then** the previously accepted plan remains active and the new item remains captured.
4. **Given** the user accepts a proposed update, **When** confirmation is given, **Then**
   the updated plan becomes active and postponed work remains visible with its status.

---

### User Story 4 - Maintain objectives, commitments, and waiting work (Priority: P2)

The user defines short-, mid-, and long-term objectives and records who requested work,
what they promised to whom, and what they are waiting for from others. The agent links
work to objectives and surfaces stalled objectives, overdue promises, and follow-ups.

**Why this priority**: Long-term outcomes and relationship commitments need to remain
visible while daily task capture and planning are in use.

**Independent Test**: Create objectives at each horizon, link work to them, record
promises and waiting items with follow-up dates, then advance dates and progress to
verify that status and briefing alerts are accurate.

**Acceptance Scenarios**:

1. **Given** an objective with a horizon and desired outcome, **When** the user creates
   it, **Then** it appears in the objective list and can be linked to captured work.
2. **Given** a task is linked to an objective, **When** the task is completed or the
   user records progress, **Then** the objective's progress history reflects that event.
3. **Given** a promise or waiting item with a person, description, and follow-up date,
   **When** the date becomes overdue or at risk, **Then** the item is surfaced in the
   briefing with its owner and follow-up date.
4. **Given** a task has repeatedly been postponed or a proposed choice conflicts with a
   stated objective or commitment, **When** the user reviews a recommendation, **Then**
   the agent respectfully calls out the pattern or conflict and explains its concern.

---

### User Story 5 - Receive delegation advice and configure an LLM (Priority: P3)

The user can ask whether work is suitable for delegation and receive advice about a
relevant role or team and the reason, without the agent contacting anyone. The user
configures a compatible LLM service using their chosen URL, secret key, and model.

**Why this priority**: Delegation and flexible model choice improve usefulness, but
depend on reliable capture and planning foundations.

**Independent Test**: Ask for delegation advice on work with different ownership
characteristics and verify advice is explanatory and has no external side effect.
Configure different compatible model choices and verify provider-specific choices are
not required by the user-facing workflow.

**Acceptance Scenarios**:

1. **Given** a task that may be better handled by another role or team, **When** the
   user requests advice, **Then** the agent names a suitable role or team and explains
   why the task may be delegated.
2. **Given** delegation advice is shown, **When** it is presented, **Then** the agent
   does not contact a person, assign the task externally, or claim delegation occurred.
3. **Given** valid user-supplied LLM connection details for a compatible provider,
   **When** the user configures them, **Then** the agent can use the selected service
   without requiring a hardcoded provider or model.
4. **Given** LLM output is malformed or incomplete, **When** the agent receives it,
   **Then** it does not apply the output as a plan or change and clearly reports that
   it could not produce a valid proposal, while preserving captured information and the
   last accepted plan.
5. **Given** the agent prepares a request to an LLM service, **When** it sends the
   request, **Then** planning data remains local except for the minimum context needed
   for that request, no telemetry is sent, and secrets are supplied only through
   environment variables or a local configuration file excluded from version control.

### Edge Cases

- A natural-language entry contains several separate tasks; the user can review and
  correct the captured items without losing the source entry.
- A date or effort estimate is ambiguous; the agent preserves uncertainty rather than
  presenting an unconfirmed guess as fact.
- The user's available time is less than the sum of urgent commitments and protected
  objective time; the briefing identifies the conflict and asks the user to choose
  rather than silently dropping work or exceeding the time budget.
- No captured work is suitable for a requested time budget; the agent may propose
  objective progress, follow-up, or available capacity without inventing a commitment.
- A linked objective is completed, paused, or deleted; linked work remains available
  and the user is told when it needs a new objective association.
- The configured LLM is unreachable, rejects a request, or returns unusable content;
  captured data and the last accepted plan remain intact, and the user receives a clear
  error rather than a fabricated result.
- Local storage or a local configuration file cannot be read or written; the agent
  reports the failure and does not claim that data or preferences were saved.
- The same item is captured more than once; the agent may identify a possible duplicate
  but must let the user decide whether to merge it.
- A user attempts to make an item depend on itself or creates a dependency cycle; the
  agent rejects the change and leaves existing relationships intact.
- A user attempts to delete a prerequisite with dependent items; the agent identifies
  affected items and requires explicit confirmation before removing those dependency
  links.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The user MUST be able to capture tasks, requests, incidents, and notes
  using conversational text.
- **FR-002**: The agent MUST preserve each captured entry and make it reviewable.
  Completed and archived entries MUST remain searchable until the user explicitly
  deletes them.
- **FR-003**: The agent MUST structure applicable details including description,
  requester, deadline, estimated effort, related objective, commitment, and waiting
  party; unknown details MUST remain visibly unknown rather than being asserted as fact.
- **FR-004**: The agent MUST ask a clarifying question only when missing information is
  important to the requested capture, decision, or plan, and MUST preserve the item
  while awaiting an answer.
- **FR-005**: The user MUST be able to create and review short-, mid-, and long-term
  objectives and link captured work to one or more objectives.
- **FR-006**: The agent MUST track objective progress and surface objectives without
  progress for the configured inactivity period, including the last progress date.
- **FR-007**: The user MUST be able to set available planning time and protected time
  for mid- and long-term objectives.
- **FR-008**: On request, the agent MUST provide a concise, ordered daily plan that
  fits available time and accounts for deadlines, value, urgency, cost of delay,
  effort, dependencies, commitments, and objective progress.
- **FR-009**: Every recommended priority MUST include a visible, concise rationale.
- **FR-010**: The agent MUST identify when stated time constraints conflict and MUST
  not silently exceed the available time or discard a commitment or protected
  objective time to hide the conflict.
- **FR-011**: The user MUST be able to accept, reorder, edit, or reject a proposed plan.
  The agent MUST apply changes only after user confirmation and retain confirmed
  corrections as preferences.
- **FR-012**: For intraday re-planning, the agent MUST show a diff from the accepted
  plan, explain postponements and changes, and describe affected objectives and
  commitments. The accepted plan MUST remain unchanged until the user confirms.
- **FR-013**: The agent MUST track promises made to people and work awaited from people,
  including follow-up dates, and MUST surface overdue items and items due within the
  user's configured at-risk warning period in a briefing. The default warning period
  MUST be two working days.
- **FR-014**: The agent MUST flag repeated postponement and conflicts between a proposed
  decision and a stated objective or commitment, with a concise explanation.
- **FR-015**: The agent MAY recommend delegation to a role or team with a rationale,
  but MUST NOT contact people, create external assignments, or claim to have delegated.
- **FR-016**: The user MUST be able to configure a compatible LLM using a user-selected
  service URL, secret key, and model; no provider or model may be required by default.
- **FR-017**: The agent MUST validate LLM output before using it and MUST report
  unusable output without applying it as a plan or change.
- **FR-018**: Planning data and preferences MUST be stored locally. The agent MUST NOT
  send telemetry. For LLM requests, it MUST send only the minimum context needed to the
  user-configured service and MUST NOT prompt for per-request approval.
- **FR-019**: The user MUST be able to use v1 through a CLI and manual conversational
  input only. V1 MUST NOT import from or take actions in external systems.
- **FR-020**: V1 MUST NOT provide plugins, connectors, or integration extension points.
  Future external data sources are outside this feature's scope.
- **FR-021**: When LLM service, local data, or configuration operations fail, the
  agent MUST report the failure and preserve the last accepted plan and captured
  information.
- **FR-022**: The user MUST be able to link a captured item to prerequisite captured
  items. The agent MUST reject self-dependencies and cycles, MUST NOT schedule an item
  before unfinished prerequisites, and MUST identify affected dependent items before
  removing a prerequisite link.

### Key Entities *(include if feature involves data)*

- **Source Capture**: One immutable natural-language entry supplied by the user; one
  source capture may produce multiple captured items.
- **Captured Item**: A task, request, incident, or note linked to its source capture,
  with extracted details, status, clarification state, and optional prerequisites.
- **Item Dependency**: A directional relationship stating that one captured item is
  blocked until another captured item is completed.
- **Objective**: A desired outcome with a short-, mid-, or long-term horizon, progress
  history, and linked work.
- **Plan**: An ordered proposal for a stated time budget, including rationale,
  protected objective time, and the user's acceptance state.
- **Commitment**: A promise made to a person, with a description, due date, status, and
  potential link to a captured item or objective.
- **Waiting Item**: An expected response or action from a person, with a follow-up
  date, status, and optional objective link.
- **Planning Preference**: A user-confirmed correction or setting used to inform later
  proposals, including protected time and inactivity thresholds.
- **LLM Configuration**: The user-selected service URL, secret key, and model used to
  request judgment or text interpretation.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: During a two-week trial, the user reports that no known task, request,
  incident, or commitment was forgotten after it had been entered.
- **SC-002**: During the second week of a two-week trial, the user completes morning
  planning in five minutes or less on at least 8 of 10 workdays.
- **SC-003**: 100% of priority recommendations in evaluated briefings include a
  visible, one-line rationale.
- **SC-004**: 100% of objectives with no progress for the configured inactivity
  period are surfaced in the next requested briefing.
- **SC-005**: In a review of 10 intraday re-planning proposals, every proposal shows
  affected and postponed work and consequences for linked objectives or commitments;
  no proposal changes the accepted plan before user confirmation.
- **SC-006**: The user rates the usefulness of daily prioritization at least 4 out of
  5 after the two-week trial.
- **SC-007**: With 1,000 active captured items and no LLM network call, p95 latency for
  capture persistence, item listing, and deterministic schedule generation is under
  500 ms.

## Assumptions

- The first release serves one user on one local environment and starts with a CLI;
  a future UI is not part of this feature.
- The user supplies available work time for each planning period. Protected objective
  time is established by the user, not inferred as a fixed percentage.
- The default objective inactivity threshold is 14 calendar days; the user can change
  it.
- The default at-risk warning period for commitments and follow-ups is two working
  days; the user can change it.
- Completed and archived items remain locally searchable until the user explicitly
  deletes them; the agent does not automatically purge them.
- A task is considered repeatedly postponed after it is deferred in two consecutive
  user-confirmed plans. The agent calls out the pattern but does not force a decision.
- The agent may apply WSJF-style reasoning but does not require a fixed numerical
  scoring formula; it explains the factors that materially affect the recommendation.
- The reference Product Owner / DevOps lead persona informs delegation suggestions,
  but the user can provide or correct their role and team context.
- In the two-week success measures, “known item” means an item the user has entered
  or confirmed, and workdays means the user's scheduled workdays during the trial.
- The selected LLM service is compatible with the user-provided connection details;
  secrets are provided through environment variables or a local configuration file
  excluded from version control. The user has configured the service to receive the
  minimum context needed for LLM-assisted work, without per-request approval prompts.
