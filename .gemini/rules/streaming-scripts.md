---
globs:
  - "scripts/streaming/**"
---

# Streaming Scripts Rules

## Kafka Conventions
- Topic names: kebab-case (e.g., `urban-safety-alerts`, `urban-safety-raw`)
- Consumer group: `{service-name}-group`
- Serialization: JSON with UTF-8 encoding
- Key: `camera_id` (string)

## Error Handling
- Luôn catch `KafkaException` và log rõ ràng
- Retry logic: exponential backoff, max 3 retries
- Dead letter: gửi message lỗi sang topic `{original-topic}-dlq`

## Producer Pattern
```python
# Standard producer pattern
producer = KafkaProducer(
    bootstrap_servers=os.getenv('KAFKA_BOOTSTRAP_SERVERS', 'localhost:19092'),
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8') if k else None
)
```

## Message Schema
Mọi message gửi vào Kafka PHẢI tuân theo Data Contract:
- Xem chi tiết: `docs/agent-guides/data-contracts.md`

## Files
- `producerRTSP.py`: Thu thập RTSP stream, gửi metadata vào Kafka
- `simulateRTSP.py`: Giả lập multi-camera RTSP feeds
- `metadataRTSP.py`: Trích xuất và publish RTSP metadata
- `inference_mock.py`: Mock AI inference output cho testing
