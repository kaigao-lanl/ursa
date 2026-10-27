# Change Log

## Unreleased

### New agent: ModelCardAgent (`#fm`)

- Added `ModelCardAgent`, which recognizes scientific foundation-model cards,
  reads a user's requirements, recommends models with explained scores, and runs
  the chosen model through its MCP tool or the `ExecutionAgent`. The TUI exposes
  it as `#fm`, and the dashboard lists it as "Foundation Model Advisor".
- Added the `ursa.model_cards` package. It reads Hugging Face repositories,
  URLs, local files, MCP servers such as Nomad, and URSA-native YAML cards in the
  Hugging Face, DOE Genesis, NVIDIA Model Card++, and free-form dialects. Cards
  become one normalized `ModelCard` with exact input and output formats,
  provenance for key facts, conflict and quality issues, and a completeness
  score. LLM refinement adds facts only with quotes checked against the card;
  a changed number never passes the check.
- Added a physics taxonomy with domain and task vocabularies that span the
  geosciences, continuum physics, plasma and space physics, astrophysics,
  materials, chemistry, and biomolecules; teams can extend it from
  configuration.
- Added explainable model ranking with weighted criteria, hard blockers for
  license, hardware, and size limits, and an evidence-bound LLM review.
- `ModelCardAgent` discovers models on every connected MCP server that offers
  `get_model_card`, even when several servers use that tool name;
  `load_mcp_tools_by_server` keeps MCP tools grouped by server. Discovered
  cards keep the deterministic reading, so the first answer needs no LLM call
  per model; `refine_discovered=True` adds the LLM review.
- In the dashboard, the agent reads only the newest message of the session
  transcript as the request and resolves "the top model" from the previous
  report, so an earlier run is never repeated; `ursa-dashboard --use-web` now
  enables its Hugging Face search.
- The agent answers questions about the catalog and the user's data: how many
  models the catalog holds and which match a filter, which models can use a
  data file, and whether a model can run inference on that data as is, after a
  stated conversion, or only after fine-tuning. Each answer starts with a
  verdict and cites the card lines and data facts behind it. A model runs only
  when a sentence of the request orders it, and a tool that takes one molecule
  or sequence runs once per item of the user's file.
- Added `ursa.model_cards.data`, which reads NetCDF, HDF5 (including The Well),
  NumPy, MATLAB, CSV, SMILES, FASTA, XYZ, CIF, PDB, image, and JSON files into a
  `DataProfile`, and `ursa.model_cards.compat`, which checks a profile against a
  card and picks the pathway. Cards now record the size and range of their
  training data (`training_scope`), so an answer warns when data lie outside
  that range.
- Added ten curated inference tasks for Nomad's demo server, with data, a
  scoring script, and figures, in `examples/model_cards/curated_tasks`.
- Added a shared group model catalog and five LangChain tools
  (`model_card_tools()`) that give any tool-using agent model-card skills.
- `TorchModuleTool` accepts a `model_card` and serves it through a
  Nomad-compatible `get_model_card` tool, so `ModelCardAgent` discovers models
  served by URSA and by Nomad in the same way.
- Added the `examples/model_cards` tutorial with a toy physics model served
  over MCP, a native card template, and a recognition benchmark of 24 public
  cards, plus documentation in `docs/agents/model-cards.md`.

### Fixes

- On Claude models, `invoke_structured` now tries Claude's native structured
  output (`method="json_schema"`) first for small schemas without nullable
  fields. Claude Opus 5.5, Sonnet 5.5, and Fable 5.1 reject forced tool calls,
  so LangChain's default structured output became an optional tool call that
  the model could skip, which cost retries. Larger schemas keep the previous
  methods: the API took three minutes to reject the requirements schema as
  "too complex".
- The dashboard now releases a session whose active run has ended when it
  starts. A restart during a run marked the run failed but left the session
  "Running": the page replayed the finished run's log in a loop, and neither
  "Cancel run" nor "Delete" could free the session.
