# Architecture inventory request

You are documenting the current state of this repository for an architecture review. Produce a
single Markdown document called `ARCHITECTURE_STATE.md`. Describe what exists, not what is
planned. Do not include source code, SQL, prompts, credentials, hostnames, account IDs, table
row counts, or any customer data. Component names, file paths, library names, and interface
shapes are fine.

Work through the repository and answer every section below. Where something does not exist,
write "not built" rather than omitting it. Where you are inferring rather than reading, say so.

## 1. Inventory

For every component in the repo, one row: name, what it does in one sentence, the language
and main libraries, where it runs (local, container, Lambda, Airflow worker, etc.), and its
status: running in production, running in a dev environment, prototype, or stub.

## 2. Data plane

- Warehouse: platform, databases and schemas the code reads from and writes to, and which
  role or service account does each.
- Ingestion: how data lands in the raw layer, what tool, who owns it.
- Transformation: is dbt used? If so: project layout by layer, materializations, how many
  models per layer, which models have tests and what kinds, whether any model has an
  enforced contract, which macros are custom, and whether any SQL is vendor-specific
  (Snowflake-only functions). If not dbt, describe what does the transformation.
- Semantic layer: does a metrics layer exist (MetricFlow, Snowflake semantic views, LookML,
  other)? List the metric names defined and where. If none, say where metric logic lives now.
- Orchestration: what schedules or triggers runs (Airflow, EventBridge, Snowflake Tasks,
  cron, manual). Describe the DAG or job graph in words: nodes and dependencies.
- Serving: how dashboards, tables, or feeds are exposed and to whom.
- Environments and CI: dev / test / prod separation, what runs on a pull request, what blocks
  a merge.

## 3. Control plane

- Intake: which channels exist (chat UI, MCP, listener, other). For each: what it receives,
  how identity and entitlements are attached, and what the normalized request looks like
  (field names only).
- Spec compilation: is a request turned into a structured spec? Which model, on which
  platform (Bedrock, direct API, other), with what output schema (field names). Is there a
  human confirm step before execution?
- Guardrails: what checks run on a request before execution (PII, entitlement, injection,
  scope) and what implements them.
- Logging and system of record: where requests, decisions, and outputs are recorded
  (Monday, a warehouse table, files). Is it append-only? What events or statuses exist?
- Routing: is there a deterministic / probabilistic / data-engineering split? What decides
  the route? What does "asked and answered" reuse look like, if it exists?
- Deterministic path: how a known-metric question is answered. Is the SQL generated or
  compiled from definitions? Is it shown to the requester?
- Probabilistic path: what tools the agent has, whether SQL execution is read-only and
  capped, whether every executed statement is captured, and whether outputs are held for
  human review before release. Who reviews, where, and what happens on rejection.
- Data-engineering path: what the agent can read and write, whether it builds and tests in
  a sandbox, whether it can touch production models, and how a proposal becomes a change.
- Output: formats produced (table, chart, Excel, HTML, code), delivery channels, and how
  outputs are logged.
- Orchestration of agents: hand-rolled loop, LangGraph, Claude Agent SDK, other. Describe
  the state machine in words: states and transitions.
- Blocked requests: what happens to a question that could not be answered for lack of data.
  Does anything re-fire it later?

## 4. Access and ownership

- Roles and service accounts in the warehouse and what each can read or write.
- The line between platform-team ownership and analyst ownership: which databases, schemas,
  repos, and jobs sit on each side.
- How secrets and tokens reach the code (env vars, secret manager, config files).

## 5. Interfaces

For each boundary between components, the interface in one line: what is passed, in what
shape, by what mechanism (function call, HTTP, MCP, file, table). Field names are fine.

## 6. Diagrams

Two diagrams in Mermaid: the data plane (sources to serving, with the ownership boundary
marked) and the request flow (intake to delivery, with any human gates marked).

## 7. Known gaps and open questions

What the code itself shows is incomplete, stubbed, or hard-coded. What decisions appear to
be unmade. Anything that only works in one environment.

Keep it factual and specific. A reader will compare this document against a target
architecture and a reference implementation, so precision about what exists matters more
than completeness of narrative.
