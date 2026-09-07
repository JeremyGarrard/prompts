# dbt project build instructions

You are working inside this dbt project. Your job is to make it safe for machines to build
and query: proven grains, an enforced contract on the analyst table, isolated builds by
object prefix, and a semantic layer whose consumer-facing metrics are described. Work in
the order below. Each phase is one pull request, or several small ones. Do not skip ahead.

Read `DBT_STATE.md` first if it exists; it is the inventory this plan was written against.

## Ground rules

- Never build to production. Every build uses the sandbox target and an object prefix (Phase 3
  creates it; until then use the existing manual prefix variable with a value that starts with
  your branch name).
- Never relax a test, delete a test, or widen an accepted range to make a build pass. If a
  test fails, the model or the data is wrong; say which and stop.
- Never change a model's SQL in the same pull request as its tests, except in Phase 2 where the
  change is the point. Test migrations must be behavior-neutral and proven so.
- Do not rename models, columns, or metrics. Consumers and saved queries depend on them.
- Do not add Snowflake Tasks, Streams, Dynamic Tables, or stored procedures.
- Keep every pull request under about 400 changed lines of YAML or SQL so a person can review
  it. Split by folder if needed.
- Every pull request description states: what changed, how it was verified, the sandbox
  prefix used, and the run results summary (models built, tests passed, tests failed).

## Phase 1: tests that prove grain and range

### 1a. Migrate generic tests to the `arguments:` form

Behavior-neutral. Prove it:

1. `dbt compile` on the current branch base. Copy `target/compiled/**/*.sql` for tests to a
   scratch folder.
2. Rewrite every parameterized generic test (`accepted_values`, `relationships`,
   `dbt_utils.*`, custom generics) from inline arguments to `arguments: {...}`. Leave
   `unique` and `not_null` alone unless they carry arguments.
3. `dbt compile` again. Diff the compiled test SQL against the scratch copy. The diff must be
   empty apart from whitespace. If it is not, you changed behavior; fix it before continuing.
4. Confirm `dbt parse` reports zero deprecation warnings for the inline-argument form.

### 1b. State every model's grain

For each model whose YAML has neither a `unique` test nor a `unique_combination_of_columns`
test:

1. Read the SQL and the upstream model or source. Determine the grain: the smallest set of
   columns that identifies one row.
2. Add a `dbt_utils.unique_combination_of_columns` test on those columns (or `unique` for a
   single column), plus `not_null` on each of them.
3. Build the model and its tests in the sandbox. If the uniqueness test fails, do not weaken
   it. Record the failure in the pull request as a data-quality finding with a row count of
   duplicates, and leave the test in place with `severity: warn` and a `meta.grain_note`
   explaining that duplicates exist upstream. That is a finding for the owner, not for you.
4. If a model genuinely has no grain (an append-only event feed with no key), add
   `meta: {grain: none, grain_note: "<why>"}` to its YAML instead of a test.

Staging models are the bulk of this. Their grain is usually the source key plus a period
column; the source documentation or an upstream `qualify` clause usually tells you which.

### 1c. Keys, foreign keys, ranges

- Every model with zero tests gets at least `not_null` on its grain columns.
- Every column that joins to a dimension in a mart gets a `relationships` test to that
  dimension. Start with the account and customer keys on the core facts and the analyst
  table.
- Every metric input that is a rate gets `dbt_utils.accepted_range` with `min_value: 0` and
  `max_value: 1` (or the documented ceiling if rates can exceed 1, such as utilization). Every
  amount that cannot be negative gets `min_value: 0`. Read the semantic-model measures to find
  these columns.
- The reconciliation singular test whose body was replaced with a zero-row query: restore the
  real comparison if the seed it needs still exists, or delete the test and say so in the pull
  request. A test that always passes is worse than no test.

### 1d. A full run artifact

Run `dbt build` for the whole project in the sandbox, with `--full-refresh` so incremental
models materialize from scratch under the prefix. Save `target/run_results.json` and
`target/manifest.json` under `docs/evidence/<date>/`. The pull request states the counts.
Zero failures is the target; if there are failures, list them by test with a one-line cause.

Done when: no deprecation warnings, every model has a stated grain or a documented reason,
the full-run artifact is committed, and every failure is attributed.

## Phase 2: the analyst table as a contract

### 2a. Resolve the columns emitted as NULL

The analyst table emits a set of legacy columns as typed NULL with TODO markers. For each:

1. Search exposures, saved queries, the semantic models, the report definitions, and the
   renderer for any reference to the column name.
2. If nothing references it, drop it from the SQL and YAML. List the dropped columns in the
   pull request.
