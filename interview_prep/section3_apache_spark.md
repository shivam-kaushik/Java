# Section 3: Apache Spark

---

## 3.1 Spark Architecture

Spark is a distributed in-memory compute engine. The Driver is your program — it builds the
execution plan. Executors are JVM processes on worker nodes that run the actual tasks. The
DAG Scheduler breaks your transformations into stages; the Task Scheduler dispatches tasks
to executors. At Ontario Government, Spark on EMR processed case note files from S3 using
this architecture — 16 executor cores across 4 m5.xlarge nodes, processing 2M records in
under 4 minutes instead of 45 minutes with the old batch job.

**One-sentence interview answer:** "The Driver builds a DAG of transformations, divides it
into stages at shuffle boundaries, and the Task Scheduler dispatches tasks to Executors
running on worker nodes."

**Most likely follow-ups:**
1. What happens if the Driver dies during a job?
2. How does Spark achieve fault tolerance?
3. What is the difference between a stage and a task?

```
┌──────────────────────────────────────────────────────────────────┐
│                    SPARK CLUSTER ARCHITECTURE                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐         ┌─────────────────────────────────┐   │
│  │   Driver    │         │       Cluster Manager           │   │
│  │  (your app) │◄───────►│  (YARN / Kubernetes / Standalone)│  │
│  │             │         └─────────────────────────────────┘   │
│  │ SparkContext│                    ▲                           │
│  │ DAG Sched.  │                    │ allocates                 │
│  │ Task Sched. │                    ▼                           │
│  └─────────────┘    ┌──────────────────────────────────────┐   │
│        │            │           Worker Node                 │   │
│        │            │  ┌──────────────────────────────┐    │   │
│        │ tasks       │  │        Executor JVM          │    │   │
│        └────────────►│  │  ┌────────┐ ┌────────┐      │    │   │
│                      │  │  │ Task 1 │ │ Task 2 │ ...  │    │   │
│                      │  │  └────────┘ └────────┘      │    │   │
│                      │  │  [Cache] [Shuffle Buffer]    │    │   │
│                      │  └──────────────────────────────┘    │   │
│                      └──────────────────────────────────────┘   │
│                                                                  │
│  DAG: filter(col > 0) → map(transform) → groupBy → agg          │
│       │___Stage 1_______│               │___Stage 2___│         │
│                         ^                                        │
│                  Shuffle boundary: Stage 1 writes               │
│                  shuffle files; Stage 2 reads them              │
└──────────────────────────────────────────────────────────────────┘

Key concepts:
  Job       = one action (e.g., count(), write())
  Stage     = group of tasks with no shuffle between them
  Task      = one unit of work on one partition on one executor
  Partition = a slice of the data (one task per partition)

  Fault tolerance via lineage (RDD) / WAL (Streaming):
    If an executor dies, the Driver re-runs the failed tasks
    on another executor using the RDD lineage to recompute data.
```

---

## 3.2 RDD vs DataFrame vs Dataset

All three are Spark's distributed data abstractions, but they differ in type safety,
optimization, and API style. DataFrames (with Catalyst optimizer) are the standard choice
for ETL work — Spark can push down filters, prune columns, and generate efficient execution
plans automatically. Datasets add compile-time type safety in Scala/Java. Raw RDDs have no
optimizer — avoid unless you need very custom low-level operations.

**One-sentence interview answer:** "Use DataFrames for most ETL work (Catalyst optimizes
them automatically), Datasets when compile-time type safety matters in Scala, and RDDs only
when you need fine-grained control that DataFrames can't express."

**Most likely follow-ups:**
1. Why is DataFrame generally faster than a well-written RDD?
2. What is the Catalyst optimizer?
3. When would you actually use an RDD in 2024?

