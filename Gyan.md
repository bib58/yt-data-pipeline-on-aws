# YouTube Data Engineering Pipeline on AWS

### 1. Prepare the Amazon S3 Storage

1. Create or identify the S3 bucket used for the Bronze layer. The notes refer to `bibhu-data-pipeline-brozne` (check the spelling against the actual bucket name in AWS).
2. Upload the raw dataset using `scripts/aws_copy.sh`.
3. Keep the raw statistics and reference data under the appropriate prefixes, such as:
   - `youtube/raw_statistics/`
   - `youtube/raw_statistics_reference_data/`
4. Configure an S3 Lifecycle rule for the Bronze data if archival is required: **S3 bucket → Management → Create lifecycle rule → transition to S3 Glacier Flexible Retrieval**.

### 2. Create IAM Roles and Permissions

1. In IAM, create the Lambda execution role `lambda-role-bibhu`.
2. Attach the `AWSLambdaBasicExecutionRole` managed policy.
3. Add `iam_permission/data-pipeline-lambda-access.json` as an inline policy.
4. Create the Glue role `glue-role-bibhu`.
5. Attach the `AWSGlueServiceRole` managed policy.
6. Add `iam_permission/data-pipeline-glue-access.json` as an inline policy.
7. Later, add the permissions required by Athena, data quality, and Step Functions when those components are configured. Grant only the permissions each role needs.

### 3. Configure YouTube API Ingestion with Lambda

1. In Google Cloud, enable the **YouTube Data API v3** and create the credentials required by the ingestion code.
2. In AWS Lambda, create a function for API ingestion using `lambdas/youtube_api_integstion/lambda_function.py`.
3. Configure the function to use `lambda-role-bibhu`.
4. Add the required environment variables in **Configuration → Environment variables**. Use the variable names expected by the code, and do not put credentials directly in source code.
5. Set the runtime to the Python version supported by the function and its dependencies. The original notes specify Python 3.11 for the pandas/AWS SDK for pandas layer; ensure the runtime and layer versions match.
6. Test the function and confirm that raw JSON data is written to the Bronze S3 location.

### 4. Set Up the JSON-to-Parquet Lambda

1. Create a Lambda function named `json-to-parquet`.
2. Use `lambdas/json_to_parquet/lambda_function.py` as its code.
3. Configure the execution role with the permissions needed to read from the Bronze bucket and write to the Silver bucket.
4. Add the **AWSSDKPandas-Python311** Lambda layer (also known as AWS SDK for pandas; it includes `awswrangler`) if the function runtime is Python 3.11.
5. Configure the environment variables expected by the function.
6. Set suitable memory and timeout values, then test with a sample JSON object.
7. Add an S3 event trigger so that a newly created JSON object in the Bronze prefix invokes this function. Confirm that the function writes a Parquet file to the Silver location.

### 5. Configure Amazon SNS Notifications

1. Open Amazon Simple Notification Service (SNS) and create a topic.
2. Create an email subscription to the topic.
3. Confirm the subscription using the confirmation email.
4. Update the Lambda role's inline policy with the topic ARN and the required SNS publish permission, if the Lambda code sends notifications.
5. Test that a notification is delivered when the relevant event occurs.

### 6. Create AWS Glue Databases and Crawlers

1. Open **AWS Glue → Data Catalog → Databases**.
2. Create databases for the Bronze, Silver, and Gold layers.
3. Create crawlers for the S3 prefixes that need catalog metadata. The original notes list these Bronze paths:
   - `s3://bibhu-data-pipeline-brozne/youtube/raw_statistics/`
   - `s3://bibhu-data-pipeline-brozne/youtube/raw_statistics_reference_data/`
4. Assign `glue-role-bibhu` to each crawler.
5. Select the appropriate target database (the notes name `glue-bronze-bibhu` for Bronze; use the corresponding database for each other layer).
6. Run the crawlers and verify that tables appear in the Glue Data Catalog.

### 7. Create an Amazon Athena Query-Results Bucket

1. Create an S3 bucket for Athena query results, using `glue-athena-query-result-bibhu` if that name is available.
2. In the Athena console, open **Settings / Query result configuration**.
3. Set the results location to the bucket created above.
4. Verify that the configured role or user has permission to write query results to that bucket.

### 8. Configure a Data-Quality Lambda

