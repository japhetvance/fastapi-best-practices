# Deployment: Docker, Gunicorn, Health Checks, Cloud Patterns

## Production Dockerfile

Use multi-stage builds to keep the final image small and secure.

```dockerfile
# ---- Build stage ----
FROM python:3.12-slim AS builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY pyproject.toml ./
RUN pip install --no-cache-dir --prefix=/install .

# ---- Runtime stage ----
FROM python:3.12-slim AS runtime

WORKDIR /app

# Runtime dependencies only (libpq for psycopg, no gcc)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

# Copy installed packages from builder
COPY --from=builder /install /usr/local

# Copy application code
COPY app/ ./app/
COPY migrations/ ./migrations/
COPY alembic.ini ./

# Non-root user
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000

CMD ["gunicorn", "app.main:create_app()", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--timeout", "120", \
     "--access-logfile", "-"]
```

### Key Decisions

- **Non-root user:** Always run as a non-root user in production. Many orchestrators enforce this.
- **No .env in image:** Environment variables come from the runtime (docker-compose, Kubernetes, cloud provider). Never bake secrets into the image.
- **curl for health checks:** Include `curl` so Docker and orchestrators can probe the health endpoint.

## Gunicorn Configuration

For more control, use a config file instead of CLI flags:

```python
# gunicorn.conf.py
import multiprocessing

bind = "0.0.0.0:8000"
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn.workers.UvicornWorker"
timeout = 120
keepalive = 5
accesslog = "-"
errorlog = "-"
loglevel = "info"

# Graceful restart
graceful_timeout = 30
max_requests = 1000          # Restart workers after N requests (leak protection)
max_requests_jitter = 50     # Stagger restarts
```

Then simplify the CMD:

```dockerfile
CMD ["gunicorn", "app.main:create_app()", "-c", "gunicorn.conf.py"]
```

### Worker Count Guidelines

| Deployment | Workers | Reasoning |
|---|---|---|
| Single container, 2 vCPU | 4–5 | 2×cores + 1 |
| Kubernetes pod, 1 vCPU | 2–3 | Match resource limits |
| I/O-heavy (many DB/HTTP calls) | Higher (3×cores) | Workers spend time waiting |
| CPU-heavy (ML inference) | Lower (1×cores) | Workers compete for CPU |

If you're behind a load balancer with multiple replicas, prefer fewer workers per replica and more replicas — it gives better fault isolation.

## Docker Compose (Development + Staging)

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  migrate:
    build: .
    command: alembic upgrade head
    env_file: .env
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

Run migrations as a separate service that exits after completion. Don't embed migration logic in your app startup.

## Health Check Endpoint

```python
# app/main.py or a dedicated health router
from fastapi import APIRouter
from sqlalchemy import text
from app.database import engine

health_router = APIRouter(tags=["health"])

@health_router.get("/health")
async def health_check():
    """Liveness + readiness check. Verifies DB connectivity."""
    try:
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        return JSONResponse(
            status_code=503,
            content={"status": "unhealthy", "database": str(e)},
        )
```

For Kubernetes, split into separate endpoints:
- `/health/live` — app is running (always 200 unless the process is dead)
- `/health/ready` — app can accept traffic (checks DB, Redis, etc.)

## Azure Container Apps / Azure Container Registry

Push to ACR and deploy to Container Apps:

```bash
# Build and push
az acr build --registry myregistry --image myapi:v1.0.0 .

# Deploy
az containerapp update \
  --name my-api \
  --resource-group my-rg \
  --image myregistry.azurecr.io/myapi:v1.0.0
```

Environment variables in Azure Container Apps:
```bash
az containerapp update \
  --name my-api \
  --resource-group my-rg \
  --set-env-vars \
    DATABASE_URL=secretref:db-url \
    JWT_SECRET_KEY=secretref:jwt-secret
```

Use Azure Key Vault references for secrets — don't put them in plain env vars in the portal.

## AWS ECS / Fargate

Task definition snippet:

```json
{
  "containerDefinitions": [{
    "name": "api",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapi:v1.0.0",
    "portMappings": [{"containerPort": 8000}],
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3,
      "startPeriod": 10
    },
    "secrets": [
      {"name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:..."},
      {"name": "JWT_SECRET_KEY", "valueFrom": "arn:aws:secretsmanager:..."}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/myapi",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "api"
      }
    }
  }],
  "cpu": "512",
  "memory": "1024"
}
```

## CI/CD Pipeline Pattern

```yaml
# .github/workflows/deploy.yml (simplified)
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e ".[test]"
      - run: pytest --cov=app

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push Docker image
        run: |
          docker build -t myregistry/myapi:${{ github.sha }} .
          docker push myregistry/myapi:${{ github.sha }}
      - name: Deploy
        run: |
          # Your deployment command (az containerapp update, aws ecs update-service, etc.)
```

## .dockerignore

Keep images clean:

```
.git
.env
.env.*
__pycache__
*.pyc
.pytest_cache
.mypy_cache
.ruff_cache
tests/
docs/
*.md
docker-compose*.yml
```
