---
name: backend-fastapi-dev
description: Guiding principles for Python backend development with FastAPI. Use whenever building APIs, designing routes, setting up databases, writing tests, configuring Docker/Nginx deployment, integrating LLMs (cloud or local), implementing background tasks with Celery, or working on any Python backend code. Trigger on any mention of "backend", "FastAPI", "API", "endpoint", "route", "deploy", "Docker", "Celery", "OpenRouter", "Ollama", "local LLM", "pgvector", "embeddings", "RAG", "test the backend", or requests to build, update, refactor, or deploy a Python backend service. Also trigger when the user asks about structuring a multi-app backend, setting up background tasks, configuring reverse proxies, designing REST APIs, implementing authentication, or switching between cloud and local LLM providers. Even casual requests like "set up my backend", "create an API for X", or "use a local model" should trigger this skill.
version: 2.0.0
---

# Backend Development with FastAPI

Production-quality Python backend development.

## Core Philosophy

**Never ship code you can't explain.** If you can't articulate what every line does and why, it doesn't belong in the codebase.

**Spec before code.** Generate a `spec.md` with requirements, architecture decisions, and constraints before implementation. The spec guides the code; the code validates the spec.

**Architecture first.** Decide the patterns, then generate the code. AI provides velocity — you provide judgment on trade-offs, boundaries, and failure modes.

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Framework | **FastAPI** | Async-native, auto OpenAPI docs, dependency injection, Pydantic integration |
| Database | **PostgreSQL + pgvector** | Relational data + vector embeddings in one DB — no separate vector service |
| ORM | **SQLAlchemy 2.0 async** | Type-safe models, mature migration story with Alembic |
| Migrations | **Alembic** | Schema versioning, auto-generation from models |
| Cache / Broker | **Redis** | Response caching, Celery broker, rate limiting, session store |
| Task Queue | **Celery** | Background/long-running work — LLM calls, file processing, embeddings |
| LLM Gateway | **OpenRouter + Ollama** | Cloud (OpenRouter) for hosted models, local (Ollama) for self-hosted — swappable via config |
| Embeddings | **OpenRouter / sentence-transformers** | Cloud embeddings via OpenRouter, or local via sentence-transformers + Ollama — per-project choice |
| Auth | **JWT (PyJWT)** | Stateless auth across multiple frontends from one backend |
| Reverse Proxy | **Nginx** | TLS termination, request buffering, rate limiting, static files |
| Containers | **Docker + Compose** | Reproducible builds, one-command local dev and deployment |
| Logging | **structlog** | Structured JSON logs with bound context — filterable, parseable |
| Validation | **Pydantic v2** | Request/response contracts, settings management |
| Testing | **pytest + httpx + factory_boy** | Async test client, model factories, coverage reporting |

---

## Project Structure

Organize by **domain**, not by technical layer. Each domain module owns its business logic. The API layer stays thin. This scales from a single-purpose API to a modular monolith.

```
project-root/
├── docker-compose.yml
├── Dockerfile
├── nginx/
│   └── conf.d/default.conf
├── alembic/
│   └── versions/
├── src/
│   ├── main.py                     # App factory, middleware, startup/shutdown
│   ├── core/
│   │   ├── config.py               # Pydantic Settings — all config from env vars
│   │   ├── security.py             # JWT encode/decode, password hashing
│   │   ├── dependencies.py         # Shared FastAPI deps (DB session, current user)
│   │   ├── logging.py              # structlog setup, context-bound loggers
│   │   ├── llm.py                  # LLM client abstraction (OpenRouter / Ollama)
│   │   └── embeddings.py           # Embedding client abstraction (cloud / local)
│   ├── domain/
│   │   └── <module>/               # One directory per business domain
│   │       ├── service.py          # Business logic lives HERE
│   │       └── models.py           # Domain models
│   ├── api/v1/
│   │   ├── router.py               # Aggregates all domain routers
│   │   └── <module>/
│   │       ├── routes.py           # Thin handlers: validate → delegate → respond
│   │       └── schemas.py          # Pydantic request/response models
│   ├── infrastructure/
│   │   ├── database.py             # Async engine + session factory
│   │   ├── redis.py                # Connection pool
│   │   ├── models/                 # SQLAlchemy ORM models
│   │   └── repositories/           # Data access layer
│   └── workers/
│       ├── celery_app.py           # Celery instance + config
│       └── <module>_tasks.py       # Background tasks per domain
├── tests/
│   ├── conftest.py
│   ├── factories/
│   ├── unit/domain/
│   ├── integration/api/
│   └── e2e/
├── docs/
├── pyproject.toml
└── .env.example
```

**Multi-app backends:** create a `StrEnum` registry giving each app its own route prefix (`/api/v1/<app>/`), Celery queue, and log tag. Keep domain modules isolated — app A never imports from app B's domain.

---

## Code Patterns

**Thin handlers, fat services** — the single most important structural rule:

```python
@router.post("/items", status_code=201)
async def create_item(
    request: CreateItemRequest,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db_session),
):
    service = ItemService(db=db)
    result = await service.create(request, user_id=current_user.id)
    return SuccessResponse(data=result)
```

**Dependency injection** for DB sessions, auth, config — all via `Depends()`. Testing becomes trivial: swap deps via `app.dependency_overrides`.

**Pydantic on all boundaries.** Every request body, response, and query param set gets a model. `ConfigDict(str_strip_whitespace=True)`, field constraints, custom validators.

**Async for I/O, sync for CPU.** Routes and DB calls are async. CPU-heavy work goes to sync Celery tasks.

**Custom exceptions → HTTP codes.** `AppException` base with `code`, `message`, `status_code`, `details`. One global handler serializes into the error envelope. Never leak internals.

---

## LLM Gateway (Cloud + Local)

All LLM calls go through one abstraction — domain code never knows (or cares) whether it's hitting OpenRouter or Ollama. Switch providers via config, not code changes.

### Provider Config

```python
# src/core/config.py
class Settings(BaseSettings):
    # Provider selection: "openrouter" or "ollama"
    LLM_PROVIDER: str = "openrouter"
    EMBEDDING_PROVIDER: str = "openrouter"      # "openrouter" or "local"

    # OpenRouter (cloud)
    OPENROUTER_API_KEY: str = ""
    OPENROUTER_BASE_URL: str = "https://openrouter.ai/api/v1"
    OPENROUTER_DEFAULT_MODEL: str = "anthropic/claude-sonnet-4-20250514"

    # Ollama (local)
    OLLAMA_BASE_URL: str = "http://ollama:11434"  # Docker service name
    OLLAMA_DEFAULT_MODEL: str = "llama3.1"

    # Embeddings
    EMBEDDING_MODEL_CLOUD: str = "openai/text-embedding-3-small"
    EMBEDDING_MODEL_LOCAL: str = "nomic-embed-text"  # Ollama model
    EMBEDDING_DIMENSION: int = 1536                   # Must match pgvector column

    LLM_TIMEOUT: int = 120
```

### Abstract Client

```python
# src/core/llm.py
from abc import ABC, abstractmethod

class LLMClient(ABC):
    @abstractmethod
    async def completion(self, messages: list[dict], model: str | None = None, **kwargs) -> dict: ...
    @abstractmethod
    async def close(self): ...

class OpenRouterClient(LLMClient):
    def __init__(self):
        self._client = httpx.AsyncClient(
            base_url=settings.OPENROUTER_BASE_URL,
            headers={"Authorization": f"Bearer {settings.OPENROUTER_API_KEY}"},
            timeout=httpx.Timeout(settings.LLM_TIMEOUT),
        )

    async def completion(self, messages, model=None, **kwargs):
        model = model or settings.OPENROUTER_DEFAULT_MODEL
        response = await self._client.post(
            "/chat/completions", json={"model": model, "messages": messages, **kwargs},
        )
        response.raise_for_status()
        return response.json()

class OllamaClient(LLMClient):
    def __init__(self):
        self._client = httpx.AsyncClient(
            base_url=settings.OLLAMA_BASE_URL,
            timeout=httpx.Timeout(settings.LLM_TIMEOUT),
        )

    async def completion(self, messages, model=None, **kwargs):
        model = model or settings.OLLAMA_DEFAULT_MODEL
        response = await self._client.post(
            "/api/chat", json={"model": model, "messages": messages, "stream": False, **kwargs},
        )
        response.raise_for_status()
        return response.json()

def create_llm_client() -> LLMClient:
    if settings.LLM_PROVIDER == "ollama":
        return OllamaClient()
    return OpenRouterClient()

llm_client = create_llm_client()  # Singleton — init at startup
```

### Embedding Client

```python
# src/core/embeddings.py
class EmbeddingClient(ABC):
    @abstractmethod
    async def embed(self, texts: list[str]) -> list[list[float]]: ...

class OpenRouterEmbeddings(EmbeddingClient):
    async def embed(self, texts):
        response = await self._client.post("/embeddings", json={
            "model": settings.EMBEDDING_MODEL_CLOUD, "input": texts,
        })
        return [item["embedding"] for item in response.json()["data"]]

class OllamaEmbeddings(EmbeddingClient):
    async def embed(self, texts):
        embeddings = []
        for text in texts:  # Ollama embeds one at a time
            response = await self._client.post("/api/embed", json={
                "model": settings.EMBEDDING_MODEL_LOCAL, "input": text,
            })
            embeddings.append(response.json()["embeddings"][0])
        return embeddings

class LocalSentenceTransformerEmbeddings(EmbeddingClient):
    """For CPU/GPU-local embeddings without Ollama."""
    def __init__(self):
        from sentence_transformers import SentenceTransformer
        self._model = SentenceTransformer("all-MiniLM-L6-v2")
    async def embed(self, texts):
        return self._model.encode(texts).tolist()  # sync, run in executor for async

def create_embedding_client() -> EmbeddingClient:
    if settings.EMBEDDING_PROVIDER == "local":
        return OllamaEmbeddings()  # or LocalSentenceTransformerEmbeddings()
    return OpenRouterEmbeddings()
```

