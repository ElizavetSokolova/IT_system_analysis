# CASE DESCRIPTION

Downtime in modern IT systems is extremely costly. To avoid it, monitoring systems must help people identify anomalies in system behavior based on collected telemetry data and promptly notify operators and the technical team.

I analyzed time series data to identify relationships between various system metrics at the moment anomalies occur.

By "relationships," I mean clear correlations in metric behavior at specific times within the system's operation. For example, as the number of requests processed by the system increases, the time required to process them also rises. Solving such problems allows monitoring systems to highlight only critical data during incidents and simplifies root cause analysis of faults.

# Input Data 

A snapshot of telemetry data from a real system covering a month of operation, recorded at one-minute intervals for analysis and solution development.

## Data explonation

The data is an export from Clickhouse in TSV format.

### Column Set for metrics_collector:

account_id, name, point, call_count, total_call_time, total_exclusive_time, min_call_time, max_call_time, sum_of_squares, instances, language, app_name, app_id, scope, host, display_host, pid, agent_version, labels

![TS](https://github.com/user-attachments/assets/227f5e90-0b84-4296-acef-9ac6ca3fa6bb)

## Key Metrics for Initial Analysis

This document outlines the SQL queries required to extract key metrics from the `metrics_collector` database for monitoring service performance.

### Metrics

#### 1. Web Response
**Description:** Service response time for external HTTP requests.

**Query:**
```sql
SELECT
    point AS time,
    SUM(NULLIF(total_call_time, 0)) / SUM(NULLIF(call_count, 0)) AS response_time
FROM
    metrics_collector
WHERE
    language = 'java'
    AND app_name = '[GMonit] Collector'
    AND scope = ''
    AND name = 'HttpDispatcher'
GROUP BY
    time
ORDER BY
    time;
```
#### 2. Throughput
**Description:** Service throughput, measured in requests per minute.

**Query:**
```sql
SELECT
    point AS time,
    SUM(NULLIF(call_count, 0)) AS throughput
FROM
    metrics_collector
WHERE
    language = 'java'
    AND app_name = '[GMonit] Collector'
    AND scope = ''
    AND name = 'HttpDispatcher'
GROUP BY
    time
ORDER BY
    time;
```

#### 3. APDEX
**Description:** A synthetic health index for the service, ranging from 0 to 1. The closer to 1, the better.

**Query:**
```sql
WITH
    SUM(NULLIF(call_count, 0)) AS s,
    SUM(NULLIF(total_call_time, 0)) AS t,
    SUM(NULLIF(total_exclusive_time, 0)) AS f
SELECT
    point AS time,
    (s + t / 2) / (s + t + f) AS apdex
FROM
    metrics_collector
WHERE
    language = 'java'
    AND app_name = '[GMonit] Collector'
    AND scope = ''
    AND name = 'Apdex'
GROUP BY
    time
ORDER BY
    time;
```

#### 4. Error Rate
**Description:** Percentage of errors in processed requests.

**Query:**
```sql
SELECT
    point AS time,
    SUM(NULLIF(call_count, 0), name = 'Errors/allWeb') / SUM(NULLIF(call_count, 0), name = 'HttpDispatcher') AS error_rate
FROM
    metrics_collector
WHERE
    language = 'java'
    AND app_name = '[GMonit] Collector'
    AND scope = ''
    AND name IN ('HttpDispatcher', 'Errors/allWeb')
GROUP BY
    time
ORDER BY
    time;
```

---

Each metric includes a description and its corresponding SQL query. These metrics are essential for monitoring and analyzing the performance and health of the service.
