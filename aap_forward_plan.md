# Agentic analytics platform: forward plan

Written against the architecture-state document of 2026-09-07 and a reference implementation
that runs the same design end to end on a laptop. Names are generic on purpose: "the platform
repo" is the control plane, "the dbt project" is the sibling project that owns the warehouse,
transformations, and the MetricFlow manifest, "the gateway" is the enterprise LLM endpoint.

The plan has two tracks that run in parallel and one dependency between them: the platform's
data-engineering agent cannot be real until the dbt project has an isolated build target. Everything
else in each track is independent.

Each phase lists what to build, what blocks it, and a "done when" check. Do phases in order
inside a track. Do not start a phase whose blocker is unresolved; build the unblocked phases
around it.

---

## Principles carried forward

1. One definition per metric, in the dbt project's semantic layer. The platform never
   re-derives a metric in its own SQL or Python.
2. The platform reads; only dbt writes. The platform's warehouse access is MetricFlow plus,
   if granted, a select-only role on marts.
3. Deterministic answers are released at once with the SQL that produced them. Generated
   answers wait for a human. Questions the data cannot answer become proposals, never guesses.
4. One routing path. The legacy one-shot path is retired, not maintained alongside.
5. Append-only events are the record. Any projection (Monday, a dashboard) is a view of the
   events, never the other way round.
6. Portable SQL in the dbt project so a second engine can run CI without warehouse credentials.
7. Every phase ships with tests that run without a model or a warehouse.

---

## Track A: the platform repo

### A0. Consolidate routing and fix reuse

Blocked by: nothing.

- Retire the one-shot spec path and its registry. The conversational input agent becomes the
  only intake for chat, JSON, and MCP. Delete the legacy classifier once its tests are ported.
- Replace the dedup fingerprint with a spec hash over: route, sorted metrics, sorted group-by,
  sorted filters (field, operator, values), time range, comparison, schema version. Add a reuse
  window (default seven days) and require the prior thread to be trusted (deterministic or
  approved). Record `reused_from` on the new thread.
- Make the audit log the single lifecycle source. Derive status by replaying events; the Monday
  cache and session store become projections and stop carrying lifecycle of their own.
- Lifecycle events: asked, aligned, guarded, clarified, confirmed, routed, reused, answered,
  delivered, review_requested, reviewed, proposed, built, blocked, refired, error.

Done when: one classifier exists; a comparison or filter change produces a different hash; a
replayed answer older than the window is recomputed; every status in the UI is derived from
events alone.

### A1. Trust surface on the deterministic path

Blocked by: nothing.

- Capture the compiled SQL from MetricFlow with the explain flag on every deterministic run and
  return it with the rows. Store it with the result.
- Add `assumptions` and `work_required` to the aligned ask, and show assumptions on the confirm
  card. A reviewer catches a wrong default by reading them.
- Implement comparison in the deterministic path: run the confirmed window and the prior window
  (prior period or prior year) as two governed queries, join on the non-time dimensions, emit
  value, prior, change, and change percent. Both compiled SQL statements are shown. Stop routing
  comparisons to a human.
- Reject a deterministic ask that names any metric, dimension, or grain outside the catalog as
  a logged error. Never fall through to the probabilistic path.

Done when: every deterministic answer in the UI has an expandable SQL block; a year-over-year
question is answered without a person; an unknown metric produces an error event and no query.

### A2. Escrow becomes a workflow

Blocked by: nothing.

- Persist held products with the executed SQL, caveats, and the agent's stated confidence.
- Reviewer queue for the analytics role: approve or reject with a note. Approval releases the
  product to the asker's history. Rejection records the note.
- Revise loop: a rejected thread re-enters alignment with the reviewer note as context and
  resubmits under the same thread id, so the log shows both decisions.
- Monday projection carries status and reviewer identity, no row data, as it does now.

Done when: a held answer is invisible to the asker until approved; a rejected answer can be
revised and resubmitted from the asker's history page; the log shows two review events.

### A3. Outputs, delivery, MCP, CI

Blocked by: nothing.

- Renderers: CSV and SQL always; Excel workbook with answer, spec, and SQL sheets; standalone
  HTML report. Files land under a per-thread output folder. Charts render from the same rows.
- Delivery adapters: chat (inline), file (path returned), MCP (rows in the response). Email and
  file-share are one function each when the platform team provides the transport.
- Network MCP server over the existing registry: tools ask, clarify, confirm, status, catalog.
  Callers pass identity and role; guardrails and escrow apply unchanged.
- CI workflow on every pull request: unit tests, a stub-model end-to-end run of all three
  routes, and a MetricFlow validate against a checked-in sample manifest. Protected branches
  already exist; wire the check as required.

