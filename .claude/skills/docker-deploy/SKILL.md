# Skill: Docker Deploy

Deploy toàn bộ hoặc một phần Docker stack.

## Steps

1. **Check prerequisites**:
   - Verify `docker/.env` exists (copy from `.env.example` if missing)
   - Verify Docker daemon is running

2. **Build images** (nếu có thay đổi Dockerfile):
   ```bash
   docker compose -f docker/docker-compose.yml build --no-cache <service>
   ```

3. **Start services** (theo thứ tự dependency):
   ```bash
   # Core infrastructure first
   docker compose -f docker/docker-compose.yml up -d kafka minio

   # Wait for Kafka healthy, then setup topics
   docker exec -it kafka bash /scripts/setup/create-topics.sh

   # Data services
   docker compose -f docker/docker-compose.yml up -d hive-metastore trino-coordinator trino-worker-1 trino-worker-2

   # Application services
   docker compose -f docker/docker-compose.yml up -d producer rtsp-pusher chatbot

   # Monitoring
   docker compose -f docker/docker-compose.yml up -d prometheus grafana
   ```

4. **Verify health**:
   ```bash
   docker compose -f docker/docker-compose.yml ps
   # All services should show "healthy" or "running"
   ```

5. **Quick smoke test**:
   - Kafka UI: http://localhost:18085
   - MinIO Console: http://localhost:9001
   - Trino: http://localhost:8080
   - Grafana: http://localhost:3000
   - Prometheus: http://localhost:9090

## Teardown
```bash
# Stop all
docker compose -f docker/docker-compose.yml down

# Stop + remove volumes (DESTRUCTIVE)
docker compose -f docker/docker-compose.yml down -v
```
