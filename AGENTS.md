# AGENTS.md

This is the operating contract for people and coding agents working in `springdale-dbt`. Follow it for every repository change unless a more specific `AGENTS.md` exists below the file being changed.

## Project purpose

Build a governed analytics layer for Springdale's business-development and M&A pipelines. Matia lands operational data in Snowflake. dbt Core owns the transformations, snapshots, tests, and documentation. Inngest orchestrates the production run on a native Python Vercel Function; GitHub Actions validates changes.

Start with [projects/springdale-pipeline-analytics.md](projects/springdale-pipeline-analytics.md) for the current requirements and open business decisions.

## Required data flow

```text
RAW → VIEWS → STAGING → REPORT → REVERSE_ETL
         └→ SNAPSHOTS ───┘
```

- Data moves forward through these layers. Do not skip a layer for convenience or introduce backward dependencies.
- Snapshots capture mutable source state exposed by the view layer. Staging may consume views, snapshots, seeds, and other staging models.
- Reverse-ETL models consume report models and must not reach around the report layer.
- Treat `RAW` as Matia-owned and read-only. Never create, update, or delete source records from dbt.
- Do not invent physical source schemas or columns before inspecting Matia's landed data.

## Model layers

### Sources

Declare every Matia relation in a source YAML file. Include descriptions, freshness rules where a reliable ingestion timestamp exists, and tests for source keys used downstream. Preserve source-system IDs and ingestion metadata.

### Views

`models/views/` is the protective contract over raw data.

- Maintain a 1:1 relationship between each view model and one raw source table.
- Materialize every model in this layer as a Snowflake view.
- Rename raw fields into stable, standard, snake-case column names.
- Cast types, normalize nulls, standardize timestamps, and create only simple row-level fields.
- Do not aggregate, join source tables, map business concepts, or change the source grain.
- Select columns explicitly so an upstream schema change is reviewed rather than silently propagated.
- Organize views by source system, such as `origent_hubspot`, `eagle_hubspot`, and `ma_airtable`.

Example names:

- `origent_hubspot__companies`
- `origent_hubspot__deals`
- `eagle_hubspot__companies`
- `ma_airtable__pipeline`

### Staging

`models/staging/` contains business logic. Use it for aggregation, mapping, identity resolution, lifecycle reconstruction, and conceptually distinct intermediate entity tables.

- Keep staging models to the minimum needed to make report models clear and reusable.
- Give each model one conceptual purpose and document its grain.
- Do not create pass-through staging models; that is the views layer's job.
- A staging model may depend on views, snapshots, seeds, or earlier staging models.
- Mapping models must end in `_mapping.sql`.
- Store small, governed manual mappings as seeds when appropriate.

Aggregation naming is semantic:

- `<concept>_by_<dimension>_<dimension>_month` is aggregated by those dimensions and month. It contains observed combinations only.
- `<concept>_by_<dimension>_<dimension>_monthly` is date-spined and contains every valid dimension combination for every month, including zero-value rows.

Examples:

- `opportunities_by_source_stage_month.sql`
- `opportunities_by_source_stage_monthly.sql`
- `pipeline_stage_mapping.sql`
- `company_identity_mapping.sql`

Every staging model must begin with one or more clearly named input CTEs using `select * from {{ ref(...) }}`. Staging must not call `source()` directly; the view layer owns raw-source access. This initial `select *` is allowed to make lineage obvious; downstream CTEs and the final projection should be explicit.

```sql
with opportunities as (

    select * from {{ ref('origent_hubspot__deals') }}

),

stage_mapping as (

    select * from {{ ref('pipeline_stage_mapping') }}

),

mapped as (

    select
        opportunities.opportunity_id,
        stage_mapping.canonical_stage
    from opportunities
    left join stage_mapping
        on opportunities.raw_stage_id = stage_mapping.raw_stage_id

)

select
    opportunity_id,
    canonical_stage
from mapped
```

### Report

`models/report/` contains the reporting layer used by dashboards, analysis, and governed consumers.