```scala
// ── Same operation in RDD, DataFrame, and Dataset ─────────────────────────
// Use case: filter investor transactions > $10,000, sum by portfolio type.
// This mirrors the Ontario Government case data pipeline — here adapted to
// investor data for the RBC context.

import org.apache.spark.sql._
import org.apache.spark.sql.functions._
import org.apache.spark.rdd.RDD

object RDDvsDataFramevsDataset {

  case class InvestorTransaction(
    transactionId: String,
    investorId:    String,
    portfolioType: String,
    amount:        Double,
    txnType:       String   // "BUY", "SELL", "FEE"
  )

  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("RBC-API-Comparison")
      .master("local[*]")  // local dev; in prod: yarn or k8s
      .getOrCreate()

    import spark.implicits._

    // Sample data
    val data = Seq(
      InvestorTransaction("T1", "INV-001", "EQUITY",       15000.0, "BUY"),
      InvestorTransaction("T2", "INV-001", "EQUITY",        8000.0, "BUY"),
      InvestorTransaction("T3", "INV-002", "FIXED_INCOME", 25000.0, "BUY"),
      InvestorTransaction("T4", "INV-002", "BALANCED",      4000.0, "SELL"),
      InvestorTransaction("T5", "INV-003", "EQUITY",       30000.0, "BUY")
    )

    // ── RDD approach ──────────────────────────────────────────────────────
    // Pro: full control over serialization, partitioning, custom algorithms
    // Con: NO Catalyst optimization, NO column pruning, verbose API,
    //      Java serialization overhead (unless using Kryo)
    val rdd: RDD[InvestorTransaction] = spark.sparkContext.parallelize(data)

    val rddResult: Map[String, Double] = rdd
      .filter(t => t.txnType == "BUY" && t.amount > 10_000)
      .map(t => (t.portfolioType, t.amount))    // PairRDD
      .reduceByKey(_ + _)                       // shuffle: groups by key across partitions
      .collect()
      .toMap

    println("RDD result: " + rddResult)

    // ── DataFrame approach ────────────────────────────────────────────────
    // Pro: Catalyst optimizer, Tungsten execution engine (off-heap, no GC),
    //      column pruning, predicate pushdown, code generation
    // Con: No compile-time type safety — column("typo") fails at runtime
    val df: DataFrame = data.toDF()   // converts Seq to DataFrame

    val dfResult = df
      .filter($"txnType" === "BUY" && $"amount" > 10_000)
      .groupBy("portfolioType")
      .agg(sum("amount").as("totalBuyAmount"))
      .orderBy($"totalBuyAmount".desc)

    dfResult.show()
    // Catalyst optimizes this to: push filter before groupBy, skip unneeded columns

    // ── Dataset approach ──────────────────────────────────────────────────
    // Pro: compile-time type safety (typo in field name = compile error),
    //      Catalyst optimization still applies,
    //      functional API like RDD (map, filter with lambdas on typed objects)
    // Con: JVM object overhead vs DataFrame's row format; slightly slower for pure SQL ops
    val ds: Dataset[InvestorTransaction] = data.toDS()

    val dsResult = ds
      .filter(t => t.txnType == "BUY" && t.amount > 10_000)  // typed lambda
      .groupByKey(_.portfolioType)                             // type-safe groupBy
      .mapGroups { (portfolioType, transactions) =>
        val total = transactions.map(_.amount).sum
        (portfolioType, total)
      }

    dsResult.show()

    // ── Performance comparison summary ─────────────────────────────────
    // RDD:       Slowest. No optimizer. Use for custom partitioning/algorithms.
    // DataFrame: Fastest for SQL-style ops. Catalyst + Tungsten. Use 90% of the time.
    // Dataset:   Same speed as DataFrame for agg/join/filter on columns.
    //            Slightly slower for map/mapGroups on typed objects (JVM deserialization).
    //            Use when type safety is worth the slight overhead.

    spark.stop()
  }
}
```

---

## 3.3 Transformations vs Actions + Lazy Evaluation

Spark is lazy: transformations build a DAG of *instructions* but do no work. Only when an
action is called does Spark actually execute. This lets Spark optimize the entire pipeline
as a whole — e.g., push a filter down before an expensive join. Wide transformations
(`groupByKey`, `join`, `repartition`) trigger a shuffle — data moves across the network —
marking stage boundaries. Narrow transformations (`map`, `filter`, `flatMap`) stay within
a partition.

**One-sentence interview answer:** "Transformations are lazy — they build a DAG without executing;
actions trigger execution; the DAG Scheduler optimizes the plan and divides it into stages at
shuffle boundaries."

**Most likely follow-ups:**
1. Why is `reduceByKey` better than `groupByKey` for aggregation?
2. What is a shuffle and why is it expensive?
3. Can you explain a real scenario where lazy evaluation helped performance?

```scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.expressions.Window

// ── Authentication Log Classification Pipeline ────────────────────────────
// Ties to Thales: millions of daily auth events classified as REST vs gRPC.
// Now running as a Spark job on EMR, processing S3 logs.
object AuthLogClassificationPipeline {

  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("Thales-Auth-Log-Classifier")
      .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
      .getOrCreate()

    import spark.implicits._

    // ── Transformations (lazy — nothing executes here) ─────────────────
    val rawLogs = spark.read           // LAZY: no data read yet
      .option("header", "true")
      .option("inferSchema", "true")
      .csv("s3://thales-logs/auth-events/year=2025/month=08/")

    // Narrow transformation: filter — stays within partitions, no shuffle
    val validLogs = rawLogs            // LAZY
      .filter($"event_status".isNotNull)
      .filter($"source_ip".isNotNull)

    // Narrow: map-style column derivation
    val classified = validLogs         // LAZY
      .withColumn("protocol_type",
        when($"request_path".startsWith("/grpc."), "GRPC")
        .when($"content_type" === "application/grpc", "GRPC")
        .when($"request_path".startsWith("/api/"), "REST")
        .otherwise("UNKNOWN")
      )
      .withColumn("is_authenticated",
        $"response_code" === 200 || $"response_code" === 201
      )
      .withColumn("processing_date",
        to_date($"event_timestamp")
      )

    // Wide transformation: groupBy + agg — triggers SHUFFLE (stage boundary)
    // Data with the same protocol_type moves to the same partition
    val dailyStats = classified        // LAZY
      .groupBy("processing_date", "protocol_type")
      .agg(
        count("*").as("total_requests"),
        sum(when($"is_authenticated", 1).otherwise(0)).as("successful_auths"),
        avg("response_time_ms").as("avg_response_time"),
        max("response_time_ms").as("max_response_time"),
        countDistinct("source_ip").as("unique_sources")
      )

    // Window function: running total per protocol (no shuffle needed in same partition)
    val windowSpec = Window
      .partitionBy("protocol_type")
      .orderBy("processing_date")
      .rowsBetween(Window.unboundedPreceding, Window.currentRow)

    val withRunningTotal = dailyStats  // LAZY
      .withColumn("running_total",
        sum("total_requests").over(windowSpec)
      )

    // ── Actions (TRIGGER execution — DAG executes here) ──────────────
    // Everything above has been just building the plan. Now Spark optimizes
    // the full DAG (e.g., predicate pushdown on CSV read) and executes.

    // count() — action: triggers full pipeline execution
    val recordCount = withRunningTotal.count()
    println(s"Total daily stat records: $recordCount")

    // show() — action: triggers execution, collects first 20 rows to Driver
    // WARNING: never use show() on production large datasets in a loop
    withRunningTotal.show(20, truncate = false)

    // write — action: triggers execution, writes output
    withRunningTotal
      .repartition(4, $"protocol_type")  // control output file count
      .write
      .mode(SaveMode.Overwrite)
      .partitionBy("processing_date", "protocol_type")  // Hive-style partitioning
      .parquet("s3://thales-output/auth-stats/")

    // ── groupByKey vs reduceByKey (RDD API) ───────────────────────────
    // groupByKey: ALL values for a key move to same partition → huge shuffle
    //   rdd.groupByKey().mapValues(_.sum)  ← BAD: transfers all values
    //
    // reduceByKey: partial aggregation happens BEFORE shuffle (map-side combine)
    //   rdd.reduceByKey(_ + _)             ← GOOD: only partial sums transferred
    //
    // DataFrame groupBy().agg() automatically uses map-side combine — prefer this.

    spark.stop()
  }
}
```

---

## 3.4 Spark SQL + DataFrames

SparkSQL lets you write SQL queries against DataFrames, making data transformation accessible
to data engineers without requiring deep Spark knowledge. Schema definition (explicit vs
inferred) is a common interview topic — for production pipelines, always define explicit
schemas to fail fast on bad data rather than silently inferring wrong types. Window functions
are essential for financial analytics like rolling 30-day portfolio valuations.

**One-sentence interview answer:** "Spark SQL runs SQL queries against DataFrames using the
same Catalyst optimizer — explicit schema definition catches data quality issues early and
window functions handle time-series financial analytics efficiently."

**Most likely follow-ups:**
1. What is the difference between `spark.sql(...)` and the DataFrame DSL?
2. How does schema inference cause performance problems?
3. How do you read from S3 and write to PostgreSQL in the same job?

```scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._
import org.apache.spark.sql.expressions.Window
import java.util.Properties

// ── Investor Portfolio Aggregation with SparkSQL ──────────────────────────
// Read raw portfolio snapshots from S3, aggregate, write to PostgreSQL.
// This mirrors the Ontario Government pipeline: S3 → Spark → PostgreSQL.
object InvestorPortfolioPipeline {

  // ── Explicit Schema (always preferred over inferSchema in production) ───
  // inferSchema reads the full file twice — once to infer, once to load.
  // It also guesses types which can be wrong (e.g., ZIP codes as integers).
  val portfolioSchema: StructType = StructType(Array(
    StructField("snapshot_id",     StringType,     nullable = false),
    StructField("investor_id",     StringType,     nullable = false),
    StructField("portfolio_type",  StringType,     nullable = false),
    StructField("total_value",     DecimalType(18,2), nullable = false),
    StructField("equity_pct",      DoubleType,     nullable = true),
    StructField("bond_pct",        DoubleType,     nullable = true),
    StructField("cash_pct",        DoubleType,     nullable = true),
    StructField("snapshot_date",   DateType,       nullable = false),
    StructField("region",          StringType,     nullable = true)
  ))

  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("RBC-Portfolio-Aggregation")
      .config("spark.sql.session.timeZone", "America/Toronto")
      .config("spark.sql.adaptive.enabled", "true")       // AQE: auto-optimize at runtime
      .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
      .getOrCreate()

    // ── Read from S3 with explicit schema ─────────────────────────────
    val rawPortfolios = spark.read
      .schema(portfolioSchema)                  // explicit schema: fail fast on type mismatch
      .option("mode", "FAILFAST")               // throw exception on malformed records
      .option("dateFormat", "yyyy-MM-dd")
      .parquet("s3://rbc-gam-data/portfolio-snapshots/year=2025/month=08/")

    // ── Register as temp view for SQL queries ─────────────────────────
    rawPortfolios.createOrReplaceTempView("portfolio_snapshots")

    // ── SparkSQL: aggregation ─────────────────────────────────────────
    val monthlyAgg = spark.sql("""
      SELECT
        investor_id,
        portfolio_type,
        DATE_TRUNC('month', snapshot_date)   AS month,
        AVG(total_value)                     AS avg_value,
        MAX(total_value)                     AS peak_value,
        MIN(total_value)                     AS trough_value,
        LAST(total_value, TRUE)              AS end_of_month_value,
        STDDEV(total_value)                  AS value_volatility
      FROM portfolio_snapshots
      WHERE total_value > 0
        AND snapshot_date >= DATE_SUB(CURRENT_DATE, 90)
      GROUP BY investor_id, portfolio_type, DATE_TRUNC('month', snapshot_date)
      ORDER BY investor_id, month
    """)

    // ── Window functions (DataFrame DSL) ──────────────────────────────
    // Window functions: compute over a "window" of rows related to current row
    // without collapsing them like groupBy does.

    val investorWindow = Window
      .partitionBy("investor_id")
      .orderBy("snapshot_date")

    val rollingWindow = Window
      .partitionBy("investor_id")
      .orderBy("snapshot_date")
      .rowsBetween(-29, 0)   // 30-day rolling window (current row + 29 prior)

    val withWindowMetrics = rawPortfolios
      .withColumn("prev_value",
        lag($"total_value", 1).over(investorWindow)
      )
      .withColumn("daily_return_pct",
        (($"total_value" - $"prev_value") / $"prev_value" * 100).cast(DecimalType(8, 4))
      )
      .withColumn("rolling_30d_avg",
        avg($"total_value").over(rollingWindow)
      )
      .withColumn("rolling_30d_max",
        max($"total_value").over(rollingWindow)
      )
      .withColumn("rank_by_value",
        dense_rank().over(
          Window.partitionBy("portfolio_type", "snapshot_date")
                .orderBy($"total_value".desc)
        )
      )

    // ── Write to PostgreSQL via JDBC ──────────────────────────────────
    // Connection props: use environment variables in production (ECS task env)
    val jdbcProps = new Properties()
    jdbcProps.put("user",     sys.env("DB_USER"))
    jdbcProps.put("password", sys.env("DB_PASSWORD"))
    jdbcProps.put("driver",   "org.postgresql.Driver")
    jdbcProps.put("batchsize", "10000")           // batch insert for performance
    jdbcProps.put("numPartitions", "10")          // parallel write partitions

    // Write in batches — avoid overwhelming the database
    withWindowMetrics
      .filter($"snapshot_date" >= date_sub(current_date(), 7))  // last 7 days only
      .select("investor_id", "portfolio_type", "snapshot_date",
              "total_value", "daily_return_pct", "rolling_30d_avg", "rank_by_value")
      .write
      .mode(SaveMode.Append)
      .jdbc(
        sys.env("DB_JDBC_URL"),     // jdbc:postgresql://rds-host:5432/investor_db
        "portfolio_analytics",
        jdbcProps
      )

    println("Portfolio aggregation pipeline complete")
    spark.stop()
  }
}
```

---

## 3.5 Spark Optimization

Spark optimization is one of the most-asked interview topics at data engineering roles. The
three biggest wins are: (1) broadcast joins for small tables, (2) proper partitioning to avoid
skew, and (3) caching intermediate DataFrames that are used multiple times. At Centennial
Innovates, an unoptimized Databricks job ran for 45 minutes; after broadcast join + repartition
tuning it ran in 6 minutes.

**One-sentence interview answer:** "I optimize Spark jobs by broadcasting small lookup tables,
repartitioning on the join key to avoid skew, caching DataFrames reused in multiple actions,
and using predicate pushdown to minimize data scanned."

**Most likely follow-ups:**
1. How do you detect data skew and what do you do about it?
2. What storage level do you use for `persist()`?
3. What is Adaptive Query Execution (AQE)?

```scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._
import org.apache.spark.storage.StorageLevel

object OptimizedSparkJob {

  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("RBC-Optimized-Pipeline")
      // Adaptive Query Execution (Spark 3.0+): dynamically re-plans at runtime
      // - auto-coalesces small shuffle partitions
      // - switches to broadcast join when runtime size < broadcast threshold
      // - handles skew by splitting skewed partitions
      .config("spark.sql.adaptive.enabled", "true")
      .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
      .config("spark.sql.adaptive.skewJoin.enabled", "true")
      // Shuffle partitions: default 200 (too many for small data, too few for large)
      // Rule of thumb: target 100-200MB per partition
      // For 10GB data: 10GB / 128MB = ~80 partitions
      .config("spark.sql.shuffle.partitions", "80")
      .getOrCreate()

    import spark.implicits._

    // ── cache() vs persist() ───────────────────────────────────────────
    // cache()  = persist(StorageLevel.MEMORY_AND_DISK_DESER) — deserialized in memory
    //          = fastest access (no deserialize step), most memory usage
    //          = spills to disk when memory full
    //
    // persist(MEMORY_ONLY)             — fastest, OOM if doesn't fit
    // persist(MEMORY_AND_DISK)         — safe default: spills to disk, serialized
    // persist(MEMORY_AND_DISK_DESER)   — = cache(), deserialized = faster access
    // persist(DISK_ONLY)               — for data too large for memory
    // persist(OFF_HEAP)                — Tungsten off-heap, reduces GC pressure
    //
    // Use cache()/persist() when:
    //   - DataFrame is used in 2+ actions (avoids recomputation)
    //   - DataFrame is the result of an expensive join or aggregation
    //
    // Always unpersist() when done to free memory

    val portfolioSnapshots = spark.read
      .parquet("s3://rbc-gam/portfolio-snapshots/")
      .filter($"snapshot_date" >= "2025-01-01")
      // Predicate pushdown: Parquet files store min/max stats per row group.
      // Spark passes this filter to the Parquet reader which skips row groups
      // where all values < "2025-01-01" — reduces I/O significantly.

    // Cache because we use this DataFrame in multiple downstream operations
    val cachedSnapshots = portfolioSnapshots
      .persist(StorageLevel.MEMORY_AND_DISK)  // persist before the first action

    // First use
    println(s"Snapshots: ${cachedSnapshots.count()}")

    // ── Broadcast join ─────────────────────────────────────────────────
    // Standard join: both DataFrames shuffle (sort-merge join) → expensive
    // Broadcast join: small table sent to EVERY executor → NO shuffle
    //
    // Rule: broadcast when one side < spark.sql.autoBroadcastJoinThreshold (default 10MB)
    // Set threshold: spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "50m")
    //
    // When to broadcast explicitly:
    //   - You KNOW the table is small (e.g. lookup/reference table)
    //   - AQE hasn't picked it up automatically
    //   - Table is within 100-200MB

    // Small reference table: investor tier lookup (a few thousand rows)
    val investorTiers = spark.read
      .parquet("s3://rbc-gam/reference/investor-tiers/")
    // investorTiers is ~500KB — perfect for broadcast

    val enrichedPortfolios = cachedSnapshots.join(
      broadcast(investorTiers),  // broadcast hint: explicit
      Seq("investor_id"),
      "left"
    )

    // ── Data skew detection + remediation ─────────────────────────────
    // Skew: one key has disproportionately many rows → one executor handles
    // most of the shuffle → that executor becomes the bottleneck.
    //
    // Detect: portfolioSnapshots.groupBy("investor_id").count().orderBy(desc("count")).show(10)
    // If one investor has 10M rows and others have 10K → severe skew.
    //
    // Fix 1: Salt the key (spread skewed key across multiple buckets)
    val numSaltBuckets = 10

    // Add a random salt (0-9) to the key to distribute the skewed partition
    val saltedPortfolios = cachedSnapshots
      .withColumn("salt", (rand() * numSaltBuckets).cast("int"))
      .withColumn("salted_investor_id",
        concat($"investor_id", lit("_"), $"salt")
      )

    // Must also salt the right side of the join — explode the lookup for each salt value
    val explodedTiers = investorTiers
      .withColumn("salt", explode(array((0 until numSaltBuckets).map(lit(_)): _*)))
      .withColumn("salted_investor_id",
        concat($"investor_id", lit("_"), $"salt")
      )

    val skewHandledJoin = saltedPortfolios.join(
      broadcast(explodedTiers),  // now explodedTiers may still be broadcastable
      Seq("salted_investor_id"),
      "left"
    ).drop("salt", "salted_investor_id")

    // Fix 2 (Spark 3.0+ AQE): just enable spark.sql.adaptive.skewJoin.enabled=true
    // AQE detects at runtime which partitions are skewed and splits them automatically.
    // Always enable AQE in Spark 3+ — it handles most skew cases automatically.

    // ── Repartition strategy ───────────────────────────────────────────
    // repartition(n):       full shuffle, produces exactly n partitions (evenly distributed)
    // repartition(n, col):  shuffle by column — all rows with same col value in same partition
    //                       great before a groupBy on that column → avoids double-shuffle
    // coalesce(n):          reduces partition count WITHOUT shuffle (merges local partitions)
    //                       use when reducing partition count to avoid unnecessary shuffle

    val optimizedForGroupBy = enrichedPortfolios
      .repartition(80, $"portfolio_type")  // pre-partition by groupBy key
      .groupBy("portfolio_type", "region")
      .agg(
        count("*").as("portfolio_count"),
        sum("total_value").as("total_aum")
      )

    optimizedForGroupBy
      .coalesce(4)   // reduce output files without shuffle — 4 output Parquet files
      .write
      .mode(SaveMode.Overwrite)
      .parquet("s3://rbc-gam/analytics/portfolio-summary/")

    // ── Always unpersist to free executor memory ───────────────────────
    cachedSnapshots.unpersist()

    spark.stop()
  }
}
```

