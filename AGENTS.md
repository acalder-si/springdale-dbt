# AGENTS.md

This file is the operating contract for people and coding agents working in `springdale-dbt`. Follow it for every repository change unless a more specific `AGENTS.md` exists below the file being changed.

## Project purpose

Build a governed analytics layer for Springdale's business-development and M&A pipelines. Matia lands operational data in Snowflake. dbt Core owns every transformation from raw source models through reporting-ready marts. Railway runs the production dbt image; GitHub Actions validates changes.

Start with [projects/springdale-pipeline-analytics.md](projects/springdale-pipeline-analytics.md) for the current business requirements and open decisions.

## Ownership boundaries

- Treat `RAW` as Matia-owned and read-only. Never create, update, or delete source records from dbt.
- dbt owns `TRANSFORM` and `MARTS`; do not maintain production objects there by hand.
- Staging models belong in `TRANSFORM.STAGING`.
- Reusable intermediate models belong in `TRANSFORM.INTERMEDIATE`.
- Governed dimensions and facts belong in `MARTS.CORE`.
- Consumer-specific tables and views belong in `MARTS.REPORTING`.
- Do not invent physical source schemas or columns before inspecting Matia's landed data.

## Required model layers

### Sources

Declare every Matia relation in a source YAML file. Include source and table descriptions, freshness rules where an ingestion timestamp exists, and tests for source keys used downstream. Preserve the raw system identifier and ingestion metadata.

### Staging

Use one staging model per source relation. Staging may rename, cast, normalize nulls, standardize timestamps, and deterministically deduplicate duplicate extracts. It must not implement cross-source joins or business definitions.

Name models with a source-system prefix:

- `stg_origent_hubspot__companies`
- `stg_origent_hubspot__deals`
- `stg_eagle_hubspot__companies`
- `stg_eagle_hubspot__deals`
- `stg_ma_airtable__pipeline`
- `stg_ma_airtable__diligence`

### Intermediate

Use intermediate models for reusable business logic, especially identity resolution, stage mapping, and lifecycle reconstruction. Suggested models include:

- `int_company_identity_candidates`
- `int_company_identity_resolved`
- `int_pipeline_opportunities_unioned`
- `int_pipeline_stage_events_normalized`

Materialize a single-use, inexpensive model as ephemeral. Materialize reused or expensive models as views or tables based on measured performance, not habit.

### Core marts

Core marts define stable business entities and events:

- `dim_company`: one row per canonical Springdale company.
- `dim_pipeline_stage`: one row per canonical stage and lifecycle version.
- `fct_pipeline_opportunity`: one row per canonical opportunity.
- `fct_pipeline_stage_event`: one row per opportunity-stage interval or transition.

### Reporting marts

Reporting models aggregate core marts for a named consumer or use case:

- `rpt_company_pipeline_current`
- `rpt_pipeline_funnel_daily`

Do not put entity resolution or lifecycle reconstruction directly in a reporting model.

## Grain and contracts

- State the grain in every model description.
- Give every dimension and fact a stable primary key and test it with `unique` and `not_null`.
- Do not treat a company as an opportunity. One company may have multiple opportunities over time, by platform, or by acquisition process.
- Keep current state separate from event history.
- Preserve source-system record IDs in all core facts for traceability.
- Use deterministic surrogate keys with explicitly ordered input fields.
- Document whether timestamps are source-local, UTC, or Snowflake-native. Normalize analytical timestamps to UTC unless a requirement says otherwise.

## Company identity resolution

Use normalized web domain as the primary cross-source matching signal, not company name. Normalize case, protocols, `www`, paths, and trailing dots before matching.

Apply matching precedence in this order:

1. An approved manual identity override or alias.
2. An exact normalized-domain match.
3. Another explicit, deterministic identifier approved in the project roadmap.

Never auto-merge companies solely on fuzzy name similarity. Surface ambiguous, missing-domain, and conflicting-domain records in a data-quality model for review. Store approved overrides in a version-controlled seed with provenance and an explanatory note.

## Pipeline lifecycle

- Preserve each raw pipeline, stage ID, stage label, source portal, and change timestamp.
- Map raw stages to a governed canonical stage in a version-controlled seed or dimension.
- Treat unmapped active stages as a failing quality condition rather than silently assigning them.
- Reconstruct `entered_at`, `exited_at`, and duration for each stage interval.
- Define explicit behavior for reopened, skipped, regressed, and simultaneously active opportunities before implementing those cases.
- Do not overwrite history when a source record changes its current stage.

