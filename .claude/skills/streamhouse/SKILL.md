# Skill: Streamhouse — Fluss + Paimon + Iceberg with Flink

Hướng dẫn triển khai kiến trúc Streamhouse Trio cho hệ thống Violence Detection.

## Streamhouse Là Gì

Streamhouse = kiến trúc dữ liệu thế hệ 3, kết hợp streaming real-time với lakehouse storage.
- **Coined by**: Jing Ge (CTO Ververica, Flink PMC) tại Flink Forward Seattle 10/2023
- **Evolution**: Data Warehouse → Data Lakehouse → **Streamhouse**
- **Core idea**: Ghi 1 lần vào Fluss, tự động tiering xuống Paimon/Iceberg

```
Write once → Fluss (hot, <100ms)
               ↓ Tiering Service (Flink job)
             Paimon (warm, seconds-minutes)
               ↓ Archival (scheduled)
             Iceberg (cold, minutes+, years retention)
```

## Tại Sao Không Dùng Lakehouse Thuần (Iceberg)?

4 gaps mà Lakehouse thuần không giải quyết được:
1. **Metadata overhead**: Mỗi Iceberg commit rewrite metadata.json + manifests → bloat
2. **Polling-based reads**: Không có push notification, consumer phải poll → +5-15s latency
3. **No enforced PK**: Iceberg V2 chấp nhận PK trong DDL nhưng không enforce uniqueness
4. **Write amplification**: Merge-on-read + delete files → small-file problem

Fluss giải quyết cả 4 bằng hot streaming layer + automatic batched tiering (~1 commit/min).

---

## Flink Catalog Setup

### Fluss Catalog (Hot Layer)

```sql
CREATE CATALOG fluss_catalog WITH (
    'type' = 'fluss',
    'bootstrap.servers' = 'fluss-coordinator:9123'
);

-- Với Paimon làm lake backend
CREATE CATALOG fluss_with_paimon WITH (
    'type' = 'fluss',
    'bootstrap.servers' = 'fluss-coordinator:9123',
    'paimon.s3.endpoint' = 'http://minio:9000',
    'paimon.s3.access-key' = '${MINIO_ACCESS_KEY}',
    'paimon.s3.secret-key' = '${MINIO_SECRET_KEY}',
    'paimon.s3.path.style.access' = 'true'
);

-- Với Iceberg làm lake backend
CREATE CATALOG fluss_with_iceberg WITH (
    'type' = 'fluss',
    'bootstrap.servers' = 'fluss-coordinator:9123',
    'iceberg.catalog-impl' = 'org.apache.iceberg.jdbc.JdbcCatalog',
    'iceberg.uri' = 'jdbc:postgresql://postgres:5432/iceberg',
    'iceberg.warehouse' = 's3://warehouse/iceberg',
    'iceberg.s3.endpoint' = 'http://minio:9000',
    'iceberg.s3.access-key-id' = '${MINIO_ACCESS_KEY}',
    'iceberg.s3.secret-access-key' = '${MINIO_SECRET_KEY}'
);
```

### Paimon Catalog (Warm Layer)

```sql
-- Paimon với filesystem metastore
CREATE CATALOG paimon_catalog WITH (
    'type' = 'paimon',
    'warehouse' = 's3://warehouse/paimon'
);

-- Paimon với Hive metastore (cho Trino federation)
CREATE CATALOG paimon_hive WITH (
    'type' = 'paimon',
    'metastore' = 'hive',
    'uri' = 'thrift://hive-metastore:9083',
    'warehouse' = 's3://warehouse/paimon'
);
```

### Iceberg Catalog (Cold Layer)

```sql
-- Iceberg với Hive catalog
CREATE CATALOG iceberg_catalog WITH (
    'type' = 'iceberg',
    'catalog-type' = 'hive',
    'uri' = 'thrift://hive-metastore:9083',
    'warehouse' = 's3://warehouse/iceberg'
);
```

---

## Table Definitions — Violence Detection

### Fluss: Hot Violence Alerts (PK table — supports upserts & lookups)

```sql
USE CATALOG fluss_catalog;
CREATE DATABASE IF NOT EXISTS security;
USE security;

CREATE TABLE hot_violence_alerts (
    `alert_id`    BIGINT,
    `camera_id`   STRING,
    `timestamp`   TIMESTAMP(3),
    `risk_score`  DOUBLE,
    `confidence`  DOUBLE,
    `is_violent`  BOOLEAN,
    `event_type`  STRING,
    `ptime`       AS PROCTIME(),
    PRIMARY KEY (`alert_id`) NOT ENFORCED
) WITH (
    'table.datalake.enabled' = 'true',
    'table.datalake.freshness' = '30s'
);
```

Khi `table.datalake.enabled = 'true'`, Fluss **tự động tạo** Paimon/Iceberg table cùng schema. Không cần tạo thủ công.