Done when: a request for a spreadsheet returns a downloadable file; a second agent can ask a
question over MCP and get the same escrow status a person would; a failing test blocks merge.

### A4. Real tools for the probabilistic agent

Blocked by: a decision on warehouse access (see Decisions, D1).

Option 1, no new access: the agent's only data tool is MetricFlow (metrics, dimensions, filters,
windows), plus a Python sandbox over the returned CSV for ranking, thresholds, and narrative.
This is "no free SQL" applied to the agent and is the recommended first version.

Option 2, with a select-only role: add a read-only SQL tool with a statement guard (select or
with only, no stacked statements), a 200-row cap, and capture of every executed statement into
the held product.

Either way: citations are the executed queries, not a static marker; the product is untrusted
and enters escrow.

Done when: a ranking question produces a held answer whose evidence is a list of executed
queries a reviewer can rerun.

### A5. Real data-engineering agent

Blocked by: Track B phase B3 (sandbox target) and a branch on the dbt project.

- Tools: list project files, read file (dbt project only), run a governed query, write a
  proposal file (never into the live models folder), build_and_test.
- build_and_test stages the dbt project on a copy, overlays the proposal, and runs
  `dbt build --select +<new models>` against the sandbox target with cautious indirect
  selection. The agent iterates until models and tests pass or reports a source-data gap.
- The proposal is a branch plus a pull request draft: model SQL, YAML contract and tests,
  semantic model and metric additions, and a one-minute summary with the sandbox result.
- The original thread is parked as blocked on data.

Done when: a question about a nonexistent feature yields a branch that builds green in the
sandbox and a pull request a human can review; the live models folder is untouched.

### A6. Triggers, re-fire, hosting

Blocked by: platform decisions D3 and D4.

- Minimal version: the trigger seam is called after each successful production dbt run (a
  post-hook, a scheduler step, or a webhook). It re-aligns every thread blocked on data and
  answers the ones that have become deterministic.
- Full version: the orchestrator the platform team standardises on runs the dbt project and
  fires the re-fire job on success.
- Hosting: the HTTP server moves off the workstation to a container; identity comes from the
  identity provider rather than the OS user; entitlements come from group claims.

Done when: a question parked last week is answered the morning after its model lands, with no
person involved; the UI is reachable by a colleague.

---

## Track B: the dbt project

Re-scoped on 2026-09-07 after the dbt inventory. The project is larger and further along
than the platform document implied: roughly 140 models in a staging, intermediate, core,
reporting layering that already matches the reference design, a real analyst table at
account-month grain, a time spine, two semantic models, and about 550 metrics that MetricFlow
can compile. It is also single-target, single-schema, Snowflake-locked, thinly tested at the
grain level, human-run from wrapper scripts, and has no CI. Track B is therefore not "build a
data plane"; it is "make the one that exists safe for machines to build and query."

### B1. Tests that prove grain and range

Blocked by: nothing.

- Migrate every generic test to the `arguments:` form (mechanical; removes about 90
  deprecation warnings and future breakage).
- Every model with an undetermined grain (about 30, mostly staging) gets a uniqueness test that
  states its grain, or a comment explaining why it has none. Every untested model (about 17)
  gets at least key tests.
- `relationships` on every foreign key the marts join on; `accepted_range` on every rate and
  amount metric input; restore the hollowed-out reconciliation test or delete it.
- Run the full suite once and keep the artifact. Right now only a selected report suite has a
  retained passing run.

Done when: a full `dbt build` artifact exists with zero failures, every model has a stated
grain, no deprecation warnings.

### B2. The analyst table as a contract

Blocked by: nothing.

- Decide the fate of the legacy columns emitted as NULL: drop them or fill them. A contract
  cannot be enforced on columns nobody can type.
- Build a dense account-period calendar so inactive months exist as rows. Active rate,
  delinquency rate, and every "share of open accounts" metric need that denominator.
- Enforce `contract: enforced` with explicit types on the analyst table, then on the core
  dimensions and facts.
- Fix the layering violation where an intermediate model reads reporting marts.

Done when: a deliberate type change on the analyst table fails the build; an account open in
a month with no activity has a row with activity flags false.

### B3. Isolation for sandbox and CI builds

Blocked by: nothing for the prefix approach; a grants decision for the schema approach.

Per-layer schemas are blocked by grants and the profile runs everything into one flat schema.
Do not wait on that. The object-prefix mechanism that already exists is the isolation unit:

- A `sandbox` target that derives the object prefix from a thread or branch id, so a proposal
  builds as `<prefix>_<model>` beside production objects without touching them.
