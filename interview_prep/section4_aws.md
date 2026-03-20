# Section 4: AWS Cloud Migration

---

## 4.1 AWS Core Services for Data Engineering

AWS provides the full data engineering stack: S3 for durable object storage, Glue for
serverless ETL, EMR for managed Spark clusters, Lambda for event-driven ingestion, ECS for
containerized microservices, RDS for managed PostgreSQL, and CloudWatch for observability.
At RBC the entire auth service stack ran on ECS + RDS + CloudWatch. At Ontario Government,
Spark ETL jobs ran on EMR reading from S3 and writing to RDS PostgreSQL.

**One-sentence interview answer:** "My AWS data engineering stack is S3 as the data lake
landing zone, Glue for lightweight serverless ETL, EMR for heavy Spark workloads, Lambda for
event-driven triggers, ECS for microservices, and CloudWatch for end-to-end observability."

```
AWS DATA ENGINEERING ARCHITECTURE (RBC-style):
─────────────────────────────────────────────────────────────────
Sources        Landing Zone    Processing        Serving
─────────────────────────────────────────────────────────────────
Investor API ──► S3 (raw/)  ──► Glue ETL ──────► RDS PostgreSQL
                              ──► EMR Spark ─────► S3 (parquet/)
Kafka stream ──► Kinesis    ──► Lambda ──────────► ElastiCache
                              ──► Glue Streaming   (Redis)
Batch files ───► S3 (raw/)  ──► Glue Crawlers ──► Glue Catalog
                                                    (metadata)
─────────────────────────────────────────────────────────────────

S3 Partitioning Strategy (Hive-style):
  s3://bucket/table-name/year=2025/month=08/day=15/part-00001.parquet
  → Glue/Athena/Spark can use partition pruning to skip irrelevant partitions
  → Critical for cost: scanning 1 day vs full table

S3 Lifecycle Policies (cost optimization):
  0-30 days:   S3 Standard (frequent access)
  31-90 days:  S3 Standard-IA (infrequent access, 40% cheaper)
  91-365 days: S3 Glacier Instant Retrieval (archive)
  365+ days:   S3 Glacier Deep Archive (cheapest, hours to retrieve)
```

---

## 4.2 Data Migration to AWS

Cloud migration strategy depends on the business objective and risk tolerance. RBC migrated
from on-premise batch PostgreSQL to AWS RDS + S3 + EMR incrementally (strangler fig pattern)
— new pipelines wrote to AWS while legacy continued, then cutover once parity was proven.

**One-sentence interview answer:** "For database migrations I use AWS DMS for ongoing
replication with minimal downtime; for data pipelines I prefer a parallel-run approach where
new and old systems both produce output until the new system proves parity."

**Most likely follow-ups:**
1. How do you handle schema changes during a live migration?
2. What is the difference between DMS full-load and CDC mode?
3. How do you validate data integrity after migration?

