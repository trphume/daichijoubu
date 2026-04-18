# Python Patterns and Anti-Patterns

## Design Principles

### KISS — Keep It Simple
Choose the simplest solution that works. Complexity must be justified by a concrete requirement. Prefer a dict lookup over a factory+registry pattern; prefer a plain function over a class hierarchy.

```python
# Over-engineered: factory with a registration decorator
class OutputFormatterFactory:
    _formatters: dict[str, type[Formatter]] = {}
    @classmethod
    def register(cls, name: str): ...
    @classmethod
    def create(cls, name: str) -> Formatter: ...

# Simple: a dict
FORMATTERS: dict[str, type[Formatter]] = {
    "json": JsonFormatter,
    "csv": CsvFormatter,
    "xml": XmlFormatter,
}

def get_formatter(name: str) -> Formatter:
    if name not in FORMATTERS:
        raise ValueError(f"Unknown format: {name}")
    return FORMATTERS[name]()
```

### Single Responsibility
Each class or function should have one reason to change. Split transport/serialization, business rules, and persistence into separate units.

```python
# BAD: handler does everything
class UserHandler:
    async def create_user(self, raw: dict) -> dict:
        if not raw.get("email"):                          # Validation
            return {"error": "email required"}
        row = await db_execute("INSERT INTO users ...")   # Persistence
        return {"id": row["id"], "email": row["email"]}   # Formatting

# GOOD: separated concerns
class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

    async def create_user(self, data: CreateUserInput) -> User:
        return await self._repo.save(User(email=data.email, name=data.name))

class UserHandler:
    def __init__(self, service: UserService) -> None:
        self._service = service

    async def create_user(self, raw: dict) -> dict:
        data = CreateUserInput.from_dict(raw)   # validated at the boundary
        user = await self._service.create_user(data)
        return user.to_dict()
```

### Separation of Concerns (Layered Architecture)
Organize into Transport → Service → Repository layers. Dependencies only point downward. A service must not import from handlers; both share a DTO/types module.

### Composition Over Inheritance
Build behavior by combining objects. Use inheritance only for genuine "is-a" relationships. Pass collaborators through the constructor.

```python
class NotificationService:
    def __init__(
        self,
        email_sender: EmailSender,
        sms_sender: SmsSender | None = None,
        push_sender: PushSender | None = None,
    ) -> None:
        self._email = email_sender
        self._sms = sms_sender
        self._push = push_sender

    async def notify(self, user: User, message: str, channels: set[str] | None = None) -> None:
        channels = channels or {"email"}
        if "email" in channels:
            await self._email.send(user.email, message)
        if "sms" in channels and self._sms and user.phone:
            await self._sms.send(user.phone, message)
```

### Rule of Three
Wait until three instances before abstracting. Duplication is often cheaper than the wrong abstraction. Override this when copies are already diverging incorrectly and causing bugs.

### Function Size
Extract when a function exceeds ~20–50 lines, mixes distinct purposes, or has 3+ levels of nesting. Compose from smaller, focused functions.

### Dependency Injection
Pass collaborators via the constructor; accept `Protocol`s for testability.

```python
from typing import Protocol

class Cache(Protocol):
    async def get(self, key: str) -> str | None: ...
    async def set(self, key: str, value: str, ttl: int) -> None: ...

class UserService:
    def __init__(self, repository: UserRepository, cache: Cache, logger: Logger) -> None:
        self._repo, self._cache, self._logger = repository, cache, logger
```

If the constructor has 7+ parameters, that is a signal the class has too many responsibilities — split it before blaming DI.

## Anti-Patterns Checklist

Before finalizing code, verify none of these are present.

### Architecture

**Exposed internal types — leaking persistence models through the API.** Translate to a plain DTO (dataclass, `TypedDict`, or equivalent) at the boundary.

```python
# BAD — returning the ORM/row object directly
def get_user(id: str) -> UserRow:
    return db_fetch_user(id)

# GOOD — translate at the boundary
def get_user(id: str) -> UserResponse:
    row = db_fetch_user(id)
    return UserResponse(id=row.id, email=row.email, name=row.name)
```

**Mixed I/O and business logic.** Keep domain logic pure; accept data as arguments.

