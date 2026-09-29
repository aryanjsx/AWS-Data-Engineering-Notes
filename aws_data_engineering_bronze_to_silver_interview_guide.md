# AWS Data Engineering Project --- Bronze to Silver Foundation

## 1. Project Objective

This project is designed as a hands-on, production-style AWS Data
Engineering implementation rather than a basic tutorial. The objective
is to build and understand a complete S3-based Medallion Architecture:

``` text
SOURCE → BRONZE → SILVER → GOLD → ANALYTICS
```

The implementation is being performed in an AWS Cloud Playground
environment and is structured to develop both practical implementation
skills and the ability to explain architectural decisions in Data
Engineering interviews.

The current phase focuses on establishing the **Bronze ingestion and
Bronze-to-Silver data-quality foundation**.

------------------------------------------------------------------------

## 2. Current AWS Environment

The project is being implemented in:

-   AWS Cloud Playground
-   AWS Region: `us-east-1`
-   S3 Bucket: `aryan-data-engineering-lab-2026`

Current Bronze location:

``` text
s3://aryan-data-engineering-lab-2026/raw/sales/
```

Source file:

``` text
sales_source_bronze.csv
```

------------------------------------------------------------------------

## 3. Source Data and Business Grain

The source sales dataset contains:

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

### Business Grain

The grain of the dataset is:

> One row represents one product line within one customer order.

This is an important interview concept because every downstream fact
table, aggregation, deduplication strategy, and metric should respect
the defined grain.

### Natural / Business Key

The natural key is:

``` text
(order_id, order_line_id)
```

This combination identifies an individual order line.

It is important not to assume that either `order_id` or `order_line_id`
alone uniquely identifies a record.

------------------------------------------------------------------------

## 4. Source Data Quality Issues

The Bronze dataset intentionally contains data-quality problems:

-   Negative `quantity`
-   `INVALID_DATE` in `order_date`
-   Missing `customer_id`
-   Duplicate source record

These issues are intentionally preserved in the Bronze layer so that the
pipeline can demonstrate real-world data-quality processing.

### Interview Perspective

A common interview question is:

**Why not clean the data immediately when it lands in S3?**

A strong answer is:

> The Bronze layer should preserve the source representation as much as
> practical. This provides auditability, data provenance, and the
> ability to reprocess the source when transformation logic changes or
> downstream processing fails. Data-quality and business transformations
> are applied in downstream layers rather than destroying the original
> source representation.

------------------------------------------------------------------------

# 5. Bronze Layer

## 5.1 Bronze Architecture

The current Bronze layer is:

``` text
Source CSV
    ↓
Amazon S3
    ↓
raw/sales/
    ↓
AWS Glue Crawler
    ↓
AWS Glue Data Catalog
```

The Bronze layer is intended to be:

-   Raw
-   Immutable in principle
-   Auditable
-   Traceable to the source
-   Reprocessable

The project requirements explicitly define Bronze as the layer
responsible for preserving the raw source representation, auditability,
provenance, and reprocessing capability.

------------------------------------------------------------------------

## 5.2 S3 Bronze Landing Zone

The first implementation step was establishing the S3 Bronze landing
location:

``` text
s3://aryan-data-engineering-lab-2026/raw/sales/
```

The source CSV was uploaded without modifying the source records.

### Why S3?

Amazon S3 provides the object-storage foundation for the data lake.

In this architecture, S3 acts as the durable storage layer while Glue
provides metadata and transformation capabilities.

### Interview Questions

**Q: Why use S3 for a data lake?**

Expected discussion:

-   Durable object storage
-   Separation of storage and compute
-   Supports multiple processing engines
-   Supports different file formats
-   Integrates with Glue, Athena, Redshift and other AWS services
-   Enables low-cost long-term storage and lifecycle management

**Q: Why preserve raw data?**

Expected answer:

> Raw data provides a source of truth for reprocessing, auditing,
> debugging, lineage, and recovery when downstream logic changes.

------------------------------------------------------------------------

# 6. Glue Data Catalog Database

A Glue Data Catalog database named:

``` text
de_bronze
```

was created.

The Data Catalog provides metadata describing datasets stored in the
data lake.

It does not physically store the source data.

### Interview Question

**Q: What is the AWS Glue Data Catalog?**

Expected answer:

> The Glue Data Catalog is a centralized metadata repository that stores
> information about datasets, including tables, schemas, locations and
> related metadata. Services such as Glue jobs and Athena can use this
> metadata to discover and query data stored in S3.

------------------------------------------------------------------------

