---
globs:
  - "scripts/transform/**"
---

# Flink Transform Rules

## Streamhouse Storage Pattern
- **Hot** (Fluss): Real-time validated data, <100ms latency, 1-2hr retention
- **Warm** (Paimon): Validated incidents + aggregations, minutes latency, 7-30 day retention
- **Cold** (Iceberg): Historical archive, Parquet, years retention

## Validated Data (Warm — Paimon)
- Input: Kafka topic `hot-violence-alerts-valid`
- Output: Paimon `violence_incidents`
- Merge engine: `deduplicate` on `incident_id`

## Aggregation Tables (Warm — Paimon)
- Input: Paimon `violence_incidents` (CDC changelog)
- Aggregations: daily counts, hourly trends, camera-level stats
- Output: Paimon `daily_incident_stats`, `camera_stats`
- Merge engine: `aggregation`

## Flink SQL Conventions
- Table names: snake_case, descriptive (no layer prefix)
- Watermark: `WATERMARK FOR timestamp AS timestamp - INTERVAL '5' SECOND`
- Processing time: chỉ dùng khi event time không available

## Important
- Exactly-once semantics: dùng Flink checkpointing
- State backend: RocksDB cho production, HashMapStateBackend cho dev
- Checkpoint interval: 30 seconds (Paimon requires checkpointing to commit)
