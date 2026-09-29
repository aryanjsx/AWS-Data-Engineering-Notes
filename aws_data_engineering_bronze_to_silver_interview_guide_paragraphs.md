# AWS Data Engineering Project --- Bronze to Silver Foundation

## Project Objective

This project is being developed as a hands-on, production-style AWS Data
Engineering implementation rather than a basic tutorial. The primary
objective is to build and understand a complete S3-based Medallion
Architecture in which data flows from the source layer through Bronze,
Silver, Gold, and finally into analytics. The current phase focuses on
establishing the Bronze ingestion layer and building the foundation of
the Bronze-to-Silver data-quality pipeline. The implementation is also
being approached from an interview perspective so that every technical
decision can be explained in terms of architecture, failure handling,
security, scalability, cost, and production practices.

## Current AWS Environment

The project is being implemented in an AWS Cloud Playground environment
using the `us-east-1` region. The primary S3 bucket is
`aryan-data-engineering-lab-2026`, and the Bronze sales data is stored
under `s3://aryan-data-engineering-lab-2026/raw/sales/`. The source file
is `sales_source_bronze.csv`. This S3 location represents the raw
landing area of the data lake and is treated as the Bronze layer.

## Source Data and Business Grain

The source sales dataset contains the fields `order_id`,
`order_line_id`, `order_date`, `customer_id`, `product_id`, `quantity`,
`unit_price`, and `region`. The business grain of the dataset is one row
representing one product line within one customer order. This definition
of grain is important because downstream transformations, deduplication
logic, aggregations, and eventual fact-table design must preserve a
clear understanding of what one record represents. The natural business
key is the combination of `order_id` and `order_line_id`, rather than
either column independently.

## Source Data Quality Problems

The Bronze source intentionally contains several data-quality problems,
including negative quantity values, `INVALID_DATE` values in the
`order_date` field, missing `customer_id` values, and a duplicate source
record. These issues are intentionally preserved in the Bronze layer
because the purpose of the project is to demonstrate how a
production-style data platform detects, isolates, and processes bad data
instead of silently modifying or discarding the source.

From an interview perspective, the reason for preserving these records
in Bronze is that the raw layer provides auditability, data provenance,
and reprocessing capability. If transformation logic changes, if a
downstream failure occurs, or if an investigation is required, the
original source representation remains available for replay and
comparison.

## Bronze Data Landing Zone

The first implementation step was to establish the S3 Bronze landing
location at `s3://aryan-data-engineering-lab-2026/raw/sales/` and upload
the source sales CSV without modifying its contents. Amazon S3 acts as
the durable storage foundation of the data lake, while downstream AWS
services use the data stored in S3 for metadata discovery,
transformation, validation, and analytics.

In an interview, the use of S3 can be explained by its durable object
storage capabilities, separation of storage from compute, integration
with AWS analytics and data-processing services, support for multiple
file formats, and suitability for scalable and relatively low-cost
data-lake storage. The Bronze layer is intentionally kept close to the
source representation so that downstream processing can be reproduced
when required.

## Glue Data Catalog

An AWS Glue Data Catalog database named `de_bronze` was created to
provide centralized metadata for the Bronze layer. The Data Catalog
stores metadata describing datasets, schemas, locations, and related
information; it does not physically store the underlying S3 data. This
metadata can then be consumed by services such as AWS Glue and Athena.

From an interview perspective, the Glue Data Catalog can be described as
the metadata layer of the data lake. It allows processing and analytics
services to discover and understand datasets stored in S3 without
requiring every service to independently infer the structure of the
underlying files.

## Glue Crawler

A Glue crawler named `crawler-bronze-sales` was configured against
`s3://aryan-data-engineering-lab-2026/raw/sales/`. The crawler was
executed successfully and registered the source dataset in the Glue Data
Catalog as the `de_bronze.sales` table. The crawler's primary
responsibility is metadata discovery and schema registration rather than
performing the complete ETL transformation process.

In an interview, the distinction between a crawler and an ETL job is
important. A crawler discovers metadata and creates or updates Catalog
tables, whereas a Glue ETL job performs data processing and
transformation. Schema inference is useful during discovery, but
production pipelines generally require controlled schemas and data
contracts to prevent unexpected source changes from silently propagating
downstream.

## IAM Configuration

The Glue crawler uses an IAM execution role named `glue-crawler-role`.
The role was configured with the required Glue service permissions and
controlled S3 read access to the Bronze sales location. The S3 resource
is restricted to the appropriate Bronze path rather than granting
unrestricted access to the entire AWS account or bucket.

This implementation follows the principle of least privilege. In an
interview, an IAM role can be explained as an AWS identity that a
service such as Glue can assume to perform operations, while IAM
policies define the permissions available to that role. The trust policy
determines which service or principal can assume the role, while
permission policies determine which AWS resources and actions the role
can access.