- Build entity tables and time-series tables from staging models or other report models. Do not reach around staging to views, snapshots, seeds, or raw sources.
- Joins and final reporting aggregations belong here when they are specific to the published output.
- Every report model must declare a single unique grain in its YAML description.
- Give every report model a stable key and test it with `unique` and `not_null` where the grain permits.
- Keep current-state entity models separate from historical time-series models.
- Do not repeat identity resolution, stage mapping, or other shared business logic across reports; move it into staging.

Initial report outputs are expected to include a current company/pipeline entity table and a daily funnel time series. Supporting opportunity and stage-event entity tables may be published where they provide a useful governed grain.

### Reverse ETL

`models/reverse_etl/` contains tables intentionally exposed for data leaving Snowflake.

- Include only the rows and columns required by the destination workflow.
- Exclude internal keys, sensitive fields, and helper columns unless the destination requires them.
- Document the destination, sync key, update behavior, and data owner.
- Give each model a stable unique key suitable for the outbound sync.
- Never place general-purpose reporting logic in this layer.

## Snapshots

Begin collecting pipeline history as soon as the corresponding raw integrations are available.

- Snapshot mutable pipeline state such as opportunities/deals, companies when relevant attributes change analytically, and Airtable pipeline records.
- Snapshot the standardized view rather than directly coupling snapshots to raw column names.
- Do not duplicate append-only activities, native property-history tables, or other relations that already preserve complete history.
- Prefer a trustworthy source `updated_at` timestamp when available; otherwise use an explicit check strategy over the fields whose history matters.
- Enable hard-delete tracking when deletion is analytically meaningful and supported by the chosen dbt version.
- Document snapshot grain, tracked fields, strategy, unique key, and known source limitations.
- Treat snapshot tables as historical inputs to staging, not as dashboard-ready outputs.

The goal is to preserve state transitions early, not to snapshot every column without a reason.

## Materializations

- The views layer is always materialized as views.
- Start staging and report models as views or full-refresh tables based on their semantics; optimize only after measuring a problem.
- Do not use incremental models at Springdale by default.
- Introduce incremental materialization only when runtime, volume, or warehouse cost becomes a demonstrated concern.
- An incremental model must define a stable `unique_key`, watermark, late-arriving-data strategy, delete behavior, and safe full-refresh path.

## SQL style

- Write SQL in lowercase and use snake case for identifiers. Preserve case only where a literal value requires it.
- Use named CTEs; do not use subqueries.
- Start staging models with `select *` input CTEs as described above.
- Use explicit column lists in transformations and final projections.
- Put the final `select` last and keep it simple.
- Prefix booleans with `is_` or `has_`; suffix timestamps with `_at`.
- Use `_id` for source/business identifiers, `_key` for warehouse surrogate keys, and `_count` for counts.
- Use fixed-precision numeric types for currency and retain currency codes when values may vary by currency.
- Avoid hard-coded mappings in SQL; use mapping models or seeds.
- Add comments only where they explain a non-obvious business rule or source-system behavior.

## Grain and identity

- State the grain in every staging, report, snapshot, and reverse-ETL model description.
- Do not treat a company as an opportunity. A company may have multiple opportunities over time or across platforms.
- Keep current state separate from event history.
- Preserve source record IDs through the report layer for traceability.
- Use deterministic surrogate keys with explicitly ordered inputs.
- Normalize analytical timestamps to UTC unless a documented requirement says otherwise.

Use normalized web domain as the primary cross-source company match, not company name. Apply matching precedence in this order:

1. An approved manual identity override or alias.
2. An exact normalized-domain match.
3. Another explicit deterministic identifier approved in the roadmap.

Never auto-merge companies solely on fuzzy name similarity. Surface ambiguous, missing-domain, and conflicting-domain records for review. Keep approved overrides in a version-controlled mapping seed or model with provenance and an explanatory note.

## Pipeline lifecycle and attribution

