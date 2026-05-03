# Skill: Git Commit

Quy trình commit code theo Conventional Commits.

## Commit Message Format
```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

### Types
| Type | Khi nào dùng |
|------|-------------|
| `feat` | Tính năng mới |
| `fix` | Sửa bug |
| `docs` | Thay đổi documentation |
| `refactor` | Refactor code (không thêm feature, không fix bug) |
| `test` | Thêm/sửa test |
| `chore` | Build, CI/CD, dependencies |
| `perf` | Cải thiện performance |

### Scopes
| Scope | Thư mục |
|-------|---------|
| `docker` | docker/ |
| `kafka` | config/kafka/, scripts/streaming/ |
| `flink` | scripts/transform/ |
| `trino` | config/trino/ |
| `rag` | scripts/chatbot/ |
| `frontend` | frontend/ |
| `infra` | config/, scripts/setup/ |
| `docs` | docs/, CLAUDE.md, README.md |

### Examples
```
feat(flink): add data contract validation for bronze layer
fix(docker): resolve kafka healthcheck timeout issue
docs(rag): update agentic RAG architecture diagram
refactor(kafka): extract producer config to env vars
chore(infra): upgrade MinIO to latest version
```

## Pre-commit Checklist
1. `docker compose -f docker/docker-compose.yml config` (validate compose file)
2. Check no secrets in staged files (no hardcoded passwords, API keys)
3. Verify `.env` is NOT being committed

## Steps
1. Review changes: `git diff --staged`
2. Stage files: `git add <specific-files>` (avoid `git add .`)
3. Commit: `git commit -m "<type>(<scope>): <description>"`
4. Push: `git push origin <branch>`