# 7. Glue Crawler

A Glue crawler named:

``` text
crawler-bronze-sales
```

was created.

The crawler target is:

``` text
s3://aryan-data-engineering-lab-2026/raw/sales/
```

The crawler was executed successfully and created the Catalog table:

``` text
de_bronze.sales
```

### Purpose of the Crawler

The crawler discovers the structure of the source data and registers
metadata in the Glue Data Catalog.

The crawler does not perform the complete business transformation
pipeline.

### Interview Question

**Q: What does a Glue crawler do?**

Expected answer:

> A Glue crawler scans configured data locations, infers metadata and
> schema information, and registers or updates tables in the Glue Data
> Catalog.

### Important Interview Distinction

Do not describe the crawler as the ETL engine.

A useful distinction is:

``` text
Crawler → Metadata discovery

Glue ETL → Transformation and processing
```

------------------------------------------------------------------------

# 8. IAM Role for Glue Crawler

The crawler uses an IAM execution role:

``` text
glue-crawler-role
```

The role was configured with the required Glue service permissions and
S3 access to the Bronze source location.

The S3 resource is restricted to:

``` text
arn:aws:s3:::aryan-data-engineering-lab-2026/raw/sales/*
```

### Security Principle

The project uses the principle of:

> Least privilege

The crawler should receive only the permissions necessary to perform its
intended operation.

### Interview Questions

**Q: Why does Glue need an IAM role?**

Expected answer:

> AWS Glue assumes an IAM role when executing operations that require
> access to AWS resources such as S3. The role defines what the Glue
> service is allowed to access.

**Q: What is the difference between an IAM role and an IAM policy?**

Expected discussion:

-   Policy → defines permissions
-   Role → identity that can be assumed
-   Trust policy → defines who/what can assume the role
-   Permission policies → define what the role can do

**Q: Why should the S3 policy be restricted?**

Expected answer:

> To follow least privilege and reduce the blast radius if the execution
> role is misused or compromised.

------------------------------------------------------------------------

# 9. Glue Catalog Table

After the crawler completed successfully, the following table became
available:

``` text
de_bronze.sales
```

The crawler inferred the source schema.

The source currently contains types similar to:

``` text
order_id        string
order_line_id   bigint
order_date      string
customer_id     string
product_id      string
quantity        bigint
unit_price      double
region          string
```

### Important Data-Type Observation

`order_date` remains a string because the source contains:

``` text
INVALID_DATE
```

Therefore, blindly converting the entire column to a DATE would be
unsafe.

### Interview Question

**Q: Why not immediately cast order_date from STRING to DATE?**

Expected answer:

> Because the Bronze source contains invalid date values. A direct cast
> can produce errors or null values and may obscure the underlying
> data-quality issue. The pipeline should validate the value first,
> handle invalid records deterministically, and then standardize valid
> records.

------------------------------------------------------------------------

# 10. Bronze-to-Silver Glue ETL Job

A Glue Studio Visual ETL job named:

``` text
job-bronze-to-silver
```

was created.

The current pipeline begins with:

``` text
AWS Glue Data Catalog
        ↓
de_bronze.sales
```

The job uses Spark-based processing.

------------------------------------------------------------------------

# 11. Change Schema Transformation

A **Change Schema** transformation was added after the Catalog source.

Current architecture:

``` text
de_bronze.sales
      ↓
Change Schema
```

The transformation is being used to explicitly manage source-to-target
schema mapping.

The current implementation intentionally does not convert `order_date`
yet.

### Why Use Explicit Schema Mapping?

Explicit schema management makes transformations easier to understand
and control.

It also provides a clear place to standardize data types as the pipeline
progresses.

### Interview Question

**Q: Why not rely entirely on schema inference?**

Expected answer:

> Schema inference is useful for discovery, but production pipelines
> generally need controlled schemas and data contracts. Explicit schema
> management helps prevent unexpected type changes from silently
> propagating downstream.

------------------------------------------------------------------------

# 12. Data Quality Layer

An **Evaluate Data Quality** transformation was introduced after the
Change Schema step.

Current architecture:

``` text
S3 Bronze
   ↓
Glue Catalog
   ↓
Change Schema
   ↓
Evaluate Data Quality
```

The initial rules validate:

``` text
order_id       → complete
order_line_id  → complete
customer_id    → complete
product_id     → complete
quantity       → > 0
unit_price     → >= 0
region         → complete
```

The project intentionally does not include `order_date` in this initial
ruleset because date validation requires separate handling.

------------------------------------------------------------------------

