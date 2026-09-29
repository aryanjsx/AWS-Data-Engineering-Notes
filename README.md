# AWS Data Engineering --- Production-Style Medallion Data Platform

A hands-on, production-style AWS Data Engineering project focused on
building, understanding, troubleshooting, and explaining an end-to-end
data platform using **S3-based Medallion Architecture**.

The project is intentionally designed beyond a basic AWS tutorial. Each
implementation step is approached from a production and interview
perspective, with emphasis on data quality, reliability, idempotency,
data lineage, schema evolution, incremental processing, dimensional
modeling, orchestration, observability, security, cost optimization, and
Infrastructure as Code.

## Architecture

``` text
                         SOURCE
                           │
                           ▼
                    ┌─────────────┐
                    │   BRONZE    │
                    │     S3      │
                    └──────┬──────┘
                           │
                           ▼
                  Glue Data Catalog
                           │
                           ▼
                    Change Schema
                           │
                           ▼
                 Data Quality Checks
                           │
                           ▼
                 Conditional Router
                    ┌──────┴──────┐
                    │             │
                  PASS           FAIL
                    │             │
                    ▼             ▼
                 SILVER       QUARANTINE
                    │
                    ▼
                  GOLD
                    │
                    ▼
             Athena / Redshift
                    │
                    ▼
                ANALYTICS
```

The long-term orchestration architecture is planned as:

``` text
S3 Event
   ↓
EventBridge
   ↓
Step Functions
   ↓
Glue Crawler
   ↓
Glue ETL / PySpark
   ↓
Data Quality
   ↓
Silver
   ↓
Gold
   ↓
Redshift
```

## Project Environment

The current implementation uses an AWS Cloud Playground environment.

The current AWS region is `us-east-1`.

The primary S3 bucket is:

``` text
s3://aryan-data-engineering-lab-2026/
```

The current Bronze source location is:

``` text
s3://aryan-data-engineering-lab-2026/raw/sales/
```

The source file is:

``` text
sales_source_bronze.csv
```

## Source Dataset

The sales dataset contains the following fields:

``` text
order_id
order_line_id
order_date
customer_id
product_id
quantity
unit_price
region
```

The business grain is:

> One row represents one product line within one customer order.

The natural/business key is:

``` text
(order_id, order_line_id)
```

The source intentionally contains data-quality problems, including
negative quantity values, invalid dates such as `INVALID_DATE`, missing
`customer_id`, and a duplicate source record.

These problems are intentionally retained in the Bronze layer so the
project can demonstrate production-style validation, quarantine,
remediation, and reprocessing.

## Medallion Architecture

### Bronze

The Bronze layer is the raw/immutable representation of source data. Its
primary responsibilities are source preservation, auditability, data
provenance, and reprocessing capability.

The current Bronze location is:

``` text
s3://aryan-data-engineering-lab-2026/raw/sales/
```

### Silver

The Silver layer will contain validated, cleaned, standardized,
deduplicated, and quality-controlled records. The target storage format
will be Parquet.

### Gold

The Gold layer will provide an analytics-ready dimensional model
consisting of:

``` text
dim_customer
dim_product
dim_date
dim_region
fact_sales
```

The fact-table grain will be explicitly defined before the fact table is
designed.

## Current AWS Implementation

The first phase established the S3 Bronze landing zone and uploaded the
raw sales dataset without modifying the source representation.

An AWS Glue Data Catalog database named `de_bronze` was created to
provide centralized metadata for the Bronze layer.

A Glue crawler named `crawler-bronze-sales` was configured against:

``` text
s3://aryan-data-engineering-lab-2026/raw/sales/
```

The crawler uses the IAM role:

``` text
glue-crawler-role
```

The role has the required Glue service permissions and controlled S3
read access to the Bronze sales location.

After successful crawler execution, the source was registered as:

``` text
de_bronze.sales
```

The crawler inferred the source schema. The `order_date` field remains a
string because the source contains invalid date values. It is
intentionally not blindly converted to a DATE before validation.

## Bronze-to-Silver ETL

A Glue Studio Visual ETL job named:

``` text
job-bronze-to-silver
```

was created.

The current transformation flow is:

``` text
de_bronze.sales
      ↓
Change Schema
      ↓
Evaluate Data Quality
      ↓
Conditional Router
```

The Change Schema transformation provides controlled schema mapping.