1. Create a Lambda function using `lambdas/data_quality/dq_lambda.py`.
2. Attach the **AWSSDKPandas-Python311** layer if the function uses `awswrangler` and runs on Python 3.11.
3. Configure the required environment variables.
4. Add the data-quality and Athena permissions from `iam_permission/data-quality-bibhu-permissions.json` to the appropriate role. Keep permissions scoped to the resources used by the function.
5. Test the function against representative data and verify its output and notifications, if configured.

### 9. Build the Bronze-to-Silver ETL Job with AWS Glue

1. Open **AWS Glue → ETL jobs** and create a job in the script editor.
2. Use `glue_jobs/bronze_to_silver_statistics.py` as the job script.
3. Configure the Glue job role (`glue-role-bibhu`) and add the job parameters expected by the script in **Job details**.
4. Confirm that the job can read the Bronze data and write to the Silver S3 location.
5. Run the job and check the logs for errors.
6. Verify that the transformed statistics are written under the `youtube/` prefix in the Silver bucket.
7. Run or configure a crawler for the Silver output so that its tables and schema are available in the Glue Data Catalog.

### 10. Build the Silver-to-Gold Analytics Job

1. Create a second AWS Glue ETL job using `glue_jobs/silver_to_gold_analytics.py`.
2. Configure its role, input/output locations, and job parameters.
3. Read the curated Silver data and write the analytics-ready Gold output to the Gold S3 location.
4. Run the job and validate the output files.
5. Run or configure a crawler for the Gold location so Athena can discover the Gold tables.

### 11. Query the Data with Amazon Athena

1. Open Athena and select the correct Glue Data Catalog database.
2. Run SQL queries against the cataloged Silver or Gold tables.
3. Check that the query-results bucket is configured and that the querying identity has the required S3 and Athena permissions.
4. Validate a few counts and sample rows against the source data.

### 12. Build Dashboards with Amazon QuickSight

1. Connect QuickSight to the relevant Athena datasets (or another supported source).
2. Select the Gold tables intended for reporting.
3. Create charts and dashboards for views, likes, trending categories, regions, and other useful measures.
4. Validate the dashboard values against Athena query results.

### 13. Orchestrate the Pipeline with AWS Step Functions

1. Create an IAM role for Step Functions.
2. Add the permissions in `iam_permission/data-pipeline-sfn-access.json` as an inline policy, reviewing the resources and actions before use.
3. Create a state machine using `step_functions/pipeline_orchestation.json`.
4. Confirm that the state machine can invoke the relevant Lambda functions and Glue jobs.
5. Execute a test run and inspect each state for success or failure.
6. Configure retries and failure handling as needed, then run the complete workflow end to end.

---
---
---

## AWS Services Notes

### Amazon S3 and S3 Glacier

- **Amazon S3** stores the raw, transformed, and analytics-ready datasets in the Bronze, Silver, and Gold layers.
- **S3 Glacier Flexible Retrieval** is a lower-cost archival storage class for data that is rarely accessed. A lifecycle rule can transition older Bronze data to an archive storage class. Consider retrieval time and transition/retrieval charges before applying a rule.

### Amazon EC2 Spot Instances

Spot Instances use spare EC2 capacity at a discount compared with On-Demand pricing. AWS can reclaim the capacity, so they suit fault-tolerant or interruptible workloads rather than workloads that require uninterrupted compute.

### AWS Glue

AWS Glue is a managed, serverless data integration and ETL service. It can extract data from S3, transform it, and write results to destinations such as S3 or Amazon Redshift.

Key points:
- **ETL jobs** run data transformation scripts.
- **Glue Data Catalog** stores metadata such as table names, column names, data types, and S3 locations.
- **Glue Crawlers** scan data sources, infer schemas, and create or update catalog tables.
- Glue integrates with services such as S3, Athena, and Redshift.

### Amazon EMR

Amazon EMR is a managed big-data platform for running frameworks such as Apache Spark, Hadoop, and Hive. It is suited to distributed processing of large datasets and offers more control over the processing environment than Glue's serverless ETL approach.

**Quick comparison:** AWS Glue is primarily a serverless data integration/ETL service; Amazon EMR is a managed platform for big-data frameworks and cluster-based processing.

### AWS Lambda

AWS Lambda runs code in response to events or requests without requiring the user to manage servers. In this pipeline, Lambda is used for API ingestion, JSON-to-Parquet conversion, and data-quality tasks.

### Amazon EC2 vs. AWS Lambda