---

## 3.6 Complete Scala Spark ETL — Ontario Government Case Data Pipeline

This is the exact pipeline from the Ontario Government work: read raw case notes from S3,
define typed schema with case classes, pattern match on case type, normalize, write to
PostgreSQL. Scala's pattern matching is more expressive than SQL CASE WHEN for complex
multi-field classification logic.

**One-sentence interview answer:** "Scala's case classes give Spark Datasets compile-time type
safety; pattern matching cleanly handles multi-condition business logic that would be unwieldy
in SQL CASE WHEN statements."

**Most likely follow-ups:**
1. What is the difference between a Scala `case class` and a regular class in Spark context?
2. How does pattern matching compile to bytecode?
3. How would you handle a malformed record in this pipeline?

```scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._
import java.util.Properties

// ── Case Data Pipeline — Ontario Government ────────────────────────────────
object OntarioCaseDataPipeline {

  // Case classes define schema AND provide typed access to fields.
  // Spark Encoder is automatically derived for case classes — no manual schema definition.
  case class RawCaseNote(
    caseId:      String,
    caseType:    String,   // raw, inconsistent strings from legacy system
    workerName:  String,
    clientName:  String,
    noteContent: String,
    statusCode:  String,
    createdDate: String,   // raw string; will parse to date
    region:      String,
    urgencyFlag: String    // "Y", "N", "1", "0", "true", "false" — inconsistent source
  )

  // Normalized output schema — clean, typed
  case class NormalizedCaseRecord(
    caseId:          String,
    classifiedType:  String,   // "ADOPTION", "CHILDCARE", "CRISIS", "GENERAL", "UNKNOWN"
    subType:         String,
    workerName:      String,
    clientName:      String,
    isUrgent:        Boolean,
    processingTier:  Int,      // 1=immediate, 2=same-day, 3=standard
    parsedDate:      java.sql.Date,
    region:          String,
    wordCount:       Int
  )

  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("Ontario-CaseData-ETL")
      .config("spark.sql.adaptive.enabled", "true")
      .getOrCreate()

    import spark.implicits._

    // ── Read raw JSON from S3 ─────────────────────────────────────────
    // S3 path uses Hive-style partitioning: Spark only reads relevant partitions
    val rawDf = spark.read
      .option("multiLine", "false")
      .option("mode", "PERMISSIVE")    // bad records go to _corrupt_record column
      .option("columnNameOfCorruptRecord", "_corrupt")
      .json("s3://ontario-gov-cases/raw/year=2025/month=*/")

    // Log and quarantine malformed records before processing
    val corruptCount = rawDf.filter($"_corrupt".isNotNull).count()
    if (corruptCount > 0) {
      println(s"WARNING: $corruptCount corrupt records found — writing to quarantine")
      rawDf.filter($"_corrupt".isNotNull)
        .write.mode(SaveMode.Append)
        .json("s3://ontario-gov-cases/quarantine/")
    }

    val cleanDf = rawDf
      .filter($"_corrupt".isNull)
      .drop("_corrupt")

    // Convert to typed Dataset
    val rawCases: Dataset[RawCaseNote] = cleanDf.as[RawCaseNote]

    // ── Pattern matching for multi-field classification ────────────────
    // Pattern matching is cleaner than nested if-else or SQL CASE WHEN
    // for complex business rules with multiple fields.
    val normalized: Dataset[NormalizedCaseRecord] = rawCases.map { raw =>

      // Normalize urgency flag (source system is inconsistent)
      val isUrgent = raw.urgencyFlag match {
        case "Y" | "1" | "true" | "TRUE" | "YES" => true
        case _                                    => false
      }

      // Classify case type using pattern matching with guards
      // (lowercase for case-insensitive matching)
      val (classifiedType, subType) = raw.caseType.toLowerCase.trim match {
        case t if t.contains("adopt")                         => ("ADOPTION", "STANDARD")
        case t if t.contains("adopt") && t.contains("inter")  => ("ADOPTION", "INTERNATIONAL")
        case t if t.contains("child") && t.contains("care")   => ("CHILDCARE", "STANDARD")
        case t if t.contains("foster")                        => ("CHILDCARE", "FOSTER")
        case t if t.contains("crisis") || t.contains("emerg") => ("CRISIS", "IMMEDIATE")
        case t if t.contains("abuse") || t.contains("neglect")=> ("CRISIS", "ABUSE_NEGLECT")
        case t if t.contains("support")                       => ("GENERAL", "SUPPORT")
        case ""                                               => ("UNKNOWN", "UNCLASSIFIED")
        case _                                                => ("GENERAL", "OTHER")
      }

      // Determine processing tier based on type + urgency
      val processingTier = (classifiedType, isUrgent) match {
        case ("CRISIS", _)          => 1   // always immediate
        case (_, true)              => 1   // urgent regardless of type
        case ("ADOPTION", false)    => 2   // same-day
        case ("CHILDCARE", false)   => 2
        case _                      => 3   // standard
      }

      val parsedDate = try {
        java.sql.Date.valueOf(raw.createdDate.take(10))  // take first 10 chars: yyyy-MM-dd
      } catch {
        case _: Exception => java.sql.Date.valueOf("1970-01-01")  // sentinel for bad dates
      }

      NormalizedCaseRecord(
        caseId         = raw.caseId,
        classifiedType = classifiedType,
        subType        = subType,
        workerName     = raw.workerName.trim,
        clientName     = raw.clientName.trim,
        isUrgent       = isUrgent,
        processingTier = processingTier,
        parsedDate     = parsedDate,
        region         = raw.region.trim.toUpperCase,
        wordCount      = raw.noteContent.split("\\s+").length
      )
    }

    // ── Cache before multi-use ────────────────────────────────────────
    normalized.persist(StorageLevel.MEMORY_AND_DISK)

    // ── Write to PostgreSQL via JDBC ──────────────────────────────────
    val jdbcUrl = sys.env("ONTARIO_DB_URL")  // jdbc:postgresql://rds-host/ontario_cases
    val jdbcProps = new Properties()
    jdbcProps.put("user",       sys.env("DB_USER"))
    jdbcProps.put("password",   sys.env("DB_PASSWORD"))
    jdbcProps.put("driver",     "org.postgresql.Driver")
    jdbcProps.put("batchsize",  "5000")

    // Repartition by processing_tier before write — keeps related records together
    // and controls number of output JDBC connections
    normalized.toDF()
      .repartition(20, $"classifiedType", $"region")
      .write
      .mode(SaveMode.Append)
      .jdbc(jdbcUrl, "case_records_normalized", jdbcProps)

    // ── Summary stats (second use — benefits from persist) ────────────
    val summary = normalized.groupBy("classifiedType", "subType", "processingTier")
      .agg(
        count("*").as("record_count"),
        sum(when($"isUrgent", 1).otherwise(0)).as("urgent_count"),
        avg("wordCount").as("avg_word_count")
      )
    summary.show()

    normalized.unpersist()
    spark.stop()
  }
}
```

