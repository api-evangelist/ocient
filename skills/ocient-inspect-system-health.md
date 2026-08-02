---
name: Inspect Ocient System health and configuration
description: Read the version, status, statistics, and configuration of an Ocient System through the read-only System Information REST endpoints.
api: openapi/ocient-http-query-api-openapi-original.json
generated: '2026-08-02'
method: generated
source: openapi/ocient-http-query-api-openapi-original.json
operations:
  - getSystemInformationRestEndpointsGetVersion
  - getSystemInformationRestEndpointsGetStatus
  - getSystemInformationRestEndpointsGetStatistics
  - getSystemInformationRestEndpointsGetConfiguration
  - getSystemInformationRestEndpointsGetConfigurationOfWholeSystem
  - getSystemInformationRestEndpointsGetConfigurationParameters
---

# Inspect Ocient System health and configuration

The System Information REST endpoints are the **read-only** operational surface of an Ocient
System. Use them for monitoring, health checks, and confirming configuration across a cluster.
Every operation in this skill is a `GET` and none of them mutate state.

## Order of operations

Work outside-in: confirm the software, then the node, then the numbers, then the config.

### 1. Version

`getSystemInformationRestEndpointsGetVersion` (`GET /v1/version`) returns the version of the
software running. Check this first — the docs publish a Version Compatibility table pairing
each Ocient System release with supported JDBC driver and Java versions, and a mismatch
explains a large share of client failures.

### 2. Status

`getSystemInformationRestEndpointsGetStatus` (`GET /v1/status`) returns the status of the
running software. Use it as the liveness check for a node.

### 3. Statistics

`getSystemInformationRestEndpointsGetStatistics` (`GET /v1/stats`) returns statistics on each
node in the database. It accepts an optional JSON body with a `filter` property — use it to
narrow the response rather than pulling every metric from every node on a polling loop.

### 4. Configuration

Three operations, increasing in scope:

- `getSystemInformationRestEndpointsGetConfiguration` (`GET /v1/sysconfig`) — the
  configuration of **this node**, in JSON.
- `getSystemInformationRestEndpointsGetConfigurationOfWholeSystem` (`GET /v1/dbconfig`) —
  the configuration of the **system as a whole**, in JSON.
- `getSystemInformationRestEndpointsGetConfigurationParameters` (`GET /v1/config`) — the
  values of named configuration parameters.

Diff `/v1/sysconfig` across nodes against `/v1/dbconfig` to find nodes that have drifted from
the intended system configuration.

## Complementary in-database signals

Several things an agent may want are **not** on this REST surface and must be read with SQL
via the HTTP Query API, JDBC, or pyocient:

- `sys.queries` and `sys.completed_queries` — running and finished queries, with
  `transaction_id`.
- `sys.pipelines`, `sys.pipeline_errors_historical`, `sys.pipeline_events_historical`,
  `sys.pipeline_files_historical`, `sys.pipeline_metrics_historical` — data pipeline state.
- `sys.sql_messages` — the full error and warning code registry.
- `information_schema.pipeline_status_historical`,
  `information_schema.pipelines_historical`,
  `information_schema.transactional_pipeline_status`.

## Cautions

- Configuration responses can include host, storage, and network topology detail. Treat the
  output as sensitive; do not paste it into an untrusted context.
- These endpoints live on Ocient SQL Nodes inside a customer deployment, not on a public
  Ocient host. Base URL is `https://{sql_node}`.