The Evaluate Data Quality transformation validates the initial
data-quality rules.

The current validation requirements include completeness checks for:

``` text
order_id
order_line_id
customer_id
product_id
region
```

and value checks for:

``` text
quantity > 0
unit_price >= 0
```

`order_date` is intentionally handled separately because the Bronze
source contains invalid dates.

## Data Quality and Quarantine

The pipeline distinguishes between deterministic data-quality failures
and transient processing failures.

Deterministic data-quality failures include invalid dates, negative
quantities, missing mandatory fields, invalid business values, and
duplicate business keys.

These records should be routed to a quarantine layer rather than
repeatedly retried.

Transient processing failures include network timeouts, temporary AWS
service unavailability, throttling, and other infrastructure failures.

These failures should be handled using retry policies, exponential
backoff, maximum retry attempts, and eventually DLQ/failure handling
where appropriate.

The intended routing pattern is:

``` text
Data Quality
     │
     ▼
Conditional Router
   ┌───┴───┐
   ▼       ▼
 PASS     FAIL
   │       │
   ▼       ▼
Silver  Quarantine
```

The quarantine layer is intended to preserve invalid records and, where
possible, associated error or failure information for investigation,
correction, and data reprocessing.

## Composite-Key Deduplication

The source business key is:

``` text
(order_id, order_line_id)
```

Uniqueness must therefore be evaluated on the combination of these two
fields rather than by independently checking each field.

Composite-key deduplication will be implemented using Spark/SQL logic as
part of the Silver processing layer.

## Data Engineering Concepts

This project is designed to provide hands-on exposure to production Data
Engineering concepts including:

-   Retry Mechanism
-   Retry Policy
-   Exponential Backoff
-   Dead-Letter Queue (DLQ)
-   Quarantine
-   Data Reprocessing
-   Idempotency
-   Data Lineage
-   Data Provenance
-   Auditability
-   Archival
-   Retention Policy
-   Backfill
-   Late-arriving Data
-   Watermark
-   Checkpoint
-   Schema Evolution
-   Schema Drift
-   Data Contract
-   SLA
-   SLO

These concepts are connected to actual pipeline implementation rather
than treated only as theoretical definitions.

## SCD and Dimensional Modeling

The Gold layer will use dimensional modeling with:

``` text
dim_customer
dim_product
dim_date
dim_region
fact_sales
```

The project will implement and explain:

-   SCD Type 0
-   SCD Type 1
-   SCD Type 2
-   SCD Type 3
-   SCD Type 4
-   SCD Type 6
-   Hybrid SCD strategies

SCD Type 2 will preserve historical changes using concepts such as
surrogate keys, effective start dates, effective end dates, and
current-record indicators.

## Incremental Processing and CDC

After the initial full-load implementation, the project will progress
toward incremental processing using:

-   Watermarks
-   High-water marks
-   Last-modified timestamps
-   Incremental extraction
-   Incremental transformation
-   Backfills
-   Late-arriving data
-   Checkpoints

Change Data Capture concepts will also be covered, including:

``` text
INSERT
UPDATE
DELETE
```

and applying CDC changes to Silver and Gold datasets.

## PySpark and Spark Engineering

The project will use PySpark to develop practical understanding of Spark
processing, including:

-   DataFrames
-   Transformations
-   Actions
-   Lazy evaluation
-   DAGs
-   Partitions
-   Shuffle
-   Repartition
-   Coalesce
-   Broadcast joins
-   Join strategies
-   Data skew
-   Caching
-   Predicate pushdown
-   Column pruning
-   Spark optimization
-   Performance troubleshooting

The goal is to understand not only PySpark syntax but also how Spark
behaves internally and how to troubleshoot production workloads.

## File Formats

The project will compare and use:

-   CSV
-   JSON
-   XML
-   Parquet
-   Avro

Raw ingestion may use CSV, JSON, or XML depending on the source, while
analytics-oriented layers will favor columnar formats such as Parquet
where appropriate.

The project will cover row-oriented versus columnar storage,
compression, schema management, predicate pushdown, query performance,
and storage cost.

## Partitioning

Partitioning will be designed according to data volume, query patterns,
cardinality, ingestion frequency, and file-size considerations.

An example partition layout is:

``` text
sales/
├── year=2026/
│   ├── month=09/
│   └── month=10/
```

