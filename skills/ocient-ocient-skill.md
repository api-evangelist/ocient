---
name: Ocient
description: Use when building data pipelines, querying petabyte-scale datasets, managing database administration, configuring system infrastructure, or integrating with third-party analytics tools. Agents should reach for this skill when working with data ingestion, SQL queries, performance tuning, user management, or system monitoring.
metadata:
    mintlify-proj: ocient
    version: "1.0"
---

# Ocient Skill Reference

## Product Summary

Ocient is a petabyte-scale OLAP data platform that unifies data ingest, query optimization, security, and governance. It uses Compute-Adjacent Storage Architecture (CASA) to colocate NVMe storage with compute, enabling ultra-fast queries on massive datasets without separate storage layers. Agents work with Ocient through SQL interfaces (JDBC, pyocient, HTTP Query API), data pipelines for ETL, and system administration tools. Key files: connection strings use `jdbc:ocient://<host>:<port>/<database>` format; pipelines are defined with `CREATE PIPELINE` DDL; queries execute via standard SQL. Primary docs: https://docs.ocient.com

## When to Use

Reach for this skill when:
- **Loading data**: Building batch or continuous data pipelines from S3, Kafka, HDFS, or filesystem sources
- **Querying**: Writing SQL queries against large time-series datasets, using geospatial or machine learning functions
- **Performance tuning**: Designing tables with TimeKeys, Clustering Keys, or secondary indexes
- **Administration**: Managing users, groups, roles, workload management, or service classes
- **System setup**: Installing, configuring, monitoring, or maintaining Ocient infrastructure
- **Integration**: Connecting Ocient to Looker, Superset, Tableau, DBeaver, Metabase, or SQLAlchemy
- **Troubleshooting**: Debugging pipeline errors, query performance issues, or system configuration problems

## Quick Reference

### Connection Methods

| Method | Use Case | Example |
|--------|----------|---------|
| JDBC CLI | Interactive queries, testing | `java -classpath ocient-jdbc4-jar-with-dependencies.jar com.ocient.cli.CLI` |
| pyocient | Python applications | `import pyocient; conn = pyocient.connect(...)` |
| HTTP Query API | REST-based queries, integrations | `POST https://sql-node/v1/execute` with JSON payload |
| Spark Connector | Apache Spark jobs | DataSourceV2 implementation for read/write |

### Data Pipeline Modes

| Mode | Source | Use Case |
|------|--------|----------|
| BATCH | S3, Filesystem, HDFS | One-time or periodic loads; static file list |
| CONTINUOUS | S3, Filesystem, HDFS | Monitor for new files; dynamic file discovery |
| TRANSACTIONAL | S3, Filesystem, HDFS | All-or-nothing loads; rollback on error |
| KAFKA | Kafka only | Real-time streaming ingestion |

### Key SQL Statements

| Statement | Purpose |
|-----------|---------|
| `CREATE PIPELINE` | Define data source, format, transformations, and target table |
| `START PIPELINE` | Begin pipeline execution with optional error handling |
| `STOP PIPELINE` | Halt pipeline; can resume without duplication |
| `PREVIEW PIPELINE` | Test pipeline logic before creating it |
| `CREATE TABLE` | Define table with TimeKey, Clustering Key, indexes |
| `CREATE MLMODEL` | Train machine learning models in SQL |
| `ALTER CONNECTIVITY_POOL` | Enable OpenAPI port for HTTP Query API |

### System Nodes

| Node Type | Role |
|-----------|------|
| SQL Nodes | Parse SQL, administer system, serve as query interface (port 4050) |
| Loader Nodes | Manage ETL ingestion, index data, run pipelines |
| Foundation Nodes | Store data, perform query processing |

### Data Sources for Pipelines

- **S3**: Bucket, prefix, filter (glob/regex), credentials, endpoint
- **Kafka**: Bootstrap servers, topic, consumer config, offset reset
- **HDFS**: Endpoint (namenode), config, filter
- **Filesystem**: Local paths (NFS mount required on all Loader Nodes)

### Extract Formats

- `delimited` / `csv`: Field/record delimiters, headers, quoting, array/tuple markers
- `json`: Selectors like `$field.subfield`, `$array[]`
- `parquet`: Schema inference from sample file
- `avro`: Inline schema, schema registry, or infer from files
- `binary`: Fixed record length, endianness, padding
- `xml`: No format-specific options
- `asn.1`: Schema URL, record type

