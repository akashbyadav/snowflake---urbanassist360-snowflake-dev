# UrbanAssist 360 implementation runbook

This runbook builds the project directly in Snowflake and uses a Git-connected
Snowflake Workspace for development. Execute the stages in order. Each checkpoint
must pass before continuing because later objects depend on earlier state.

## 1. Prerequisites

### Snowflake

- An account on AWS or a Snowflake region allowed to read the selected S3 bucket
- `ACCOUNTADMIN` access for initial grants and Snowflake CoWork configuration
- Permission to create a storage integration
- Cortex AI Functions available locally, or an approved cross-region inference
  policy
- Cortex Agents and Snowflake CoWork enabled for the account
- A user with a default role and default warehouse

### AWS

- An S3 bucket in an approved region
- Permission to create or modify an IAM policy and role
- Permission to create S3 event notifications
- No existing event notification with an overlapping `urbanassist/bookings/` or
  `urbanassist/providers/` prefix

### GitHub

- A non-empty repository with at least one branch
- A development branch, for example `feature/urbanassist-360`
- Permission to authorize the Snowflake GitHub App for the repository
- A Snowflake Git API integration that allows the repository HTTPS URL

Never place AWS keys, Snowflake passwords, GitHub tokens, or private keys in this
repository or in SQL comments.

## 2. Prepare the Git-connected Snowflake Workspace

1. Add the `urbanassist-360` folder to the GitHub repository.
2. Commit and push it to the intended development branch. Snowflake cannot create
   a Git Workspace from an empty repository.
3. From a regular Snowsight SQL worksheet, run the commands in the README's
   **Connect GitHub to a Snowflake Workspace** section. This creates
   `URBANASSIST_GITHUB_API_INT` and grants it to `URBANASSIST_ENGINEER`. If the
   role does not exist yet, copy and run `sql/00_setup/01_account_and_roles.sql`
   from the GitHub file view first.
4. In Snowsight, open **Projects → Workspaces**.
5. Choose **From Git repository**.
6. Paste the repository HTTPS URL.
7. Select `URBANASSIST_GITHUB_API_INT`.
8. Complete the Snowflake GitHub App authorization and create the Workspace.
9. Open **Changes**, fetch all branches if needed, and switch to the project
   branch.
10. Confirm that `urbanassist-360/sql/00_setup/01_account_and_roles.sql` opens as
    a SQL file.

The Workspace is the development surface. Running a SQL file executes it against
the role, warehouse, database, and schema selected by the statements in that
file. Git synchronization versions the files; it does not deploy Snowflake
objects automatically.

## 3. Generate and separate the source data

The repository includes generated files. To reproduce them, run from the project
root in any standard Python 3 environment:

```bash
python3 scripts/generate_data.py
```

No third-party Python package is required. The manifest is a lightweight source
inventory containing file names, physical record counts, and file sizes; it is
not loaded by the Snowflake pipeline.

Separate the upload into two groups.

### Initial files

```text
reference/customers.json.gz
reference/services.json.gz
providers/provider_initial.json.gz
bookings/booking_batch_001.json.gz
bookings/booking_batch_002.json.gz
bookings/booking_batch_003.json.gz
bookings/booking_batch_004.json.gz
bookings/booking_batch_005.json.gz
```

### Held-back live files

```text
providers/provider_changes_live.json.gz
bookings/booking_live_batch.json.gz
```

Do not upload the held-back files until the streams and Tasks have been created.

## 4. Create the S3 prefix

Create or reuse one bucket, then create these object-key prefixes:

```text
urbanassist/bookings/
urbanassist/providers/
urbanassist/reference/customers/
urbanassist/reference/services/
```

The prefix names are part of the SQL contract. If they change, update the stage
paths in `sql/01_bronze/02_snowpipe_auto_ingest.sql`,
`sql/01_bronze/03_initial_reference_load.sql`, and
`sql/05_operations/02_live_event_demo.sql`.

