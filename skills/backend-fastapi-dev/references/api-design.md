# API Design Reference

Patterns for REST API design in Python/FastAPI backends. Read when designing endpoints, response formats, authentication, pagination, RAG pipelines, or versioning strategies.

## Table of Contents

- [Protocol Selection](#protocol-selection)
- [Route Conventions](#route-conventions)
- [HTTP Methods and Status Codes](#http-methods-and-status-codes)
- [Response Envelope Standards](#response-envelope-standards)
- [Request Validation](#request-validation)
- [Pagination](#pagination)
- [Filtering and Search](#filtering-and-search)
- [Authentication](#authentication)
- [Authorization](#authorization)
- [RAG with pgvector](#rag-with-pgvector)
- [Rate Limiting](#rate-limiting)
- [Versioning](#versioning)
- [Streaming Responses](#streaming-responses)
- [Idempotency](#idempotency)
- [Bulk Operations](#bulk-operations)
- [OpenAPI Documentation](#openapi-documentation)
- [Query Optimization](#query-optimization)
- [gRPC Patterns](#grpc-patterns)

---

## Protocol Selection

| Protocol | Use Case | When to Choose |
|----------|----------|----------------|
| **REST** | Client-to-backend, external APIs | Default. Simple, cacheable, widely understood |
| **gRPC** | Internal service-to-service | High throughput, you control both ends |

REST is the right default for any endpoint consumed by browsers, mobile apps, or external clients. gRPC makes sense only when you extract an internal service and need high-performance binary communication between endpoints you fully control.

---

## Route Conventions

### URL Structure

- Resources are **plural nouns**: `/users`, `/orders`, `/documents`
- IDs in paths: `/users/{user_id}`
- Nested for ownership: `/users/{user_id}/orders`
- Actions as sub-resources via POST: `/orders/{order_id}/cancel`
- Max 2-3 nesting levels; flatten with query params beyond that
- Cross-domain references use query params, not deep nesting

### Multi-App Prefixes

When one backend serves multiple apps, give each a prefix under the API version:

```
/api/v1/
├── auth/              # Shared services
├── health/
├── <app-a>/           # App A's resources
│   ├── POST   /items
│   ├── GET    /items
│   ├── GET    /items/{id}
│   └── POST   /items/{id}/process
└── <app-b>/           # App B's resources
    ├── POST   /documents
    └── POST   /query
```

Define prefixes in a central registry (e.g., a `StrEnum`). Set the prefix at router registration, not on individual routes — this keeps route files portable.

### Input Handling

**Path parameters** — required identifiers:
```
GET /users/123
GET /users/456/orders/789
```

**Query parameters** — optional filters:
```
GET /users?role=admin&sort=-created_at&limit=20
```

**Request body** — data for creation/updates (POST, PUT, PATCH):
```python
@router.post("/items")
async def create_item(item: CreateItemRequest):
    ...

@router.patch("/items/{item_id}")
async def update_item(item_id: str, updates: UpdateItemRequest):
    ...
```

---

## HTTP Methods and Status Codes

### Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET | Read | Yes | Yes |
| POST | Create | No | No |
| PUT | Full replace | Yes | No |
| PATCH | Partial update | Yes | No |
| DELETE | Remove | Yes | No |

### Status Codes

```
2xx Success
  200  OK — GET/PUT/PATCH success
  201  Created — POST success, include Location header
  202  Accepted — async task queued, return task_id
  204  No Content — DELETE success

4xx Client Errors
  400  Bad Request — malformed syntax, validation failure
  401  Unauthorized — missing/invalid auth
  403  Forbidden — authenticated but not permitted
  404  Not Found — resource doesn't exist
  409  Conflict — duplicate, version mismatch
  422  Unprocessable Entity — valid syntax, semantic error
  429  Too Many Requests — rate limit hit

5xx Server Errors
  500  Internal Server Error — unexpected failure
  502  Bad Gateway — upstream failure (LLM provider, etc.)
  503  Service Unavailable — overload/maintenance
  504  Gateway Timeout — upstream timeout
```

---

## Response Envelope Standards

Every response follows a consistent envelope. Include `request_id` in `meta` for traceability.

### Success

```json
{
  "data": { ... },
  "meta": {
    "request_id": "uuid",
    "timestamp": "ISO8601"
  }
}
```

### Collection

```json
{
  "data": [ ... ],
  "meta": {
    "total_count": 142,
    "page": 1,
    "page_size": 20,
    "has_more": true
  },
  "links": {
    "self": "/items?page=1",
    "next": "/items?page=2",
    "prev": null
  }
}
```

### Async Task (202)

```json
{
  "data": {
    "task_id": "celery-task-uuid",
    "status": "queued"
  },
  "meta": { "request_id": "uuid" }
}
```

### Error

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable description",
    "details": { "field": "email", "reason": "Invalid format" }
  },
  "meta": { "request_id": "uuid" }
}
```

### Pydantic Envelope Models

```python
from pydantic import BaseModel, Field
from typing import Generic, TypeVar

T = TypeVar("T")

class Meta(BaseModel):
    request_id: str
    timestamp: datetime = Field(default_factory=datetime.utcnow)

class SuccessResponse(BaseModel, Generic[T]):
    data: T
    meta: Meta

class ErrorDetail(BaseModel):
    code: str
    message: str
    details: dict | list | None = None

class ErrorResponse(BaseModel):
    error: ErrorDetail
    meta: Meta
```

---

## Request Validation

### Pydantic Models

```python
class CreateUserRequest(BaseModel):
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)
    age: int = Field(..., ge=0, le=150)

    model_config = ConfigDict(str_strip_whitespace=True)

class UpdateUserRequest(BaseModel):
    email: EmailStr | None = None
    name: str | None = Field(default=None, min_length=1, max_length=100)
```

### Validation Error Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {"field": "email", "message": "Invalid email format"},
      {"field": "age", "message": "Must be >= 0"}
    ]
  }
}
```

---

## Pagination

### Offset-Based (Simple)

```
GET /items?limit=20&offset=40
```
Simple, supports random access. Degrades on large offsets, inconsistent with concurrent writes.

### Cursor-Based (Recommended)

```
GET /items?limit=20&cursor=eyJpZCI6MTAwfQ==
```
Consistent results, efficient on large datasets. No random access, needs stable sort.

### Implementation

```python
@router.get("/items")
async def list_items(
    limit: int = Query(default=20, le=100),
    cursor: str | None = Query(default=None),
) -> PaginatedResponse[Item]:
    # Decode cursor → WHERE id > cursor_id → encode next cursor from last item
```

---

## Filtering and Search

```
GET /orders?status=pending&created_after=2024-01-01
GET /products?category=electronics&price_min=100&price_max=500
GET /users?q=john&sort=-created_at
```

### Conventions

- snake_case for all param names
- Date ranges: `created_after`, `created_before`
- Numeric ranges: `price_min`, `price_max`
- Descending sort: `-` prefix → `sort=-created_at`
- Multiple values: `status=pending,processing`
- Text search: `q=` param

---

## Authentication

### JWT (Primary — Recommended)

```
Authorization: Bearer <jwt_token>
```

Stateless — token carries claims. Include `sub` (user ID), `email`, `role`, `iat`, `exp`, `iss`.

```python
def create_access_token(user_id: str, role: str) -> str:
    payload = {
        "sub": user_id,
        "role": role,
        "iat": datetime.now(timezone.utc),
        "exp": datetime.now(timezone.utc) + timedelta(minutes=settings.JWT_EXPIRE_MINUTES),
        "iss": "your-backend",
    }
    return jwt.encode(payload, settings.JWT_SECRET, algorithm=settings.JWT_ALGORITHM)

def decode_jwt(token: str) -> dict | None:
    try:
        return jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.JWT_ALGORITHM])
    except jwt.PyJWTError:
        return None
```

### Auth Dependency

```python
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(HTTPBearer()),
    db: AsyncSession = Depends(get_db_session),
) -> User:
    payload = decode_jwt(credentials.credentials)
    if not payload:
        raise HTTPException(401, "Invalid or expired token")
    user = await db.get(User, payload["sub"])
    if not user:
        raise HTTPException(401, "User not found")
    return user
```

### API Key (Service-to-Service)

```
X-API-Key: <scoped_key>
```
For internal callers, webhooks, CI/CD. Scope keys to specific permissions.

### Error Responses

- 401 for missing/invalid/expired credentials
- 403 for valid credentials, insufficient permissions
- Include `WWW-Authenticate: Bearer` on 401

---

## Authorization

### Role-Based Actions

```python
PERMISSIONS = {
    "create_item": ["admin", "editor"],
    "delete_item": ["admin"],
    "view_item": ["admin", "editor", "viewer"],
}

def require_permission(permission: str):
    def checker(current_user: User = Depends(get_current_user)):
        if current_user.role not in PERMISSIONS.get(permission, []):
            raise HTTPException(403, "Insufficient permissions")
        return current_user
    return Depends(checker)
```

### Resource Ownership

Never trust client-provided user IDs for authorization decisions:

```python
@router.delete("/items/{item_id}")
async def delete_item(
    item_id: str,
    current_user: User = Depends(get_current_user),
):
    item = await get_item(item_id)
    if item.owner_id != current_user.id and current_user.role != "admin":
        raise HTTPException(403, "Not authorized")
    await remove_item(item_id)
```

---

## RAG with pgvector

PostgreSQL + pgvector keeps vector search in the same database as relational data — no extra infrastructure.

### Schema

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL,
    source_url TEXT,
    content_hash TEXT UNIQUE,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE document_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index INT NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536),        -- dimension MUST match your model (1536 for OpenAI, 768 for nomic, 384 for MiniLM)
    token_count INT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_chunks_embedding
ON document_chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

### SQLAlchemy Model

```python
from pgvector.sqlalchemy import Vector
from src.core.config import settings

class DocumentChunk(Base):
    __tablename__ = "document_chunks"
    id = Column(UUID, primary_key=True, default=uuid4)
    document_id = Column(UUID, ForeignKey("documents.id", ondelete="CASCADE"))
    chunk_index = Column(Integer, nullable=False)
    content = Column(Text, nullable=False)
    embedding = Column(Vector(settings.EMBEDDING_DIMENSION))  # Matches chosen model
    token_count = Column(Integer)
    metadata_ = Column("metadata", JSONB, default={})
```

### Semantic Search

```python
async def semantic_search(
    db: AsyncSession,
    query_embedding: list[float],
    top_k: int = 10,
    threshold: float = 0.7,
) -> list[dict]:
    result = await db.execute(text("""
        SELECT id, content, document_id,
               1 - (embedding <=> :embedding::vector) AS similarity
        FROM document_chunks
        WHERE 1 - (embedding <=> :embedding::vector) >= :threshold
        ORDER BY embedding <=> :embedding::vector
        LIMIT :top_k
    """), {"embedding": str(query_embedding), "top_k": top_k, "threshold": threshold})
    return [dict(r) for r in result.mappings()]
```

### RAG Query Flow

```
1. User question → POST /query
2. Generate query embedding via embedding_client.embed()
     Cloud: OpenRouter /embeddings → text-embedding-3-small (1536d)
     Local: Ollama /api/embed → nomic-embed-text (768d)
     Local: sentence-transformers → all-MiniLM-L6-v2 (384d)
3. Semantic search → pgvector returns top-K chunks
4. Build prompt: system instructions + retrieved chunks + question
5. LLM completion via llm_client.completion()
     Cloud: OpenRouter → any hosted model
     Local: Ollama → llama3.1, mistral, etc.
6. Return synthesized answer + source chunk references
```

The embedding dimension must be consistent per project — you can't mix 1536d and 768d embeddings in the same column. Choose one provider for embeddings and stick with it, or maintain separate columns if you need both. Set `EMBEDDING_DIMENSION` in config and reference it in your SQLAlchemy model and Alembic migrations.

Embedding generation is CPU/IO-heavy — run it as a Celery task. Store embeddings alongside content so re-embedding is only needed on content changes. When switching embedding models, you must re-embed all existing content.

### Embedding Provider Comparison

| Provider | Model | Dimension | Speed | Cost | Quality |
|----------|-------|-----------|-------|------|---------|
| OpenRouter (cloud) | text-embedding-3-small | 1536 | ~50ms/batch | ~$0.02/1M tokens | High |
| OpenRouter (cloud) | text-embedding-3-large | 3072 | ~80ms/batch | ~$0.13/1M tokens | Highest |
| Ollama (local) | nomic-embed-text | 768 | ~200ms/item | Free | Good |
| Ollama (local) | mxbai-embed-large | 1024 | ~300ms/item | Free | High |
| sentence-transformers | all-MiniLM-L6-v2 | 384 | ~10ms/item | Free | Good |
| sentence-transformers | all-mpnet-base-v2 | 768 | ~30ms/item | Free | High |

Use cloud embeddings when quality matters most and volume is manageable. Use local when you're iterating fast, running on a budget, or processing sensitive data that shouldn't leave your infrastructure.

---

## Rate Limiting

Apply at two layers: **Nginx** (per-IP, connection-level) and **application** (per-user, business-logic-level).

### Headers

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
Retry-After: 60
```

### Error (429)

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests",
    "details": { "retry_after": 60 }
  }
}
```

Separate Nginx rate-limit zones for general endpoints (e.g., 30r/s) and LLM-heavy endpoints (e.g., 5r/s). LLM calls are expensive — protect them aggressively.

---

## Versioning

### URL Path (Recommended)

```
/api/v1/items
/api/v2/items
```

- Major version in URL for breaking changes
- Support N-1 versions minimum (current + previous)
- 6-month deprecation notice minimum
- Document all changes in CHANGELOG

### Header-Based (Alternative)

```
Accept: application/vnd.api+json; version=2
```

---

## Streaming Responses

For endpoints where users expect real-time LLM output, use Server-Sent Events:

```python
from fastapi.responses import StreamingResponse

@router.post("/generate/stream")
async def stream_response(request: GenerateRequest):
    async def generate():
        async for chunk in llm_client.completion_stream(
            messages=build_messages(request),
        ):
            yield f"data: {json.dumps(chunk)}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

Both OpenRouter and Ollama support streaming — OpenRouter uses the OpenAI-compatible `stream: true` parameter, Ollama streams by default. The abstraction layer should normalize the chunk format so consumers don't care which provider is active.

---

## Idempotency

```
Idempotency-Key: <client-generated-uuid>
```

Store key + response for 24-48 hours. Return cached response on duplicate key. Required for POST, optional for other methods. Especially important for LLM-calling endpoints — avoids duplicate charges on cloud providers and wasted compute on local inference.

---

## Bulk Operations

```
POST /items/batch
{
  "items": [
    {"name": "A", "value": 1},
    {"name": "B", "value": 2}
  ]
}
```

```json
{
  "data": {
    "succeeded": [{"id": "1", "name": "A"}],
    "failed": [
      {"index": 1, "error": {"code": "DUPLICATE", "message": "Already exists"}}
    ]
  },
  "meta": { "total": 2, "succeeded": 1, "failed": 1 }
}
```

---

## OpenAPI Documentation

```python
@router.post(
    "/items",
    response_model=ItemResponse,
    status_code=201,
    summary="Create an item",
    description="Creates an item with the provided details.",
    responses={
        409: {"model": ErrorResponse, "description": "Already exists"},
        502: {"model": ErrorResponse, "description": "Upstream LLM failure"},
    },
)
async def create_item(request: CreateItemRequest) -> ItemResponse:
    ...
```

Group routes by domain using tags set at router registration:

```python
api_router.include_router(items_router, prefix="/api/v1/items", tags=["Items"])
```

Use `openapi_tags` on the FastAPI app for tag descriptions in the docs UI.

---

## Query Optimization

### N+1 Problem

```python
# Problem: 10 items → 10 separate owner queries
# Fix: DataLoader for batching

from aiodataloader import DataLoader

async def batch_load_owners(ids: list[str]) -> list[User]:
    users = await db.execute(select(User).where(User.id.in_(ids)))
    user_map = {u.id: u for u in users.scalars()}
    return [user_map.get(id) for id in ids]

owner_loader = DataLoader(batch_load_owners)
```

### HATEOAS Links (When Warranted)

```json
{
  "data": { "id": "123", "status": "pending" },
  "links": {
    "self": "/items/123",
    "approve": "/items/123/approve",
    "cancel": "/items/123/cancel"
  }
}
```

---

## gRPC Patterns

For high-performance internal service communication when you extract from the monolith.

```protobuf
syntax = "proto3";
service ItemService {
  rpc GetItem(GetItemRequest) returns (Item);
  rpc ListItems(ListItemsRequest) returns (stream Item);
}
```

```python
class ItemServicer(items_grpc.ItemServiceServicer):
    async def GetItem(self, request, context):
        item = await db.get(Item, request.item_id)
        if not item:
            context.abort(grpc.StatusCode.NOT_FOUND, "Not found")
        return item.to_proto()
```

Choose gRPC over REST when: both ends are services you control, binary format matters (~10x smaller), you need bi-directional streaming, or strong cross-service typing is worth the proto toolchain.