## Decision Guidance

### When to Use BATCH vs CONTINUOUS vs TRANSACTIONAL

| Scenario | Mode | Reason |
|----------|------|--------|
| Load historical data once | BATCH | Static file list, no monitoring overhead |
| Monitor S3 bucket for new files | CONTINUOUS | Dynamic discovery via SQS/Kafka monitor |
| Ensure all-or-nothing semantics | TRANSACTIONAL | Rollback on error; all data visible after success |
| Real-time streaming | KAFKA (CONTINUOUS) | Kafka is only source; always continuous |

### When to Use TimeKey vs Clustering Key vs Secondary Index

| Index Type | Use Case | Storage Cost | Query Benefit |
|------------|----------|--------------|---------------|
| TimeKey | Filter by timestamp column; partition data by time | None (no extra storage) | Skip irrelevant segments; essential for time-series |
| Clustering Key | Frequently queried column combinations | None (no extra storage) | Subdivide segments on disk; faster reference |
| Secondary Index | Filter on numeric, string, partial string, or geospatial columns | Extra storage required | Dramatically reduce query time on large datasets |

### When to Use S3 vs Kafka vs Filesystem

| Source | Batch | Continuous | Streaming | Credentials | Best For |
|--------|-------|-----------|-----------|-------------|----------|
| S3 | ✓ | ✓ | ✗ | AWS keys or IAM role | Cloud-native, large files, cost-effective |
| Kafka | ✗ | ✓ | ✓ | Consumer config | Real-time, high-volume, event-driven |
| HDFS | ✓ | ✓ | ✗ | Hadoop config | On-premises, Hadoop ecosystem |
| Filesystem | ✓ | ✓ | ✗ | NFS mount | Local testing, shared storage |

## Workflow

### Load Data from S3

1. **Create target table**: Define schema with TimeKey on timestamp column, Clustering Key on frequently filtered columns, secondary indexes on filter columns.
   ```sql
   CREATE TABLE orders (
       id INT, user_id INT, created_at TIMESTAMP TIME KEY BUCKET(1, DAY),
       amount DECIMAL(10,2), status VARCHAR(20)
   );
   CREATE INDEX idx_status ON orders(status);
   ```

2. **Create pipeline**: Specify S3 source, CSV/JSON format, field mappings, transformations.
   ```sql
   CREATE BATCH PIPELINE orders_load
       SOURCE S3
           ENDPOINT 'https://s3.us-east-1.amazonaws.com'
           BUCKET 'my-bucket'
           FILTER '*.csv'
       EXTRACT FORMAT csv NUM_HEADER_LINES 1
       INTO orders
       SELECT $1 AS id, $2 AS user_id, $3 AS created_at, $4 AS amount, $5 AS status;
   ```

3. **Preview (optional)**: Test with sample data before full load.
   ```sql
   PREVIEW PIPELINE orders_load
       SOURCE INLINE 'id,user_id,created_at,amount,status|1,100,2024-01-01,99.99,pending'
       EXTRACT FORMAT csv NUM_HEADER_LINES 0
       INTO orders SELECT $1 AS id, $2 AS user_id, ...;
   ```

4. **Start pipeline**: Execute with error handling.
   ```sql
   START PIPELINE orders_load ERROR LIMIT 10;
   ```

5. **Monitor**: Check status and errors.
   ```sql
   SHOW PIPELINE_STATUS;
   SELECT * FROM sys.pipeline_errors WHERE pipeline_name = 'orders_load';
   ```

6. **Validate**: Verify row counts, time ranges, NULL values.
   ```sql
   SELECT COUNT(*), MIN(created_at), MAX(created_at) FROM orders;
   ```

### Query Data

1. **Connect**: Use JDBC, pyocient, or HTTP API.
   ```sql
   connect to jdbc:ocient://10.10.1.1:4050/system;
   ```

2. **Write query**: Leverage TimeKey and indexes for performance.
   ```sql
   SELECT user_id, SUM(amount) AS total
   FROM orders
   WHERE created_at >= TIMESTAMP '2024-01-01' AND status = 'completed'
   GROUP BY user_id
   ORDER BY total DESC;
   ```

