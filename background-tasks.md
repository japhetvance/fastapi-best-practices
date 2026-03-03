# Background Tasks & Async Workers

## FastAPI Built-in BackgroundTasks

Use for lightweight, fire-and-forget work that doesn't need retries or monitoring.

```python
from fastapi import BackgroundTasks

async def send_welcome_email(email: str, name: str):
    """Runs after the response is sent."""
    # Send email via SMTP / SES / SendGrid
    ...

@router.post("/", response_model=UserResponse, status_code=201)
async def create_user(
    data: UserCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
):
    service = UserService(db)
    user = await service.create_user(data)
    background_tasks.add_task(send_welcome_email, user.email, user.name)
    return user
```

**Good use cases:** sending emails, logging analytics events, cleaning up temp files, pushing to a webhook.

**Bad use cases:** anything that takes more than a few seconds, needs retries, needs monitoring, or must not be lost if the process crashes. For those, use a task queue.

## Task Queue with ARQ (Lightweight, Redis-backed)

ARQ is a good fit for FastAPI because it's async-native and simple.

```bash
pip install arq
```

### Define tasks

```python
# app/workers/tasks.py
import httpx
from arq import create_pool
from arq.connections import RedisSettings
from app.config import settings

async def process_document(ctx: dict, document_id: int):
    """Heavy processing — OCR, extraction, AI analysis."""
    db = ctx["db"]
    doc = await get_document(db, document_id)
    result = await run_extraction_pipeline(doc)
    await save_results(db, document_id, result)

async def generate_report(ctx: dict, report_id: int, params: dict):
    """Generate a PDF/Excel report."""
    ...

class WorkerSettings:
    """ARQ worker configuration."""
    functions = [process_document, generate_report]
    redis_settings = RedisSettings.from_dsn(settings.REDIS_URL)
    max_jobs = 10
    job_timeout = 300  # 5 minutes

    @staticmethod
    async def on_startup(ctx: dict):
        """Runs when worker starts — initialize shared resources."""
        ctx["db"] = await create_db_session()

    @staticmethod
    async def on_shutdown(ctx: dict):
        await ctx["db"].close()
```

### Enqueue from FastAPI

```python
# app/workers/client.py
from arq import create_pool
from arq.connections import RedisSettings
from app.config import settings

_pool = None

async def get_task_pool():
    global _pool
    if _pool is None:
        _pool = await create_pool(RedisSettings.from_dsn(settings.REDIS_URL))
    return _pool

# In a router or service
async def enqueue_document_processing(document_id: int):
    pool = await get_task_pool()
    job = await pool.enqueue_job("process_document", document_id)
    return job.job_id
```

### Run the worker

```bash
arq app.workers.tasks.WorkerSettings
```

Add to docker-compose as a separate service:

```yaml
  worker:
    build: .
    command: arq app.workers.tasks.WorkerSettings
    env_file: .env
    depends_on:
      - redis
      - db
```

## Task Queue with Celery (Heavy-duty)

Use Celery when you need task scheduling (crontab), complex workflows (chains, groups, chords), or when the team already uses it.

```bash
pip install celery[redis]
```

### Setup

```python
# app/workers/celery_app.py
from celery import Celery
from app.config import settings

celery = Celery(
    "worker",
    broker=settings.REDIS_URL,
    backend=settings.REDIS_URL,
)

celery.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="UTC",
    task_track_started=True,
    task_acks_late=True,           # Re-deliver if worker crashes
    worker_prefetch_multiplier=1,  # Fair task distribution
)
```

### Define tasks

```python
# app/workers/tasks.py
from .celery_app import celery

@celery.task(bind=True, max_retries=3, default_retry_delay=60)
def process_document(self, document_id: int):
    """Celery tasks are sync by default. Use sync DB driver here."""
    try:
        # Heavy processing
        ...
    except ExternalServiceError as exc:
        self.retry(exc=exc)
```

**Important:** Celery workers run synchronously by default. If your task needs async (e.g., async DB driver), either use a sync DB driver in workers or use `asgiref.sync.async_to_sync`.

### Enqueue from FastAPI

```python
from app.workers.tasks import process_document

@router.post("/documents/{doc_id}/process")
async def trigger_processing(doc_id: int):
    task = process_document.delay(doc_id)
    return {"task_id": task.id, "status": "queued"}

@router.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    from app.workers.celery_app import celery
    result = celery.AsyncResult(task_id)
    return {
        "task_id": task_id,
        "status": result.status,
        "result": result.result if result.ready() else None,
    }
```

## Choosing Between Options

| Need | Use |
|---|---|
| Quick, non-critical side effect | `BackgroundTasks` |
| Async-native, Redis available, moderate complexity | ARQ |
| Complex workflows, scheduling, large team, existing Celery infra | Celery |
| Extreme throughput, custom requirements | Raw Redis streams or Kafka consumers |

## Patterns for Task Status Tracking

If clients need to poll for task completion:

```python
# Simple status tracking with Redis
import redis.asyncio as redis

class TaskTracker:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def set_status(self, task_id: str, status: str, result: dict | None = None):
        data = {"status": status, "result": result}
        await self.redis.set(f"task:{task_id}", json.dumps(data), ex=3600)

    async def get_status(self, task_id: str) -> dict | None:
        data = await self.redis.get(f"task:{task_id}")
        return json.loads(data) if data else None
```

For real-time updates, use WebSockets or Server-Sent Events (SSE) instead of polling:

```python
from sse_starlette.sse import EventSourceResponse

@router.get("/tasks/{task_id}/stream")
async def stream_task_status(task_id: str):
    async def event_generator():
        while True:
            status = await tracker.get_status(task_id)
            yield {"event": "status", "data": json.dumps(status)}
            if status and status["status"] in ("completed", "failed"):
                break
            await asyncio.sleep(1)
    return EventSourceResponse(event_generator())
```