```python
# BAD
def calculate_discount(user_id: str) -> float:
    user = db_query("SELECT * FROM users WHERE id = ?", user_id)
    orders = db_query("SELECT * FROM orders WHERE user_id = ?", user_id)
    return 0.15 if len(orders) > 10 else 0.0

# GOOD — pure function, easily testable
def calculate_discount(user: User, orders: list[Order]) -> float:
    return 0.15 if len(orders) > 10 else 0.0
```

### Error Handling

**Bare `except:` or `except Exception: pass` — swallows all errors silently.** Catch specific exceptions and either log-and-reraise or translate them at a boundary. Never catch `BaseException` (covers `KeyboardInterrupt`, `SystemExit`).

**Ignored partial failures in batches.**
```python
@dataclass
class BatchResult:
    succeeded: dict[int, Any]
    failed: dict[int, Exception]

def process_batch(items: list[Item]) -> BatchResult:
    succeeded, failed = {}, {}
    for idx, item in enumerate(items):
        try:
            succeeded[idx] = process(item)
        except Exception as e:
            failed[idx] = e
    return BatchResult(succeeded, failed)
```

**Missing input validation.** Validate at boundaries — never unpack a raw `dict` directly into a domain model. Convert to a typed input object (dataclass with a `from_dict` classmethod, `TypedDict` + runtime check, or equivalent) first.

**Re-raising without `raise from`.** When translating an exception, preserve the cause:

```python
try:
    parse(value)
except ValueError as e:
    raise BadInputError("invalid value") from e
```

**Using exceptions for control flow.** Exceptions are for exceptional conditions. Do not use `try/except` to replace a simple `if` check.

### Resources

**Unclosed resources.** Always use context managers for files, sockets, DB connections, locks.

```python
# BAD
f = open(path); return f.read()
# GOOD
with open(path) as f:
    return f.read()
```

**Blocking calls in async code.** `time.sleep`, synchronous sockets, and synchronous file-I/O block the event loop. Inside `async def`, use `asyncio.sleep` and async-native APIs. If blocking code is unavoidable, run it via `asyncio.to_thread`.

**Mutable default arguments.** `def f(x=[])` shares the same list across every call.

```python
# BAD
def add_item(item: Item, bag: list[Item] = []) -> list[Item]:
    bag.append(item)
    return bag

# GOOD
def add_item(item: Item, bag: list[Item] | None = None) -> list[Item]:
    bag = bag if bag is not None else []
    bag.append(item)
    return bag
```

### Configuration

**Hard-coded configuration or secrets.** Load from environment variables (`os.environ`) or configuration files at the edge; keep the rest of the code free of direct environment access. The specific config-parsing library belongs in its own skill.

**Scattered timeout/retry logic duplicated across call sites.** Centralize in a helper or wrapper object so retry semantics are defined once.

**Double retry.** If a lower layer already retries, do not add application-level retry on top — you will multiply attempts and blow past backoff budgets.

### Type Safety

**Missing or partial type hints.** Annotate every public function parameter, return type, and class attribute. Untyped collections (`list`, `dict`) must be parameterized (`list[User]`, `dict[str, int]`).

## Summary Table

| Anti-Pattern                            | Fix                                                 |
|-----------------------------------------|-----------------------------------------------------|
| Scattered retry/timeout                 | Centralize in a helper or wrapper                   |
| Hard-coded config                       | Load from environment / config file at the edge     |
| Exposed persistence models at API boundary | DTO / response schemas                           |
| Mixed I/O + business logic              | Repository pattern; pure domain functions           |
| `except Exception: pass`                | Catch specific exceptions; log or re-raise          |
| Re-raise without `raise from`           | `raise NewError(...) from e`                        |
| Batch aborts on first error             | Return a result with successes and failures         |
| No input validation                     | Validate at the boundary                            |
| Unclosed resources                      | Context managers                                    |
| Blocking in async                       | Async-native APIs or `asyncio.to_thread`            |
| Mutable default arguments               | Default to `None`, build inside the body            |
| Missing types / untyped collections     | Annotate public APIs; parameterize collections      |
| Only happy-path tests                   | Test error paths and edge cases                     |
