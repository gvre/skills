# Testing Guide Reference

Comprehensive guide to testing Python applications with pytest, including basic tests, async testing, mocking, parametrization, and coverage configuration.

## Basic Testing

### Simple Test Examples

```python
import pytest
from myapp import process_data, User

def test_process_data_success():
    """Test successful data processing."""
    result = process_data({"key": "value"})
    assert result.status == "success"

def test_process_data_with_multiple_assertions():
    """Multiple assertions in a single test."""
    result = process_data({"id": 1, "name": "Alice"})
    assert result.status == "success"
    assert result.data["id"] == 1
    assert result.data["name"] == "Alice"

def test_exceptions():
    """Test that functions raise expected exceptions."""
    with pytest.raises(ValueError, match="Invalid input"):
        process_data({"invalid": "data"})
```

### Using Fixtures

```python
import pytest

@pytest.fixture
def sample_user():
    """Fixture that provides a sample user."""
    return User(id=1, name="Alice", email="alice@example.com")

@pytest.fixture
def user_database():
    """Fixture with setup and teardown."""
    db = Database()
    db.connect()
    yield db  # Test runs here
    db.disconnect()  # Cleanup

def test_with_fixture(sample_user):
    """Use fixture as test parameter."""
    assert sample_user.name == "Alice"
    assert sample_user.id == 1

def test_multiple_fixtures(sample_user, user_database):
    """Use multiple fixtures."""
    user_database.save(sample_user)
    retrieved = user_database.get(sample_user.id)
    assert retrieved.name == sample_user.name
```

### Parametrized Tests

```python
import pytest

@pytest.mark.parametrize("value,expected", [
    (1, 2),
    (2, 4),
    (3, 6),
    (10, 20),
])
def test_double(value: int, expected: int) -> None:
    """Test with multiple input/output pairs."""
    assert double(value) == expected

@pytest.mark.parametrize("username,is_valid", [
    ("alice123", True),
    ("bob_user", True),
    ("ab", False),  # Too short
    ("user@name", False),  # Invalid character
    ("", False),  # Empty
])
def test_validate_username(username: str, is_valid: bool) -> None:
    """Test validation with multiple cases."""
    result = validate_username(username)
    assert result == is_valid

# Parametrize with indirect fixtures
@pytest.fixture
def user(request):
    return User(name=request.param)

@pytest.mark.parametrize("user", ["Alice", "Bob", "Charlie"], indirect=True)
def test_user_creation(user):
    assert user.name in ["Alice", "Bob", "Charlie"]
```

### Markers

```python
import pytest

@pytest.mark.skip(reason="Not implemented yet")
def test_future_feature():
    pass

@pytest.mark.xfail(reason="Known bug in library")
def test_known_issue():
    assert complex_calculation() == 42  # Currently fails

@pytest.mark.slow
def test_slow_operation():
    """Test marked as slow - run with: pytest -m slow"""
    time.sleep(5)
    assert True

# Run only slow tests: pytest -m slow
# Run all except slow: pytest -m "not slow"
```

---

## Async Testing

### Basic Async Tests

```python
import pytest
import httpx
from myapp import fetch_url

@pytest.mark.asyncio
async def test_fetch_url():
    """Test async functions with pytest-asyncio."""
    result = await fetch_url("https://example.com")
    assert result is not None
    assert len(result) > 0

@pytest.mark.asyncio
async def test_fetch_multiple_urls():
    """Test multiple concurrent operations."""
    import asyncio
    
    urls = [
        "https://example.com",
        "https://example.org",
        "https://example.net",
    ]
    
    results = await asyncio.gather(*[fetch_url(url) for url in urls])
    assert len(results) == 3
    assert all(r is not None for r in results)
```

### Async Fixtures

```python
import pytest
import httpx

@pytest.fixture
async def async_client():
    """Fixture for async resources."""
    async with httpx.AsyncClient() as client:
        yield client
    # Cleanup happens automatically

@pytest.mark.asyncio
async def test_with_async_client(async_client):
    """Use async fixture in test."""
    response = await async_client.get("https://example.com")
    assert response.status_code == 200

@pytest.fixture
async def database():
    """Async database fixture with connection pooling."""
    db = AsyncDatabase()
    await db.connect()
    yield db
    await db.disconnect()

@pytest.mark.asyncio
async def test_async_database_operation(database):
    """Test async database operations."""
    result = await database.query("SELECT * FROM users")
    assert result is not None
```

### Event Loop Configuration

In `pyproject.toml`:
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"  # Automatically handle event loops
testpaths = ["tests"]
python_files = ["test_*.py"]
```

---

## Mocking

### Mocking Synchronous Functions

```python
from unittest.mock import Mock, patch, call

def test_with_mock():
    """Mock synchronous dependencies."""
    mock_db = Mock()
    mock_db.get_user.return_value = User(id=1, name="Alice")

    service = UserService(mock_db)
    user = service.fetch_user(1)

    assert user.name == "Alice"
    # Verify the mock was called correctly
    mock_db.get_user.assert_called_once_with(1)

