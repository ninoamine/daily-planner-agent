# CLI Contracts

The installed command is `daily-planner`. Commands use human-readable Rich output on
interactive terminals and plain text when output is redirected. All prompts require
confirmation only for persistent plan changes and destructive deletion. No command
contacts people or writes to external work systems.

## Commands

| Command | Contract |
|---|---|
| `daily-planner init` | Create the local data/config directories and initial database. Safe to repeat; never overwrites existing planning data. |
| `daily-planner capture [TEXT]` | Capture the supplied natural-language text. With no `TEXT` on an interactive terminal, read one conversational entry from the prompt. Save the original text before optional LLM extraction; show extracted items and any needed clarification. |
| `daily-planner items list [--status STATUS] [--objective ID] [--search TEXT] [--all]` | List captured work; default includes active/inbox items. `--all` includes done and archived items. Search includes retained history. |
| `daily-planner items show ITEM_ID` | Show the captured item, source text, known details, status, linked objectives, and unresolved clarification. |
| `daily-planner items edit ITEM_ID` | Prompt for editable fields or accept explicit field options including repeatable `--blocked-by ITEM_ID` and `--remove-blocker ITEM_ID`; unknown/ambiguous values can remain unset. Reject self-dependencies and cycles without changing existing links. |
| `daily-planner items complete ITEM_ID` | Mark work done and record completion time. |
| `daily-planner items archive ITEM_ID` | Archive without deleting; archived items remain searchable. |
| `daily-planner items restore ITEM_ID` | Restore archived work to actionable status. |
| `daily-planner items delete ITEM_ID` | Permanently delete the selected item after explicit confirmation; if other items depend on it, list them and require confirmation to remove those links too. No bulk delete in v1. |
| `daily-planner objectives add` | Create an objective with title, outcome, and `short`, `mid`, or `long` horizon. |
| `daily-planner objectives list [--status STATUS]` | List objectives and progress freshness. |
| `daily-planner objectives progress OBJECTIVE_ID --note TEXT` | Record user-confirmed progress and update last-progress time. |
| `daily-planner commitments add` | Record a promise, person label, optional due date, and optional item/objective links. |
| `daily-planner waiting add` | Record an expected response/action, person label, optional follow-up date, and optional links. |
| `daily-planner brief [--date DATE] [--available MINUTES] [--protect MINUTES]` | Generate and display a proposed daily plan with rationale, risks, and objective alerts. Persist as `proposed`; do not change the accepted plan. Interactive sessions may accept, edit/reorder, or reject. Non-interactive use prints the proposal ID and requires a separate plan command. |
| `daily-planner replan --item ITEM_ID` | Propose an updated plan using an already captured interruption. Show the diff, moved/postponed work, and consequences. The accepted plan remains active until confirmation. |
| `daily-planner plan show PLAN_ID` | Show a proposal or historical plan, including its status and change comparison where applicable. |
| `daily-planner plan accept PLAN_ID` | Explicitly confirm and apply a proposal; only this operation changes the accepted plan. |
| `daily-planner plan edit PLAN_ID` | Edit/reorder a proposal interactively, then require explicit confirmation before accepting the revised proposal. |
| `daily-planner plan reject PLAN_ID` | Reject a proposal without changing the accepted plan or incrementing deferral counts. |
| `daily-planner advise delegate ITEM_ID` | Request delegation advice for a captured item. Display a role/team suggestion and rationale only; never contact or assign anyone. |
| `daily-planner config show` | Show effective non-secret settings; mask or omit secret values. |
| `daily-planner config validate` | Check required LLM settings and perform a minimal test request. Do not expose the API key in output. |
| `daily-planner export --format json\|markdown [--output PATH]` | Export a versioned portable copy. If `--output` is omitted, write to stdout. Never include secrets. |
| `daily-planner backup --output PATH` | Create a consistent SQLite backup at the specified path. Do not copy a live DB file directly. |

`--available` and `--protect` are nonnegative minute counts. The protected amount must
not exceed available time. Date inputs use ISO `YYYY-MM-DD`; a date without a time uses
the user's configured timezone and a documented end-of-day interpretation for deadlines.
Dates the parser cannot interpret unambiguously remain unset and are shown for correction.

## Confirmation and Error Semantics

- A generated proposal is not an accepted plan. In non-interactive use, `brief` and
  `replan` never infer confirmation from command success or output redirection.
- `plan accept` performs a final deterministic validation before changing state.
- Dependency edits reject self-links and cycles. Schedule generation will not schedule a
  dependent item until every prerequisite item is complete. An archived or deleted
  prerequisite is not considered complete.
- Invalid IDs, options, or field values print actionable errors and exit with code `2`.
- Local configuration or persistence failures print errors and exit with code `3`;
  the application does not claim data was saved.
- LLM connection, timeout, or response validation failures print errors and exit with
  code `4`. Existing data and the accepted plan remain intact; local-only commands remain
  usable.
- Exit code `0` indicates the requested operation completed. A proposal-producing
  command can succeed with code `0` while leaving the proposal unaccepted.
- Errors go to stderr. Rich styling is disabled or simplified when stdout is not a TTY.

## Configuration Inputs

The TOML file holds the user-selected LLM `base_url` and `model`, timezone, and planning
defaults. `DAILY_PLANNER_LLM_API_KEY` provides the secret. Environment variables override
TOML; explicit command options override both. A configured compatible service is needed
only for LLM-assisted operations.
