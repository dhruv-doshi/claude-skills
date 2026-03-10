# Testing Reference

Testing patterns for Python/FastAPI backends. Read when writing tests, setting up fixtures, mocking external services, or configuring CI pipelines.

## Table of Contents

- [Testing Pyramid](#testing-pyramid)
- [Coverage Targets](#coverage-targets)
- [Test Structure](#test-structure)
- [Fixtures](#fixtures)
- [Mocking Patterns](#mocking-patterns)
- [Integration Testing](#integration-testing)
- [Async Testing](#async-testing)
- [Error Case Testing](#error-case-testing)
- [Property-Based Testing](#property-based-testing)
- [Test Data Management](#test-data-management)
- [Running Tests](#running-tests)
- [CI/CD Integration](#cicd-integration)

---

## Testing Pyramid

```
        /\
       /  \      E2E Tests (few)
      /----\     Critical user journeys only
     /      \
    /--------\   Integration Tests (moderate)
   /          \  Real DB, mocked externals
  /------------\
 /              \ Unit Tests (many)
/________________\ Domain logic, pure functions
```

---

## Coverage Targets

| Layer | Target | Rationale |
|-------|--------|-----------|
| Domain logic | 80%+ | Core business rules must be reliable |
| Infrastructure | 60%+ | Integration points need coverage |
| API routes | 50%+ | Thin layer, mostly tested via integration |
| Overall | 60%+ | Balance thoroughness with maintenance cost |

---

## Test Structure

### Arrange-Act-Assert

Every test follows this pattern. If a test doesn't fit cleanly into AAA, it's probably testing too many things.

```python
def test_user_creation_with_valid_email():
    # Arrange
    user_data = {"email": "test@example.com", "name": "Test User"}
    service = UserService(repository=mock_repo)

    # Act
    result = service.create_user(user_data)

    # Assert
    assert result.email == "test@example.com"
    assert result.id is not None
```

### Naming Convention

```
test_<action>_<expected_outcome>[_<condition>]

test_create_user_201                          # Happy path
test_create_user_401_no_auth                  # Auth failure
test_create_user_422_invalid_email            # Validation
test_create_user_409_duplicate_email          # Conflict
test_process_item_retries_on_timeout          # Retry behavior
test_delete_item_403_not_owner                # Authorization
```

### Directory Layout

```
tests/
├── conftest.py              # Shared fixtures (DB, client, mocks)
├── factories/               # factory_boy model factories
├── unit/
│   ├── domain/              # Business logic — fast, no I/O
│   └── core/                # Utility/helper tests
├── integration/
│   ├── api/                 # HTTP route tests via AsyncClient
│   └── infrastructure/      # DB/Redis connection tests
└── e2e/
    └── test_workflows.py    # Full-stack smoke tests
```

---

## Fixtures

### Factory Pattern (Recommended)

Factories produce consistent test data without hitting the database:

```python
# tests/factories.py
import factory
from src.domain.user import User

class UserFactory(factory.Factory):
    class Meta:
        model = User

    id = factory.Sequence(lambda n: f"user_{n}")
    email = factory.LazyAttribute(lambda o: f"{o.id}@example.com")
    name = factory.Faker("name")
    created_at = factory.LazyFunction(datetime.utcnow)

# Usage
def test_user_deactivation():
    user = UserFactory(status="active")
    user.deactivate()
    assert user.status == "inactive"
```

### Database Fixture

```python
@pytest_asyncio.fixture
async def db_session():
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async with AsyncSession(engine) as session:
        yield session
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()
```

### FastAPI Test Client

```python
@pytest_asyncio.fixture
async def client(db_session):
    app.dependency_overrides[get_db_session] = lambda: db_session
    app.dependency_overrides[get_current_user] = lambda: FakeUser(id="test", role="user")
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

### Mock External Services

```python
@pytest.fixture
def mock_llm(monkeypatch):
    """Mock LLM responses for unit tests — works regardless of provider."""
    async def fake_completion(*args, **kwargs):
        return {
            "choices": [{"message": {"content": "Mocked response"}, "finish_reason": "stop"}],
            "usage": {"prompt_tokens": 10, "completion_tokens": 20, "total_tokens": 30},
        }
    monkeypatch.setattr("src.core.llm.llm_client.completion", fake_completion)
```

---

## Mocking Patterns

### Mock at Boundaries

Mock external services at the infrastructure boundary, not deep inside domain logic:

```python
# Good — mock the gateway
@pytest.fixture
def mock_payment_gateway(mocker):
    return mocker.patch(
        "src.infrastructure.payment.StripeGateway.charge",
        return_value=PaymentResult(success=True, transaction_id="txn_123"),
    )

# Bad — mocking internal domain methods couples tests to implementation
```

### Repository Mocking

```python
@pytest.fixture
def mock_repo():
    repo = Mock(spec=UserRepository)
    repo.get_by_id.return_value = UserFactory(id="123")
    repo.save.side_effect = lambda user: user
    return repo

def test_user_update(mock_repo):
    service = UserService(repository=mock_repo)
    result = service.update_user("123", {"name": "New Name"})

    mock_repo.save.assert_called_once()
    assert result.name == "New Name"
```

### Async Mocking

```python
@pytest.fixture
def mock_async_repo(mocker):
    mock = mocker.AsyncMock(spec=UserRepository)
    mock.get_by_id.return_value = UserFactory()
    return mock
```

---

## Integration Testing

### Database Tests

```python
@pytest.mark.integration
async def test_user_persistence(db_session):
    # Create
    user = User(email="test@example.com", name="Test")
    db_session.add(user)
    await db_session.commit()

    # Read
    result = await db_session.get(User, user.id)
    assert result.email == "test@example.com"

    # Verify constraints
    duplicate = User(email="test@example.com", name="Dup")
    db_session.add(duplicate)
    with pytest.raises(IntegrityError):
        await db_session.commit()
```

### API Integration Tests

```python
@pytest.mark.integration
async def test_create_item_201(client: AsyncClient, mock_llm):
    response = await client.post(
        "/api/v1/items",
        json={"name": "Test Item", "type": "standard"},
    )
    assert response.status_code == 201
    body = response.json()
    assert "data" in body
    assert body["data"]["name"] == "Test Item"
    assert "X-Request-ID" in response.headers

async def test_create_item_401_no_auth(db_session):
    """Unauthenticated request returns 401."""
    app.dependency_overrides.clear()
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        response = await ac.post("/api/v1/items", json={"name": "Test"})
    assert response.status_code == 401

async def test_create_item_422_bad_input(client: AsyncClient):
    response = await client.post("/api/v1/items", json={"name": ""})
    assert response.status_code == 422
```

### Contract Testing (External Services)

```python
@pytest.mark.integration
@pytest.mark.vcr  # Record/replay HTTP interactions
def test_stripe_charge():
    gateway = StripeGateway(api_key=TEST_API_KEY)
    result = gateway.charge(amount=1000, currency="usd", source="tok_visa")
    assert result.success
    assert result.transaction_id.startswith("ch_")
```

---

## Async Testing

```python
import pytest

@pytest.mark.asyncio
async def test_async_creation():
    service = AsyncItemService()
    item = await service.create({"name": "Test"})
    assert item.id is not None

@pytest.mark.asyncio
async def test_concurrent_operations():
    results = await asyncio.gather(
        service.get("1"),
        service.get("2"),
        service.get("3"),
    )
    assert len(results) == 3
```

---

## Error Case Testing

```python
def test_not_found_raises_exception():
    repo = Mock(spec=ItemRepository)
    repo.get_by_id.return_value = None
    service = ItemService(repository=repo)

    with pytest.raises(NotFoundError) as exc_info:
        service.get_item("nonexistent")
    assert exc_info.value.error_code == "NOT_FOUND"

def test_validation_error_422(client):
    response = client.post("/api/v1/items", json={"name": ""})
    assert response.status_code == 422
    error = response.json()["error"]
    assert error["code"] == "VALIDATION_ERROR"

async def test_handles_upstream_failure(monkeypatch):
    """Service handles LLM provider failure gracefully (cloud or local)."""
    async def failing(*args, **kwargs):
        raise httpx.HTTPStatusError("502", request=Mock(), response=Mock())
    monkeypatch.setattr("src.core.llm.llm_client.completion", failing)

    service = ItemService(db=db_session)
    with pytest.raises(UpstreamError):
        await service.process("item-1")
```

---

## Property-Based Testing

Use Hypothesis to find edge cases you wouldn't think to write manually:

```python
from hypothesis import given, strategies as st

@given(st.emails())
def test_email_validation_accepts_valid_emails(email):
    user = User(email=email, name="Test")
    assert user.email == email

@given(st.text(min_size=1, max_size=100))
def test_name_accepts_any_non_empty_string(name):
    user = User(email="test@example.com", name=name)
    assert user.name == name

@given(st.integers(min_value=0, max_value=150))
def test_age_within_bounds(age):
    request = CreateUserRequest(email="a@b.com", name="T", age=age)
    assert request.age == age
```

---

## Test Data Management

### Deterministic IDs

```python
@pytest.fixture(autouse=True)
def deterministic_ids(mocker):
    counter = itertools.count(1)
    mocker.patch("uuid.uuid4", side_effect=lambda: f"test-uuid-{next(counter)}")
```

### Freeze Time

```python
from freezegun import freeze_time

@freeze_time("2024-01-15 12:00:00")
def test_expiration():
    item = Item(created_at=datetime.utcnow(), ttl_hours=24)

    with freeze_time("2024-01-16 12:00:01"):
        assert item.is_expired()
```

---

## Running Tests

```bash
# All tests
pytest -v --tb=short

# Unit only (fast)
pytest tests/unit/ -v

# Integration (needs DB + Redis)
pytest tests/integration/ -v

# Single file
pytest tests/unit/domain/test_user_service.py -v

# Single test
pytest tests/unit/domain/test_user_service.py::test_create_user_201 -v

# With coverage
pytest --cov=src --cov-report=term-missing --cov-fail-under=60

# Parallel
pytest tests/unit/ -n auto  # requires pytest-xdist

# Skip slow tests
pytest tests/ -m "not slow"
```

### Test Markers

```python
# conftest.py
def pytest_configure(config):
    config.addinivalue_line("markers", "unit: Unit tests")
    config.addinivalue_line("markers", "integration: Integration tests")
    config.addinivalue_line("markers", "e2e: End-to-end tests")
    config.addinivalue_line("markers", "slow: Slow tests (>1s)")
```

```ini
# pytest.ini
[pytest]
markers =
    unit: Unit tests
    integration: Integration tests
    e2e: End-to-end tests
    slow: Slow tests
testpaths = tests
asyncio_mode = auto
```

---

## CI/CD Integration

```yaml
# GitHub Actions
test:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: pgvector/pgvector:pg16
      env:
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
        POSTGRES_DB: test_db
      ports: ["5432:5432"]
      options: >-
        --health-cmd pg_isready
        --health-interval 10s
        --health-timeout 5s
        --health-retries 5
    redis:
      image: redis:7-alpine
      ports: ["6379:6379"]

  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v5
      with:
        python-version: "3.12"
    - run: pip install -r requirements-test.txt
    - name: Unit tests
      run: pytest tests/unit/ -v --cov=src --cov-report=xml
    - name: Integration tests
      run: pytest tests/integration/ -v
      env:
        DATABASE_URL: postgresql+asyncpg://test:test@localhost:5432/test_db
        REDIS_URL: redis://localhost:6379/0
    - uses: codecov/codecov-action@v4
      with:
        file: coverage.xml
```