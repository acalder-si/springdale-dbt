# springdale-dbt

`springdale-dbt` is the dbt Core project for Springdale's business-development and M&A analytics in Snowflake. It will turn operational data from HubSpot and Airtable into governed company entities, pipeline history, and reporting-ready funnel models.

The first use case is documented in [Springdale pipeline analytics](projects/springdale-pipeline-analytics.md). Repository conventions for people and coding agents live in [AGENTS.md](AGENTS.md).

## Architecture

```text
Control: Inngest schedule ─> Vercel Python/dbt

Origent HubSpot ─┐
Eagle HubSpot ───┼─> Matia ─> RAW ─> VIEWS ─> STAGING ─> REPORT ─> Snowflake MCP / BI
M&A Airtable ────┘                         │                    │
                                          └─> SNAPSHOTS        └─> REVERSE_ETL
```

- **Matia** extracts source data and owns loading into Snowflake's `RAW` database.
- **dbt Core** maintains the protective view layer, snapshots mutable pipeline state, applies business logic in staging, and publishes report and reverse-ETL models.
- **Vercel** hosts a native Python function containing the pinned dbt Core and `dbt-snowflake` packages.
- **Inngest** schedules the run, limits production concurrency to one, retries transient failures, and records run status.
- **GitHub Actions** will run pull-request validation before changes are merged.
- **Snowflake** is the system of record for transformed and reporting-ready data.

The initial schedule will be deliberately simple: Matia loads at the top of the agreed interval, then Inngest invokes the Vercel function after a buffer. One production step runs source freshness, snapshots, and `dbt build` sequentially so runs cannot overlap.

## Initial data products

The reporting layer is expected to include:

- A governed company entity table with one row per canonical company.
- A pipeline opportunity entity table with one row per business-development or M&A opportunity.
- A pipeline stage-event entity table with one row per stage interval or transition.
- A current company/pipeline report for operational reporting.
- A daily funnel time series of inventory, entries, exits, conversion, and aging by stage and attribution dimensions.

The opportunity and stage-event tables are intentional. A company table and a funnel time series alone cannot faithfully represent multiple opportunities, reopened processes, or stage history.

## Planned repository layout

```text
.
├── AGENTS.md
├── README.md
├── api/
│   └── inngest.py
├── dbt_project.yml
├── packages.yml
├── requirements.txt
├── macros/
├── models/
│   ├── views/
│   │   ├── origent_hubspot/
│   │   ├── eagle_hubspot/
│   │   └── ma_airtable/
│   ├── staging/
│   │   ├── identity/
│   │   └── pipeline/
│   ├── report/
│   └── reverse_etl/
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
- A dbt service user that can read raw data and build `VIEWS`, `STAGING`, `SNAPSHOTS`, `REPORT`, and `REVERSE_ETL`.
- Read-only reporting roles for analysts, applications, and MCP/BI consumers.
- Separate developer credentials or roles using `dev_<schema>_<user>` schemas.

Prefer Snowflake key-pair authentication for machine users. Keep credentials in Vercel or CI secrets and reference them from `profiles.yml` through environment variables. Never commit tokens, passwords, private keys, `.env` files, or a populated dbt profile.

## Local development

Once the dbt project is initialized, the expected workflow will be:

```bash
dbt deps
dbt debug --target dev
dbt build --target dev
```

Pin the Python, dbt Core, `dbt-snowflake`, and Inngest SDK versions used by the Vercel function. Developers should build into isolated Snowflake schemas rather than shared production schemas.

## Current prerequisites

- [ ] Provision the Matia account and Snowflake destination.
- [ ] Create the Matia Snowflake service user and role.
- [ ] Create the dbt Snowflake service user, role, warehouse, and developer access pattern.
- [ ] Create read-only HubSpot private-app credentials for the Origent portal.
- [ ] Create read-only HubSpot private-app credentials for Nate's Eagle portal.
- [ ] Create an Airtable personal access token scoped to the required M&A bases.
- [ ] Confirm the canonical stage map, opportunity grain, and attribution definitions with business owners.
- [ ] Configure the Vercel project and Inngest production environment for the dbt runner.

See the [pipeline analytics roadmap](projects/springdale-pipeline-analytics.md) for the source inventory, transformation plan, decisions, and definition of done.