- `llm_model.max_completion_tokens` now reaches Anthropic models as
  `max_tokens`. Under the OpenAI name, it reached the Anthropic API as an
  unknown field and the request failed.
- `DeepReviewAgent` emitted no progress events: its module used
  `from __future__ import annotations`, so LangGraph did not inject the
  `RunnableConfig` into graph nodes. The module now uses real annotations.
- `BaseAgent.aclose` now stops the async store's batch worker, which removes
  "Task was destroyed but it is pending" warnings at shutdown.
- `TorchModuleTool.batch_as_completed` accepts `max_concurrency`; the
  misspelled `max_concurency` keyword still works with a deprecation warning.
  The module's TypeVar names now match their variables, and `name` is typed as
  optional.
- Fixed a test in `tests/tools/test_fm_base_tool.py` that checked tool metadata
  without an `assert`, and a typo in the HITL unknown-agent message.
- A `BaseAgent` test no longer leaves `graph_store.sqlite` in the current
  directory; it now runs in a temporary workspace.

## v0.17.2

Changes since `v0.17.1`:

### New environment: Agent Elo

- Added Agent Elo, URSA's evolutionary competition environment: multiple agents
  work independently on the same task, are randomly paired each generation, and
  compete head-to-head; an LLM judge compares completed and timed-out competitors
  against the task's evaluation criteria to pick a winner or a draw, and Elo
  ratings track the standings (#321).
- Losing competitors leave the active population while survivors produce
  descendants that inherit their parent's files and persistent state, so
  solutions are developed and refined over successive generations (#321).
- Added validated `AgentEloConfig` configuration covering population size,
  unique member names, automatically managed member workspaces, and persistent
  agent identities, along with a runnable example project under
  `examples/environments/agent_elo` and documentation in
  `docs/environments/agent-elo.md` (#321).
- Dashboard environment runs now support Agent Elo end to end, including run
  management, the run UI, and worker wiring (#321).

### Configuration and security

- Fixed and hardened layered configuration merging: higher-priority layers no
  longer clobber or incorrectly promote lower-priority defaults, optional models
  can be completed across layers, and invalid or secret-valued overrides from
  higher-priority sources are rejected instead of silently inherited (#352).
- Switching a model between the `base_url` and `inference_provider` endpoint
  styles now cleanly supersedes the other style without turning the internal
  null into an explicit model override, and merged configurations no longer
  alias mutable inputs (#352).
- The default group can now have an endpoint allowlist: previously the default
  group always permitted every endpoint; it now follows a persisted group policy
  when one is configured and remains unrestricted when none is, with the
  configured policy validated like any other group's (#350).
- `ursa groups update` now supports persisting a policy for the otherwise
  implicit default group by creating its directories on demand (#350).

## v0.17.1

Changes since `v0.17.0`:

### Agents and tool reliability

- Hardened message-history preparation and summarization for broader provider
  compatibility: URSA now removes invalid mid-history system messages, orphaned
  tool results, and crashed-step prompt tails before model calls; keeps the
  system prompt first; and summarizes older tool-call transcripts as plain text
  instead of sending provider-invalid tool-call slices (#339).
- Fixed unregistered typed state declarations for Materials Project, RAG, and
  recall agents so state registration warnings/errors reflect real issues
  instead of agent implementation mismatches (#341).
- Added an async implementation for persisted RAG tools, skipped empty RAG
  progress bars, and installed a thread-only `tqdm` lock in the TUI path to avoid
  Textual stderr/resource-tracker `bad value(s) in fds_to_keep` failures and
  follow-on deadlocks during RAG use (#330, #337).
- Made `URSA_SAFETY_LEVEL=none` and `URSA_SAFETY_LEVEL=yolo` bypass the command
  safety LLM check entirely; the previous permissive prompt behavior is now the
  `trusted` safety level (#343).
- Closed async SQLite resources more completely by joining their worker threads,
  preventing lingering non-daemon threads from blocking shutdown (#339).

### Dashboard

- Returned HTTP 400 responses for invalid named-agent inputs and added matching
  browser-side validation/alerts for creating, saving, copying, and deleting
  named agents (#322).
- Added dashboard `--use-web` / `URSA_DASHBOARD_USE_WEB` support so
  dashboard-created agents, including Deep Review, can opt in to web, arXiv, and
  OSTI tools consistently with the CLI (#334).
- Fixed dashboard environment credential handling so safe secret references such
  as `api_key: {env: ...}` are accepted while literal API keys remain rejected in
  persisted or worker configuration (#327, #346).
- Initialized dashboard worker OpenAI chat and embedding clients with URSA's
  explicit HTTPX/truststore clients to avoid connection errors seen by some
  users on OpenAI endpoints (#346).

### Configuration, TUI, and inference providers

- Corrected chat and embedding model parsing so only known provider prefixes are
  split from `provider:model` strings, colon-containing model names are
  preserved, and users get clearer provider-inference errors with supported
  provider hints (#337).
- Fixed layered configuration merging for model settings, including sparse
  overrides, nested model objects, and the precedence between `base_url` and
  `inference_provider` (#337).
- Improved inference-provider validation errors by listing known models for a
  provider and separating endpoint/credential guidance from the underlying
  validation failure (#337).
- Fixed TUI prompt scrolling after multi-line edits and formatted total token
  counts with thousands separators (#337).

### Image handling and provider compatibility

- Converted SVG files passed to `read_image_tool` into PNG image payloads while
  preserving the original filename as a separate text block (#329).
- Stopped placing local workspace paths in `ImageContentBlock.file_id`, which is
  reserved for provider-side file references and could break providers such as
  Gemini that prioritize `file_id` over embedded image bytes (#340).

### Dependencies, documentation, and maintenance

- Capped `aiosqlite` below 0.22 pending upstream compatibility (#344),
  relaxed `langchain-mcp-adapters` to the 0.2 series, allowed `justext` 3.0.1
  through the 3.x series, and updated `uv.lock` accordingly.
- Updated docs and examples for dashboard web-tool opt-in and fixed several
  documentation typos (#333, #334).
- Updated the Ruff workflow to run the current lint-check target (#337).

## v0.17.0

Changes since `v0.16.4`:

### Terminal UI (TUI)

- Replaced the Rich REPL with a new Textual terminal application, split into
  focused modules with per-event "cards" for source code, command output,
  files, diffs, plans, tools, search, and agent/artifact activity. Added an
  expandable exception card, an explicit `exit` command, theming, a growing
  multi-line prompt, runtime hardening, and randomized startup tips with
  platform-aware keymaps (#315).
- Surfaced persistent named-agent sessions and agent routing directly in the
  TUI, and marked each user turn in the scrollback so prompts stay visible
  between agent output blocks (#311, #315).
- Hardened TUI teardown so a Mount dispatch racing app shutdown can no longer
  crash the app, and stabilized the associated timing tests (#324).

### Configuration and inference providers

- Added layered CLI configuration with XDG-based overrides and reusable,
  provider-aware inference-provider definitions, so models can be declared once
  and shared across agents and environments (#314).
- Added external secret references resolved from the OS credential store,
  including support for injecting secrets into MCP request headers (#314).
- Expanded the `print_config` command and hardened configuration resolution and
  source merging, with correct XDG config paths on Windows (#314).

### CLI

- Added an `ursa self` command to inspect and manage a `uv tool` installation,
  with `status`, `update` (preserving the install recipe), and `modify`
  (extras, extra packages, exact version, or Git ref) subcommands (#325).
- Added the running URSA version to the startup banner (#307).
- Reported missing/invalid API keys and model-initialization errors cleanly,
  without a traceback, for OpenAI and non-OpenAI endpoints (#302).

### Agents

- Made the acquisition agents concurrent (async search, materialization, and
  cached-item loading) and registered their typed graph state; added a
  `build_config` helper and `SourceTask`/`ProcessedSource` structures (#314,
  #315).
- Added an `UnregisteredAgentStateWarning` that flags `BaseAgent` subclasses
  declaring a typed state they never register (#297).
- Landed context summaries as framed human-role messages so a summarized
  history never ends on an assistant summary, keeping message sequences
  provider-valid (#309).
- Kept deep-review role/phase prompts out of the persisted message channel,
  ensured each debate phase leads with its own role prompt, and propagated
  phase model failures immediately instead of swallowing them (#308).
- Fixed `print_visited_sites` to return only the visited-sites update, so
  deep-review outputs report exactly one entry per real iteration (#313).
- Added cross-agent regression coverage pinning provider-valid message
  sequences across agents (#297).

### Observability

- Repaired the OpenTelemetry OTLP/HTTP export path (Tier 1 of #259): each export
  builds a private, per-call tracer provider (shut down in-call) that never
  touches the global provider, emits a real-time root span plus one GenAI
  semantic-convention child span per LLM call, and returns a structured,
  truthfully reported result. Endpoint resolution follows a documented
  precedence (parameter, field, then `OTEL_EXPORTER_OTLP_*` env vars); headers
  accept a mapping or an env-style string; a missing `otel` extra now warns with
  an install hint; and the inert `otel_metrics` constructor argument is
  deprecated (#305).
- Captured reasoning and cached-token counts from all usage carriers
  (langchain `usage_metadata` details, raw OpenAI Responses/Anthropic shapes,
  service-tier-prefixed keys) without double-counting (#316).
- Recorded error samples when `on_llm_error` has no matching start, so failed
  LLM calls are no longer silently dropped from the per-LLM metrics (#293).
- Emitted arXiv and web search progress events asynchronously via `aemit`
  (#320).

### Dashboard

- Gave the dashboard chat visible failure feedback so pre-run send errors name
  the real problem instead of silently orphaning the user's message (#301,
  #306).

### Documentation and maintenance

- Reorganized and expanded the getting-started, configuration (files/env,
  models, secrets, MCP), CLI/TUI, and reference (OTel, CLI) documentation; added
  a filterable example catalog with per-example READMEs and runnable project
  scaffolding; and added versioned-docs build hooks (#314, #315, #318, #320).
- Migrated packaged TUI stylesheets into the distribution, bumped `rich`,
  `langchain-mcp-adapters`, `textual`, and `keyring` dependencies, narrowed the
  `otel` extra to the OTLP/HTTP exporter, and added `hypothesis` to the dev
  group (#314, #315, #305).

## v0.16.4

Changes since `v0.16.3`:

### Agents and CLI

- Expanded the LAMMPS agent with user-supplied potentials, input templates and
  structure data, GPU launch support, potential-selection-only mode, automatic
  result summarization, improved repair history, and structured progress events
  (#277).
- Added richer default CLI logging with MIME-aware rendering for source code,
  command output, diffs, plans, and tool/agent progress. Fixed artifact
  rendering for named agents (#278, #288).
- Fixed PlanningAgent state and message accumulation, and simplified its
  review/revision flow to avoid invalid model message sequences (#285, #286).
- Added `--config` support to `ursa mcp-server` (#275).

### Dashboard

- Added secure API-key management using the operating-system credential store,
  while retaining environment-variable and keyless configuration options
  (#275).
- Added dashboard launching, monitoring, continuation, and cancellation of agent
  teams and symposia (#280).
- New sessions now require an explicit persistent or temporary workspace instead
  of receiving a hidden default workspace; legacy session workspaces remain
  recoverable (#282).
- Refined dashboard layout, controls, settings, and model-provider handling
  (#275, #276).

### Persistence

- Unnamed CLI sessions are now ephemeral and no longer create checkpoint
  databases. Named sessions remain persistent, with a migration warning for
  legacy workspace checkpoints (#279, #287).
- Persistent-agent SQLite databases now prune obsolete checkpoint history after
  successful runs, substantially reducing long-running storage growth (#286).

### Documentation and maintenance

- Reorganized and expanded the agent, logging, API, environment, persistence,
  installation, and tutorial documentation (#278).
- Added versioned documentation publishing and updated MCP server guidance
  (#281).
