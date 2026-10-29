# springdale-dbt

`springdale-dbt` is the dbt Core project for Springdale's business-development and M&A analytics in Snowflake. It will turn operational data from HubSpot and Airtable into a governed company master, pipeline history, and reporting-ready funnel models.

The first use case is documented in [Springdale pipeline analytics](projects/springdale-pipeline-analytics.md). Repository conventions for people and coding agents live in [AGENTS.md](AGENTS.md).

## Architecture

```text
Origent HubSpot ─┐
Eagle HubSpot ───┼─> Matia ─> Snowflake RAW ─> dbt Core on Railway ─> TRANSFORM ─> MARTS
M&A Airtable ────┘                                                       │
                                                                         └─> Snowflake MCP / BI
```

- **Matia** extracts source data and owns loading into Snowflake's `RAW` database.
- **dbt Core** validates, standardizes, joins, tests, and documents that data.
- **Railway** runs the versioned dbt Docker image on a schedule and stores runtime environment variables.
- **GitHub Actions** will run pull-request validation before changes are merged.
- **Snowflake** is the system of record for transformed and reporting-ready data.

The initial schedule will be deliberately simple: Matia loads at the top of the agreed interval, then Railway runs dbt after a buffer. A dbt run should execute source-freshness checks before `dbt build` and exit when complete.

## Initial data products

The reporting layer is expected to include:

- `dim_company`: one governed row per canonical company.
- `fct_pipeline_opportunity`: one row per business-development or M&A opportunity.
- `fct_pipeline_stage_event`: one row per stage interval or stage transition.
- `rpt_company_pipeline_current`: the current company and pipeline state for operational reporting.
- `rpt_pipeline_funnel_daily`: a daily time series of inventory, entries, exits, conversion, and aging by stage and attribution dimensions.

The opportunity and stage-event facts are intentional. A company table and a funnel time series alone cannot faithfully represent multiple opportunities, reopened processes, or stage history.

## Planned repository layout

```text
.
├── AGENTS.md
├── README.md
├── dbt_project.yml
├── packages.yml
├── macros/
├── models/
│   ├── staging/
│   │   ├── origent_hubspot/
│   │   ├── eagle_hubspot/
│   │   └── ma_airtable/
│   ├── intermediate/
│   │   ├── identity/
│   │   └── pipeline/
│   └── marts/
│       ├── core/
│       └── reporting/
├── projects/
├── seeds/
├── snapshots/
└── tests/
```

Only the documentation scaffold exists initially. Source names and physical schemas will be added after Matia has provisioned the integrations and we have profiled the landed data; they should not be guessed in advance.

## Environments and access

Snowflake: [Springdale Snowflake](https://app.snowflake.com/ubsxwsi/lj73493/#/homepage)

We will use separate machine identities and least-privilege roles:

- A Matia service user that can write only to the raw landing area it owns.
- A dbt service user that can read raw data and build the transformation and mart layers.
- Read-only reporting roles for analysts, applications, and MCP/BI consumers.
- Separate developer credentials or roles for local development.

Prefer Snowflake key-pair authentication for machine users. Keep credentials in Railway or CI secrets and reference them from `profiles.yml` through environment variables. Never commit tokens, passwords, private keys, `.env` files, or a populated dbt profile.

## Local development

Once the dbt project is initialized, the expected workflow will be:

```bash
dbt deps
dbt debug --target dev
dbt build --target dev
```

Pin the dbt Core and `dbt-snowflake` versions in the runtime image. Developers should build into isolated Snowflake schemas rather than shared production schemas.

## Current prerequisites

- [ ] Provision the Matia account and Snowflake destination.
- [ ] Create the Matia Snowflake service user and role.
- [ ] Create the dbt Snowflake service user, role, warehouse, and developer access pattern.
- [ ] Create read-only HubSpot private-app credentials for the Origent portal.
- [ ] Create read-only HubSpot private-app credentials for Nate's Eagle portal.
- [ ] Create an Airtable personal access token scoped to the required M&A bases.
- [ ] Confirm the canonical stage map, opportunity grain, and attribution definitions with business owners.

See the [pipeline analytics roadmap](projects/springdale-pipeline-analytics.md) for the source inventory, transformation plan, decisions, and definition of done.
