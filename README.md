# dbt-duckdb-jaffle-shop-tutorial

`dbt` project example based on: https://docs.getdbt.com/guides/duckdb?step=1

Example repo: https://github.com/dbt-labs/jaffle_shop_duckdb

## Generate data

```sh
uv run jafgen 4
```

This command will generate 4 years of data.

## dbt commands

### `uv run dbt debug`

```sh
uv run dbt debug
```

Sanity check. Confirms dbt can find `dbt_project.yml` and `profiles.yml`, that
the config parses, and that it can actually open a connection to the DuckDB
database. Run this first when something is misconfigured.

### `uv run dbt seed`

```sh
uv run dbt seed
```

Loads the CSV files in `seeds/` (`raw_customers`, `raw_orders`, `raw_payments`
— the output of `jafgen`) into the database as tables. This is the raw source
data every model is built on top of, so run it before `dbt run`.

### `uv run dbt run`

```sh
uv run dbt run
```

Executes the models in `models/`, in dependency order. The three `stg_*` models
in `models/staging/` are built as **views** and the `customers` / `orders` marts
as **tables** (per the materializations set in `dbt_project.yml`). This is the
core transform step — SQL in, tables/views out.

### `uv run dbt test`

```sh
uv run dbt test
```

Runs the data tests declared in the `schema.yml` files (e.g. `unique`,
`not_null`, `accepted_values`, relationship tests). Validates that the tables
`dbt run` produced actually hold — use it after building to catch bad data.

### `uv run dbt build`

```sh
uv run dbt build
```

The all-in-one command: runs `seed`, `run`, `snapshot`, and `test` together in a
single DAG. Each node is tested as soon as it's built, and a failing test stops
its downstream models. In day-to-day use this replaces running
`seed` → `run` → `test` separately.

### `uv run dbt deps`

```sh
uv run dbt deps
```

Installs the packages listed in `packages.yml` (e.g. `dbt_utils`,
`dbt_expectations`) into `dbt_packages/`. This repo has no `packages.yml` yet, so
there's nothing to install — but run it after adding one, and after every change
to it, before `run`/`build`.

### `uv run dbt snapshot`

```sh
uv run dbt snapshot
```

Runs the snapshots in `snapshots/` — Type-2 slowly-changing-dimension tables that
record how a row changed over time. None are defined in this project yet;
`dbt build` will call this step automatically once you add some.

### `uv run dbt source freshness`

```sh
uv run dbt source freshness
```

Checks how recently each declared source was loaded against the `freshness`
thresholds in the source's `schema.yml`, and reports warn/error. Only meaningful
once sources with freshness rules are defined.

### `uv run dbt ls` (alias `list`)

```sh
uv run dbt ls --resource-type model
uv run dbt ls --select staging
```

Lists the resources (models, tests, seeds, sources…) that match a selector —
**without building anything**. The fastest way to preview exactly which nodes a
`--select` expression will hit before you run it.

### `uv run dbt run-operation`

```sh
uv run dbt run-operation <macro_name> --args '{key: value}'
```

Invokes a macro from `macros/` directly, outside of a model. Useful for one-off
maintenance tasks (granting permissions, custom SQL) driven by dbt's Jinja
context.

### `uv run dbt parse`

```sh
uv run dbt parse
uv run dbt parse --no-partial-parse   # re-parse everything, ignore the cache
```

Reads and validates the **whole project** — every `.sql`, `.yml`, and `.md`
file — resolves `ref()`/`source()`/configs, builds the DAG, and writes
`target/manifest.json`, all **without touching the database or building
anything**. It's the metadata step every other command runs first; on its own
it's a fast sanity check that the project parses cleanly and surfaces config
errors and deprecation warnings. Add `--no-partial-parse` to re-parse from
scratch (the cache can skip unchanged files and hide warnings).

### `uv run dbt compile`

```sh
uv run dbt compile
```

Renders every model's Jinja/`ref()`/`source()` into the raw SQL that would be
sent to the warehouse and writes it to `target/`, **without executing anything**.
Handy for debugging what a model actually resolves to.

### `uv run dbt docs generate`

```sh
uv run dbt docs generate
```

Builds the documentation site data — a `manifest.json` and `catalog.json` in
`target/` describing every model, column, test, and the DAG.

### `uv run dbt docs serve`

```sh
uv run dbt docs serve
```

Serves the generated docs (run `docs generate` first) as a local website,
including the interactive lineage graph. Defaults to <http://localhost:8080>.

### `uv run dbt clean`

```sh
uv run dbt clean
```

Deletes the generated artifact directories listed under `clean-targets` in
`dbt_project.yml` (`target/`, `logs/`, `dbt_modules/`). Purely a housekeeping
step to start from a clean slate.

## Typical workflow

```sh
uv run jafgen 4        # 1. generate raw CSV data into seeds/
uv run dbt seed        # 2. load the CSVs into DuckDB
uv run dbt run         # 3. build the staging views and marts
uv run dbt test        # 4. validate the results
uv run dbt docs generate && uv run dbt docs serve   # 5. browse the docs + lineage
```

Steps 2–4 can be collapsed into a single `uv run dbt build`.

## Node selection (`--select`, `--exclude`, …)

By default `run`/`test`/`build`/`ls` act on **every** node. These flags narrow
that down to a subset of the DAG. They work the same across all those commands.

### `--select` (`-s`)

Build only the matching nodes.

```sh
uv run dbt run --select customers            # just this one model
uv run dbt run --select staging              # everything in models/staging/
uv run dbt run --select stg_orders orders    # multiple nodes (space = OR)
```

**Graph operators** pull in a node's neighbours in the DAG:

```sh
uv run dbt run --select stg_customers+   # the model AND everything downstream
uv run dbt run --select +customers       # the model AND everything upstream
uv run dbt run --select +orders+         # full lineage, both directions
uv run dbt run --select 2+customers      # only 2 layers of ancestors
```

**Selector methods** match by attribute (`method:value`):

```sh
uv run dbt run  --select path:models/staging     # by directory
uv run dbt build --select tag:nightly            # by config tag
uv run dbt build --select config.materialized:view
uv run dbt test  --select test_type:generic      # only schema.yml tests
uv run dbt build --select source:raw+            # a source and its children
```

### `--exclude`

The inverse — drop matching nodes. Combine with `--select`:

```sh
uv run dbt run --exclude staging                 # everything except the views
uv run dbt build --select +orders --exclude orders   # orders' deps but not orders
```

### `--selector`

Run a named, reusable selection defined in a `selectors.yml` file, instead of
retyping a long expression:

```sh
uv run dbt run --selector nightly
```

### Intersections

A single `--select` argument with **commas** (no spaces) means AND — a node must
match every part:

```sh
uv run dbt run --select staging,tag:nightly      # staging models that are ALSO tagged nightly
```

### `--state` / `--defer` (advanced)

Compare against a previous run's artifacts to build only what changed —
`--select state:modified+` rebuilds modified models and their children. Powers
CI "slim" runs; needs a saved `manifest.json` to diff against.

### Handy run flags

```sh
uv run dbt run --full-refresh    # rebuild incremental models from scratch
uv run dbt build --fail-fast     # stop at the first failure
uv run dbt --debug run           # verbose logging + the SQL being run
uv run dbt run --threads 4       # override the thread count from profiles.yml
```

> Tip: preview any selector with `dbt ls` before running it —
> `uv run dbt ls --select state:modified+` shows exactly what would build.

