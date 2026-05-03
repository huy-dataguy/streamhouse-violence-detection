# Skill: Test Pipeline End-to-End

Kiểm tra pipeline từ đầu đến cuối: Kafka → Flink → Storage → Query.

## Steps

### 1. Verify Infrastructure
```bash
# Check all services running
docker compose -f docker/docker-compose.yml ps

# Check Kafka topics exist
docker exec kafka kafka-topics --list --bootstrap-server localhost:9092
# Expected: urban-safety-raw, urban-safety-alerts, urban-safety-quarantine
```

### 2. Test Data Ingestion (Mock Inference)
```bash
# Start mock inference to generate test data
docker compose -f docker/docker-compose.yml up -d inference-mock

# Verify messages in Kafka UI
# Open http://localhost:18085 → Topics → urban-safety-alerts
# Should see new messages arriving
```

### 3. Test Data Contract Validation
```bash
# Send valid record manually
docker exec kafka kafka-console-producer \
  --broker-list localhost:9092 \
  --topic urban-safety-raw \
  <<< '{"camera_id":"cam_01","risk_score":0.85,"timestamp":"2025-01-15T10:00:00Z","is_violent":true,"event_type":"FIGHTING","confidence":0.9}'

# Send invalid record (bad camera_id)
docker exec kafka kafka-console-producer \
  --broker-list localhost:9092 \
  --topic urban-safety-raw \
  <<< '{"camera_id":"INVALID","risk_score":2.5,"timestamp":"2099-01-01T00:00:00Z"}'

# Check: valid → urban-safety-alerts, invalid → urban-safety-quarantine
```

### 4. Test Trino Queries
```bash
docker exec -it trino-coordinator trino

# Test Iceberg
SELECT COUNT(*) FROM iceberg.security.historical_violence_incidents;

# Test Paimon (when available)
SELECT COUNT(*) FROM paimon.security.violence_incidents;
```

### 5. Test Agentic RAG
```bash
# Test chatbot endpoint
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Hôm nay có bao nhiêu vụ bạo lực?"}'
```

### 6. Verify Monitoring
- Prometheus targets: http://localhost:9090/targets (all UP)
- Grafana dashboards: http://localhost:3000 (data flowing)

## Expected Results
- Mock inference generates ~1 msg/second
- Valid messages appear in alerts topic
- Invalid messages routed to quarantine
- Trino queries return data
- RAG chatbot responds with SQL-backed answers
- Grafana shows live metrics