- Preserve raw pipeline and stage IDs, labels, source portal, and change timestamps.
- Map raw stages to governed canonical stages in a `_mapping.sql` model backed by seeds where useful.
- Fail quality checks for unmapped active stages instead of silently assigning them.
- Reconstruct `entered_at`, `exited_at`, and duration for each stage interval.
- Define explicit behavior for reopened, skipped, regressed, and simultaneously active opportunities.
- Never overwrite history when current state changes.

Keep these attribution concepts separate: `source_channel`, `capture_system`, `owning_platform`, and `owner`. Do not infer one from another without a documented rule. Retain raw and canonical values.

## Tests and reconciliation

At minimum, add:

- `unique` and `not_null` tests for model keys.
- `relationships` tests for report entity relationships.
- Accepted-value tests for governed categories and statuses.
- Source-freshness checks where timestamps are reliable.
- Tests preventing invalid or overlapping stage intervals.
- Tests surfacing unmapped stages and unresolved identity conflicts.
- Reconciliation queries comparing view-layer record counts and keys with Matia's raw relations.

Every bug fix should add a regression test when practical. Use singular tests for multi-row business invariants that generic tests cannot express.

## Schemas and development environments

Production model schemas follow the layers: `VIEWS`, `STAGING`, `REPORT`, `REVERSE_ETL`, and `SNAPSHOTS` in the dbt-owned database selected during setup.

Developer schemas use:

```text
dev_<schema>_<user>.<relation>
```

Examples:

- `dev_views_andrew.origent_hubspot__companies`
- `dev_staging_andrew.pipeline_stage_mapping`
- `dev_report_andrew.pipeline_funnel_daily`

Sanitize usernames to lowercase snake case in the schema-generation macro. Never build development models into shared production schemas.

## Secrets and access

- Never commit `.env`, `profiles.yml`, tokens, passwords, private keys, account recovery material, or copied production data.
- Read runtime credentials from environment variables.
- Prefer Snowflake key-pair authentication for Matia and dbt machine users.
- Keep Matia and dbt identities separate and least-privileged.
- Use read-only, base- or portal-scoped credentials for HubSpot and Airtable ingestion.
- Redact credentials and sensitive source values from logs, fixtures, and pull requests.

## Development workflow

1. Read the project roadmap and confirm the model grain and business definition.
2. Inspect source metadata and sample data before writing a model.
3. Develop in `dev_<schema>_<user>` schemas.
4. Add or update YAML documentation and tests with each SQL change.
5. Run `dbt deps`, `dbt parse`, and the narrowest valid `dbt build --select ...` command.
6. Build affected downstream models before requesting review.
7. Record new business decisions and unresolved questions in the roadmap.

Do not commit generated dbt artifacts such as `target/`, `logs/`, or downloaded packages.

## Deployment and change discipline

- Deploy dbt as a native Python Vercel Function, not a custom container function.
- Pin Python, dbt Core, `dbt-snowflake`, and Inngest SDK versions.
- Invoke dbt through its Python API where practical; do not introduce shell orchestration without a specific need.
- Configure the Vercel function for a 1,800-second maximum duration on a Pro or Enterprise project with Fluid compute. Extended duration is currently a beta capability, so verify project eligibility during setup.
- Keep the entire production sequence—source freshness, snapshots, and the dependent build/tests—inside one Inngest step unless an explicit cross-step lock is implemented.
- Configure Inngest production concurrency at one and use bounded retries for transient failures.
- Schedule dbt after Matia with an explicit buffer until a reliable completion trigger exists.
- Treat a normal runtime above 25 minutes as a migration trigger. Move only the runner to a true batch/container service while preserving the dbt project and Inngest orchestration.
- Store Snowflake credentials and the private key in Vercel environment variables. Use Vercel Static IPs if a Snowflake network policy requires egress allowlisting.
- A freshness, snapshot, test, build, or timeout failure must fail the Inngest run and alert the configured owner.
- Prefer small pull requests organized around one business outcome.
- Do not rename published report models or change their grain without a migration plan.
- Treat mapping changes as governed code changes requiring review.
- If a request changes opportunity grain, canonical stages, attribution, or historical behavior and the roadmap does not decide it, ask for the business decision rather than guessing.
