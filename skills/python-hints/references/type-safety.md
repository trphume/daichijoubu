# Python Type Safety

Python is dynamically typed; type annotations move errors from runtime to static-analysis time when a type checker runs them. Treat types as enforceable documentation. The specific type-checker choice and its configuration belong in a separate skill; this file covers language-level typing guidance only.

## Annotate All Public Signatures

Every public function, method, and class attribute should be annotated.

```python
def get_user(user_id: str) -> User: ...

def process_batch(
    items: list[Item],
    max_workers: int = 4,
) -> BatchResult[ProcessedItem]: ...

class UserRepository:
    def __init__(self, db: Database) -> None:
        self._db = db

    async def find_by_id(self, user_id: str) -> User | None: ...
    async def save(self, user: User) -> User: ...
```

## Modern Union Syntax (3.10+)

Use `T | None` instead of `Optional[T]`, and `A | B` instead of `Union[A, B]`.

```python
# Preferred
def find_user(user_id: str) -> User | None: ...
def parse_value(v: str) -> int | float | str: ...

# Older (only when supporting 3.9)
from typing import Optional
def find_user(user_id: str) -> Optional[User]: ...
```

## Parameterize Collections

Never use bare `list`, `dict`, `set`, or `tuple` in annotations — always parameterize.

```python
# BAD
def get_users() -> list: ...

# GOOD
def get_users() -> list[User]: ...
def count_by_tag() -> dict[str, int]: ...
def all_ids() -> tuple[str, ...]: ...
```

## Type Narrowing

Use guards (`is None`, `isinstance`, `assert`) so the type checker narrows the type inside the block.

```python
def process_user(user_id: str) -> UserData:
    user = find_user(user_id)
    if user is None:
        raise UserNotFoundError(user_id)
    # user is User here, not User | None
    return UserData(name=user.name, email=user.email)

def only_valid(items: list[Item | None]) -> list[Item]:
    return [item for item in items if item is not None]
```

For exhaustive checks on `Literal` / `Enum` unions, end with `typing.assert_never(x)` so the type checker fails when a new variant is added without handling:

```python
from typing import assert_never

def describe(event: Created | Updated | Deleted) -> str:
    match event:
        case Created():
            return "created"
        case Updated():
            return "updated"
        case Deleted():
            return "deleted"
        case _:
            assert_never(event)
```

## Generics

Use `TypeVar` and `Generic` for reusable containers that preserve the inner type.

```python
from typing import TypeVar, Generic

T = TypeVar("T")
E = TypeVar("E", bound=Exception)

class Result(Generic[T, E]):
    def __init__(self, value: T | None = None, error: E | None = None) -> None:
        if (value is None) == (error is None):
            raise ValueError("Exactly one of value or error must be set")
        self._value, self._error = value, error

    def unwrap(self) -> T:
        if self._error is not None:
            raise self._error
        assert self._value is not None
        return self._value
```

### TypeVar with Bounds and Constraints

Restrict a generic to a base class (bound) or a fixed set of types (constraints).

```python
from typing import TypeVar

# Bound — T must be a subclass of Comparable
ComparableT = TypeVar("ComparableT", bound="Comparable")

def largest(xs: list[ComparableT]) -> ComparableT:
    return max(xs)

# Constraints — T must be exactly one of these
StrOrBytes = TypeVar("StrOrBytes", str, bytes)

def first_char(s: StrOrBytes) -> StrOrBytes:
    return s[:1]
```

## Protocols (Structural Typing)

Define interfaces by *shape*, not inheritance. Any class with matching methods satisfies the protocol. Use `@runtime_checkable` when `isinstance` checks are required.

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Serializable(Protocol):
    def to_dict(self) -> dict: ...

class User:  # does NOT inherit Serializable
    def __init__(self, id: str, name: str) -> None:
        self.id, self.name = id, name
    def to_dict(self) -> dict:
        return {"id": self.id, "name": self.name}

def serialize(obj: Serializable) -> str:
    import json
    return json.dumps(obj.to_dict())

serialize(User("1", "Alice"))  # works — User structurally matches
```

Common reusable protocols:

```python
class Closeable(Protocol):
    def close(self) -> None: ...

class AsyncCloseable(Protocol):
    async def close(self) -> None: ...

class HasId(Protocol):
    @property
    def id(self) -> str: ...
```

## Type Aliases

Name complex types. PEP 695 `type` statement requires Python 3.12+; on 3.10/3.11 use `TypeAlias`.

```python
# Python 3.12+
type UserId = str
type Handler[T] = Callable[[Request], T]

# Python 3.10/3.11
from typing import TypeAlias
from collections.abc import Callable

UserId: TypeAlias = str
Handler: TypeAlias = Callable[[Request], Response]
```

## Callable Types

Use `collections.abc.Callable` (not `typing.Callable`) and `collections.abc.Awaitable` for async callbacks. Use a `Protocol` with `__call__` when keyword-only parameters matter.

```python
from collections.abc import Callable, Awaitable

ProgressCallback = Callable[[int, int], None]
AsyncHandler = Callable[[Request], Awaitable[Response]]

class OnProgress(Protocol):
    def __call__(self, current: int, total: int, *, message: str = "") -> None: ...
```

## TypedDict and NamedTuple

For lightweight records, prefer a `@dataclass`. Use `TypedDict` only when interoperating with existing dict-shaped data (JSON parsing, external APIs). Use `NamedTuple` for small, immutable, positional records.

```python
from typing import TypedDict, NamedTuple

class UserPayload(TypedDict):
    id: str
    email: str
    name: str

class Point(NamedTuple):
    x: float
    y: float
```

## Minimize `Any`

`Any` silently disables type checking. Acceptable only for:
- Truly dynamic data (parsed JSON before validation).
- Interfacing with untyped third-party code that has no stubs.

Otherwise reach for `object`, a `Protocol`, a `TypeVar`, or a union.

## `# type: ignore` Hygiene

- Never add a blanket `# type: ignore` — always include the specific error code: `# type: ignore[attr-defined]`.
- Every ignore must have a short comment explaining why the type system is wrong here.
- Review ignores during code review the same as any other assertion.

## Checklist

- [ ] All public function parameters and return types annotated.
- [ ] All class attributes annotated.
- [ ] No bare collection types (`list`, `dict`) — always parameterized.
- [ ] Modern union syntax (`T | None`) in new code.
- [ ] `Protocol` used for structural interfaces instead of nominal base classes where appropriate.
- [ ] `Any` usage reviewed and justified.
- [ ] Every `# type: ignore` is narrowed to a specific error code and commented.
- [ ] Exhaustive matches end with `assert_never`.
