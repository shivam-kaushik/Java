# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

This is a plain Java project with no build tool configured yet. Compile and run manually:

```bash
# Compile
javac Demo.java

# Run
java Demo
```

## Project Structure

Currently a single-file Java playground (`Demo.java`) at the repo root. No package structure, build system (Maven/Gradle), or test framework has been set up yet.

## Interview Prep

A complete RBC interview prep guide lives in `interview_prep/`.

| File | Contents |
|------|----------|
| `section1_java_core.md` | OOP, Collections, Concurrency, Java 8+, Design Patterns, Exceptions, JVM |
| `section2_spring_boot.md` | Auto-config, REST API, JWT Security, Spring Data + PostgreSQL |
| `section3_apache_spark.md` | Architecture, RDD/DF/DS, Transformations, Spark SQL, Optimization, Scala ETL, Streaming |
| `section4_aws.md` | S3/Glue/EMR/Lambda, Data Migration to AWS, Glue Deep Dive, EMR via boto3 |
| `section5_azure.md` | ADLS/ADF/Databricks, ADF Pipeline JSON, Delta Lake, MLflow |
| `section6_7_pipelines_cicd.md` | End-to-end AWS + Azure pipelines, Jenkins, GitHub Actions, Testing |
| `section8_interview_qa.md` | All Q&A with one-liners, full answers, follow-up questions |

### Quick Reference — One-Liners

**Java**
- **HashMap vs ConcurrentHashMap**: HashMap is unsafe; CHM uses bin-level CAS/sync for concurrent reads/writes without locking the whole map.
- **GC**: Mark reachable from GC roots → sweep unreachable; G1GC collects garbage-dense regions first; ZGC is concurrent + sub-1ms pauses.
- **== vs equals**: `==` is reference equality; `.equals()` is logical equality defined by the class.
- **Deadlock prevention**: Always acquire multiple locks in consistent global order (lower ID first).
- **Functional interface**: Exactly one abstract method — can be a lambda target.
- **Abstract class vs interface**: Abstract class = shared state + behavior; interface = pure contract, multiple inheritance.

**Spring Boot**
- **Auto-configuration**: Scans classpath for `@ConditionalOn*` beans; override by declaring your own bean.
- **@Transactional pitfalls**: Self-invocation, checked exceptions don't rollback, non-public methods ignored.
- **JWT**: Sign claims on login; filter validates signature + expiry on every request; set SecurityContext.
- **Connection pooling**: Reuse pre-established TCP connections; HikariCP maxPoolSize must match DB max_connections / num_instances.

**Spark**
- **Transformation vs action**: Transformations are lazy (DAG); actions trigger execution.
- **Lazy evaluation**: Allows Catalyst to optimize the full plan before running — e.g., push filters before joins.
- **cache() vs persist()**: cache() = MEMORY_AND_DISK; persist(MEMORY_ONLY) = fastest but OOM risk; always unpersist() when done.
- **Data skew**: Salt the key (append 0-N random suffix); explode the lookup table; or enable AQE skewJoin.
- **Broadcast join**: Send small table to every executor — no shuffle; use when one side < ~100MB.
- **Fault tolerance**: RDD lineage recomputes lost partitions; Streaming uses checkpoint files.

**Cloud**
- **AWS Glue vs EMR**: Glue = serverless short ETL; EMR = full control, long jobs, custom libs.
- **Delta Lake**: ACID transactions on ADLS/S3 — MERGE upserts, time travel, schema enforcement.
- **PostgreSQL → AWS migration**: DMS full-load + CDC replication → validate → cutover (< 15 min downtime).
- **Data security layers**: Encrypt at rest (KMS) + in transit (TLS) + IAM roles + Secrets Manager + audit logs.

## Notes

- `.class` files are not gitignored — consider adding a `.gitignore` if the project grows.
- If a build tool is added (Maven/Gradle), update this file with the relevant commands (`mvn compile`, `./gradlew build`, etc.).