# 13. Why Data Quality Is a Separate Pipeline Concern

Data quality is treated as a first-class component rather than an
afterthought.

The purpose is to identify deterministic data-quality failures before
the records enter the trusted Silver layer.

Examples include:

``` text
Missing customer_id
Negative quantity
Invalid numeric values
Missing mandatory fields
Invalid date
Duplicate business key
```

### Interview Question

**Q: What is the difference between a processing failure and a
data-quality failure?**

Expected answer:

> A processing failure is generally a failure of the pipeline or
> infrastructure, such as a timeout, service unavailability, or
> throttling. A data-quality failure is a deterministic problem with the
> data itself, such as an invalid date or negative quantity.

------------------------------------------------------------------------

# 14. Retry vs Quarantine

This project explicitly distinguishes transient processing failures from
deterministic data-quality failures.

### Transient Processing Failure

Examples:

``` text
Network timeout
AWS service unavailable
Throttling
Temporary infrastructure failure
```

Expected handling:

``` text
Retry Policy
      ↓
Exponential Backoff
      ↓
Retry
      ↓
Success → Continue
      ↓
Maximum Attempts
      ↓
DLQ / Failure Handling
```

### Deterministic Data-Quality Failure

Examples:

``` text
INVALID_DATE
quantity <= 0
missing customer_id
invalid business value
```

Expected handling:

``` text
Data Quality Failure
        ↓
Quarantine
        ↓
Investigation / Correction
        ↓
Data Reprocessing
```

### Interview Question

**Q: Why shouldn't you endlessly retry an invalid record?**

Expected answer:

> Because the failure is deterministic. Retrying the same invalid record
> without changing the data or processing logic will produce the same
> failure. Retries should be used primarily for transient failures,
> while deterministic bad records should be quarantined.

------------------------------------------------------------------------

# 15. Conditional Routing

A **Conditional Router** was added after Data Quality evaluation.

The intended architecture is:

``` text
Evaluate Data Quality
        ↓
Conditional Router
       / \
      /   \
   PASS   FAIL
    |       |
    ↓       ↓
 Silver  Quarantine
```

The PASS path is intended for records that satisfy the defined quality
requirements.

The failure path will be used for deterministic data-quality failures.

------------------------------------------------------------------------

# 16. Quarantine Design

The quarantine layer should eventually contain:

-   Deterministically invalid records
-   Invalid dates
-   Missing mandatory fields
-   Invalid numeric values
-   Duplicate records
-   Data-quality failure information
-   Error/reason information where possible

The quarantine design is important because a production pipeline should
not simply discard invalid records.

### Interview Question

**Q: What is the purpose of a quarantine zone?**

Expected answer:

> A quarantine zone isolates invalid records from trusted datasets while
> preserving the failed records for investigation, correction, auditing,
> and possible reprocessing.

------------------------------------------------------------------------

# 17. Composite Key Uniqueness

The business key is:

``` text
(order_id, order_line_id)
```

The project intentionally does not treat these as two independent
uniqueness checks.

For example:

``` text
order_id = 1001
order_line_id = 1
```

is different from:

``` text
order_id = 1001
order_line_id = 2
```

The uniqueness requirement applies to the **combination**.

Composite uniqueness will therefore be handled using Spark/SQL logic
rather than incorrectly applying independent `IsUnique` rules.

### Interview Question

**Q: Why can't you simply apply IsUnique to order_id and order_line_id
separately?**

Expected answer:

> Because the business key is composite. Neither column individually
> defines record identity. Uniqueness must be evaluated on the combined
> `(order_id, order_line_id)` key.

------------------------------------------------------------------------

# 18. Silver Layer Target

The eventual Silver layer should contain:

-   Valid records
-   Standardized data types
-   Cleaned records
-   Validated records
-   Deduplicated records
-   Parquet format
-   Quality-controlled data

The intended conceptual structure is:

``` text
Bronze
   ↓
Validation
   ↓
Cleaning
   ↓
Deduplication
   ↓
Type Standardization
   ↓
Silver Parquet
```

------------------------------------------------------------------------

# 19. Production Concepts Being Introduced

The project is intentionally being designed around production Data
Engineering terminology.

Important concepts to understand throughout the implementation include:

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

These concepts should be connected to actual implementation decisions
rather than memorized independently.

------------------------------------------------------------------------

# 20. Interview Explanation of the Current Pipeline

A concise interview explanation can be:

