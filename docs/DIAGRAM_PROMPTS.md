# Diagram generation specifications

The PNG diagrams in this repository were generated with the built-in ImageGen
workflow and then visually inspected for label accuracy and readability. These
specifications preserve the intended content if a future visual refresh is
required.

## Architecture diagram

```text
Use case: infographic-diagram
Asset type: replacement README architecture diagram
Primary request: Redesign the complete project flow in light mode as a polished hand-drawn technical architecture sketch titled "UrbanAssist 360 — Event-Driven Snowflake Architecture".
Canvas: warm-white graph paper, landscape 16:9, generous margins.
Style: senior solution architect's hand-drawn whiteboard sketch; black marker outlines, readable hand-lettering, pastel watercolor fills, purposeful icons, professional rather than childish.
Main flow: Synthetic JSON Files → Amazon S3 prefixes → External Stage → Snowpipe Auto-Ingest with S3 ObjectCreated → SQS → Bronze → Silver → Stream and Task automation → Gold → Semantic View → Cortex Analyst → Cortex Agent → Snowflake CoWork.
Bronze labels: RAW_BOOKING_EVENTS, RAW_PROVIDER_EVENTS, RAW_CUSTOMERS, RAW_SERVICES.
Silver labels: DT_BOOKINGS_CLEAN, DT_PROVIDER_CHANGES_CLEAN, Streams + Tasks, AI_SENTIMENT + AI_CLASSIFY.
Gold labels: DIM_DATE, DIM_CUSTOMER, DIM_SERVICE, DIM_PROVIDER_SCD2, FACT_BOOKING, FACT_REVIEW_INSIGHT, DT_DAILY_SERVICE_KPI.
Automation branches: Provider Stream → TASK_PROCESS_PROVIDER_SCD2 → SP_APPLY_PROVIDER_SCD2; Booking Review Stream → TASK_ENRICH_NEW_REVIEWS → AI_SENTIMENT + AI_CLASSIFY.
Observability rail: PIPE STATUS • COPY HISTORY • TASK HISTORY • DYNAMIC TABLE REFRESH HISTORY • DATA QUALITY CHECKS • AGENT OBSERVABILITY.
Constraints: light background, no glossy 3D blocks, no arrow from Gold directly to CoWork, Cortex Analyst and Cortex Agent must remain separate, no crossing arrows, no people, no watermark.
```

## Data-model diagram

```text
Use case: infographic-diagram
Asset type: replacement README dimensional data-model diagram
Primary request: Redesign the complete model in light mode as a clean hand-drawn star-schema sketch titled "UrbanAssist 360 — Gold Star Schema".
Canvas: warm-white graph paper, landscape 16:9, faint gray dot grid.
Style: senior data architect's notebook diagram; black marker outlines, readable hand-lettering, pastel highlighter fills.
Center: FACT_BOOKING with every physical key, timestamp, money field, status, rating, duration, and additive booking counter from the Gold table.
Dimensions: DIM_DATE, DIM_CUSTOMER, DIM_SERVICE, DIM_PROVIDER_SCD2, each connected one-to-many to FACT_BOOKING with the joined key labeled.
Review fact: FACT_REVIEW_INSIGHT connected one-to-zero-or-one through booking_id and showing the complete stored AI result fields.
SCD2 inset: P00001 Standard → P00001 Premium with effective_start_at, effective_end_at, is_current, plus the half-open temporal join predicate.
Footer: "Fact grain: one row per booking using the latest booking state".
Color mapping: pale-yellow booking fact, pale-blue dimensions, pale-mint review fact, pale-lavender SCD2 inset.
Constraints: white/light background, exact identifiers and cardinalities, clean non-crossing connectors, no 3D, no logos, no people, no watermark.
```