## Glue Catalog Table and Schema

After the crawler completed successfully, the Glue Data Catalog
contained the `de_bronze.sales` table. The source schema was inferred
with fields corresponding to the original dataset. The important
observation is that `order_date` remains a string because the source
contains an invalid value such as `INVALID_DATE`.

The project intentionally avoids blindly converting `order_date` to a
DATE type at this stage. In a production pipeline, directly casting an
invalid date field can cause errors or produce null values and may hide
the underlying data-quality problem. The correct approach is to validate
the value, separate invalid records, and then standardize valid records.

## Bronze-to-Silver Glue ETL Job

A Glue Studio Visual ETL job named `job-bronze-to-silver` was created to
implement the Bronze-to-Silver processing pipeline using Spark-based
processing. The `de_bronze.sales` Glue Catalog table is configured as
the source of the ETL job. The current processing flow begins with the
Catalog source and continues into schema management and data-quality
validation.

The purpose of this job is not simply to convert one file format into
another. It is being developed as the transformation boundary between
the raw Bronze layer and the trusted Silver layer, where records will
eventually be validated, standardized, cleaned, deduplicated, and stored
in an analytics-friendly format.

## Change Schema Transformation

A Change Schema transformation was added after the Glue Data Catalog
source. The transformation is used to explicitly manage source-to-target
schema mapping. The current implementation intentionally leaves
`order_date` as a string because invalid date values exist in the source
and require explicit validation before type conversion.

From an interview perspective, explicit schema management is preferable
to relying entirely on schema inference for production transformations
because it makes data contracts and expected data types easier to
control and review. Schema inference can be useful for initial
discovery, but production pipelines need mechanisms for detecting schema
drift and managing schema evolution safely.

## Data Quality Validation

An Evaluate Data Quality transformation was introduced after the Change
Schema step. The initial data-quality rules validate that `order_id`,
`order_line_id`, `customer_id`, `product_id`, and `region` are complete.
The rules also validate that `quantity` is greater than zero and that
`unit_price` is greater than or equal to zero. The `order_date` field is
intentionally excluded from this initial ruleset because invalid dates
require separate validation and standardization logic.

Data quality is treated as a first-class pipeline concern rather than
something performed after the data has already entered the trusted
layer. The purpose is to identify deterministic data-quality failures
before records are allowed to proceed into the Silver dataset.

## Processing Failure Versus Data-Quality Failure

A key production concept in this project is the distinction between
transient processing failures and deterministic data-quality failures. A
transient processing failure can include a network timeout, temporary
AWS service unavailability, throttling, or another
infrastructure-related issue. Such failures can normally be handled
using a retry mechanism, a defined retry policy, exponential backoff,
and a maximum number of attempts. If the maximum number of retries is
reached, the workflow can transition into DLQ or other failure-handling
mechanisms.

A deterministic data-quality failure is different. Examples include
`INVALID_DATE`, a quantity less than or equal to zero, a missing
mandatory customer identifier, or an invalid business value. Retrying
the same invalid record without changing the input data or processing
logic will generally produce the same result. Therefore, deterministic
data-quality failures should be routed to a quarantine process rather
than subjected to endless retries.

## Conditional Routing

A Conditional Router was added after the Data Quality transformation.
The intended architecture is that records passing the defined validation
rules continue through the PASS path toward the Silver layer, while
records failing validation are routed toward a quarantine path. This
establishes the production-style processing pattern of separating
trusted records from deterministic bad records.

The PASS route is intended for records that satisfy the currently
defined data-quality requirements. The failure route will eventually be
used to retain invalid records together with appropriate failure
information so that the records can be investigated, corrected, and
potentially reprocessed.

## Quarantine Design

The quarantine layer is intended to contain deterministic bad records
such as invalid dates, missing mandatory fields, invalid numeric values,
duplicate records, and other data-quality failures. Where possible, the
quarantine dataset should also contain information explaining why a
record failed validation.

The purpose of quarantine is not to discard bad data. It is to isolate
invalid records from trusted datasets while preserving them for
investigation, auditability, correction, and data reprocessing. This is
an important distinction in production Data Engineering because simply
dropping invalid records can result in silent data loss.

## Composite-Key Uniqueness

The business key for the sales dataset is the composite key
`(order_id, order_line_id)`. The project intentionally does not treat
`order_id` and `order_line_id` as two independent uniqueness constraints
because neither field individually defines the identity of a sales-line
record.

Composite uniqueness will therefore be implemented through Spark or SQL
logic. This approach allows the pipeline to detect duplicate
combinations of the complete business key and route those records
appropriately.

In an interview, this can be explained by stating that the uniqueness
requirement belongs to the business grain. Since one row represents one
product line within one order, the combination of order identifier and
line identifier is what defines record identity.

## Silver Layer