> I implemented the initial Bronze-to-Silver foundation of an AWS data
> lake using S3 and AWS Glue. Raw sales data is landed in an S3 Bronze
> zone and catalogued using the Glue Data Catalog through a crawler. I
> then created a Glue Studio Spark-based ETL job that reads the Catalog
> table, applies controlled schema mapping, performs data-quality
> validation, and conditionally routes records based on quality
> outcomes. I intentionally preserve invalid source data in Bronze and
> distinguish deterministic data-quality failures from transient
> infrastructure failures. Valid records are intended to continue toward
> the Silver layer, while invalid records are routed toward quarantine
> for investigation and reprocessing.

------------------------------------------------------------------------

# 21. Likely Interview Questions

## Architecture

1.  Explain your Bronze-to-Silver architecture.
2.  Why did you choose Medallion Architecture?
3.  What belongs in Bronze versus Silver?
4.  Why is Bronze generally preserved?
5.  Where would business transformations happen?

## S3

6.  Why use S3 as the data lake storage layer?
7.  How would you partition the Silver data?
8.  How would you solve the small-files problem?
9.  How would you implement S3 lifecycle policies?
10. How would you secure the bucket?

## Glue

11. What is a Glue crawler?
12. What is the Glue Data Catalog?
13. Why use a crawler instead of manually defining every table?
14. What are the limitations of schema inference?
15. What IAM permissions does Glue require?

## Data Quality

16. How do you handle missing mandatory fields?
17. How do you handle invalid dates?
18. How do you validate numeric business rules?
19. How do you handle duplicate records?
20. How do you handle composite-key uniqueness?

## Failure Handling

21. What is the difference between retry and quarantine?
22. When would you use exponential backoff?
23. What is a DLQ?
24. How do you prevent endless retries?
25. How would you reprocess quarantined records?

## Data Engineering

26. What is idempotency?
27. What is data lineage?
28. What is data provenance?
29. What is a data contract?
30. What is schema drift?
31. What is schema evolution?
32. What is a watermark?
33. What is a checkpoint?
34. What is a backfill?
35. How do you handle late-arriving data?

------------------------------------------------------------------------

# 22. Current Implementation Status

### Completed

-   [x] S3 Bronze bucket
-   [x] Raw sales source uploaded
-   [x] `de_bronze` Glue Catalog database
-   [x] `crawler-bronze-sales`
-   [x] `glue-crawler-role`
-   [x] Glue service permissions
-   [x] S3 access policy
-   [x] Successful crawler execution
-   [x] `de_bronze.sales` Catalog table
-   [x] `job-bronze-to-silver`
-   [x] Glue Data Catalog source
-   [x] Change Schema transformation
-   [x] Evaluate Data Quality transformation
-   [x] Initial DQ rules
-   [x] Conditional Router
-   [x] PASS processing group

### Next Implementation Areas

The remaining implementation should progressively cover:

1.  Complete PASS/FAIL routing.
2.  Build the Silver output.
3.  Validate and standardize `order_date`.
4.  Implement composite-key deduplication.
5.  Create the quarantine dataset with failure information.
6.  Write Silver data in Parquet.
7.  Introduce partitioning based on actual query and ingestion patterns.
8.  Validate Silver output.
9.  Build the Gold dimensional model.
10. Implement `dim_customer`, `dim_product`, `dim_date`, `dim_region`,
    and `fact_sales`.
11. Implement SCD Types 0, 1, 2, and 3, followed by advanced SCD
    strategies.
12. Implement full and incremental loads.
13. Introduce watermark/high-water-mark processing.
14. Introduce CDC concepts and processing.
15. Add Athena and Redshift analytics.
16. Add orchestration with EventBridge and Step Functions.
17. Add CloudWatch observability.
18. Apply IAM, KMS and data-access controls.
19. Optimize cost and Spark performance.
20. Implement Terraform and CI/CD.

------------------------------------------------------------------------

# 23. Interview Preparation Rule for Every Major Component

For every major implementation step, the project should be explained
using the following framework:

1.  **What are we using?**
2.  **Why are we using it?**
3.  **Where does it fit in the architecture?**
4.  **What happens if it fails?**
5.  **How do we retry?**
6.  **How do we handle bad data?**
7.  **How do we prevent duplicates?**
8.  **How do we monitor it?**
9.  **How do we secure it?**
10. **How do we optimize cost?**
11. **How would this work in production?**
12. **How would I explain it in an interview?**
13. **What interview questions can be asked about it?**

This framework should be applied throughout the project so that the
final result demonstrates not only AWS implementation knowledge, but
also production-oriented Data Engineering reasoning.