```python
# ── AWS Glue ETL Job: Authentication Log Classification ────────────────────
# Reads raw Thales authentication logs, classifies by protocol,
# writes structured Parquet output to S3.
# Deployed as a Glue 3.0 job (Python Shell) on the Centennial Innovates project.

import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.sql import functions as F
from pyspark.sql.types import *
import boto3
import logging

# ── Job initialization ────────────────────────────────────────────────────
args = getResolvedOptions(sys.argv, [
    'JOB_NAME',
    'source_bucket',       # s3://thales-raw-logs
    'target_bucket',       # s3://thales-processed
    'source_prefix',       # auth-events/
    'target_prefix',       # classified/
    'processing_date'      # 2025-08-01 (passed by Glue trigger or EventBridge rule)
])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# ── Read raw logs using GlueDynamicFrame ──────────────────────────────────
# DynamicFrame advantage over DataFrame: tolerates schema inconsistencies
# (mixed types, extra/missing columns) common in raw log files.
# DynamicFrame.toDF() converts to DataFrame when you need Spark SQL operations.
source_path = f"s3://{args['source_bucket']}/{args['source_prefix']}date={args['processing_date']}/"

logger.info(f"Reading from: {source_path}")

raw_dynamic_frame = glueContext.create_dynamic_frame.from_options(
    connection_type = "s3",
    connection_options = {"paths": [source_path], "recurse": True},
    format = "json",
    format_options = {
        "multiLine": False,
        "jsonPath": "$[*]"
    },
    transformation_ctx = "raw_logs"
)

# Log record count (Glue job metrics)
record_count = raw_dynamic_frame.count()
logger.info(f"Read {record_count} raw records")

# ── Convert to DataFrame for complex transformations ──────────────────────
df = raw_dynamic_frame.toDF()

# ── Classify authentication protocol ─────────────────────────────────────
classified_df = df \
    .withColumn("protocol_type",
        F.when(
            F.col("request_path").startswith("/grpc.") |
            (F.col("content_type") == "application/grpc"),
            "GRPC"
        ).when(
            F.col("request_path").startswith("/api/"),
            "REST"
        ).when(
            F.col("request_path").startswith("/interac/"),
            "INTERAC"
        ).otherwise("UNKNOWN")
    ) \
    .withColumn("is_authenticated",
        F.col("response_code").isin([200, 201])
    ) \
    .withColumn("response_tier",
        F.when(F.col("response_time_ms") < 100, "FAST")
        .when(F.col("response_time_ms") < 500, "NORMAL")
        .otherwise("SLOW")
    ) \
    .withColumn("event_date",
        F.to_date(F.col("event_timestamp"))
    ) \
    .withColumn("event_hour",
        F.hour(F.to_timestamp(F.col("event_timestamp")))
    ) \
    .filter(F.col("request_path").isNotNull()) \
    .filter(F.col("event_timestamp").isNotNull())

# ── Data quality check ────────────────────────────────────────────────────
unknown_protocol_count = classified_df.filter(
    F.col("protocol_type") == "UNKNOWN"
).count()

if unknown_protocol_count > record_count * 0.1:  # >10% unknown = data quality issue
    logger.warning(
        f"Data quality alert: {unknown_protocol_count}/{record_count} records "
        f"have UNKNOWN protocol ({100*unknown_protocol_count/record_count:.1f}%)"
    )
    # In production: publish to SNS topic for alerting
    sns = boto3.client('sns', region_name='ca-central-1')
    sns.publish(
        TopicArn=f"arn:aws:sns:ca-central-1:{args.get('account_id', 'UNKNOWN')}:data-quality-alerts",
        Message=f"Auth log classification: {unknown_protocol_count} UNKNOWN records on {args['processing_date']}",
        Subject="Data Quality Alert: Auth Logs"
    )

# ── Convert back to DynamicFrame and write to S3 ─────────────────────────
output_dynamic_frame = DynamicFrame.fromDF(classified_df, glueContext, "classified_output")

target_path = f"s3://{args['target_bucket']}/{args['target_prefix']}"

# Write partitioned Parquet — Glue handles S3 output partitioning
glueContext.write_dynamic_frame.from_options(
    frame = output_dynamic_frame,
    connection_type = "s3",
    connection_options = {
        "path": target_path,
        "partitionKeys": ["event_date", "protocol_type"]
    },
    format = "glueparquet",   # optimized Parquet writer with statistics
    format_options = {
        "compression": "snappy",
        "blockSize": 134217728,    # 128MB Parquet block size
        "pageSize": 1048576        # 1MB page size
    },
    transformation_ctx = "write_output"
)

logger.info(f"Successfully wrote classified records to {target_path}")

# Update Glue Data Catalog partition metadata
glue_client = boto3.client('glue', region_name='ca-central-1')
try:
    glue_client.batch_create_partition(
        DatabaseName='thales_auth',
        TableName='auth_events_classified',
        PartitionInputList=[{
            'Values': [args['processing_date'], protocol],
            'StorageDescriptor': {
                'Location': f"{target_path}event_date={args['processing_date']}/protocol_type={protocol}/",
                'InputFormat': 'org.apache.hadoop.mapred.TextInputFormat',
                'OutputFormat': 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat',
                'SerdeInfo': {'SerializationLibrary': 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'}
            }
        } for protocol in ['REST', 'GRPC', 'INTERAC', 'UNKNOWN']]
    )
except Exception as e:
    logger.warning(f"Partition registration failed (non-fatal): {e}")

job.commit()  # marks the job as successful in Glue job history
```

