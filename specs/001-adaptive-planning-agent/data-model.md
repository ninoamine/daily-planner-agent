# Data Model: Adaptive Personal Planning Agent

This model describes local, single-user records. Identifiers are stable opaque IDs.
Timestamps are stored in UTC; user-entered deadlines retain their original wording
and interpreted date/timezone so ambiguous dates are not silently converted.

## Entities

### Source Capture

Represents one conversational entry exactly as supplied by the user. It is persisted
before any LLM extraction and remains available even if extraction fails or the text
produces multiple items.

| Field | Type | Rules |
|---|---|---|
| `id` | ID | Stable, unique |
| `original_text` | Text | Required; never overwritten during item edits |
| `created_at` | DateTime | Required |

### Captured Item

Represents one task, request, incident, or note identified from a conversational entry.
Multiple items may refer to the same Source Capture.

| Field | Type | Rules |
|---|---|---|
| `id` | ID | Stable, unique |
| `source_capture_id` | ID | Required reference to its original Source Capture |
| `title` | Text | Required once identified; concise, user-editable |
| `description` | Text | Optional detail |
| `kind` | Enum | `task`, `request`, `incident`, `note` |
| `status` | Enum | `inbox`, `actionable`, `in_progress`, `waiting`, `done`, `archived` |
| `requester` | Text? | Optional; who asked, not an external account/contact |
| `deadline_at` | DateTime? | Optional; only set when sufficiently clear |
| `deadline_text` | Text? | Original date/deadline phrase for audit and correction |
| `estimate_minutes` | Integer? | Optional; positive when present |
| `clarification_question` | Text? | Optional unresolved question; item remains visible |
| `defer_count` | Integer | Nonnegative; incremented only when a confirmed plan defers the item |
| `created_at`, `updated_at` | DateTime | Required |
| `completed_at`, `archived_at` | DateTime? | Set on corresponding state transition |

### Item Dependency

An item dependency means that one captured item is blocked by another captured item.
The relationship is directional: `item_id` is the dependent item and
`prerequisite_item_id` is work that must be completed first.

| Field | Type | Rules |
|---|---|---|
| `item_id` | ID | Required captured-item reference; dependent |
| `prerequisite_item_id` | ID | Required captured-item reference; prerequisite |
| `created_at` | DateTime | Required |

The `(item_id, prerequisite_item_id)` pair is unique. Both references must point to
distinct existing items. Adding a link that would create a cycle is rejected, leaving
existing links unchanged. A dependent item is schedulable only after all prerequisites
are `done`; archived or deleted prerequisites do not count as completed. Before
explicitly deleting a prerequisite with dependents, the CLI identifies those dependents
and requires confirmation to remove the affected links. Dependency links are not
silently cascaded.

Captured-item state transitions:

```text
inbox -> actionable | waiting | archived
actionable -> in_progress | waiting | done | archived
in_progress -> actionable | waiting | done | archived
waiting -> actionable | in_progress | done | archived
done -> archived
archived -> actionable (user restores)
any retained state -> deleted (explicit user action only)
```

Changing state never removes the original capture. Done and archived items remain
searchable until explicit deletion. Deletion is a user action, not an automatic retention
policy.

### Objective

| Field | Type | Rules |
|---|---|---|
| `id` | ID | Stable, unique |
| `title` | Text | Required |
| `outcome` | Text | Required description of desired result |
| `horizon` | Enum | `short`, `mid`, `long` |
| `status` | Enum | `active`, `paused`, `completed` |
| `last_progress_at` | DateTime? | Updated when progress is recorded |
| `created_at`, `updated_at` | DateTime | Required |

An objective may link to zero or more captured items. A captured item may link to zero
or more objectives. Progress is recorded in Objective Progress entries rather than
overwriting the desired outcome.

### Objective Progress

| Field | Type | Rules |
|---|---|---|
| `id` | ID | Stable, unique |
| `objective_id` | ID | Required reference to one objective |
| `occurred_at` | DateTime | Required |
| `note` | Text | Required user-confirmed progress description |
| `captured_item_id` | ID? | Optional related item |

Recording progress updates the objective's `last_progress_at`. An objective is stale
when it is active and elapsed time since last progress (or creation, if no progress
exists) reaches the configured inactivity threshold.

### Commitment

Represents a promise made by the user to a person.

