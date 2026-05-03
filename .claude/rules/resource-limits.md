# Resource Limits — 16GB Windows Machine

## Hardware Budget
- **Total RAM**: 16GB (OS + Docker ~4GB → containers ~12GB max)
- **CPU cores**: 8 logical (shared across all containers)
- **ML Model**: Runs on separate machine — NOT on this host

## Resource Budget Table
Mọi service PHẢI có `deploy.resources.limits` với cả `memory` VÀ `cpus`.

### Core Services (always on) — 9.6GB RAM, 7.85 CPU
| Service | Memory | CPU | Profile |
|---------|--------|-----|---------|
| kafka | 512m | 0.50 | core |
| minio | 512m | 0.50 | core |
| minio_client | 64m | 0.10 | core |
| inference-mock | 256m | 0.25 | core |
| jobmanager | 1g | 0.50 | core |
| taskmanager | 2g | 2.00 | core |
| fluss-zookeeper | 256m | 0.25 | core |
| fluss-coordinator | 512m | 0.50 | core |
| fluss-tablet | 512m | 0.50 | core |
| mysql | 512m | 0.50 | core |
| hive-metastore | 512m | 0.25 | core |
| trino-coordinator | 1536m | 1.00 | core |
| chatbot | 1536m | 1.00 | core |

### Optional Services (profile-gated) — 3.6GB RAM
| Service | Memory | CPU | Profile |
|---------|--------|-----|---------|
| kafka-ui | 384m | 0.25 | ui |
| producer | 256m | 0.50 | streaming |
| rtsp_pusher | 256m | 0.50 | streaming |
| mediamtx | 128m | 0.25 | streaming |
| prometheus | 256m | 0.25 | monitoring |
| grafana | 256m | 0.25 | monitoring |
| node-exporter | 64m | 0.10 | monitoring |
| trino-worker-1 | 1g | 1.00 | scaling |
| trino-worker-2 | 1g | 1.00 | scaling |

## Rules for Adding New Services
1. **PHẢI** có `deploy.resources.limits.memory` VÀ `deploy.resources.limits.cpus`
2. Tổng RAM core KHÔNG được vượt **10GB** (giữ 2GB buffer)
3. Service không thiết yếu cho dev/test → đặt vào profile
4. CPU limit tối đa cho 1 service: **2.00** (chỉ taskmanager được phép)
5. Memory tối đa cho 1 service: **2g** (chỉ taskmanager được phép)
6. Nếu cần thêm service > 512m RAM → phải giảm service khác để bù

## Profile Usage
```bash
# Core only (AI test) — ~9.6GB
docker compose -f docker/docker-compose.yml up -d

# + streaming
docker compose -f docker/docker-compose.yml --profile streaming up -d

# + monitoring
docker compose -f docker/docker-compose.yml --profile monitoring up -d

# Near-full (no trino workers)
docker compose -f docker/docker-compose.yml --profile streaming --profile monitoring --profile ui up -d
```

## Flink TaskManager Notes
- Task slots: 3 (giảm từ 5 để tiết kiệm RAM)
- Process size: 1536m (phải < memory limit 2g)
- Nếu cần thêm slots → tăng memory limit tương ứng (mỗi slot ~400m)