The project will also cover when partitioning is useful, when it can
become harmful, partition pruning, high-cardinality partition columns,
and the small-file problem.

## Schema Evolution and Data Contracts

The project will cover schema drift and schema evolution, including new
columns, breaking changes, non-breaking changes, backward compatibility,
forward compatibility, and data contracts.

The goal is to understand how production data platforms detect and
safely handle changes in upstream schemas.

## Analytics

The analytics layer will use:

``` text
S3 Data Lake
     ↓
Athena / Redshift
     ↓
Analytics
```

Amazon Athena will be used for querying data in the data lake, while
Amazon Redshift will be used to understand warehouse-oriented analytics
and dimensional workloads.

## Orchestration

The eventual production-style orchestration design will use:

``` text
S3 Event
   ↓
EventBridge
   ↓
Step Functions
   ↓
Glue Crawler
   ↓
Glue ETL
   ↓
Data Quality
   ↓
Silver
   ↓
Gold
   ↓
Redshift
```

The orchestration phase will cover workflow dependencies, scheduling,
event-driven processing, retries, state management, and failure
handling.

## Observability

The project will introduce AWS observability using:

-   CloudWatch Logs
-   CloudWatch Metrics
-   CloudWatch Alarms
-   EventBridge

Important operational metrics will include pipeline success/failure,
processing duration, record counts, data-quality failures, retry counts,
quarantine counts, data freshness, and SLA/SLO compliance.

## Security

Security will be implemented using AWS security best practices,
including:

-   IAM
-   Least privilege
-   IAM roles
-   Trust policies
-   `iam:PassRole`
-   S3 security
-   KMS
-   Encryption at rest
-   Encryption in transit
-   Secrets management
-   Lake Formation
-   Data access control

The project will emphasize securing data and service-to-service access
while maintaining operational usability.

## Cost Optimization

The project will address AWS cost optimization through:

-   Appropriate S3 storage classes
-   Lifecycle policies
-   Parquet
-   Compression
-   Partitioning
-   Athena scanned-data optimization
-   Glue job sizing
-   Spark optimization
-   Redshift cost management
-   Avoiding unnecessary crawlers and jobs
-   Small-file management

Cost decisions will be considered alongside performance and
maintainability rather than optimizing a single metric in isolation.

## Infrastructure as Code

The final stages will introduce Terraform for infrastructure
provisioning, including:

``` text
S3
IAM
Glue
Lambda
EventBridge
Step Functions
CloudWatch
Redshift
```

Terraform topics will include state management, modules, variables,
outputs, environments, and CI/CD.

## Interview Preparation

This repository is also intended to document the reasoning behind each
implementation so that the project can be discussed confidently in Data
Engineering interviews.

For every major component, the project focuses on answering the
following questions: What are we using? Why are we using it? Where does
it fit in the architecture? What happens if it fails? How do we retry
it? How do we handle bad data? How do we prevent duplicates? How do we
monitor it? How do we secure it? How do we optimize its cost? How would
the design operate in production? How would the implementation be
explained in an interview? What questions could an interviewer ask about
the component?

## Current Status

The current implementation has established the S3 Bronze layer, Glue
Data Catalog database, Glue crawler, crawler IAM role and S3
permissions, successful metadata discovery, the `de_bronze.sales`
Catalog table, the `job-bronze-to-silver` Glue Studio job, the Glue
Catalog source, Change Schema transformation, Evaluate Data Quality
transformation, initial data-quality validation, and Conditional Router
with a PASS processing path.

The next stage is to complete PASS/FAIL routing, build the Silver
output, validate and standardize `order_date`, implement composite-key
deduplication, create the quarantine dataset with failure information,
write Silver data in Parquet, and validate the resulting Silver layer.

The subsequent phases will cover Gold dimensional modeling, fact and
dimension tables, SCD implementations, incremental processing, CDC,
Athena and Redshift, orchestration, observability, security, cost
optimization, Terraform, and CI/CD.

## Repository Purpose

This repository is intended to serve as both a practical implementation
of an AWS Data Engineering platform and a technical reference for
understanding the production concepts behind each component.

The priority is not simply to complete an AWS lab. The objective is to
understand why each component exists, how the components interact, how
the system behaves under failure, how data quality is managed, how the
platform scales, how it is secured, how costs are controlled, and how
the architecture can be explained clearly in a technical interview.

## License

This project is licensed under the MIT License.