Recommended bucket controls:

- Block public access.
- Enable default server-side encryption.
- Enable bucket versioning when organizational policy requires recovery.
- Add a lifecycle policy only if the source-retention requirement is known.
- Keep Snowflake access read-only for this project.

## 5. Create the AWS IAM policy and role

Attach a least-privilege policy like the following to a dedicated IAM role.
Replace `<bucket>` with the exact bucket name.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListUrbanAssistPrefix",
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::<bucket>",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["urbanassist/*"]
        }
      }
    },
    {
      "Sid": "ReadUrbanAssistObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": "arn:aws:s3:::<bucket>/urbanassist/*"
    }
  ]
}
```

Create the role with a temporary trusted principal permitted by your AWS
administration process. Copy the role ARN. The final trust relationship cannot
be completed until Snowflake returns its IAM user ARN and external ID.

## 6. Create Snowflake account objects

Open `sql/00_setup/01_account_and_roles.sql`. If it was already run to prepare
the Git-connected Workspace, rerun it to confirm the idempotent setup and final
session-context check.

1. Replace `<YOUR_SNOWFLAKE_USER>` with the exact Snowflake user identifier.
2. Run the account-admin section.
3. Continue through the project-role section.
4. Confirm the final query reports:
   - `URBANASSIST_ENGINEER`
   - `URBANASSIST_WH`
   - `URBANASSIST_DB`

The script grants the project role to `SYSADMIN` so account administrators can
manage objects through the normal custom-role hierarchy.

## 7. Complete the Snowflake–AWS trust relationship

Open `sql/00_setup/02_s3_storage_integration.sql`.

1. Replace `<AWS_ROLE_ARN>` with the dedicated role ARN.
2. Replace `<S3_BUCKET_NAME>` with the exact bucket.
3. Run the script.
4. From `DESC INTEGRATION`, copy:
   - `STORAGE_AWS_IAM_USER_ARN`
   - `STORAGE_AWS_EXTERNAL_ID`
5. Replace the IAM role's temporary trust policy with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "<STORAGE_AWS_IAM_USER_ARN>"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "<STORAGE_AWS_EXTERNAL_ID>"
        }
      }
    }
  ]
}
```

Do not omit the external-ID condition. It binds role assumption to the intended
Snowflake integration.

## 8. Create and verify the stage

Open `sql/01_bronze/01_stage_and_raw_tables.sql`.

1. Replace `<S3_BUCKET_NAME>`.
2. Run the file.
3. `LIST @URBANASSIST_S3_STAGE` may be empty before upload, but it must not return
   an authorization error.
4. Confirm all four Bronze tables exist.

The script uses `IF NOT EXISTS` for persistent landing objects to protect data
if the file is reopened. To intentionally change a stage or file-format property,
use an explicit `ALTER` or reviewed replacement statement.

## 9. Create Snowpipes and S3 event notifications

Open `sql/01_bronze/02_snowpipe_auto_ingest.sql` and run it.

`SHOW PIPES` returns a `notification_channel` ARN. Snowflake commonly shares a
managed SQS queue among pipes in the same account and AWS region, so the two ARNs
may be identical.

In the S3 bucket, create ObjectCreated notifications:

### Booking notification

- Event: all object-create events
- Prefix: `urbanassist/bookings/`
- Suffix: `.json.gz`
- Destination: SQS queue
- Queue ARN: booking pipe `notification_channel`

### Provider notification

- Event: all object-create events
- Prefix: `urbanassist/providers/`
- Suffix: `.json.gz`
- Destination: SQS queue
- Queue ARN: provider pipe `notification_channel`

AWS rejects overlapping notification filters. Resolve any pre-existing filter
before proceeding; do not broaden these two prefixes to `urbanassist/`.

Check both `SYSTEM$PIPE_STATUS` results. `executionState` should be running and
the notification channel should not report a configuration error.

## 10. Upload the initial files

Upload the initial files to these exact keys:

| Local file | S3 object key |
|---|---|
| `reference/customers.json.gz` | `urbanassist/reference/customers/customers.json.gz` |
| `reference/services.json.gz` | `urbanassist/reference/services/services.json.gz` |
| `providers/provider_initial.json.gz` | `urbanassist/providers/provider_initial.json.gz` |
| `bookings/booking_batch_*.json.gz` | `urbanassist/bookings/booking_batch_*.json.gz` |

Reference files are loaded by `COPY INTO`; booking and provider files are loaded
by Snowpipe. If files were placed before notifications became active,
`sql/01_bronze/03_initial_reference_load.sql` issues a one-time
`ALTER PIPE … REFRESH` recovery.

Snowpipe deduplicates by staged filename and load metadata. Uploading the same
content under a different name creates additional Bronze rows. Silver still
deduplicates bookings by `booking_id`, but renamed duplicates should be removed
from the source process rather than treated as normal.

## 11. Load reference data and verify Bronze

Run `sql/01_bronze/03_initial_reference_load.sql`.

Expected logical inputs:

- `RAW_CUSTOMERS`: 5,000
- `RAW_SERVICES`: 20
- `RAW_PROVIDER_EVENTS`: 250 before the live file
- `RAW_BOOKING_EVENTS`: more than 50,000 physical rows because controlled booking
  updates add later snapshots
- Distinct Bronze booking IDs: 50,000

Review COPY history. Any `LOAD_FAILED` or non-zero error count must be resolved
before creating downstream objects.

## 12. Create Silver Dynamic Tables

Run `sql/02_silver/01_clean_dynamic_tables.sql`.

Check `SHOW DYNAMIC TABLES`:

- Both scheduling states should be active.
- `DT_BOOKINGS_CLEAN` should contain 50,000 rows.
- `DT_PROVIDER_CHANGES_CLEAN` should contain 250 initial provider rows.
- `refresh_mode` may resolve to incremental or full under `AUTO`; both are valid
  for this small implementation. Review `refresh_mode_reason` when learning why.

The five-minute target lag is a maximum staleness objective. Snowflake chooses
refresh timing; it is not equivalent to a five-minute cron expression.

## 13. Build dimensions and initialize provider history

Run `sql/03_gold/01_dimensions.sql` only after Silver data is present.

Expected state:

- `DIM_DATE`: 1,096 calendar rows
- `DIM_CUSTOMER`: 5,000
- `DIM_SERVICE`: 20
- `DIM_PROVIDER_SCD2`: 250
- Every provider has one current version
- Every current provider ends at `9999-12-31`

The initial provider seed selects the earliest valid provider event. Later files
must go through the SCD2 procedure, not the seed statement.

## 14. Establish the provider-change boundary

Run `sql/03_gold/02_provider_scd2_automation.sql`.

This moment is important: the provider stream starts after the initial 250 rows.
Do not upload `provider_changes_live.json.gz` before this point.

The Task is created suspended by Snowflake and explicitly resumed by the script.
Confirm the final `SHOW TASKS` result reports `started`.

The SCD2 procedure performs one transaction:

1. Materialize the append-only stream delta.
2. Compare the incoming attribute hash with the current provider version.
3. Close a changed current version at the incoming effective timestamp.
4. Insert the new current version.
5. Commit and advance the stream offset together.

The sample change file has at most one change per provider. If a production feed
can deliver several ordered changes for one provider in one micro-batch, extend
the procedure to process all effective timestamps rather than keeping only the
latest event.

## 15. Initialize AI enrichment and its event boundary

Run `sql/03_gold/03_review_ai_enrichment.sql`.

Before running:

- Confirm the project role has `SNOWFLAKE.CORTEX_USER`.
- Confirm the account region supports `AI_SENTIMENT` and `AI_CLASSIFY` or that an
  approved cross-region inference setting exists.
- Review `AI_BACKFILL_LIMIT`. The default is 1,000.

The script enriches the newest historical reviews, then creates the booking
stream at the current Bronze offset. Do not upload `booking_live_batch.json.gz`
before this point.

The complete function objects are stored in VARIANT fields. Normalized labels
are extracted into relational columns for the fact and semantic layer. Null
function results remain visible for operational review rather than aborting the
entire multi-row run.

## 16. Build facts and KPIs

Run `sql/03_gold/04_booking_fact_and_kpis.sql`.

Expected state before live ingestion:

- `FACT_BOOKING`: 50,000 rows
- No duplicate `booking_id`
- No missing dimension keys
- `DT_DAILY_SERVICE_KPI` contains January–August 2026 dates

The booking fact and KPI objects are Dynamic Tables. The manual refreshes at the
end of the file make the initial result deterministic; normal refresh management
continues afterward.

## 17. Create and test the Semantic View

Run `sql/04_intelligence/01_semantic_view.sql`.

The final direct `SEMANTIC_VIEW` query must return city-level metrics. If it does
not, stop before creating the Agent.

Review:

```sql
SHOW SEMANTIC VIEWS IN SCHEMA URBANASSIST_DB.INTELLIGENCE;
SHOW SEMANTIC DIMENSIONS IN SEMANTIC VIEW
  URBANASSIST_DB.INTELLIGENCE.URBANASSIST_OPERATIONS_SV;
SHOW SEMANTIC METRICS IN SEMANTIC VIEW
  URBANASSIST_DB.INTELLIGENCE.URBANASSIST_OPERATIONS_SV;
```

Two verified queries anchor cancellation rate by city and revenue by service
category. Add verified queries only after a data owner has checked the expected
SQL and result.

## 18. Create and test the Cortex Agent

Run `sql/04_intelligence/02_cortex_agent.sql`.

In Snowsight:

1. Open **AI & ML → Agents**.
2. Select **UrbanAssist Operations Agent**.
3. Open the Agent playground.
4. Ask: “What is the cancellation rate by city?”
5. Confirm the Agent invokes `UrbanAssist_Analyst` and displays generated SQL.
6. Compare the result with `GOLD.DT_DAILY_SERVICE_KPI`.

If the Agent answers from general knowledge without a tool call, inspect the
Agent specification. The tool name under `tool_resources` must exactly match the
name under `tools`.

## 19. Add the Agent to Snowflake CoWork

Open `sql/04_intelligence/03_cowork_access.sql` as `ACCOUNTADMIN`.

1. Run only `SHOW SNOWFLAKE INTELLIGENCES` first.
2. If it returns no rows, run the `CREATE SNOWFLAKE INTELLIGENCE` statement.
3. If an object already exists, skip CREATE. An account can have only one.
4. Grant the project role USAGE and MODIFY on the object.
5. Add the Agent.
6. Replace `<YOUR_SNOWFLAKE_USER>` and set default role/warehouse only if the
   existing defaults are not already appropriate.

Snowflake CoWork and Cortex Agents use the querying user's default role and
default warehouse when initializing the session. A currently selected role in a
different Snowsight worksheet does not replace that requirement.

Open [https://ai.snowflake.com](https://ai.snowflake.com), or use **Preview in
Snowflake CoWork** from the Agent page. Confirm that the Agent appears.

## 20. Run the validation gates

Run `sql/05_operations/01_monitoring_and_validation.sql`.

All of these result sets must return zero rows or zero violations:

- Duplicate Silver bookings
- Duplicate Gold bookings
- Missing dimension keys
- Invalid date keys
- Providers with zero or multiple current versions
- Invalid SCD2 intervals
- Overlapping provider intervals
- Negative amounts, invalid ratings, or negative durations

Review AI enrichment coverage separately. It is expected to be less than 100%
when the historical limit is lower than the number of source reviews.

## 21. Execute the live event-driven change

Open `sql/05_operations/02_live_event_demo.sql` and execute the BEFORE queries.

Upload:

```text
data/generated/bookings/booking_live_batch.json.gz
  -> urbanassist/bookings/booking_live_batch.json.gz

data/generated/providers/provider_changes_live.json.gz
  -> urbanassist/providers/provider_changes_live.json.gz
```

Then execute the AFTER sections sequentially:

1. Confirm stage visibility and pipe health.
2. Confirm each stream reports data.
3. Allow the one-minute Tasks to run, or manually execute them once.
4. Refresh Silver and Gold Dynamic Tables for deterministic inspection.
5. Validate 51,000 Gold booking rows.
6. Inspect provider version history.
7. Inspect September KPI rows and the increased AI-enrichment count.

Finally ask the Agent:

> What changed after September 1, 2026, and which city and service category now
> need the most attention? Support the answer with governed metrics.

Useful follow-ups:

- Compare Standard and Premium provider completion rates.
- Which providers changed tier, and how does their earlier performance compare?
- What are the most frequent themes among negative enriched reviews?
- Create a concise operations summary separating facts from recommendations.

## 22. Rerun behavior

| Object group | Safe rerun behavior |
|---|---|
| Account database/schemas/warehouse | `IF NOT EXISTS`; non-destructive |
| Bronze tables, stage, pipes | `IF NOT EXISTS`; current data and offsets preserved |
| Reference dimensions | `MERGE`; Type 1 values updated |
| Silver/Gold Dynamic Tables | `CREATE OR REPLACE`; contents reinitialize |
| Provider initial seed | Inserts only missing natural keys |
| Streams | `IF NOT EXISTS`; offsets preserved |
| Procedures/Tasks | Replaced with repository definition |
| Historical AI backfill | Review hash prevents unchanged updates, but AI source query can still incur calls |
| Semantic View/Agent | Replaced; recheck grants and Agent behavior |
| CoWork object | Conditional; never recreate an existing shared object |

Do not replace a stream merely to clear it. Consume or deliberately recreate it
only after deciding how pending changes will be recovered.

## 23. Troubleshooting

| Symptom | Check | Resolution |
|---|---|---|
| `LIST` returns access denied | Integration ARN, external ID, IAM policy, bucket prefix | Correct trust and least-privilege policy, then retry `LIST` |
| File is visible but Bronze is unchanged | Pipe status and S3 notification prefix/suffix | Correct notification; use one reviewed `ALTER PIPE … REFRESH` for recent missed files |
| Pipe reports stopped | `SYSTEM$PIPE_STATUS`, COPY history | Resolve file-format or permission error, then resume/recreate only if required |
| Silver count is zero | Bronze count and standalone SELECT | Correct JSON path/casts, then replace and refresh the Dynamic Table |
| Dynamic Table uses full refresh | `SHOW DYNAMIC TABLES.refresh_mode_reason` | Accept for this data size or simplify unsupported query constructs |
| Provider stream has no data | Upload timing and exact S3 key | Ensure the live file arrived after stream creation and was loaded by the provider pipe |
| Provider has two current rows | SCD2 validation and effective timestamps | Suspend Task, correct conflicting interval in a transaction, rerun validation |
| AI labels are null | Cortex role, regional support, function result VARIANT | Review privilege/region; inspect raw result and function errors |
| AI Task repeats costs | Review hash and stream offset | Confirm existing review hash matches and stream is not being recreated |
| Agent does not call Analyst | Agent tool/resource names and semantic privileges | Make names identical; grant semantic view access and table SELECT as required |
| CoWork does not display Agent | CoWork object membership, agent USAGE, default role/warehouse | Add Agent, correct grants, and reauthenticate |
| Git branch is missing | Workspace Changes menu | Fetch All, then select the remote branch |

## 24. Cleanup

`sql/05_operations/03_cleanup.sql` contains a safe teardown order with every destructive
statement commented out. Suspend Tasks first, remove the Agent from the shared
CoWork object, then remove only project-owned database, warehouse, integration,
and role objects. The SQL does not delete S3 files.