3. If something references it, either fill it from the correct upstream (if you can identify
   one from the column name and the intermediate models) or leave it NULL and add it to a
   `docs/analyst_table_open_columns.md` list with the consumer that needs it. Do not guess a
   source.

### 2b. Dense account-period calendar

The analyst table only has rows for account-months present in the source. Inactive months
are missing rows, which makes every "share of open accounts" denominator wrong.

1. Add an intermediate model `int_account_month_spine`: one row per account per month-end
   from the account's open month through its close month or the latest period in the time
   spine, whichever is earlier. Build it from the account dimension and the existing time
   spine, not from activity. Grain: (account key, month-end). Test it.
2. Rebuild the analyst table as the spine left-joined to the activity, balance, and score
   models it already uses. Every measure that comes from activity is coalesced to zero;
   every flag that means "had activity" is derived from the join, not from the source.
3. The population and inactive-classification columns that are currently NULL become
   real: inactive-six-months is computed from the spine and activity, not carried from a
   source that no longer supplies it.
4. Keep the incremental strategy. The spine bounds the incremental window the same way the
   source did.
5. Verify against a small sample: pick five accounts, one that closed, one that opened in the
   window, one with a gap in activity. Query the old and new tables in the sandbox and show
   the row counts per account in the pull request.

### 2c. Enforce the contract

1. In the analyst table's YAML, add `config: {contract: {enforced: true}}` and a `data_type`
   for every column. Read the types from the current sandbox build with `describe table` and
   pin them exactly.
2. Add explicit casts in the SQL so the types are stable, especially for sums of decimals and
   counts. A contract that passes because of an implicit type is a contract that breaks
   later.
3. Build in the sandbox. Then, as a check, change one column's type in the YAML and confirm the
   build fails. Revert.
4. Repeat 2c for the core dimensions and facts once the analyst table is done.

### 2d. Fix the layering violation

An intermediate model reads reporting marts. Move whatever it needs from the reporting marts
into an intermediate model, or move the model itself to the reporting layer. Intermediate
never reads marts.

Done when: the analyst table has an enforced contract with every column typed, an account
open in a quiet month has a row with activity flags false, no NULL-forever columns remain
without a documented consumer, and no intermediate model refs a mart.

## Phase 3: isolation by object prefix

The profile writes everything into one flat schema and per-layer schemas are blocked by
grants. Isolation is by object name prefix, using the mechanism that already exists.

1. Add two profile targets, `sandbox` and `ci`, both pointing at the sandbox database and
   schema, both using the same authentication path as the current target.
2. Change the alias macro so that, on the `sandbox` and `ci` targets, the object prefix is
   taken from an environment variable `DBT_SANDBOX_ID` (for example a thread id or a branch
   name, sanitized to letters, digits, and underscores) and prepended as
   `<prefix>__<model>`. On every other target the macro behaves exactly as it does today.
   Prove that with a compile diff on the production naming path.
3. Add a macro `drop_prefixed_objects` callable with `dbt run-operation`, that drops every
   table and view in the sandbox schema whose name starts with the given prefix and nothing
   else. It must refuse an empty prefix.
4. Add one entry point, a script `build_sandbox` taking a prefix and a dbt selector, that:
   sets `DBT_SANDBOX_ID`, runs `dbt build --target sandbox --select <selector>
   --indirect-selection cautious --full-refresh`, copies `run_results.json` to a path named
   by the prefix, and exits with dbt's exit code. Document it in the README. This is the
   command the platform's data-engineering agent will call; it must not need a person.
5. Verify: run two builds with different prefixes at the same time on a small selector.
   Neither may see the other's objects. Then tear both down and confirm the schema holds no
   prefixed objects.

Done when: a proposal can be built and tested under a prefix beside production objects with
one command, and torn down with one command.

## Phase 4: semantic layer descriptions and catalog hygiene

1. Every metric the semantic models expose gets a `description` that states what it measures,
   its unit, and any exclusion, in one or two sentences a reviewer would accept. Do
   consumer-facing metrics first: the ones referenced by saved queries, exposures, and the
   report definitions.
2. Mark internal numerator, denominator, and component metrics so a catalog can hide them:
   `meta: {internal: true}` on each, and a naming note in the semantic README. Do not rename
   them.
3. Run `mf validate-configs` and commit the output under `docs/evidence/<date>/`. Fix anything
   it reports.
4. If the time spine only declares day granularity, add month, quarter, and year so offset
   metrics can be defined later.

Done when: every non-internal metric has a description, internal metrics are marked,
`mf validate-configs` is green and its output is committed.

## Report

At the end of each phase, write `docs/build_report_<phase>.md`: what was done, what was found
(data-quality findings, dropped columns, undetermined grains), what was verified and how, and
what is left. A reader will compare it against the plan's done-when checks.
