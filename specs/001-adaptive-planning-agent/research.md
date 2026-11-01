# Research: Adaptive Personal Planning Agent

## Decisions

### Python packaging and CLI

- **Decision**: Declare Python `>=3.12` and a console-script entry point in
  `pyproject.toml`; use uv for locked development and CI environments, and support
  installation with `uv tool install .` or `pipx install .`.
- **Rationale**: This supports one-command installation and repeatable project
  environments while isolating the end-user CLI from other Python packages.
- **Alternatives considered**: Plain pip/venv is viable but less convenient for the
  requested app-oriented install and lockfile workflow.
- **References**: [uv project layout](https://docs.astral.sh/uv/concepts/projects/layout/),
  [uv tools](https://docs.astral.sh/uv/guides/tools/),
  [pipx](https://pipx.pypa.io/stable/),
  [PyPA pyproject guide](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/).

### Application boundaries

- **Decision**: Keep a single package split into domain, application use cases,
  deterministic scheduling, advisor, SQLite storage, and Typer/Rich CLI.
- **Rationale**: The application boundary lets a future UI reuse use cases without
  placing CLI concerns in domain logic. The small layered package avoids premature
  services or abstractions.
- **Alternatives considered**: A web backend/frontend split and a plugin architecture
  add no v1 value. The latter is also prohibited by the constitution.
- **References**: [Typer tutorial](https://typer.tiangolo.com/tutorial/),
  [Typer testing](https://typer.tiangolo.com/tutorial/testing/),
  [Rich Console](https://rich.readthedocs.io/en/stable/console.html).

### Local storage and migrations

- **Decision**: Use Python's SQLite support and numbered SQL migrations tracked with
  `PRAGMA user_version`; apply each migration transactionally. Put the database in the
  per-user data directory and TOML settings in the per-user config directory.
- **Rationale**: SQLite is local, portable, and sufficient for one user's workload.
  `user_version` and small SQL migrations meet the migration requirement without an
  ORM or migration framework. Use the SQLite backup API for consistent database
  backups; expose portable, versioned JSON and Markdown exports separately.
- **Alternatives considered**: A third-party ORM/migration system is unnecessary for
  the initial schema size. Copying a live database file is not a reliable substitute
  for SQLite's backup operation.
- **References**: [platformdirs](https://platformdirs.readthedocs.io/en/latest/),
  [SQLite user_version](https://www.sqlite.org/pragma.html#pragma_user_version),
  [SQLite backup API](https://www.sqlite.org/backup.html),
  [Python sqlite3](https://docs.python.org/3/library/sqlite3.html).

### Configuration and secrets

- **Decision**: Read TOML using Python 3.12's `tomllib`; resolve configuration in the
  order defaults, TOML, environment, then explicit CLI option. Keep the API key in
  `DAILY_PLANNER_LLM_API_KEY` (or a user-controlled local secret environment) and
  never print or export it.
- **Rationale**: The standard library reads TOML without adding a writer dependency;
  environment variables avoid storing secrets in data exports or committed files.
  Store URL and model in TOML where convenient.
- **Alternatives considered**: A TOML writer is unnecessary because the app need not
  rewrite the user's settings. A local secrets file is permitted by the constitution
  but not the default.
- **References**: [Python tomllib](https://docs.python.org/3/library/tomllib.html),
  [platformdirs](https://platformdirs.readthedocs.io/en/latest/).

### LLM client and failures

- **Decision**: Use the official OpenAI Python SDK internally, configured with the
  user-provided `base_url`, API key, and model, targeting an OpenAI-compatible Chat
  Completions endpoint. Keep provider support at that wire-compatibility boundary;
  validate returned JSON locally with Pydantic rather than requiring provider-specific
  structured-output features.
- **Rationale**: The SDK supports a configured base URL and typed request/response
  handling. Local validation avoids requiring optional structured-output behavior that
  compatible services may implement differently. Contract-test each service the
  project claims to support; compatibility is not guaranteed by the label alone.
- **Alternatives considered**: HTTPX offers more transport control but requires the
  application to own more parsing and error mapping. Do not add a provider plugin
  framework. Add a provider-specific client only if a concrete compatibility need is
  demonstrated and governance permits it.
- **Failure policy**: Set a finite 30-second request timeout and at most two retries
  for retryable transient failures. Do not retry authentication, invalid-request, or
  response-validation failures. Return an explicit error and leave local data and the
  accepted plan untouched. Capture, browse, and deterministic plan operations remain
  available without network access.
- **References**: [OpenAI Python SDK](https://github.com/openai/openai-python),
  [HTTPX timeouts](https://www.python-httpx.org/advanced/timeouts/),
  [HTTPX transports and retries](https://www.python-httpx.org/advanced/transports/),
  [Pydantic models](https://docs.pydantic.dev/latest/concepts/models/).

### Testing and quality gates

- **Decision**: Use pytest for domain and integration tests, mock the LLM boundary,
  run contract tests without live credentials, and use stable snapshots only for
  briefing presentation. Run Ruff, mypy, and tests locally and in GitHub Actions;
  provide pre-commit as a convenience, not the only required gate.
- **Rationale**: This tests deterministic rules and external boundaries without network
  or credentials, while checking the requested CLI and formatted output.
- **References**: [pytest monkeypatch](https://docs.pytest.org/en/stable/how-to/monkeypatch.html),
  [pytest-mock](https://pytest-mock.readthedocs.io/en/latest/usage.html),
  [Ruff](https://docs.astral.sh/ruff/),
  [mypy](https://mypy.readthedocs.io/en/stable/getting_started.html),
  [pre-commit](https://pre-commit.com/),
  [GitHub Python CI](https://docs.github.com/en/actions/tutorials/build-and-test-code/python).

## Resolved Constraints and Open Assumptions

- v1 has no connector interface, plugin mechanism, or external source integration,
  including an unimplemented connector abstraction. This follows Constitution
  Principle VI despite the requested future connector interface.
- The LLM contract covers OpenAI-compatible Chat Completions. Azure, Ollama, corporate
  gateways, or other services are supported only when they implement the configured
  endpoint/model contract; provider-specific API variants are not implied.
- The selected initial license is MIT for simplicity. Maintainers should confirm the
  choice before public release if an explicit patent grant is needed.
- Local capture persistence, item listing, and deterministic scheduling have a measurable
  p95 target below 500 ms with 1,000 active items, excluding LLM network time. The
  advisor request timeout remains 30 seconds per attempt.