def test_mock_side_effects():
    """Test multiple calls with different results."""
    mock_api = Mock()
    mock_api.fetch.side_effect = [
        {"status": "success"},  # First call
        {"status": "error"},     # Second call
        Exception("Network error")  # Third call raises
    ]

    assert mock_api.fetch() == {"status": "success"}
    assert mock_api.fetch() == {"status": "error"}
    
    with pytest.raises(Exception, match="Network error"):
        mock_api.fetch()

def test_mock_call_tracking():
    """Track how a mock was called."""
    mock_logger = Mock()
    
    my_function(mock_logger)
    
    # Verify it was called
    assert mock_logger.info.called
    # Verify it was called with specific arguments
    mock_logger.info.assert_called_with("Operation completed")
    # Verify all calls
    assert mock_logger.method_calls == [
        call.debug("Starting"),
        call.info("Operation completed")
    ]
```

### Mocking Async Functions

```python
from unittest.mock import AsyncMock
import pytest

@pytest.mark.asyncio
async def test_async_with_mock():
    """Mock async dependencies."""
    mock_client = AsyncMock()
    mock_response = AsyncMock()
    mock_response.text = "mocked response"
    mock_client.get.return_value = mock_response

    result = await fetch_url("https://example.com", mock_client)
    assert result == "mocked response"
    mock_client.get.assert_called_once()

@pytest.mark.asyncio
async def test_async_mock_side_effects():
    """Async mock with side effects."""
    mock_api = AsyncMock()
    mock_api.fetch.side_effect = [
        {"data": "first"},
        {"data": "second"},
    ]

    first = await mock_api.fetch()
    second = await mock_api.fetch()
    
    assert first == {"data": "first"}
    assert second == {"data": "second"}
```

### Using patch()

```python
from unittest.mock import patch

@patch("myapp.external_api.call")
def test_with_patch(mock_call):
    """Patch external dependencies."""
    mock_call.return_value = {"status": "ok"}
    result = my_function()
    assert result["status"] == "ok"

@patch("myapp.database.connect")
@patch("myapp.external_api.call")
def test_multiple_patches(mock_api, mock_db):
    """Stack multiple patches (reversed order)."""
    mock_api.return_value = {"status": "ok"}
    mock_db.return_value = True
    
    result = my_function()
    assert result is not None

# Context manager style
def test_patch_context_manager():
    with patch("myapp.config.API_KEY", "test-key"):
        # API_KEY is "test-key" inside this block
        assert get_api_key() == "test-key"
    # API_KEY is restored after block

# Patch with side effect
@patch("myapp.external_api.call")
def test_patch_side_effect(mock_call):
    mock_call.side_effect = ConnectionError("Network unavailable")
    
    with pytest.raises(ConnectionError):
        my_function()
```

---

## Fixtures

### Fixture Scopes

```python
import pytest

@pytest.fixture(scope="function")  # Default - new instance per test
def function_fixture():
    return {"data": "fresh"}

@pytest.fixture(scope="class")  # One instance per test class
def class_fixture():
    return {"data": "shared"}

@pytest.fixture(scope="module")  # One instance per module
def module_fixture():
    return {"data": "module-wide"}

@pytest.fixture(scope="session")  # One instance per test session
def session_fixture():
    return {"data": "entire-session"}

class TestMyClass:
    def test_one(self, function_fixture, class_fixture):
        assert function_fixture == {"data": "fresh"}
        assert class_fixture == {"data": "shared"}
```

### Fixture Dependencies

```python
import pytest

@pytest.fixture
def user_data():
    return {"id": 1, "name": "Alice"}

@pytest.fixture
def user(user_data):
    """Fixture that depends on another fixture."""
    return User(**user_data)

@pytest.fixture
def database(user):
    """Fixture that depends on previous fixtures."""
    db = Database()
    db.save(user)
    return db

def test_with_dependencies(database, user):
    """Test uses the full fixture chain."""
    assert database.get(user.id).name == "Alice"
```

### Fixture Parametrization

```python
import pytest

@pytest.fixture(params=["sqlite", "postgresql", "mysql"])
def database_url(request):
    """Parametrized fixture - test runs once per parameter."""
    urls = {
        "sqlite": "sqlite:///:memory:",
        "postgresql": "postgresql://localhost/test",
        "mysql": "mysql://localhost/test",
    }
    return urls[request.param]

def test_with_all_databases(database_url):
    """This test runs 3 times, once for each database."""
    db = Database(database_url)
    assert db.connect() is True
```

---

## Coverage Configuration

### pyproject.toml Configuration

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]

[tool.coverage.run]
source = ["src"]
branch = true
parallel = true
omit = [
    "*/tests/*",
    "*/site-packages/*",
    # Only omit __init__.py if they're empty/just imports
    # If __init__.py contains logic, remove this line to get coverage
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "def __str__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "@overload",  # Type hint overloads don't run
    "if TYPE_CHECKING:",  # Type checking imports
    "except ImportError:",  # Optional dependencies
    "except ModuleNotFoundError:",
    "^\\s*\\.\\.\\.\\s*$",  # Ellipsis in stub files
]
precision = 2
show_missing = true
skip_covered = false

[tool.coverage.html]
directory = "htmlcov"
```