Domain code calls `llm_client.completion(...)` and `embedding_client.embed(...)` — it never imports a provider directly. Log every call with model name and token usage. Never call LLM/embedding APIs outside these wrappers.

---

## Logging

**structlog** with JSON output. Bind context (request ID, module tag) so every line is filterable.

```python
def setup_logging(log_level: str = "INFO"):
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer(),
        ],
        logger_factory=structlog.PrintLoggerFactory(),
    )

def get_logger(tag: str = "app") -> structlog.BoundLogger:
    return structlog.get_logger().bind(tag=tag)
```

Add request middleware that auto-binds `request_id`, `method`, `path`. Return `X-Request-ID` in response headers.

- **info** — request lifecycle, external API calls, task dispatches
- **warning** — retries, degraded responses, slow queries (>1s)
- **error** — failures, with enough context to reproduce
- **never log** — tokens, passwords, API keys, PII

---

## Celery

Use for anything blocking >2s or that shouldn't hold up a response.

```python
celery_app = Celery("backend", broker=settings.REDIS_URL, backend=settings.REDIS_URL)
celery_app.conf.update(
    task_serializer="json",
    task_acks_late=True,
    worker_prefetch_multiplier=1,
    task_soft_time_limit=300,
    task_time_limit=360,
)
```

- `bind=True` + `self.retry(countdown=2**self.request.retries)` for exponential backoff
- Named queues for domain isolation
- Return **202 Accepted** with `task_id`; clients poll for status

---

## Docker & Nginx

**Compose** orchestrates: `api`, `celery-worker`, `celery-beat`, `db` (pgvector:pg16), `redis`, `nginx`, and optionally `ollama`. Healthchecks on db/redis so API waits. Volume-mount `./src` for dev hot-reload.

For local LLM support, add Ollama to compose:
```yaml
ollama:
  image: ollama/ollama
  ports: ["11434:11434"]
  volumes:
    - ollama_data:/root/.ollama   # Persist downloaded models
  deploy:
    resources:
      reservations:
        devices:
          - capabilities: [gpu]   # Optional — remove if CPU-only
```
Pull models after the container starts: `docker exec ollama ollama pull llama3.1 && docker exec ollama ollama pull nomic-embed-text`

**Nginx in front** because even with one server you get: TLS termination, slow-client buffering, per-IP rate limiting (separate zones for cheap vs expensive endpoints), static file serving, and trivial horizontal scaling. Set `proxy_read_timeout ~120s` for LLM responses. Forward `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Request-ID`.

---

## Type Safety

- `strict` mode in pyproject.toml (mypy)
- Type hints on every function signature
- `TypedDict` for complex dicts
- Dataclasses or Pydantic — never raw dicts at boundaries

---

## Do NOT

- Use `*` imports
- Put business logic in route handlers
- Use `Any` without justification
- Commit commented-out code
- Use raw SQL strings
- Hardcode config — `.env` + Pydantic Settings
- Skip Alembic migrations
- Catch bare `Exception` without re-raising or logging
- Call LLM/embedding APIs outside the abstraction layer
- Log secrets

## Anti-Patterns

| Pattern | Problem | Fix |
|---------|---------|-----|
| God service | One class does everything | Decompose by responsibility |
| Anemic domain | Logic in handlers | Domain service layer |
| Shotgun surgery | One change, many files | Consolidate related logic |
| N+1 queries | Per-item DB calls in a loop | Eager loading / DataLoader |
| Primitive obsession | Raw strings for concepts | Value objects / enums |
| Fat handlers | Business logic in routes | Delegate to services |
| Untagged logs | No context | Bind tag + request_id |
| Direct provider calls | LLM/embed API called without wrapper | Use the abstraction layer |

---

## References

Read these when you need detailed patterns. Each is standalone — load only what's relevant.

| Reference | When to Read |
|-----------|-------------|
| [references/api-design.md](references/api-design.md) | REST conventions, response envelopes, pagination, filtering, auth (JWT), pgvector/RAG with cloud+local embeddings, rate limiting, versioning, streaming, gRPC, bulk ops |
| [references/security.md](references/security.md) | Input validation, injection prevention (SQL/XSS/command/path), file upload handling, RBAC, safe error messages, threat modeling |
| [references/testing.md](references/testing.md) | Test pyramid, coverage targets, fixtures, factory pattern, mocking, async testing, integration tests, property-based testing, CI/CD |

### External References

- FastAPI Full-Stack Template: `github.com/fastapi/full-stack-fastapi-template`
- FastAPI Best Practices: `github.com/zhanymkanov/fastapi-best-practices`
- Amazon Builders' Library for distributed systems
- OpenRouter API Docs: `openrouter.ai/docs`
- Ollama API Docs: `github.com/ollama/ollama/blob/main/docs/api.md`