### Fluss: Camera Metadata (Dimension table — lookup join)

```sql
CREATE TABLE camera_registry (
    `camera_id`   STRING,
    `location`    STRING,
    `ward_id`     STRING,
    `district`    STRING,
    `status`      STRING,
    `installed`   DATE,
    PRIMARY KEY (`camera_id`) NOT ENFORCED
);
```

### Paimon: Violence Incidents (Standalone — nếu không dùng Fluss tiering)

```sql
USE CATALOG paimon_catalog;
CREATE DATABASE IF NOT EXISTS security;
USE security;

CREATE TABLE violence_incidents (
    `incident_id` STRING,
    `camera_id`   STRING,
    `timestamp`   TIMESTAMP(3),
    `risk_score`  DOUBLE,
    `confidence`  DOUBLE,
    `is_violent`  BOOLEAN,
    `event_type`  STRING,
    `is_deleted`  BOOLEAN,
    PRIMARY KEY (`incident_id`) NOT ENFORCED
) WITH (
    'merge-engine' = 'deduplicate',
    'changelog-producer' = 'input',
    'snapshot.time-retained' = '7d',
    'bucket' = '4'
);
```

### Paimon: Daily Incident Stats (Aggregation)

```sql
CREATE TABLE daily_incident_stats (
    `incident_date` DATE,
    `ward_id`       STRING,
    `total_incidents` BIGINT,
    `avg_risk_score`  DOUBLE,
    `max_risk_score`  DOUBLE,
    `violent_count`   BIGINT,
    PRIMARY KEY (`incident_date`, `ward_id`) NOT ENFORCED
) WITH (
    'merge-engine' = 'aggregation',
    'fields.total_incidents.aggregate-function' = 'sum',
    'fields.avg_risk_score.aggregate-function' = 'last_non_null_value',
    'fields.max_risk_score.aggregate-function' = 'max',
    'fields.violent_count.aggregate-function' = 'sum'
);
```

### Iceberg: Historical Violence Incidents

```sql
USE CATALOG iceberg_catalog;
CREATE DATABASE IF NOT EXISTS security;
USE security;

CREATE TABLE historical_violence_incidents (
    `incident_id` STRING,
    `camera_id`   STRING,
    `timestamp`   TIMESTAMP(3),
    `risk_score`  DOUBLE,
    `confidence`  DOUBLE,
    `is_violent`  BOOLEAN,
    `event_type`  STRING,
    `ward_id`     STRING,
    `district`    STRING
) PARTITIONED BY (days(`timestamp`));
```

---

## Data Pipeline — Flink SQL Jobs

### Job 1: Kafka → Fluss (Ingest + Data Contract Validation)

```sql
-- Source: Kafka raw topic
CREATE TEMPORARY TABLE kafka_raw_events (
    `camera_id`   STRING,
    `timestamp`   TIMESTAMP(3),
    `risk_score`  DOUBLE,
    `confidence`  DOUBLE,
    `is_violent`  BOOLEAN,
    `event_type`  STRING,
    `raw_payload` STRING,
    WATERMARK FOR `timestamp` AS `timestamp` - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'urban-safety-raw',
    'properties.bootstrap.servers' = 'kafka:19092',
    'properties.group.id' = 'flink-ingest-group',
    'scan.startup.mode' = 'latest-offset',
    'format' = 'json'
);

-- Insert valid records into Fluss (auto-tiers to Paimon/Iceberg)
INSERT INTO fluss_catalog.security.hot_violence_alerts
SELECT
    CAST(UNIX_TIMESTAMP() * 1000 + ROW_NUMBER() OVER () AS BIGINT) as alert_id,
    camera_id,
    `timestamp`,
    risk_score,
    confidence,
    is_violent,
    event_type
FROM kafka_raw_events
WHERE
    camera_id RLIKE '^cam_\\d{2}$'          -- Data Contract: valid camera_id
    AND risk_score BETWEEN 0 AND 1           -- Data Contract: risk_score range
    AND confidence BETWEEN 0 AND 1           -- Data Contract: confidence range
    AND `timestamp` <= CURRENT_TIMESTAMP + INTERVAL '1' MINUTE  -- No future timestamps
    AND (is_violent = FALSE OR event_type IS NOT NULL);          -- event_type required when violent
```

### Job 2: Quarantine Invalid Records