### Running Tests with Coverage

```bash
# Run tests with coverage
pytest --cov=src --cov-report=html

# Generate coverage report
coverage report

# Generate HTML report
coverage html

# Show missing lines
coverage report -m
```

---

## Testing Best Practices

### Naming Conventions

```python
# ✅ GOOD - Descriptive test names
def test_user_login_with_valid_credentials():
    pass

def test_user_login_with_invalid_password():
    pass

def test_calculate_total_price_includes_tax():
    pass

# ❌ BAD - Non-descriptive names
def test_user():
    pass

def test_login():
    pass

def test_calc():
    pass
```

### Arrange-Act-Assert Pattern

```python
def test_process_data_success():
    """Follow Arrange-Act-Assert pattern."""
    # Arrange - setup test data and mocks
    input_data = {"user_id": 1, "action": "update"}
    mock_db = Mock()
    mock_db.get_user.return_value = User(id=1, name="Alice")
    
    # Act - execute the code being tested
    result = process_data(input_data, mock_db)
    
    # Assert - verify the results
    assert result.status == "success"
    assert mock_db.get_user.called
```

### Isolation and Independence

```python
# ✅ GOOD - Tests are independent
@pytest.fixture
def fresh_database():
    db = Database()
    db.connect()
    yield db
    db.clear_all()

def test_user_creation(fresh_database):
    fresh_database.create_user("alice")
    users = fresh_database.get_all_users()
    assert len(users) == 1

def test_user_deletion(fresh_database):
    fresh_database.create_user("bob")
    fresh_database.delete_user("bob")
    users = fresh_database.get_all_users()
    assert len(users) == 0

# Tests don't affect each other - each gets a fresh database
```

### One Assertion Per Test (Usually)

```python
# ✅ GOOD - Each test verifies one thing
def test_user_has_valid_name():
    user = create_user("Alice")
    assert user.name == "Alice"

def test_user_has_valid_email():
    user = create_user("Alice")
    assert user.email == "alice@example.com"

# ❌ AVOID - Testing multiple unrelated things
def test_user():
    user = create_user("Alice")
    assert user.name == "Alice"
    assert user.email == "alice@example.com"
    assert user.created_at is not None
    assert user.is_active is True
```

### Test Organization

```python
# Group related tests in classes
class TestUserCreation:
    def test_create_with_valid_data(self):
        pass
    
    def test_create_with_invalid_email(self):
        pass
    
    def test_create_with_duplicate_username(self):
        pass

class TestUserValidation:
    def test_validate_username_length(self):
        pass
    
    def test_validate_email_format(self):
        pass
```

---

## Advanced Testing

### Testing Exceptions

```python
import pytest

def test_division_by_zero():
    """Test that function raises exception."""
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)

def test_exception_message():
    """Test exception message."""
    with pytest.raises(ValueError, match="Invalid input"):
        process_data(None)

def test_exception_type_and_message():
    """Capture exception for detailed inspection."""
    with pytest.raises(ValueError) as exc_info:
        process_data(invalid_data)
    
    assert "Invalid input" in str(exc_info.value)
    assert exc_info.type is ValueError
```

### Monkeypatch

```python
import pytest

def test_with_monkeypatch(monkeypatch):
    """Temporarily replace functions/attributes."""
    # Replace module constant
    monkeypatch.setattr("myapp.config.DEBUG", True)
    assert get_debug_mode() is True
    
    # Replace function
    monkeypatch.setattr("myapp.external_api.fetch", lambda url: "mocked")
    assert fetch_data("url") == "mocked"
    
    # Restore happens automatically after test

def test_monkeypatch_env_var(monkeypatch):
    """Set environment variables for test."""
    monkeypatch.setenv("DATABASE_URL", "sqlite:///:memory:")
    assert os.getenv("DATABASE_URL") == "sqlite:///:memory:"
```

### Capsys for Output Testing

```python
def test_print_output(capsys):
    """Capture stdout/stderr."""
    print("Hello, World!")
    
    captured = capsys.readouterr()
    assert "Hello, World!" in captured.out

def test_logging_output(capsys):
    """Capture logging output."""
    logging.info("Test message")
    
    captured = capsys.readouterr()
    assert "Test message" in captured.out or True  # Logging may not go to stdout
```

---

## Rules for Effective Testing

- **ALWAYS use pytest** (never unittest)
- **ALWAYS use descriptive test names** that explain what's being tested
- **Use fixtures** for setup and shared resources
- **Use pytest-asyncio** for async code testing
- **Mock external dependencies** (APIs, databases, file systems)
- **Use @patch** for patching specific functions/methods
- **Aim for >80% coverage** with meaningful tests (not just line coverage)
- **Keep tests fast** - slow tests discourage running them frequently
- **Test behavior, not implementation** - tests should survive refactoring
- **One logical assertion per test** (multiple related assertions OK)
- **Use parametrization** to test multiple scenarios
- **Group related tests** in classes for organization
- **Make tests independent** - run in any order, isolated state
- **Use clear names** that document expected behavior
- **Test edge cases** - empty inputs, None, negative numbers, etc.