---

## 3.7 Spark Structured Streaming

Structured Streaming treats a live data stream as an unbounded table — you write the same
DataFrame/SQL API you'd use for batch, and Spark handles micro-batch execution automatically.
At RBC, replacing the nightly batch portfolio reconciliation with Structured Streaming from
Kafka reduced data latency from hours to seconds.

**One-sentence interview answer:** "Structured Streaming models a stream as an unbounded table
— micro-batches are appended as new rows; watermarks handle late-arriving data; the same SQL/
DataFrame API works for both batch and streaming."

**Most likely follow-ups:**
1. What is the difference between `append`, `update`, and `complete` output modes?
2. How does watermarking work for late data?
3. What are the guarantees of Structured Streaming (exactly-once)?

```scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming._
import org.apache.spark.sql.types._

// ── Real-Time Authentication Event Stream Processor ───────────────────────
// Reads from Kafka (auth events), classifies REST vs gRPC vs Interac,
// writes aggregates to PostgreSQL. Ties to RBC real-time portfolio analytics
// that replaced the nightly batch job.
object RealTimeAuthStreamProcessor {

  val authEventSchema: StructType = StructType(Array(
    StructField("event_id",       StringType,    nullable = false),
    StructField("investor_id",    StringType,    nullable = false),
    StructField("auth_type",      StringType,    nullable = false),  // REST/GRPC/INTERAC
    StructField("success",        BooleanType,   nullable = false),
    StructField("response_ms",    IntegerType,   nullable = true),
    StructField("event_timestamp",StringType,    nullable = false)
  ))

  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("RBC-Auth-Stream-Processor")
      .config("spark.sql.streaming.checkpointLocation", "s3://rbc-checkpoints/auth-stream/")
      .getOrCreate()

    import spark.implicits._

    // ── Read from Kafka ───────────────────────────────────────────────
    val kafkaStream = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", sys.env("KAFKA_BROKERS"))
      .option("subscribe", "investor.auth.events")
      .option("startingOffsets", "latest")
      // For production: set maxOffsetsPerTrigger to control micro-batch size
      .option("maxOffsetsPerTrigger", "10000")
      .load()

    // Kafka delivers: key (bytes), value (bytes), topic, partition, offset, timestamp
    val parsedEvents = kafkaStream
      .select(
        from_json(
          $"value".cast("string"),   // decode bytes to JSON string
          authEventSchema            // parse JSON to columns
        ).as("event"),
        $"timestamp".as("kafka_timestamp")  // Kafka message timestamp
      )
      .select(
        $"event.*",                  // flatten the struct
        $"kafka_timestamp",
        to_timestamp($"event.event_timestamp", "yyyy-MM-dd'T'HH:mm:ss")
          .as("event_time")
      )

    // ── Watermarking for late data ─────────────────────────────────────
    // Watermark: Spark tracks the max event_time seen.
    // "2 minutes" means: accept late records up to 2 min behind the watermark.
    // Records older than (max_event_time - 2 min) are dropped.
    // Required for stateful aggregations (window + groupBy) to bound state size.
    val withWatermark = parsedEvents
      .withWatermark("event_time", "2 minutes")

    // ── Windowed aggregation ──────────────────────────────────────────
    // window(timeColumn, windowDuration, slideDuration)
    // 1-minute tumbling window: [09:00-09:01), [09:01-09:02), ...
    // 1-minute sliding window (30s slide): [09:00-09:01), [09:00:30-09:01:30), ...
    val windowedMetrics = withWatermark
      .groupBy(
        window($"event_time", "1 minute"),    // tumbling 1-minute window
        $"auth_type"
      )
      .agg(
        count("*").as("total_events"),
        sum(when($"success", 1).otherwise(0)).as("successful_auths"),
        avg("response_ms").as("avg_response_ms"),
        percentile_approx($"response_ms", 0.95).as("p95_response_ms")
      )
      .select(
        $"window.start".as("window_start"),
        $"window.end".as("window_end"),
        $"auth_type",
        $"total_events",
        $"successful_auths",
        ($"successful_auths" / $"total_events" * 100).as("success_rate_pct"),
        $"avg_response_ms",
        $"p95_response_ms"
      )

    // ── Output modes ──────────────────────────────────────────────────
    // append:   only new rows since last trigger — works only for non-aggregated streams
    //           or aggregations with watermark where state is finalized
    // update:   rows that changed since last trigger — good for real-time dashboards
    // complete: ALL rows every trigger — only for aggregations (full state each time)
    //           dangerous for large state — use for small-cardinality aggregations

    // Write to PostgreSQL — foreachBatch lets you use any DataFrame write operation
    val query: StreamingQuery = windowedMetrics
      .writeStream
      .outputMode("update")           // emit updated windows each micro-batch
      .trigger(Trigger.ProcessingTime("30 seconds"))  // trigger every 30 seconds
      .foreachBatch { (batchDF: DataFrame, batchId: Long) =>
        // foreachBatch: treat each micro-batch as a static DataFrame
        // This gives full batch API access (JDBC, Delta Lake, etc.)
        println(s"Processing batch $batchId with ${batchDF.count()} records")

        // Upsert to PostgreSQL using temp view + SQL
        batchDF.createOrReplaceTempView(s"auth_metrics_batch_$batchId")

        // Write using JDBC with upsert semantics (ON CONFLICT DO UPDATE)
        batchDF
          .write
          .mode(SaveMode.Append)
          .option("driver", "org.postgresql.Driver")
          .jdbc(
            sys.env("DB_JDBC_URL"),
            "auth_realtime_metrics",
            {
              val p = new java.util.Properties()
              p.put("user",     sys.env("DB_USER"))
              p.put("password", sys.env("DB_PASSWORD"))
              p
            }
          )
      }
      .start()

    // Keep the stream running until termination signal
    query.awaitTermination()
  }
}
```

---