```python
# ── AWS Lambda: S3 Event Trigger for Real-Time Ingestion ─────────────────
# Triggered when a new file lands in s3://rbc-raw-events/auth-events/
# Parses the file and inserts records into RDS PostgreSQL.
# Used in RBC to process real-time investor authentication events.

import json
import boto3
import psycopg2
import os
import logging
from datetime import datetime
from typing import List, Dict

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def get_db_connection():
    """Get RDS connection using credentials from Secrets Manager."""
    secrets_client = boto3.client('secretsmanager', region_name='ca-central-1')
    secret = secrets_client.get_secret_value(
        SecretId=os.environ['DB_SECRET_ARN']
    )
    creds = json.loads(secret['SecretString'])

    return psycopg2.connect(
        host=creds['host'],
        port=creds['port'],
        database=creds['database'],
        user=creds['username'],
        password=creds['password'],
        connect_timeout=5
    )

def parse_auth_events(s3_content: str) -> List[Dict]:
    """Parse newline-delimited JSON auth events."""
    events = []
    for line in s3_content.strip().split('\n'):
        if not line:
            continue
        try:
            event = json.loads(line)
            # Validate required fields
            if not all(k in event for k in ['event_id', 'investor_id', 'auth_type']):
                logger.warning(f"Skipping malformed event: {line[:100]}")
                continue
            events.append(event)
        except json.JSONDecodeError as e:
            logger.error(f"JSON parse error: {e} | Line: {line[:100]}")
    return events

def batch_insert_events(conn, events: List[Dict]) -> int:
    """Batch insert using executemany for performance."""
    if not events:
        return 0

    sql = """
        INSERT INTO auth_events (
            event_id, investor_id, auth_type, success,
            response_ms, event_timestamp, protocol_classified, ingested_at
        ) VALUES (
            %(event_id)s, %(investor_id)s, %(auth_type)s, %(success)s,
            %(response_ms)s, %(event_timestamp)s, %(protocol_classified)s, NOW()
        )
        ON CONFLICT (event_id) DO NOTHING
    """

    # Classify protocol (same logic as Glue job — single source of truth
    # should be a shared Lambda layer in production)
    for event in events:
        path = event.get('request_path', '')
        if path.startswith('/grpc.') or event.get('content_type') == 'application/grpc':
            event['protocol_classified'] = 'GRPC'
        elif path.startswith('/api/'):
            event['protocol_classified'] = 'REST'
        elif path.startswith('/interac/'):
            event['protocol_classified'] = 'INTERAC'
        else:
            event['protocol_classified'] = 'UNKNOWN'

    with conn.cursor() as cur:
        cur.executemany(sql, events)
        inserted = cur.rowcount

    conn.commit()
    return inserted

def lambda_handler(event, context):
    """
    S3 event trigger:
    {
        "Records": [
            {
                "s3": {
                    "bucket": {"name": "rbc-raw-events"},
                    "object": {"key": "auth-events/2025/08/01/batch-001.jsonl", "size": 45678}
                }
            }
        ]
    }
    """
    s3_client = boto3.client('s3')
    processed = 0
    errors = []

    for record in event.get('Records', []):
        bucket = record['s3']['bucket']['name']
        key    = record['s3']['object']['key']
        size   = record['s3']['object'].get('size', 0)

        logger.info(f"Processing s3://{bucket}/{key} ({size} bytes)")

        # Skip empty files (can happen with S3 notifications)
        if size == 0:
            logger.info(f"Skipping empty file: {key}")
            continue

        try:
            # Read file from S3
            response = s3_client.get_object(Bucket=bucket, Key=key)
            content  = response['Body'].read().decode('utf-8')

            # Parse events
            auth_events = parse_auth_events(content)
            logger.info(f"Parsed {len(auth_events)} events from {key}")

            # Insert to RDS
            conn = get_db_connection()
            try:
                inserted = batch_insert_events(conn, auth_events)
                logger.info(f"Inserted {inserted} events from {key}")
                processed += inserted
            finally:
                conn.close()

            # Move file to processed/ prefix (atomic S3 copy + delete)
            processed_key = key.replace('auth-events/', 'processed/')
            s3_client.copy_object(
                Bucket=bucket,
                Key=processed_key,
                CopySource={'Bucket': bucket, 'Key': key}
            )
            s3_client.delete_object(Bucket=bucket, Key=key)

        except Exception as e:
            logger.error(f"Failed to process {key}: {e}", exc_info=True)
            errors.append({'key': key, 'error': str(e)})

            # Move to dead-letter prefix for investigation
            try:
                dl_key = key.replace('auth-events/', 'dead-letter/')
                s3_client.copy_object(
                    Bucket=bucket, Key=dl_key,
                    CopySource={'Bucket': bucket, 'Key': key}
                )
            except Exception as dl_err:
                logger.error(f"Failed to move to dead-letter: {dl_err}")

    result = {
        'statusCode': 200 if not errors else 207,
        'processed': processed,
        'errors': len(errors),
        'errorDetails': errors
    }
    logger.info(f"Lambda complete: {result}")
    return result
```

