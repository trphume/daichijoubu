# Python Testing

This file covers general testing guidance using the standard library (`unittest`, `unittest.mock`). Specific third-party test frameworks (e.g. pytest), fixtures systems, property-based testing libraries, and coverage tools belong in their own skills.

## Layout
- Tests live in a top-level `tests/` directory (or alongside source in `src/` layouts), separated by type:
  ```
  tests/
    __init__.py
    unit/
    integration/
    e2e/
  ```
- Test modules are named `test_*.py`. Test functions and methods are named `test_*`.

## AAA Pattern
Each test has three visually distinct sections:

- **Arrange** — set up inputs, fakes, and preconditions.
- **Act** — call the code under test exactly once.
- **Assert** — verify the observable outcome.

```python
import unittest

class TestCalculator(unittest.TestCase):
    def test_divide_returns_quotient(self):
        # Arrange
        calc = Calculator()

        # Act
        result = calc.divide(6, 2)

        # Assert
        self.assertEqual(result, 3)
```

## Test Naming
Use `test_<unit>_<scenario>_<expected_outcome>`. Names must describe the input and expected behavior — not `test_1`, not `test_user`.

```python
def test_create_user_with_valid_data_returns_user(self): ...
def test_create_user_with_duplicate_email_raises_conflict(self): ...
def test_get_user_with_unknown_id_returns_none(self): ...
```

## One Behavior Per Test
Each test verifies exactly one behavior. Do not chain create → update → delete assertions in a single test — split them. When a test fails, the name should tell you which behavior is broken.

## Always Test Error Paths
For every function that can fail, write a test that exercises the failure. Check the error type and message, not just that an exception was raised.

```python
def test_divide_by_zero_raises(self):
    with self.assertRaises(ZeroDivisionError) as cm:
        divide(5, 0)
    self.assertIn("Division by zero", str(cm.exception))
```

Do not use substring matching on error messages as the *only* assertion — prefer asserting on an explicit error type or sentinel instance.

## Setup and Teardown
Use `setUp` / `tearDown` for per-test preparation, and `setUpClass` / `tearDownClass` for expensive one-per-class setup. Avoid sharing mutable state across tests.

```python
class TestDatabase(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.db = Database("sqlite:///:memory:")
        cls.db.connect()

    @classmethod
    def tearDownClass(cls):
        cls.db.disconnect()

    def setUp(self):
        self.db.truncate_all()   # reset per-test state

    def test_query_returns_rows(self):
        self.db.insert(User(id=1, name="Alice"))
        self.assertEqual(len(self.db.query("SELECT * FROM users")), 1)
```

Use `self.addCleanup(fn, *args)` to register teardown from inside a helper — it runs even if the test raises, and it is clearer than `try/finally`.

## Parameterizing Tests
The stdlib has no built-in parameterization. Use `subTest` to run the same assertions over multiple inputs; each failing case is reported independently.

```python
class TestEmailValidation(unittest.TestCase):
    def test_valid_and_invalid_emails(self):
        cases = [
            ("user@example.com", True),
            ("test.user@domain.co.uk", True),
            ("invalid.email", False),
            ("@example.com", False),
            ("", False),
        ]
        for email, expected in cases:
            with self.subTest(email=email):
                self.assertEqual(is_valid_email(email), expected)
```

## Mocking at Boundaries
- Mock at the boundary (HTTP clients, filesystem, clock) — not internal collaborators.
- Prefer dependency injection + hand-written fakes over `unittest.mock.patch` when possible. Fakes behave; mocks only remember.
- When patching, patch the symbol **where it is used**, not where it is defined.

```python
from unittest.mock import patch, Mock

class TestAPIClient(unittest.TestCase):
    def test_get_user_parses_json_response(self):
        client = APIClient("https://api.example.com")
        response = Mock()
        response.json.return_value = {"id": 1, "name": "John Doe"}
        response.raise_for_status.return_value = None

        with patch("mymodule.api_client.http_get", return_value=response) as http_get:
            user = client.get_user(1)

        self.assertEqual(user["id"], 1)
        http_get.assert_called_once_with("https://api.example.com/users/1")
```

**Avoid over-mocking.** If a test mocks every collaborator, it verifies no real behavior. For critical paths, use an integration test.

## Testing Retries and Sequences
Use `side_effect` with an iterable to simulate a sequence of failures followed by success.

```python
def test_retries_on_transient_error(self):
    client = Mock()
    client.request.side_effect = [
        ConnectionError("Failed"),
        ConnectionError("Failed"),
        {"status": "ok"},
    ]
    service = ServiceWithRetry(client, max_retries=3)
    self.assertEqual(service.fetch(), {"status": "ok"})
    self.assertEqual(client.request.call_count, 3)
```

## Time-Dependent Tests
Inject a clock (`Callable[[], datetime]`) rather than calling `datetime.now()` directly. In production, pass `datetime.now`; in tests, pass a fake clock that returns a fixed or advancing value.

```python
class Token:
    def __init__(self, expires_at: datetime, now: Callable[[], datetime]) -> None:
        self._expires_at = expires_at
        self._now = now

    def is_expired(self) -> bool:
        return self._now() >= self._expires_at

def test_token_is_expired_after_expiry():
    fixed = datetime(2026, 1, 15, 12, 0, 0)
    token = Token(expires_at=datetime(2026, 1, 15, 11, 30, 0), now=lambda: fixed)
    assert token.is_expired()
```

Third-party time-freezing libraries belong in their own skill.

## Async Tests
Use `unittest.IsolatedAsyncioTestCase` for async code — it manages the event loop per test.

```python
class TestAsyncClient(unittest.IsolatedAsyncioTestCase):
    async def test_fetch_user(self):
        user = await client.fetch_user("u-1")
        self.assertEqual(user.id, "u-1")
```

## Running Tests
```bash
python -m unittest discover -s tests
python -m unittest tests.unit.test_user
python -m unittest -v tests.unit.test_user.TestUser.test_create_user
```

## What Not to Test
- Auto-generated code. Test the code that uses it.
- The standard library. Test your usage of it.
- Private attributes or internal state. Test observable behavior through the public API.
- Implementation details that will change without changing behavior (exact call ordering of internal helpers, intermediate data shapes).