```sql
-- Sink: Kafka quarantine topic
CREATE TEMPORARY TABLE kafka_quarantine (
    `camera_id`    STRING,
    `timestamp`    TIMESTAMP(3),
    `risk_score`   DOUBLE,
    `confidence`   DOUBLE,
    `is_violent`   BOOLEAN,
    `event_type`   STRING,
    `raw_payload`  STRING,
    `violation`    STRING
) WITH (
    'connector' = 'kafka',
    'topic' = 'urban-safety-quarantine',
    'properties.bootstrap.servers' = 'kafka:19092',
    'format' = 'json'
);

INSERT INTO kafka_quarantine
SELECT
    camera_id, `timestamp`, risk_score, confidence, is_violent, event_type, raw_payload,
    CASE
        WHEN NOT camera_id RLIKE '^cam_\\d{2}$' THEN 'INVALID_CAMERA_ID'
        WHEN risk_score < 0 OR risk_score > 1 THEN 'RISK_SCORE_OUT_OF_RANGE'
        WHEN confidence < 0 OR confidence > 1 THEN 'CONFIDENCE_OUT_OF_RANGE'
        WHEN `timestamp` > CURRENT_TIMESTAMP + INTERVAL '1' MINUTE THEN 'FUTURE_TIMESTAMP'
        WHEN is_violent = TRUE AND event_type IS NULL THEN 'MISSING_EVENT_TYPE'
        ELSE 'UNKNOWN_VIOLATION'
    END as violation
FROM kafka_raw_events
WHERE
    NOT camera_id RLIKE '^cam_\\d{2}$'
    OR risk_score < 0 OR risk_score > 1
    OR confidence < 0 OR confidence > 1
    OR `timestamp` > CURRENT_TIMESTAMP + INTERVAL '1' MINUTE
    OR (is_violent = TRUE AND event_type IS NULL);
```

### Job 3: Enrichment via Temporal Join (Fluss lookup)

```sql
-- Enrich alerts with camera metadata
INSERT INTO paimon_catalog.security.violence_incidents
SELECT
    CAST(a.alert_id AS STRING) as incident_id,
    a.camera_id,
    a.`timestamp`,
    a.risk_score,
    a.confidence,
    a.is_violent,
    a.event_type,
    FALSE as is_deleted
FROM fluss_catalog.security.hot_violence_alerts a
LEFT JOIN fluss_catalog.security.camera_registry
    FOR SYSTEM_TIME AS OF a.ptime AS c
ON a.camera_id = c.camera_id;
```

### Job 4: Aggregation (Paimon → Paimon)

```sql
INSERT INTO paimon_catalog.security.daily_incident_stats
SELECT
    CAST(`timestamp` AS DATE) as incident_date,
    'default' as ward_id,  -- enrich from camera_registry
    COUNT(*) as total_incidents,
    AVG(risk_score) as avg_risk_score,
    MAX(risk_score) as max_risk_score,
    COUNT(*) FILTER (WHERE is_violent = TRUE) as violent_count
FROM paimon_catalog.security.violence_incidents
WHERE is_deleted = FALSE
GROUP BY CAST(`timestamp` AS DATE);
```

---

## Tiering Service — Fluss → Lake

### Start Tiering Service (Fluss → Paimon)

```bash
${FLINK_HOME}/bin/flink run \
    ${FLUSS_HOME}/opt/fluss-flink-tiering-*.jar \
    --fluss.bootstrap.servers fluss-coordinator:9123 \
    --datalake.format paimon \
    --datalake.paimon.metastore filesystem \
    --datalake.paimon.warehouse s3://warehouse/paimon \
    --datalake.paimon.s3.endpoint http://minio:9000 \
    --datalake.paimon.s3.access.key ${MINIO_ACCESS_KEY} \
    --datalake.paimon.s3.secret.key ${MINIO_SECRET_KEY} \
    --datalake.paimon.s3.path.style.access true
```

### Start Tiering Service (Fluss → Iceberg)

```bash
${FLINK_HOME}/bin/flink run \
    ${FLUSS_HOME}/opt/fluss-flink-tiering-*.jar \
    --fluss.bootstrap.servers fluss-coordinator:9123 \
    --datalake.format iceberg \
    --datalake.iceberg.catalog-impl org.apache.iceberg.jdbc.JdbcCatalog \
    --datalake.iceberg.uri "jdbc:postgresql://postgres:5432/iceberg" \
    --datalake.iceberg.warehouse "s3://warehouse/iceberg" \
    --datalake.iceberg.io-impl org.apache.iceberg.aws.s3.S3FileIO \
    --datalake.iceberg.s3.endpoint "http://minio:9000" \
    --datalake.iceberg.s3.access-key-id ${MINIO_ACCESS_KEY} \
    --datalake.iceberg.s3.secret-access-key ${MINIO_SECRET_KEY} \
    --datalake.iceberg.s3.path-style-access true
```

---

## Query Patterns

### Union Read (Fluss hot + Lake cold — tự động merge)