| Feature | Amazon EC2 | AWS Lambda |
| --- | --- | --- |
| Definition | Virtual server in the cloud | Serverless compute service |
| Management | User manages the server and operating system | AWS manages the underlying infrastructure |
| Execution | Can run continuously until stopped | Runs code in response to events or requests |
| Scaling | Manual or configured with Auto Scaling | Scales automatically with demand within service limits |
| Billing | Based primarily on provisioned instance usage | Based on requests and execution duration |
| Common uses | Websites, applications, long-running processes | APIs, event processing, background tasks |

### AWS IAM: Users and Roles

IAM (Identity and Access Management) controls access to AWS resources. Users represent identities; roles provide permissions that trusted users or services can assume, generally using temporary credentials.

| Feature | IAM User | IAM Role |
| --- | --- | --- |
| Purpose | Represents a person or application identity | Grants permissions to a trusted entity that assumes it |
| Credentials | May have long-term credentials | Uses temporary security credentials when assumed |
| Access | Can be used directly where authorized | Assumed by an authorized principal or AWS service |
| Example | A developer accessing AWS | A Lambda function or EC2 instance accessing S3 |
| Security consideration | Long-term credentials require careful protection | Temporary credentials reduce reliance on permanent keys |

Prefer IAM roles for AWS workloads instead of embedding access keys in application code.

### Apache Parquet

Apache Parquet is a columnar file format designed for efficient analytical storage and processing.

| Feature | CSV | Parquet |
| --- | --- | --- |
| Storage layout | Text arranged by rows | Columnar binary format |
| Compression | Often less efficient for analytics | Commonly supports efficient compression |
| Query performance | May require scanning many fields | Can read selected columns efficiently |
| Data types | Values are represented as text | Stores typed values such as integers and timestamps |
| File size | Often larger for analytical datasets | Often smaller for similar analytical data |

**Remember:** Parquet combines columnar storage, typed data, and efficient compression. It can be stored in S3 and queried using Athena.

### Amazon Athena

Athena is a serverless SQL query service for analyzing data in S3 without managing query servers or loading the data into a traditional database. It commonly uses table metadata from the Glue Data Catalog.

### Amazon Redshift Spectrum

Redshift Spectrum lets Amazon Redshift query data stored in S3 without first loading that external data into Redshift tables. It can be useful for combining warehouse data with data in S3.

### AWS Glue Data Catalog

The Glue Data Catalog is a centralized metadata repository for datasets, including table names, schemas, and S3 locations. Crawlers can populate or update this metadata, which Athena can then use to query data.

### Amazon SNS

Amazon Simple Notification Service (SNS) is a publish/subscribe messaging service. A topic can distribute notifications to subscribers, such as an email endpoint. Email subscribers must confirm their subscription before receiving messages.

### Amazon QuickSight

QuickSight is a business intelligence service for building interactive dashboards, charts, and reports from AWS and other data sources. It can use Athena, S3, and Redshift data sources. **SPICE** (Super-fast, Parallel, In-memory Calculation Engine) is QuickSight's in-memory engine for accelerating analytics.

### Amazon Kinesis

Amazon Kinesis provides services for collecting and processing streaming data in near real time.

- **Kinesis Data Streams:** Collects and streams real-time data.
- **Kinesis Data Firehose:** Delivers streaming data to supported destinations such as S3 and Redshift.
- **Managed Service for Apache Flink:** Processes and analyzes streaming data.

### AWS Step Functions

Step Functions orchestrates multi-step workflows across AWS services such as Lambda and Glue.

Key features:
- **Workflow orchestration:** Coordinates service tasks in a defined sequence.
- **Error handling:** Supports retries and catch rules.
- **Conditional logic:** Uses Choice states to select workflow paths.
- **Parallel execution:** Runs independent tasks concurrently.
- **Monitoring:** Tracks workflow executions and state transitions.

## Important Configuration Checks

- Confirm the actual bucket names and spelling before configuring triggers, crawlers, lifecycle rules, or IAM policies.
- Keep Lambda runtime versions compatible with their layers; the original notes specify Python 3.11 for `AWSSDKPandas-Python311`.
- The original notes mention memory values of 512 MB and 1,024 MB and timeouts of 5 seconds for some Lambda functions, but also mention a 5-minute timeout for the data-quality function. Set values based on actual processing needs and Lambda service limits rather than applying one value to every function.
- Check that S3 event notifications do not create recursive invocations. For example, trigger on the Bronze prefix and write output to a different Silver bucket or prefix.
- Validate IAM policies, SNS topic ARNs, Lambda environment variables, Glue job parameters, and Athena result-bucket permissions before running the full pipeline.
- Treat the filenames and resource names in these notes as project-specific examples; confirm they match the repository and AWS resources you actually use.
