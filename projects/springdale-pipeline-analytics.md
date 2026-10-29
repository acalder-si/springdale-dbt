# Springdale pipeline analytics

## Project status

- **Phase:** foundation and access setup
- **Primary system:** [Springdale Snowflake](https://app.snowflake.com/ubsxwsi/lj73493/#/homepage)
- **Transformation:** dbt Core in a native Python Vercel Function, orchestrated by Inngest
- **Ingestion:** Matia
- **Initial consumers:** Snowflake MCP and conversational analysis; a BI or custom dashboard can follow once definitions are stable

## Objective

Create one governed view of Springdale's business-development and M&A funnel across Origent, Nate's Eagle workflow, and the internal M&A process. Users should be able to see a company once, understand every associated opportunity and stage change, trace values back to the source, and analyze funnel inventory, flow, conversion, and aging over time.

The first reporting surface should answer:

- What companies and opportunities are active now?
- Where did each opportunity originate, who owns it, and which platform is it for?
- How did an opportunity move through the funnel?
- How many opportunities entered, exited, converted, stalled, or regressed at each stage?
- How long do opportunities spend in each stage and end-to-end?
- How do conversion and velocity differ by source, platform, owner, and cohort?

## Scope

### Initial sources

| Source | Expected objects | Purpose | Access dependency |
| --- | --- | --- | --- |
| Origent HubSpot | Companies, contacts, deals, pipelines/stages, owners, deal property history, relevant activities | Origent-sourced BD pipeline and handoff into M&A | Read-only HubSpot private-app credential for the Origent portal |
| Nate's Eagle HubSpot | Companies, contacts, deals, pipelines/stages, owners, deal property history, relevant activities | Eagle/Nate-sourced opportunities and progression | Read-only HubSpot private-app credential for the Eagle portal |
| M&A Airtable: Airflow Pipeline | Company and opportunity records, stage/status, owner, platform, attribution, key dates | Internal M&A pipeline | Airtable PAT scoped read-only to the required base and tables |
| M&A Airtable: Diligence | Diligence records, milestones, statuses, linked companies/opportunities | Later-stage process visibility | Airtable PAT scoped read-only to the required base and tables |

The exact tables, properties, history endpoints, and Matia relation names must be confirmed by source profiling. Do not encode guessed field names into dbt.

### Out of scope for the first release

- Writing transformed data back into HubSpot or Airtable.
- Replacing the operational systems used by BD or M&A teams.
- Automatic fuzzy merging of ambiguous companies.
- A polished custom application before metric definitions have been validated.
- Predictive lead scoring or forecasting.

## Target architecture

```text
Control plane: Inngest schedule → Vercel Python/dbt

HubSpot portals + Airtable bases
             │
             ▼
           Matia
             │  service user: write raw only
             ▼
       Snowflake RAW
             │
             ▼
           VIEWS
  1:1 protective source contracts
             │
       ┌─────┴──────────┐
       ▼                ▼
   SNAPSHOTS         STAGING
 mutable history    business logic
       └─────┬──────────┘
             ▼
           REPORT
     entity + time series
       ┌─────┴──────────┐
       ▼                ▼
 Snowflake MCP / BI   REVERSE_ETL
```

### Snowflake ownership

| Area | Owner | Access pattern |
| --- | --- | --- |
| `RAW` | Matia | Matia writes; dbt reads |
| `VIEWS` | dbt | dbt builds one protective view per raw relation |
| `SNAPSHOTS` | dbt | dbt records changes to mutable pipeline state |
| `STAGING` | dbt | dbt builds mappings, aggregates, and intermediate entities |
| `REPORT` | dbt | dbt builds governed entity and time-series outputs |
| `REVERSE_ETL` | dbt | dbt exposes narrowly scoped outbound datasets |

Use separate least-privilege roles for Matia ingestion, dbt production, dbt development, and read-only reporting. Machine users should use key-pair authentication where supported.

Development objects use `dev_<schema>_<user>.<relation>`, for example `dev_report_andrew.pipeline_funnel_daily`. The views layer is always materialized as views. Staging and report models should remain non-incremental unless measured runtime, volume, or warehouse cost demonstrates a need to optimize.

## Data model

### 1. Company master

`companies` has one row per canonical company in the report layer.

Candidate fields:

- `company_key`: stable warehouse key.
- Canonical company name and normalized domain.
- Known source-system company IDs.
- Platform or platform relationship where that attribute is genuinely company-level.
- Industry, geography, size, and other descriptive attributes with documented survivorship rules.
- First-seen and last-seen timestamps.
- Identity-resolution method and review status.

Domain is the primary deterministic match signal, but it is not universally present or unique. The identity process must:

1. Normalize domains consistently.
2. Apply reviewed manual aliases and overrides first.
3. Match exact normalized domains second.
4. Quarantine conflicts and missing-domain records for review.
5. Never merge on fuzzy company name alone.

Maintain a version-controlled identity override seed containing the source record, canonical company key or alias, reason, and reviewer context.

### 2. Opportunity fact

`pipeline_opportunities` has one row per canonical pipeline opportunity, not one row per company. This entity table is necessary because the same company can be approached more than once, can have opportunities for multiple platforms, or can re-enter the funnel.

Candidate fields:

- `opportunity_key` and source opportunity ID.
- `company_key`.
- Source portal/system and source channel.
- Owning platform, current owner, and current canonical stage.
- Created, opened, current-stage-entered, closed, and last-activity timestamps.
- Outcome and outcome reason.
- Raw and canonical attribution fields.

The exact opportunity grain remains a business decision. The implementation must decide how to distinguish a genuinely new process from a reopened or duplicated record.

### 3. Stage-event fact

`pipeline_stage_events` has one row per opportunity-stage interval or transition. It is the historical backbone of the funnel.

Candidate fields:

- `opportunity_key`, `company_key`, and `stage_key`.
- Raw stage ID/name and canonical stage.
- `entered_at`, `exited_at`, and duration.
- Entry and exit sequence.
- Whether the stage is current, skipped, regressed into, or reopened.
- Source history record and ingestion metadata for traceability.

Where a source does not provide usable history, document the limitation. Begin collecting snapshots prospectively rather than fabricating historical transitions from the current value.

### 4. Current-state reporting table

`company_pipeline_current` provides the operational company table. Its exact grain should be **one row per active company-opportunity combination** unless stakeholders explicitly choose a lossy one-row-per-company presentation.

It should expose canonical company attributes, opportunity attribution, platform, owner, current stage, stage-entered date, age in stage, next step, next-step date, and traceable source links/IDs.

### 5. Funnel time series

`pipeline_funnel_daily` has one row per calendar date and selected reporting dimensions, including canonical stage. It should support:

- Beginning- and end-of-day active opportunity counts.
- Entries, exits, regressions, reopens, wins, and losses.
- Stage-to-stage and cumulative conversion rates.
- Median and percentile time in stage.
- Current aging buckets.
- Cohort analysis by opportunity-created period.
- Slicing by source channel, capture system, platform, owner, and outcome.

Build the time series from stage events and a date spine. Do not derive historical inventory from today's opportunity state.

## Canonical lifecycle proposal

The following is a discussion draft that must be mapped with the business owners before it becomes a contract:

1. Identified / sourced
2. Outreach active
3. Connected
4. Qualifying
5. Internal or client introduction
6. NDA / information requested
7. Information received
8. Diligence
9. LOI
10. Closed won
11. Closed lost

`Nurture / keep warm` and `Hold` should likely be modeled as governed states or status dimensions rather than quietly mixed into the primary forward stage sequence. Origent's detailed labels and M&A's high/medium/low holding patterns need explicit mappings. Raw values must always remain available.

## Attribution model

Reporting must distinguish:

| Concept | Question answered | Example values |
| --- | --- | --- |
| Source channel | How did this opportunity originate? | Origent, Nate/Eagle, third-party broker |
| Capture system | Where was the source record stored? | Origent HubSpot, Eagle HubSpot, M&A Airtable |
| Owning platform | Which Springdale platform or team is pursuing it? | Governed platform list |
| Owner | Who is accountable for the next action? | HubSpot or Airtable owner mapped to a person dimension |

These values must not be substituted for one another. Define first-touch, current-source, and handoff behavior before publishing conversion metrics by source.

## Transformation plan

### Phase 0: foundation

- [ ] Create the repository documentation and project operating rules.
- [ ] Initialize dbt Core with pinned `dbt-snowflake` versions.
- [ ] Add a native Python Vercel Function with pinned dbt Core, `dbt-snowflake`, and Inngest SDK versions.
- [ ] Add environment-driven `profiles.yml` configuration without secrets.
- [ ] Create Snowflake databases/schemas, warehouses, users, and least-privilege roles.
- [ ] Configure the Vercel production project, Fluid compute, a 1,800-second maximum duration, and environment variables.
- [ ] Configure the Inngest application, schedule, single-run concurrency, bounded retries, and final-failure alerting.
- [ ] Configure GitHub Actions for parse/compile and isolated CI builds.

### Phase 1: ingestion and source profiling

- [ ] Provision Matia.
- [ ] Connect both HubSpot portals with read-only private-app credentials.
- [ ] Connect the required Airtable bases with a read-only, narrowly scoped PAT.
- [ ] Land initial full-refresh extracts in `RAW`.
- [ ] Record each relation's grain, key, deletion behavior, history availability, timestamps, and row counts.
- [ ] Confirm how Matia represents HubSpot property history and Airtable linked records.
- [ ] Establish freshness expectations and reconciliation queries.

### Phase 2: protective views and snapshots

- [ ] Create source YAML definitions only after physical schemas exist.
- [ ] Build one 1:1 view for every raw relation used by the project.
- [ ] Rename fields and normalize identifiers, domains, timestamps, booleans, enums, and null values without changing source grain.
- [ ] Preserve raw IDs, labels, and ingestion metadata.
- [ ] Add source freshness, key, and reconciliation tests.
- [ ] Identify mutable pipeline relations that do not already provide complete native history.
- [ ] Start snapshots for mutable deals/opportunities and Airtable pipeline records immediately after their views are available.
- [ ] Snapshot analytically important company fields only where their change history is needed.
- [ ] Document snapshot keys, strategies, tracked fields, and hard-delete behavior.

### Phase 3: staging business logic

- [ ] Build company identity candidates and a conflict-review output.
- [ ] Establish the approved alias/override process.
- [ ] Agree on the canonical company and opportunity grains.
- [ ] Inventory every raw pipeline and stage value.
- [ ] Approve the canonical stage order and implement `pipeline_stage_mapping.sql` with BD/M&A owners.
- [ ] Reconstruct stage intervals and document gaps in historical coverage.
- [ ] Define reopened, regressed, skipped, duplicate, won, and lost behavior.
- [ ] Build conceptually distinct staging entities and aggregates only where they are reused by reports.

### Phase 4: report entities and time series

- [ ] Build and validate `companies`.
- [ ] Build and validate `pipeline_opportunities`.
- [ ] Build and validate `pipeline_stage_events`.
- [ ] Build `company_pipeline_current`.
- [ ] Build `pipeline_funnel_daily` from stage events and a date spine.
- [ ] Reconcile company, opportunity, and stage-event counts to the sources.
- [ ] Document and test the unique grain of every report model.

### Phase 5: adoption and outbound data

- [ ] Validate definitions and sample records with business owners.
- [ ] Publish dbt documentation and a metric/data dictionary.
- [ ] Connect Snowflake MCP for governed conversational analysis.
- [ ] Decide whether Hex or a custom dashboard is warranted after usage is understood.
- [ ] Add reverse-ETL models only for approved outbound workflows and expose the minimum required columns.

## Orchestration plan

The initial production pattern uses the existing Vercel and Inngest stack:

1. Matia starts the ingestion sync at the top of the agreed interval.
2. Inngest invokes the Vercel-hosted Python dbt function after a conservative buffer, initially 30 minutes.
3. One Inngest step checks source freshness, updates snapshots, and then builds and tests dependent models sequentially.
4. Inngest limits production concurrency to one and applies bounded retries to transient failures.
5. Vercel captures function logs; Inngest records run state and alerts after final failure.
6. A freshness, snapshot, test, build, or timeout failure must fail the run.

Start with a nightly schedule while the models and SLAs are being established. Increase frequency only when source sync duration, warehouse cost, and business need justify it. If Matia later provides a reliable completion webhook, it can replace the fixed buffer. Configure a 1,800-second maximum duration, verify the Vercel project's eligibility for the extended-duration beta during setup, and treat a normal runtime above 25 minutes as the point to move only the execution step to a true batch/container runtime.

Pull requests should run `dbt parse` or compile plus a modified-model build against an isolated CI schema. Production runs should execute the immutable Vercel deployment built from the merged commit.

## Access and credential checklist

### Snowflake

- [ ] Matia machine user and `MATIA_ETL_ROLE` with write access limited to its raw landing area.
- [ ] dbt machine user and `DBT_TRANSFORMER_ROLE` with `RAW` read and build permissions for `VIEWS`, `SNAPSHOTS`, `STAGING`, `REPORT`, and `REVERSE_ETL`.
- [ ] Dedicated, right-sized dbt warehouse with auto-suspend.
- [ ] Developer role and `dev_<schema>_<user>` schema-generation convention.
- [ ] Read-only reporting role for MCP/BI consumers.
- [ ] Key pairs and rotation ownership documented outside the repository.

### HubSpot

- [ ] Identify an administrator for each portal.
- [ ] Create one read-only private app per portal or another explicitly approved isolation pattern.
- [ ] Grant only the CRM objects, owners, pipeline metadata, property history, and activities needed for the project.
- [ ] Store tokens in Matia's secret store, never in dbt or GitHub.
- [ ] Confirm API access to historical deal-stage values before promising retrospective funnel reporting.

### Airtable

- [ ] Create a dedicated integration identity where practical.
- [ ] Create a PAT with record-read and schema-read access only.
- [ ] Scope it only to the Airflow Pipeline and Diligence bases.
- [ ] Store the PAT in Matia's secret store.
- [ ] Inventory linked records, formula fields, select values, and deleted-record behavior.

### Matia, Vercel, and Inngest

- [ ] Provision the Matia account and Snowflake destination.
- [ ] Configure incremental sync strategy only after source behavior is understood.
- [ ] Create the Vercel dbt project and production environment with Fluid compute and a 1,800-second maximum duration.
- [ ] Add Snowflake account, user, role, warehouse, database, schema, and private-key variables to Vercel.
- [ ] Connect the Inngest production environment and configure its signing and event keys.
- [ ] Configure one scheduled function with concurrency one and bounded retries.
- [ ] Enable Vercel Static IPs and allowlist them in Snowflake if a network policy requires fixed egress.
- [ ] Add failure monitoring and an owner for responding to failed runs.

## Decisions required before metric publication

1. What uniquely defines an opportunity across source systems and over time?
2. When a company re-enters the funnel, is that a reopened opportunity or a new one?
3. Which raw stages map to each canonical stage, and what is their ordering?
4. Are nurture/keep-warm and hold stages, statuses, or separate queues?
5. How are Origent, Nate/Eagle, and third-party broker attribution assigned after a handoff?
6. Can an opportunity have multiple owning platforms, and if so, how is that represented?
7. Which outcome and reason values are required for closed-lost analysis?
8. What is the reporting timezone and end-of-day boundary?
9. Which source is authoritative when company attributes conflict?
10. What are the freshness and reliability expectations for the first production release?

Record each decision in this document or a linked decision record before encoding it in SQL.

## Data quality gates

- Every staging and report model has a documented grain, and every report entity has a tested key.
- Every report relationship to a company, opportunity, and stage is tested.
- No active raw stage is silently unmapped.
- No ambiguous identity candidate is silently merged.
- Source row counts and keys reconcile within documented filters.
- Stage intervals do not have unexplained negative durations or prohibited overlaps.
- Current-state reporting reconciles to operational source views for an agreed sample.
- Funnel snapshots reconcile to event history for selected dates and opportunities.
- Freshness failures stop the production build rather than publishing stale data unnoticed.

## Definition of done for the first release

- All four initial source areas land reliably in Snowflake with documented refresh behavior.
- The canonical company, opportunity, stage, and attribution definitions are approved.
- Protective views, snapshots, staging logic, and report entities pass automated tests and source reconciliation.
- The current company-pipeline table and daily funnel time series are published in `REPORT`.
- Users can trace every reported row to raw source identifiers.
- Inngest runs dbt on Vercel successfully on schedule with single-run concurrency, observable failures, and no committed secrets.
- GitHub pull requests validate dbt changes in an isolated environment.
- The data dictionary documents grain, fields, limitations, freshness, and owners.

## Immediate next actions

Work that can start before Matia is provisioned:

1. Create the Snowflake roles, warehouses, machine users, and developer-access pattern.
2. Obtain the two HubSpot private-app credentials and confirm access to deal-stage history.
3. Create the Airtable PAT and confirm the precise bases, tables, and linked-record fields.
4. Convene the stage-mapping and opportunity-grain decisions with the BD/M&A owners.
5. Initialize dbt Core, the Vercel Python function, Inngest orchestration, and CI without hard-coding source schemas.

Once Matia is available, run a full raw landing and make source profiling—not dashboard construction—the first implementation milestone.