---

## 4.3 AWS Glue Deep Dive

AWS Glue consists of three components: (1) **Crawlers** — discover schemas and populate the
Data Catalog automatically, (2) **ETL Jobs** — serverless Spark jobs using Python or Scala,
and (3) **Data Catalog** — central metadata store used by Athena, EMR, and Redshift. The
key concept for interviews is `DynamicFrame` vs `DataFrame` — DynamicFrames tolerate schema
inconsistencies but are less performant for complex SQL; convert to DataFrame when you need
the full Spark SQL API. At Centennial Innovates, Glue processed clinical data from ADLS-
equivalent S3 buckets.

**One-sentence interview answer:** "Glue Crawlers populate the Data Catalog with inferred
schemas; Glue Jobs are serverless Spark that use DynamicFrames to tolerate schema drift in
raw data; the catalog makes data queryable in Athena without any ETL."

**Most likely follow-ups:**
1. When would you use Glue vs EMR?
2. What is a Glue Bookmark and when is it useful?
3. How do you handle schema evolution in Glue?

```python
# ── Glue ETL Job: Clinical Data Transformation ────────────────────────────
# Centennial Innovates use case: raw clinical CSV/JSON → normalized Parquet
# Key Glue-specific features: DynamicFrame, ResolveChoice, ApplyMapping, Bookmarks

import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.sql import functions as F
from pyspark.sql.types import *

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# ── Job Bookmark: process only NEW files since last run ───────────────────
# Without bookmark: job reprocesses ALL files every run → duplicates
# With bookmark:    Glue tracks last processed file → incremental only
# Enable in Glue console: Job bookmark = "Enable"
# Or in job params: --job-bookmark-option job-bookmark-enable
raw_clinical = glueContext.create_dynamic_frame.from_catalog(
    database    = "clinical_raw",
    table_name  = "patient_records",
    # bookmark automatically applied — reads only new partitions
    transformation_ctx = "clinical_raw_read"
)

print(f"Raw record count: {raw_clinical.count()}")
print(f"Schema: {raw_clinical.schema()}")

# ── ResolveChoice: handle type conflicts ──────────────────────────────────
# If a column has mixed types (int in some rows, string in others),
# DynamicFrame represents it as a CHOICE type.
# ResolveChoice resolves this conflict using different strategies:
#   cast:keep_as_string  — cast all to string (safe but loses types)
#   project:type         — cast all to specified type (loses non-matching)
#   make_cols            — split into type-specific columns (verbose but safe)
resolved = ResolveChoice.apply(
    frame = raw_clinical,
    choice = "cast:long",          # cast all numeric CHOICE fields to long
    transformation_ctx = "resolve_choice"
)

# ── ApplyMapping: rename + type-cast columns declaratively ───────────────
# ApplyMapping is like a SELECT with column renaming + casting — avoids verbose withColumn chains
mapped = ApplyMapping.apply(
    frame = resolved,
    mappings = [
        # (source_col, source_type, target_col, target_type)
        ("patient_id",      "string",  "patient_id",       "string"),
        ("patient_name",    "string",  "full_name",        "string"),
        ("dob",             "string",  "date_of_birth",    "string"),
        ("diagnosis_code",  "string",  "icd10_code",       "string"),
        ("treatment_type",  "string",  "treatment_type",   "string"),
        ("admission_date",  "string",  "admission_date",   "string"),
        ("discharge_date",  "string",  "discharge_date",   "string"),
        ("attending_md",    "string",  "physician_name",   "string"),
        ("visit_cost",      "double",  "visit_cost_cad",   "double"),
        ("insurance_type",  "string",  "insurance_type",   "string")
    ],
    transformation_ctx = "apply_mapping"
)

# ── Convert to DataFrame for complex transformations ──────────────────────
df = mapped.toDF()

transformed = df \
    .withColumn("date_of_birth",
        F.to_date(F.col("date_of_birth"), "yyyy-MM-dd")
    ) \
    .withColumn("admission_date",
        F.to_date(F.col("admission_date"), "yyyy-MM-dd")
    ) \
    .withColumn("discharge_date",
        F.to_date(F.col("discharge_date"), "yyyy-MM-dd")
    ) \
    .withColumn("los_days",                                       # length of stay
        F.datediff(F.col("discharge_date"), F.col("admission_date"))
    ) \
    .withColumn("age_at_admission",
        F.floor(
            F.datediff(F.col("admission_date"), F.col("date_of_birth")) / 365.25
        )
    ) \
    .withColumn("age_group",
        F.when(F.col("age_at_admission") < 18,  "PEDIATRIC")
        .when(F.col("age_at_admission") < 65,  "ADULT")
        .otherwise("SENIOR")
    ) \
    .withColumn("visit_cost_cad",
        F.round(F.col("visit_cost_cad"), 2)
    ) \
    .withColumn("icd10_chapter",
        F.substring(F.col("icd10_code"), 1, 1)  # first letter = ICD chapter
    ) \
    .withColumn("processed_at",
        F.current_timestamp()
    ) \
    .filter(F.col("patient_id").isNotNull()) \
    .filter(F.col("los_days") >= 0)            # data quality: discharge >= admission

# PII masking (compliance — PHIPA in Ontario)
# In production: use AWS Macie to detect PII automatically
pii_masked = transformed \
    .withColumn("full_name",
        F.concat(
            F.substring(F.col("full_name"), 1, 1),   # first initial
            F.lit("***")                               # mask rest
        )
    )

# ── Write to S3 as Parquet ────────────────────────────────────────────────
output_dynamic_frame = DynamicFrame.fromDF(pii_masked, glueContext, "clinical_output")

glueContext.write_dynamic_frame.from_options(
    frame = output_dynamic_frame,
    connection_type = "s3",
    connection_options = {
        "path": "s3://centennial-clinical/processed/patient-records/",
        "partitionKeys": ["icd10_chapter", "age_group"]
    },
    format = "glueparquet",
    format_options = {"compression": "snappy"},
    transformation_ctx = "write_clinical"
)

job.commit()
```

