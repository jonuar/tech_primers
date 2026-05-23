# Pytest Cheatsheet

## Mental Model

Pytest is a testing framework built around three ideas: **collect** (find test files/functions automatically), **execute** (run them with detailed output), and **report** (show exactly what failed and why). Tests are plain functions — no classes required. Fixtures handle setup/teardown and dependency injection. The goal is tests that are readable, isolated, and fast.

---

## Install & Minimal Setup

```bash
pip install pytest pytest-cov

# Run all tests
pytest

# Run with verbosity
pytest -v

# Run a specific file or function
pytest tests/test_api.py
pytest tests/test_api.py::test_create_user

# Stop after first failure
pytest -x

# Show print() output (stdout)
pytest -s
```

**File naming convention:** pytest auto-discovers files matching `test_*.py` or `*_test.py` and functions starting with `test_`.

```
project/
├── src/
│   └── myapp/
│       └── services.py
└── tests/
    ├── conftest.py       ← shared fixtures
    ├── test_services.py
    └── test_api.py
```

---

## Core Concepts

### 1. Basic Test

```python
# tests/test_math.py
def test_addition():
    assert 1 + 1 == 2

def test_string_upper():
    assert "hello".upper() == "HELLO"

# Test that an exception is raised
def test_division_by_zero():
    with pytest.raises(ZeroDivisionError):
        1 / 0

# Check exception message
def test_value_error_message():
    with pytest.raises(ValueError, match="must be positive"):
        validate(-1)
```

### 2. Fixtures
Reusable setup code injected by name into test functions.

```python
import pytest

@pytest.fixture
def sample_user():
    # Setup
    user = {"id": 1, "name": "Joshua", "role": "admin"}
    yield user
    # Teardown (optional — runs after the test)
    print("Cleaning up user")

def test_user_has_name(sample_user):
    assert sample_user["name"] == "Joshua"

def test_user_is_admin(sample_user):
    assert sample_user["role"] == "admin"
```

### 3. Fixture Scopes
Control how often a fixture is created and destroyed.

```python
@pytest.fixture(scope="function")   # default — new instance per test
@pytest.fixture(scope="class")      # shared across tests in a class
@pytest.fixture(scope="module")     # shared across the whole test file
@pytest.fixture(scope="session")    # shared across the entire test run
```

Use `session` or `module` scope for expensive resources (DB connections, ML models).

### 4. `conftest.py`
Fixtures defined here are available to all tests in the same directory and below — no import needed.

```python
# tests/conftest.py
import pytest
from myapp import create_app

@pytest.fixture(scope="session")
def app():
    app = create_app(config="testing")
    yield app

@pytest.fixture(scope="function")
def client(app):
    return app.test_client()
```

### 5. Parametrize
Run the same test with multiple inputs.

```python
import pytest

@pytest.mark.parametrize("input,expected", [
    ("hello", "HELLO"),
    ("world", "WORLD"),
    ("", ""),
])
def test_upper(input, expected):
    assert input.upper() == expected

# Parametrize with IDs for readable output
@pytest.mark.parametrize("n,result", [
    (0, True),
    (2, True),
    (3, False),
], ids=["zero", "even", "odd"])
def test_is_even(n, result):
    assert (n % 2 == 0) == result
```

### 6. Marks
Tag tests for selective execution.

```python
import pytest

@pytest.mark.slow
def test_heavy_computation():
    ...

@pytest.mark.skip(reason="not implemented yet")
def test_future_feature():
    ...

@pytest.mark.skipif(sys.platform == "win32", reason="Linux only")
def test_unix_socket():
    ...
```

```bash
# Run only marked tests
pytest -m slow

# Exclude marked tests
pytest -m "not slow"
```

Register custom marks in `pytest.ini` to avoid warnings:
```ini
[pytest]
markers =
    slow: marks tests as slow
    integration: marks integration tests
```

### 7. Mocking — `monkeypatch` (built-in)
Patch attributes, environment variables, or functions temporarily.

```python
def test_env_variable(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-key-123")
    assert os.environ["API_KEY"] == "test-key-123"

def test_patch_function(monkeypatch):
    monkeypatch.setattr("myapp.services.external_api_call", lambda: {"status": "ok"})
    result = process_data()
    assert result["status"] == "ok"
```

### 8. Mocking — `unittest.mock` (for complex mocking)

```python
from unittest.mock import MagicMock, patch

# Patch as decorator
@patch("myapp.services.requests.get")
def test_http_call(mock_get):
    mock_get.return_value.json.return_value = {"users": []}
    mock_get.return_value.status_code = 200

    result = fetch_users()
    assert result == []
    mock_get.assert_called_once_with("https://api.example.com/users")

# Patch as context manager
def test_s3_upload():
    with patch("boto3.client") as mock_boto:
        mock_s3 = MagicMock()
        mock_boto.return_value = mock_s3

        upload_file("test.txt")

        mock_s3.upload_file.assert_called_once()
```

### 9. FastAPI / HTTP Testing

```python
from fastapi.testclient import TestClient
from myapp.main import app

client = TestClient(app)

def test_health_check():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}

def test_create_user():
    payload = {"name": "Joshua", "email": "j@example.com"}
    response = client.post("/users", json=payload)
    assert response.status_code == 201
    assert response.json()["name"] == "Joshua"
```

---

## Most-Used Patterns

### Fixture with Dependency Injection

```python
@pytest.fixture
def db_session():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    session = Session(engine)
    yield session
    session.close()
    Base.metadata.drop_all(engine)

@pytest.fixture
def user_repo(db_session):
    return UserRepository(db_session)  # fixtures can depend on other fixtures

def test_create_user(user_repo):
    user = user_repo.create(name="Joshua")
    assert user.id is not None
```

### Approximate Numeric Assertions

```python
# Use pytest.approx for floats
assert 0.1 + 0.2 == pytest.approx(0.3)
assert result == pytest.approx(expected, rel=1e-3)  # 0.1% tolerance
```

### Coverage Report

```bash
pytest --cov=myapp --cov-report=term-missing
pytest --cov=myapp --cov-report=html  # generates htmlcov/index.html
```

---

## Gotchas

- **Fixture not found** — make sure it's in `conftest.py` or imported in the test file. Pytest injects by name, not import.
- **Scope mismatch** — a `function`-scoped fixture cannot depend on a `session`-scoped fixture if the session fixture has side effects. Generally, narrower scopes depend on wider ones.
- **Mocking the wrong location** — patch where the name is *used*, not where it's defined. If `myapp/services.py` does `import requests`, patch `myapp.services.requests`, not `requests.get`.
- **Parametrize with mutable defaults** — don't use lists or dicts as default param values in parametrize; they're shared across test runs.
- **Forgetting `yield` in fixtures with teardown** — if you use `return` instead of `yield`, the teardown code after it never runs.

---

## Quick Links

- [Pytest Docs](https://docs.pytest.org)
- [pytest-cov](https://pytest-cov.readthedocs.io)
- [unittest.mock](https://docs.python.org/3/library/unittest.mock.html)
- [pytest-asyncio](https://pytest-asyncio.readthedocs.io) — for testing async FastAPI endpoints
- [Factory Boy](https://factoryboy.readthedocs.io) — fixture factories for complex models