```sql
-- Query trả về cả data real-time (Fluss) + historical (Paimon/Iceberg)
SELECT * FROM fluss_catalog.security.hot_violence_alerts
WHERE `timestamp` > NOW() - INTERVAL '1' HOUR;

-- Chỉ đọc lake data (bỏ qua Fluss hot)
SELECT * FROM fluss_catalog.security.hot_violence_alerts$lake
WHERE CAST(`timestamp` AS DATE) < CURRENT_DATE;

-- Xem snapshots
SELECT snapshot_id, total_record_count
FROM fluss_catalog.security.hot_violence_alerts$lake$snapshots;
```

### Trino Federation (Cross-layer queries)

```sql
-- Hot: real-time từ Fluss (qua Flink SQL Gateway hoặc Fluss Trino connector)
-- Warm: operational từ Paimon
SELECT * FROM paimon.security.violence_incidents
WHERE `timestamp` >= CURRENT_DATE;

-- Cold: historical từ Iceberg
SELECT DATE_TRUNC('MONTH', `timestamp`) as month, COUNT(*) as total
FROM iceberg.security.historical_violence_incidents
WHERE YEAR(`timestamp`) = 2025
GROUP BY DATE_TRUNC('MONTH', `timestamp`);

-- Time Travel (Iceberg)
SELECT * FROM iceberg.security.historical_violence_incidents
FOR TIMESTAMP AS OF TIMESTAMP '2025-01-14 14:00:00';

-- Time Travel (Paimon)
SELECT * FROM paimon.security.violence_incidents /*+ OPTIONS('scan.snapshot-id' = '42') */;
```

---

## Docker Services — Streamhouse Stack

```yaml
# Thêm vào docker-compose.yml
services:
  fluss-coordinator:
    image: fluss/fluss:latest
    command: coordinatorServer
    ports:
      - "9123:9123"
    environment:
      - FLUSS_PROPERTIES=
          coordinator.host=fluss-coordinator
          remote.data.dir=s3://warehouse/fluss
          lakehouse.storage=paimon
          datalake.paimon.metastore=filesystem
          datalake.paimon.warehouse=s3://warehouse/paimon
    depends_on:
      minio:
        condition: service_healthy

  fluss-tablet-server:
    image: fluss/fluss:latest
    command: tabletServer
    depends_on:
      - fluss-coordinator
    environment:
      - FLUSS_PROPERTIES=
          coordinator.host=fluss-coordinator
          tablet-server.host=fluss-tablet-server
          tablet-server.port=9124
          data.dir=/tmp/fluss/data
          remote.data.dir=s3://warehouse/fluss
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 2G
          cpus: '1.0'

  flink-jobmanager:
    image: flink:1.20
    command: jobmanager
    ports:
      - "8081:8081"
    environment:
      - JOB_MANAGER_RPC_ADDRESS=flink-jobmanager
    deploy:
      resources:
        limits:
          memory: 2G

  flink-taskmanager:
    image: flink:1.20
    command: taskmanager
    depends_on:
      - flink-jobmanager
    environment:
      - JOB_MANAGER_RPC_ADDRESS=flink-jobmanager
      - TASK_MANAGER_NUMBER_OF_TASK_SLOTS=4
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 4G
          cpus: '2.0'
```

---

## Mapping Với Project Violence Detection

| Fraud Detection (Reference) | Violence Detection (Ours) |
|------------------------------|---------------------------|
| Transaction stream | Camera inference results (Kafka) |
| Account lookup table | Camera metadata (Fluss PK table) |
| Fraud detection logic | VioMobileNet + Data Contract validation |
| Enriched fraud alerts | Enriched violence alerts |
| Tiering to Iceberg | Tiering to Paimon + Iceberg |

## Verification Steps

1. **Fluss running**: `curl http://fluss-coordinator:9123/status`
2. **Flink UI**: http://localhost:8081 — check running jobs
3. **Tiering active**: Flink UI shows tiering job with records processed
4. **Union read works**: Query Fluss table, verify data from both hot + lake
5. **Trino federation**: Query Paimon + Iceberg tables via Trino
6. **Latency test**: Measure time from Kafka produce → Fluss queryable (<100ms)

## References
- [Apache Fluss Docs](https://fluss.apache.org/docs/)
- [Fluss Quickstart: Building a Streaming Lakehouse](https://fluss.apache.org/docs/quickstart/lakehouse/)
- [Fluss x Paimon Integration](https://fluss.apache.org/docs/streaming-lakehouse/integrate-data-lakes/paimon/)
- [Fluss x Iceberg Integration](https://fluss.apache.org/docs/streaming-lakehouse/integrate-data-lakes/iceberg/)
- [What is Streamhouse — Ververica](https://www.ververica.com/what-is-streamhouse)
- [Apache Paimon Flink SQL DDL](https://paimon.apache.org/docs/master/flink/sql-ddl/)
- [Apache Iceberg Flink Getting Started](https://iceberg.apache.org/docs/latest/flink/)