| Field | Type | Rules |
|---|---|---|
| `id` | ID | Stable, unique |
| `person_label` | Text | Required plain-text name or role |
| `description` | Text | Required promise |
| `due_at` | DateTime? | Optional; unknown remains unset |
| `status` | Enum | `open`, `fulfilled`, `cancelled` |
| `captured_item_id` | ID? | Optional related item |
| `objective_id` | ID? | Optional related objective |
| `created_at`, `updated_at` | DateTime | Required |

An open commitment is overdue after its due date. It is at risk when due within the
user-configured warning period, defaulting to two working days. Risk is computed from
the due date and preference, not stored as a stale status.

### Waiting Item

Represents a response or action expected from another person.

| Field | Type | Rules |
|---|---|---|
| `id` | ID | Stable, unique |
| `person_label` | Text | Required plain-text name or role |
| `description` | Text | Required expected response or action |
| `follow_up_at` | DateTime? | Optional; unknown remains unset |
| `status` | Enum | `open`, `resolved`, `cancelled` |
| `captured_item_id` | ID? | Optional related item |
| `objective_id` | ID? | Optional related objective |
| `created_at`, `updated_at` | DateTime | Required |

An open waiting item is overdue or at risk using the same due-date and warning-period
rules as a commitment.

### Plan and Plan Entry

Plan is an immutable proposal/revision once displayed; an explicit user decision updates
its acceptance status. A new accepted revision supersedes the previous accepted plan.

| Field | Type | Rules |
|---|---|---|
| `Plan.id` | ID | Stable, unique |
| `Plan.planning_date` | Date | Required |
| `Plan.available_minutes` | Integer | Nonnegative, supplied by user |
| `Plan.protected_minutes` | Integer | Nonnegative and no greater than available minutes |
| `Plan.status` | Enum | `proposed`, `accepted`, `rejected`, `superseded` |
| `Plan.revises_plan_id` | ID? | Optional prior accepted plan |
| `Plan.supersedes_plan_id` | ID? | Optional earlier proposal replaced by a user-edited proposal |
| `Plan.created_at`, `decided_at` | DateTime | Required creation time; decision time when decided |
| `PlanEntry.item_id` | ID | Required captured-item reference |
| `PlanEntry.position` | Integer | Unique within plan; zero-based display order |
| `PlanEntry.estimate_minutes` | Integer? | May remain unknown; positive when present |
| `PlanEntry.rationale` | Text | Required, concise rationale for recommended items |
| `PlanEntry.disposition` | Enum | `scheduled`, `deferred`, `not_selected` |
| `PlanEntry.consequence` | Text? | Explanation of impact when moved or deferred |

The deterministic planning engine checks dependency order, capacity, and protected time.
An infeasible proposal is reported with the conflict; it is not silently made to fit.
Plan diffs are derived from the previous accepted plan and the proposal. Editing a
displayed proposal creates a replacement proposal and marks the earlier proposal
superseded; it does not change the accepted plan. Accepting a proposal increments
deferral counts for items actually deferred from the accepted plan. Rejecting or
superseding a proposal leaves the accepted plan and deferral counts unchanged.

### Planning Preference

| Field | Type | Rules |
|---|---|---|
| `key` | Text | Unique setting name |
| `value` | Typed value | Validated for the setting |
| `updated_at` | DateTime | Required |

Includes objective inactivity days (default 14), at-risk working-day warning (default
2), available/protected-time defaults, timezone, and user-confirmed planning corrections.
Preferences do not override hard scheduling constraints or bypass plan confirmation.

### LLM Configuration

Configuration is kept outside planning records. TOML stores the base URL and model;
`DAILY_PLANNER_LLM_API_KEY` supplies the secret. A secret may instead come from a
user-controlled local secrets file excluded from version control. Neither secret source
is included in exports, logs, prompts, or config display.

## Relationships and Persistence Invariants

- One Source Capture may produce multiple captured items; each refers to the same
  immutable original text and has its own stable ID.
- Captured-item dependencies are directed links between distinct items and MUST remain
  acyclic. A dependent item cannot be scheduled until every prerequisite is done.
- Objective/item links are many-to-many and do not cascade-delete captured items.
- Commitments and waiting items may link to a captured item and/or objective; deleting
  linked work requires an explicit user action and does not erase the commitment record.
- Plan entries refer to captured items; historical plans remain reviewable until the
  user deletes their underlying data.
- All tables are local to one user. There are no users, roles, permissions, or external
  identity records.
- SQLite schema upgrades use ordered, transactional SQL migrations tracked by
  `PRAGMA user_version`. Export format has an independent `schema_version`.
- Exported JSON/Markdown contains planning records and preferences, never credentials,
  prompts, or hidden model request logs.