---

## 4.4 AWS EMR — Submit Spark Job via boto3

EMR (Elastic MapReduce) is a managed Spark/Hadoop cluster on EC2. Use EMR (not Glue) when
you need full Spark configuration control, custom JARs, long-running clusters for cost
amortization, or Spark versions Glue doesn't support. At Centennial Innovates, EMR clusters
ran distributed ML training jobs that needed custom Spark configurations not available in Glue.

**One-sentence interview answer:** "I use EMR when I need full Spark cluster control, custom
libraries, or long-running jobs; Glue for serverless short ETL jobs where I want zero cluster
management."

**Most likely follow-ups:**
1. EMR on EC2 vs EMR Serverless — when do you choose each?
2. How do you control EMR costs?
3. How do you submit a Spark job to a running EMR cluster?

```python
# ── Submit Spark job to EMR via boto3 ─────────────────────────────────────
import boto3
import time
import logging
from typing import Optional

logger = logging.getLogger(__name__)

def create_emr_cluster(
    cluster_name: str,
    instance_type: str = "m5.xlarge",
    num_core_nodes: int = 3,
    log_uri: str = "s3://rbc-emr-logs/clusters/"
) -> str:
    """Create a new EMR cluster. Returns cluster ID."""
    emr = boto3.client('emr', region_name='ca-central-1')

    response = emr.run_job_flow(
        Name = cluster_name,
        ReleaseLabel = "emr-6.15.0",   # Spark 3.4.1
        LogUri = log_uri,

        Instances = {
            'MasterInstanceType': instance_type,
            'SlaveInstanceType':  instance_type,
            'InstanceCount':      num_core_nodes + 1,  # +1 for master
            'KeepJobFlowAliveWhenNoSteps': False,       # auto-terminate after steps complete
            'Ec2SubnetId':   'subnet-XXXX',             # private subnet for security
            'Ec2KeyName':    'rbc-emr-key'
        },

        Applications = [
            {'Name': 'Spark'},
            {'Name': 'Hadoop'},
            {'Name': 'Hive'}    # for Glue catalog compatibility
        ],

        Configurations = [
            {
                'Classification': 'spark',
                'Properties': {
                    'maximizeResourceAllocation': 'true'   # auto-tune executor settings
                }
            },
            {
                'Classification': 'spark-defaults',
                'Properties': {
                    'spark.serializer':                   'org.apache.spark.serializer.KryoSerializer',
                    'spark.sql.adaptive.enabled':         'true',
                    'spark.sql.adaptive.coalescePartitions.enabled': 'true',
                    'spark.sql.shuffle.partitions':       '200',
                    'spark.executor.extraJavaOptions':    '-XX:+UseZGC',
                    'spark.driver.extraJavaOptions':      '-XX:+UseZGC',
                }
            },
            {
                'Classification': 'hive-site',
                'Properties': {
                    # Use Glue Data Catalog as Hive Metastore
                    'hive.metastore.client.factory.class':
                        'com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory'
                }
            }
        ],

        BootstrapActions = [
            {
                'Name': 'Install Python dependencies',
                'ScriptBootstrapAction': {
                    'Path': 's3://rbc-scripts/bootstrap/install-deps.sh'
                }
            }
        ],

        ServiceRole      = 'EMR_DefaultRole',
        JobFlowRole      = 'EMR_EC2_DefaultRole',
        EbsRootVolumeSize = 50,

        Tags = [
            {'Key': 'Project',     'Value': 'RBC-GAM-DataPlatform'},
            {'Key': 'Environment', 'Value': 'production'},
            {'Key': 'CostCenter',  'Value': 'data-engineering'}
        ]
    )

    cluster_id = response['JobFlowId']
    logger.info(f"Created EMR cluster: {cluster_id}")
    return cluster_id


def submit_spark_step(
    cluster_id: str,
    job_name: str,
    script_s3_path: str,   # e.g., s3://rbc-scripts/spark-jobs/portfolio-pipeline.py
    args: list = None,
    spark_submit_params: Optional[list] = None
) -> str:
    """Add a Spark step to an existing cluster. Returns step ID."""
    emr = boto3.client('emr', region_name='ca-central-1')

    command = [
        'spark-submit',
        '--deploy-mode', 'cluster',    # driver runs on cluster (not local)
        '--master',      'yarn',
    ]

    if spark_submit_params:
        command.extend(spark_submit_params)

    command.extend([script_s3_path])

    if args:
        command.extend(args)

    response = emr.add_job_flow_steps(
        JobFlowId = cluster_id,
        Steps = [{
            'Name': job_name,
            'ActionOnFailure': 'CONTINUE',  # or 'TERMINATE_CLUSTER' if step is critical
            'HadoopJarStep': {
                'Jar': 'command-runner.jar',
                'Args': command
            }
        }]
    )

    step_id = response['StepIds'][0]
    logger.info(f"Submitted step {step_id} to cluster {cluster_id}")
    return step_id


def wait_for_step(cluster_id: str, step_id: str, timeout_minutes: int = 60) -> str:
    """Poll until step completes. Returns final state."""
    emr = boto3.client('emr', region_name='ca-central-1')
    deadline = time.time() + timeout_minutes * 60

    while time.time() < deadline:
        response = emr.describe_step(ClusterId=cluster_id, StepId=step_id)
        state = response['Step']['Status']['State']

        if state in ('COMPLETED', 'FAILED', 'CANCELLED', 'INTERRUPTED'):
            logger.info(f"Step {step_id} finished with state: {state}")
            if state == 'FAILED':
                failure_reason = response['Step']['Status'].get('FailureDetails', {})
                logger.error(f"Step failure: {failure_reason}")
            return state

        logger.info(f"Step {step_id} state: {state} — waiting...")
        time.sleep(30)

    raise TimeoutError(f"Step {step_id} did not complete within {timeout_minutes} minutes")


# ── Example usage ─────────────────────────────────────────────────────────
def run_portfolio_pipeline(processing_date: str):
    """End-to-end: create cluster → submit job → wait → log result."""
    cluster_id = create_emr_cluster(
        cluster_name  = f"rbc-portfolio-pipeline-{processing_date}",
        instance_type = "m5.2xlarge",
        num_core_nodes = 4
    )

    step_id = submit_spark_step(
        cluster_id    = cluster_id,
        job_name      = "Portfolio Aggregation",
        script_s3_path = "s3://rbc-scripts/spark/portfolio-aggregation.py",
        args          = ['--processing_date', processing_date,
                         '--output_bucket', 'rbc-gam-output'],
        spark_submit_params = [
            '--conf', 'spark.executor.memory=8g',
            '--conf', 'spark.executor.cores=4',
            '--conf', 'spark.driver.memory=4g',
            '--num-executors', '8'
        ]
    )

    final_state = wait_for_step(cluster_id, step_id)
    return {'cluster_id': cluster_id, 'step_id': step_id, 'state': final_state}


# ── EMR vs Glue decision guide ────────────────────────────────────────────
"""
USE GLUE WHEN:
  - Serverless (no cluster management)
  - Short ETL jobs (minutes, not hours)
  - Schema discovery via crawlers is valuable
  - Simple column transformations, type mapping
  - Budget: pay per DPU-second (no idle cost)
  - Glue Studio visual editor for less-technical users

USE EMR WHEN:
  - Long-running jobs (hours) — amortize cluster startup cost
  - Need specific Spark version/config not available in Glue
  - Custom JARs or native code libraries
  - ML training jobs requiring GPU instances (p3, g4)
  - Large clusters (100+ nodes) where per-second EMR pricing beats Glue DPU
  - Need YARN/Kubernetes cluster manager features
  - Need to share cluster across multiple teams

EMR SERVERLESS (newer):
  - Combines EMR power (full Spark config) with Glue-like serverless model
  - No cluster pre-provisioning
  - Best of both worlds for most new workloads
"""
```

---