The eventual Silver layer will contain valid records with standardized
data types, validated values, cleaned fields, deduplicated records, and
quality-controlled data. The Silver layer is expected to use Parquet as
the primary storage format.

The conceptual flow is Bronze ingestion followed by validation,
cleaning, deduplication, type standardization, and Silver persistence.
The goal is for Silver to represent a trusted and consistently
structured dataset that can be consumed by downstream dimensional
modeling and analytics processes.

## Production Concepts

The project is intentionally being developed around production Data
Engineering concepts including retry mechanisms, retry policies,
exponential backoff, Dead-Letter Queues, quarantine, data reprocessing,
idempotency, data lineage, data provenance, auditability, archival,
retention policies, backfill, late-arriving data, watermarks,
checkpoints, schema evolution, schema drift, data contracts, SLAs, and
SLOs.

These concepts are not being treated as isolated definitions. Each
concept should be connected to an actual implementation decision within
the pipeline so that the final project demonstrates practical
understanding rather than only theoretical knowledge.

## Interview Explanation of the Current Implementation

A concise interview explanation of the current implementation is as
follows. The project implements the initial Bronze-to-Silver foundation
of an AWS data lake using Amazon S3 and AWS Glue. Raw sales data is
landed in an S3 Bronze zone and catalogued through AWS Glue Data Catalog
using a Glue crawler. A Glue Studio Spark-based ETL job then reads the
Catalog table, applies controlled schema mapping, performs data-quality
validation, and conditionally routes records according to their
validation outcome. The Bronze layer intentionally preserves invalid
source data for auditability and reprocessing, while deterministic
data-quality failures are separated from transient infrastructure
failures. Valid records are intended to continue toward the Silver
layer, while invalid records are routed toward quarantine for
investigation and potential reprocessing.

## Interview Questions to Prepare

The architecture should support discussion around several categories of
interview questions. For architecture, preparation should include
explaining the Bronze-to-Silver design, the reason for using Medallion
Architecture, what belongs in Bronze versus Silver, why raw data is
preserved, and where business transformations should occur. For S3,
preparation should include explaining why S3 is appropriate for a data
lake, how partitioning should be selected, how small-file problems can
be addressed, how lifecycle policies can be implemented, and how S3
access can be secured.

For AWS Glue, preparation should include explaining what a Glue crawler
does, what the Glue Data Catalog provides, why schema inference has
limitations, how IAM roles are used by Glue, and how controlled schema
management differs from automatic schema discovery. For Data Quality,
preparation should include explaining how missing mandatory fields,
invalid dates, invalid numeric values, and duplicate records are handled
and how composite-key uniqueness is validated.

For reliability and failure handling, preparation should include
explaining the difference between retry and quarantine, when exponential
backoff should be used, what a DLQ represents, how endless retries are
prevented, and how quarantined data can be reprocessed. For broader Data
Engineering concepts, preparation should include idempotency, lineage,
provenance, data contracts, schema drift, schema evolution, watermarks,
checkpoints, backfills, and late-arriving data.

## Current Implementation Status

The completed foundation currently includes the S3 Bronze bucket and raw
sales source, the `de_bronze` Glue Catalog database, the
`crawler-bronze-sales` Glue crawler, the `glue-crawler-role` IAM
configuration, successful crawler execution, the `de_bronze.sales`
Catalog table, the `job-bronze-to-silver` Glue Studio job, the Glue Data
Catalog source, the Change Schema transformation, the Evaluate Data
Quality transformation, the initial data-quality rules, and the
Conditional Router with a PASS processing group.

The next implementation areas are to complete PASS and FAIL routing,
build the Silver output, validate and standardize `order_date`,
implement composite-key deduplication, create the quarantine dataset
with failure information, write Silver data in Parquet, evaluate
partitioning, validate the Silver output, and then progress toward the
Gold dimensional model. Later stages will cover dimensions and fact
tables, SCD Types 0 through 3 and advanced SCD strategies, full and
incremental loads, watermark and high-water-mark processing, CDC, Athena
and Redshift analytics, EventBridge and Step Functions orchestration,
CloudWatch observability, IAM and KMS controls, cost optimization, Spark
performance optimization, Terraform, and CI/CD.

## Interview Framework for Every Major Component

For every major implementation step, the project should be explained
using the same production-oriented framework. The explanation should
identify what technology or component is being used, why it is being
used, where it fits within the architecture, what happens if it fails,
how retry behavior works, how bad data is handled, how duplicate
processing is prevented, how the component is monitored, how it is
secured, how cost is optimized, how the implementation would operate in
production, how the design can be explained during an interview, and
what interview questions could be asked about that component.

This approach ensures that the project demonstrates more than the
ability to configure AWS services. The final objective is to demonstrate
the ability to reason about production data platforms, including
reliability, data quality, scalability, security, observability, cost,
maintainability, and operational trade-offs.