- A `ci` target that does the same from the pull-request number.
- Teardown: a wrapper step that drops every object with the run prefix.
- Keep the flat-schema flag; per-layer schemas can come later if grants allow.

Done when: two builds with different prefixes run at once and neither sees the other; the
platform data-engineering agent can build a proposal with a prefix and report test results.

### B4. CI on Snowflake, not on a second engine

Blocked by: a service identity for the CI runner (decision D6).

The SQL is Snowflake-locked in almost every model (window qualifiers, date arithmetic, cast
syntax, generators, list aggregation, grouped-by-all). Porting 140 models to cross-engine
macros is weeks of work with little payoff. Do not build a DuckDB lane for this project.

- CI runs on Snowflake with the prefix isolation from B3: `dbt build --select state:modified+`
  with defer to the last production manifest, in the PR prefix, torn down on merge.
- The Kerberos-to-short-lived-token wrapper does not work for a headless runner. CI needs a
  service identity with its own credential path, which is a platform ask.
- Pin the utility package to an exact version and commit the lock file.
- Branch protection with the CI check required; a CODEOWNERS file naming the analyst owner.

Done when: a pull request that breaks a test cannot merge; the runner authenticates without a
person.

### B5. The semantic layer as the interface

Blocked by: nothing.

- Four hundred of the 550 metrics have no description. The platform alignment step reads
  descriptions to pick metrics; undescribed metrics are invisible to it in practice. Write
  descriptions for every consumer-facing metric first, then the internal component metrics.
- Separate consumer-facing metrics from internal numerator and denominator components by
  naming or metadata so the catalog the platform shows is short and readable.
- Run `mf validate-configs` and keep it green. Add month, quarter, and year granularities to
  the time spine if any consumer needs offset metrics.
- Publish the analyst-table semantic model as a Snowflake semantic view (OSI export when the
  account has it, DDL until then).

Done when: every metric the platform can route to has a description a reviewer would accept;
the catalog the platform shows has fewer than a hundred entries; a question answered in the
platform and in Cortex agree.

### B6. Sources, scheduling, and access

Blocked by: D1 (reader role), D3 (orchestrator), and source owners for freshness fields.

- Freshness: every source gets a `loaded_at_field`; sources without one get a documented
  owner and a reason.
- A production target in the profile so the documented production path is runnable, and a
  scheduler that runs it. The wrapper scripts become the scheduler steps, not the scheduler.
- A select-only consumer role on the core and reporting marts plus the semantic view, with one
  service user per consumer.

Done when: the production build runs without a person on the schedule; the reader role can
select the analyst table and nothing upstream.

---

## Decisions needed, in order

| # | Decision | Who | Unblocks |
|---|---|---|---|
| D1 | Will the platform get a select-only warehouse role, or is MetricFlow-only the standing policy? | Data governance | A4 option 2, B5 |
| D2 | Which identity groups map to viewer, analyst, and admin? | Analytics leadership | A6 hosting, entitlements |
| D3 | Where does the platform run in production, and which orchestrator runs the dbt project? | Platform team | A6 |
| D4 | Is the gateway the production model transport, or is direct Bedrock coming? | Platform team | A4, A5 cost and latency |
| D5 | Is Monday the human-facing queue, with the event log as the record, or the other way round? | Analytics leadership | A0 projection design |
| D6 | Can a headless CI runner get its own warehouse identity, given the current human Kerberos-to-token path? | Platform team | B4 |

The recommendation on D1 is to start with MetricFlow-only and earn the reader role with the
escrow record. On D5, the event log is the record; Monday is a projection.

---

## What not to do

- Do not build a second warehouse connector inside the platform. The dbt project owns the
  connection; the platform reaches it through MetricFlow and, if granted, one reader role.
- Do not keep two routing paths for compatibility. Port the tests and delete the old path.
- Do not let the data-engineering agent write into the live models folder, even on a branch,
  until B3 exists.
- Do not add Snowflake Tasks, Streams, or Dynamic Tables to fill the orchestration gap. One
  scheduler.
- Do not relax a contract or delete a test to make a build pass.

---

## Sequencing summary

Week one to two: A0, A1, B1 in parallel. These are pure code and fix the two correctness gaps
the state document called out.

Week three to four: A2, A3, B2, B3. The platform becomes something a colleague can use and
review; the dbt project becomes something a machine can build in isolation using object
prefixes.

After that: A4 in its MetricFlow-only form, A5 once B3 exists, B5 in parallel because it
directly improves alignment quality, B4 and B6 as the identity and access decisions land,
A6 when the platform decisions land.

Every phase leaves the system working. Nothing here requires a big-bang cutover.