3. **Analyze results**: Use aggregate, window, geospatial, or ML functions.

### Manage Users and Permissions

1. **Create user**: Define username, password, database.
   ```sql
   CREATE USER 'analyst@mydb' IDENTIFIED BY 'password';
   ```

2. **Create role**: Group permissions.
   ```sql
   CREATE ROLE analyst_role;
   GRANT SELECT ON mydb.* TO analyst_role;
   ```

3. **Assign role**: Link user to role.
   ```sql
   GRANT analyst_role TO 'analyst@mydb';
   ```

4. **Verify permissions**: Check system catalog.
   ```sql
   SELECT * FROM information_schema.role_table_grants WHERE grantee = 'analyst_role';
   ```

## Common Gotchas

- **Pipeline deduplication**: Pipelines track position and deduplicate on resume. Truncating target table resets position; dropping and recreating table maintains position.
- **NULL handling in pipelines**: Empty fields default to NULL if `EMPTY_FIELD_AS_NULL = true`. Use `COALESCE(<value>, DEFAULT)` to load default values instead of NULL.
- **Kafka compression**: Ocient auto-detects Kafka compression; do not specify `COMPRESSION_METHOD` for Kafka sources.
- **S3 credentials hierarchy**: Pipelines check (1) explicit credentials, (2) AWS SDK default chain, (3) anonymous access. Higher levels take precedence.
- **Continuous pipelines**: Cannot use `PREFIX`, `SORT_BY`, `START_FILENAME`, or timestamp filters. Use `MONITOR` (SQS/Kafka) for file discovery.
- **TimeKey bucket size**: Choose bucket granularity (DAY, HOUR, MINUTE) based on query patterns. Smaller buckets = more segments = slower queries on large time ranges.
- **Secondary indexes on high-cardinality columns**: Indexes on columns with millions of distinct values may not improve performance; test before deploying.
- **Loader Node availability**: Pipelines require active Loader Nodes with `streamloader` role. If all Loaders are down, pipelines fail.
- **HTTP Query API port**: Must explicitly enable OpenAPI port with `ALTER CONNECTIVITY_POOL` before using REST API.
- **Geospatial data format**: Ensure coordinates are in correct order (longitude, latitude for POINT) and use correct constructors (POINT, LINESTRING, POLYGON).
- **Machine learning models**: Models are SQL objects; `CREATE MLMODEL` trains in-database. Use `PREDICT` function to score new data.
- **Transactional pipelines**: All data invisible until load completes; if load fails, all changes roll back. Use for critical loads requiring atomicity.

## Verification Checklist

Before submitting work with Ocient:

- [ ] **Data pipeline**: Ran `PREVIEW PIPELINE` to validate transformations; checked `SHOW PIPELINE_STATUS` for completion; verified row counts match source.
- [ ] **Table design**: TimeKey defined on timestamp column; Clustering Key on frequently filtered columns; secondary indexes on filter columns.
- [ ] **Query performance**: Checked execution plan; used TimeKey filters to skip segments; verified indexes are used.
- [ ] **Data validation**: Checked for unexpected NULL values; verified time ranges; sampled records for type conversions.
- [ ] **User permissions**: Verified user has required privileges (SELECT, INSERT, CREATE PIPELINE, etc.); tested with actual user account.
- [ ] **Error handling**: Set appropriate `ERROR LIMIT` for pipeline; checked `sys.pipeline_errors` for record-level issues.
- [ ] **System resources**: Verified Loader Nodes are active; checked available disk space; confirmed CPU/memory not saturated.
- [ ] **Integration**: Tested third-party tool connection (Looker, Tableau, etc.); verified time zone configuration if needed.
- [ ] **Backup/restore**: For critical data, tested backup and restore procedures; documented recovery steps.

## Resources

**Comprehensive navigation**: https://docs.ocient.com/llms.txt

**Critical documentation pages**:
1. [Load and Analyze Data](https://docs.ocient.com/load-and-analyze-data) — End-to-end tutorial for pipelines and queries
2. [Data Pipelines Reference](https://docs.ocient.com/data-pipelines) — Complete CREATE PIPELINE syntax and options
3. [SQL Reference](https://docs.ocient.com/sql-reference) — Functions, operators, DDL/DML/DQL statements

---

> For additional documentation and navigation, see: https://docs.ocient.com/llms.txt