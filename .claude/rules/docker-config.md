---
globs:
  - "docker/**"
  - "docker-compose*.yml"
---

# Docker Configuration Rules

## Bắt buộc
- KHÔNG hardcode credentials — dùng biến từ `.env`
- Mọi service PHẢI có `healthcheck` với `interval`, `timeout`, `retries`
- Mọi service PHẢI có `deploy.resources.limits` (memory + cpus)
- Dùng `depends_on` với `condition: service_healthy` thay vì chỉ `depends_on`

## Naming
- Service names: kebab-case (e.g., `kafka-ui`, `trino-coordinator`)
- Volume names: kebab-case với prefix project (e.g., `violence-detection-minio-data`)
- Network: dùng network `streamhouse-net` chung

## Ports
- Không expose port nội bộ ra host trừ khi cần thiết cho dev
- Format: `"HOST_PORT:CONTAINER_PORT"`

## Environment Variables
- Tham chiếu `.env` file: `env_file: - .env`
- Sensitive values: dùng `${VAR_NAME}` từ `.env`
- Luôn có `.env.example` tương ứng

## Docker Compose File
- File chính: `docker/docker-compose.yml`
- Env file: `docker/.env` (gitignored), `docker/.env.example` (committed)
