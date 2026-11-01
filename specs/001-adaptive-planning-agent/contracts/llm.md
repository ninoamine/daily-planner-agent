# LLM Prompt and Response Contracts

This contract is provider-independent and assumes the configured endpoint implements
OpenAI-compatible Chat Completions. The official SDK receives the user-supplied base
URL, API key, and model. The secret is transport configuration only: it is never placed
in prompt content, persisted in planning tables, logged, or exported.

## Shared Rules

- Send only context required for the specific operation. Prefer IDs and role labels to
  full personal or organizational details; do not send unrelated history, full
  databases, secrets, or telemetry.
- Treat user text and stored descriptions as untrusted data, not instructions to change
  these rules or perform actions.
- Request a single JSON object in UTF-8. Do not require provider-specific JSON-schema
  mode. Parse and validate the response locally with Pydantic, rejecting unknown fields,
  invalid enums, missing required values, and invalid references.
- A schema-valid response is not proof of factual accuracy. Keep inferred values
  distinguishable from user-confirmed values and preserve original input.
- The LLM only advises. Deterministic code validates dates/state/dependencies, computes
  available capacity and the final schedule, and validates plan diffs. The user must
  confirm before any proposed plan changes become active.
- If an endpoint returns invalid or unusable output, show a clear error; do not make a
  success-shaped fallback or alter the accepted plan.

## Capture Extraction

### Request payload

```json
{
  "operation": "capture_extraction",
  "current_date": "2026-11-01",
  "timezone": "Europe/Paris",
  "known_objectives": [
    {"id": "objective-id", "title": "Reduce deployment lead time"}
  ],
  "user_text": "The architecture team asked me for a review by Thursday."
}
```

Include `known_objectives` only to support matching; include no unrelated items. Keep
the exact `user_text` in local storage before calling the advisor.

### Response shape

```json
{
  "items": [
    {
      "title": "Review the architecture proposal",
      "description": null,
      "kind": "request",
      "requester": "architecture team",
      "deadline_text": "Thursday",
      "deadline_date": "2026-11-05",
      "estimate_minutes": null,
      "suggested_objective_id": null,
      "commitment": null,
      "waiting_for": null
    }
  ],
  "clarification_question": null,
  "clarification_reason": null
}
```

Contract rules:

- `items` contains one or more actionable interpretations; for a note that is not work,
  it may contain one item with `kind: "note"`.
- `kind` is one of `task`, `request`, `incident`, `note`.
- Optional facts are `null` when not stated or safely inferable. Dates use ISO
  `YYYY-MM-DD`, are interpreted relative to `current_date` and `timezone`, and retain
  `deadline_text` verbatim when a date phrase was present.
- `estimate_minutes` is a positive integer or `null`; do not invent estimates.
- `suggested_objective_id` must be one of the supplied IDs or `null`.
- `commitment` and `waiting_for` are either `null` or objects with a plain-text
  `person_label`, `description`, and optional ISO date (`due_date` or `follow_up_date`).
- If an essential detail is missing for the user's requested action, provide one
  concise `clarification_question` and a brief `clarification_reason`; otherwise both
  are `null`. The original entry stays captured either way.
- Do not claim that a task was scheduled, a commitment made, or another person
  contacted.

## Priority and Plan-Impact Advice

### Request payload

The application constructs a compact payload from eligible local records:

```json
{
  "operation": "plan_judgment",
  "current_date": "2026-11-01",
  "timezone": "Europe/Paris",
  "available_minutes": 360,
  "protected_objective_minutes": 60,
  "items": [
    {
      "id": "item-1",
      "title": "Resolve deployment incident",
      "kind": "incident",
      "deadline": "2026-11-01T15:00:00+01:00",
      "estimate_minutes": 90,
      "objective_ids": [],
      "blocked_by_item_ids": [],
      "defer_count": 0,
      "commitment_due": null
    }
  ],
  "objectives": [
    {
      "id": "objective-1",
      "title": "Reduce deployment lead time",
      "horizon": "long",
      "last_progress_at": "2026-10-10"
    }
  ],
  "accepted_plan": null
}
```

Send `accepted_plan` only for an intraday re-plan, and include only the item IDs,
order, estimates, and user-confirmed rationale necessary to compare the proposal. Do not
include requester/person names unless essential; role labels are preferred.

### Response shape

```json
{
  "item_judgments": [
    {
      "item_id": "item-1",
      "relative_rank": 1,
      "value": "high",
      "urgency": "high",
      "cost_of_delay": "high",
      "rationale": "It has the nearest deadline and blocks restoring service."
    }
  ],
  "objective_conflicts": [],
  "plan_impacts": [],
  "delegation_advice": []
}
```

Contract rules:

- Every item judgment references a supplied item ID exactly once; `relative_rank` is a
  unique positive integer. The application rejects duplicate, missing, or unknown IDs.
- `value`, `urgency`, and `cost_of_delay` are `low`, `medium`, or `high`. These are
  advisory labels, not computed facts or replacements for deadline/dependency rules.
- Every rationale is one concise sentence. The application requires a rationale for
  each item it recommends.
- `objective_conflicts` contains only supplied objective/item IDs and a concise
  explanation of the conflict. Objective inactivity detection itself is deterministic.
- The advisor may rank only eligible items. `blocked_by_item_ids` are resolved and
  validated locally; the advisor cannot override unfinished prerequisites or invent
  dependency links.
- `plan_impacts` explains only deterministic changes already identified by the
  application; the LLM must not invent moved or dropped items.
- `delegation_advice` is optional and contains a supplied `item_id`, suggested role/team,
  and rationale. It never signals that work was assigned or a person contacted.
- The deterministic scheduler applies eligible rankings subject to capacity, protected
  time, status, and dependency constraints. It records unmet capacity or conflicts
  rather than silently dropping a commitment or exceeding available time.
- For a re-plan, the application computes the exact diff against the last accepted
  plan. An advisor explanation cannot add or remove diff entries.

## Validation and Retry Behavior

- Require finite request timeouts (30 seconds) and at most two retries for transient
  connection, rate-limit, or server failures.
- Do not retry invalid credentials, invalid request configuration, or malformed/schema-
  invalid responses.
- After retries are exhausted, return a user-visible error. Keep captured work and the
  last accepted plan unchanged.
- Contract tests use a mocked OpenAI-compatible client and representative JSON
  responses; required CI uses no live API key. Real endpoint compatibility is claimed
  only for services verified with the same contract tests.