The initial canonical stage proposal is discovery input, not a final contract. Changing stage definitions or ordering requires updating the roadmap, mapping data, documentation, and affected tests in the same pull request.

## Attribution dimensions

Keep these concepts separate:

- `source_channel`: how the opportunity first arrived, such as Origent, Nate/Eagle, or third-party broker.
- `capture_system`: the system from which the record was extracted.
- `owning_platform`: the Springdale platform or team responsible for the process.
- `owner`: the person responsible for the current action.

Do not infer one of these values from another without a documented rule. Retain both raw and canonical attribution values.

## SQL conventions

- Write Snowflake SQL in lowercase, except literal values whose case is meaningful.
- Use explicit column lists; do not use `select *` in a production model.
- Organize transformations as named CTEs, with the final projection last.
- Prefix booleans with `is_` or `has_`, dates with a clear business noun, and timestamps with `_at`.
- Use `_id` for identifiers, `_key` for warehouse surrogate keys, and `_count` for counts.
- Represent currency using a fixed-precision numeric type and include the currency code when values may vary by currency.
- Avoid hard-coded source labels and stage mappings in SQL; use seeds or governed mapping models.
- Add comments only where they explain a non-obvious business rule or source-system behavior.

## Tests and reconciliation

At minimum, add:

- `unique` and `not_null` tests for model keys.
- `relationships` tests for fact-to-dimension keys.
- Accepted-value tests for governed categories and statuses.
- Source-freshness checks when reliable ingestion timestamps are available.
- Tests preventing negative or overlapping stage intervals where the business rules disallow them.
- Tests that identify unmapped raw stages and unresolved identity conflicts.
- Reconciliation queries comparing staged record counts and source keys with Matia's landed relations.

Every bug fix should add a regression test when practical. Use singular tests for multi-row business invariants that generic tests cannot express.

## Incremental models

Start with views or full-refresh tables until data volume or runtime justifies incremental logic. An incremental model must define:

- A stable `unique_key`.
- The source watermark used for change detection.
- A lookback or late-arriving-record strategy.
- How deletes and source corrections are handled.
- A safe full-refresh path.

Test both incremental and full-refresh behavior before deploying it.

## Secrets and access

- Never commit `.env`, `profiles.yml`, tokens, passwords, private keys, account recovery material, or copied production data.
- Read runtime credentials from environment variables.
- Prefer Snowflake key-pair authentication for Matia and dbt machine users.
- Keep Matia and dbt identities separate and least-privileged.
- Use read-only, base- or portal-scoped credentials for HubSpot and Airtable ingestion.
- Redact credentials and sensitive source values from logs, examples, fixtures, and pull requests.

## Development workflow

1. Read the relevant project roadmap and confirm the model grain and business definition.
2. Inspect source metadata and sample data before writing a model.
3. Develop in an isolated Snowflake schema, never directly in production schemas.
4. Add or update model documentation and tests with the SQL change.
5. Run `dbt deps`, `dbt parse`, and the narrowest valid `dbt build --select ...` command.
6. Run the affected downstream build before requesting review.
7. Record new business decisions and unresolved questions in the project roadmap.

Do not commit generated dbt artifacts such as `target/`, `logs/`, or downloaded packages.

## Deployment and orchestration

- Pin the Python, dbt Core, and `dbt-snowflake` versions in the Docker image.
- The Railway cron process must run a finite command and exit nonzero on freshness, test, or build failure.
- Run source freshness before the production build.
- Schedule dbt after Matia with an explicit buffer until a reliable completion trigger is available.
- Prevent overlapping runs; investigate any skipped run rather than starting concurrent production builds.
- Use Railway logs and error monitoring for failures. Never make a failed build appear successful.
- Pull requests must pass parse/compile and model tests against an isolated CI target before merge once CI is configured.

## Change discipline

- Prefer small pull requests organized around one business outcome.
- Do not rename published marts or change their grain without a migration plan.
- Treat stage mappings, identity overrides, and attribution mappings as governed code changes requiring review.
- If a request would change opportunity grain, canonical stages, attribution, or historical behavior and the roadmap does not decide it, stop and ask for the business decision rather than guessing.
