# dbt project inventory request

You are documenting the current state of this dbt project for an architecture review. Produce a
single Markdown document called `DBT_STATE.md`. Describe what exists, not what is planned. Do not
include SQL bodies, credentials, hostnames, account identifiers, row counts, or any customer
data. Model names, column names, file paths, macro names, package names, and configuration keys
are fine.

Where something does not exist, write "not built". Where you infer rather than read, say so.

## 1. Project shape

- dbt version, adapter and version, packages in `packages.yml` with versions.
- `dbt_project.yml`: model paths, seed paths, default materializations by folder, custom
  schema or alias macros, vars, on-run hooks, required dbt version.
- Profile targets (names and types only, no credentials): dev, ci, prod, sandbox, other. For
  each: which database and schema pattern it writes to, and which role it runs as, if declared.
- How the project is run: manually, a wrapper script, a scheduler. Name the mechanism and where
  it lives. Does anything call MetricFlow, and how (CLI, wrapper, API)?

## 2. Sources

For each `source`: name, database and schema, tables declared, whether `loaded_at_field` and
freshness thresholds are set, who loads it (name the tool or team if the repo says; otherwise
"not declared"). Note any source table referenced in SQL that is not declared as a source.

## 3. Models by layer

One table, one row per folder or layer: folder, purpose in one sentence, number of models,
materialization, naming convention, whether models in it join across sources.

Then for every model, one row: name, layer, materialization, grain in one sentence (what one row
represents), upstream refs and sources, and whether it has a YAML entry. Mark any model whose
grain you cannot determine from the SQL.

## 4. Tests and contracts

- Counts by test type: unique, not_null, accepted_values, relationships, accepted_range,
  singular tests, custom generic tests. Which models have zero tests.
- Which models have `contract: enforced`. For those, whether every column has a `data_type`.
- Whether tests use the `arguments:` form or the older inline form.
- Any test currently failing, warning, or disabled, as far as the repo shows (a `dbt build`
  log, a CI artifact, a commented-out test).
- Seeds: names, purpose, whether column types are pinned, whether they are tested.

## 5. Portability

- Every use of a vendor-specific function or syntax (for example `iff`, `qualify`, `dateadd`,
  `datediff` without the dbt macro, `try_cast`, `flatten`, `lateral`, variant paths). List by
  model. This decides whether the project can run on a second engine for CI.
- Every custom macro: name, purpose, and whether it dispatches by adapter.
- Any use of Snowflake Tasks, Streams, Dynamic Tables, or stored procedures, in the repo or in
  hooks.

## 6. Semantic layer

- Where semantic models and metrics live (paths). For each semantic model: name, model it sits
  on, entities with type, dimensions, measures with aggregation. For each metric: name, type
  (simple, ratio, derived, cumulative, conversion), inputs, whether it has a description, and
  whether it has a label.
- Time spine configuration: which model, which column, granularities.
- Whether `mf validate-configs` passes, as far as the repo shows.
- Any Snowflake semantic view or OSI export defined here, or "not built".
- Metrics that consumers use but that are not defined here (for example computed in a
  dashboard or notebook), if the repo or docs mention them.

## 7. The analyst table

Is there a table intended as the primary analysis surface, one row per entity per period, with
pre-joined dimensions and pre-computed measures? If so: name, grain, whether it includes inactive
periods, its contract and tests. If not, name the models analysts most often query directly.

## 8. Environments, CI, and change process

- What runs on a pull request, if anything, and what blocks a merge.
- How a change reaches production: who runs what, from where.
- Whether a state-based or slim CI exists (`state:modified+`, defer).
- Whether a per-PR or per-branch schema pattern exists.
- Branch protection, code owners, review requirements, as far as the repo shows.

## 9. Access

- Roles and warehouses referenced anywhere in the repo or its docs, and what each is used for.
- Whether a read-only consumer role for downstream tools exists, or "not declared".
- Where credentials come from at runtime (env vars, profile file, secret manager, key pair).

## 10. Diagram

One Mermaid diagram of the lineage from sources to the analyst table and the semantic layer,
grouped by layer, with the materialization noted on each node.

## 11. Gaps and open questions

Models without tests, models without YAML, sources without freshness, vendor-locked SQL, missing
contracts, missing CI, unclear ownership, anything hard-coded to one environment, and decisions
that appear unmade.

Keep it factual and specific. A reader will compare this against a reference implementation that
has enforced contracts, cross-engine SQL, a sandbox target, and CI on every change, so precision
about what exists matters more than narrative.
