# Python Conventions

## Target Python Version
- Target Python 3.10+ for new code; 3.12+ when possible.
- Do not import from `typing` for generics that are available in the builtins (`list[T]`, `dict[K, V]`, `tuple[T, ...]`, `set[T]`) or in `collections.abc` (`Callable`, `Iterable`, `Iterator`, `Mapping`, `Sequence`, `Awaitable`).

## Naming (PEP 8)
- Modules and files: `snake_case`, descriptive, no abbreviations (`user_repository.py`, not `usr_repo.py`).
- Classes: `PascalCase`. Acronyms stay uppercase (`HTTPClient`, not `HttpClient`).
- Functions and variables: `snake_case`.
- Module-level constants: `SCREAMING_SNAKE_CASE`.
- Private module members use a leading underscore; exported names are listed in `__all__`.
- No stuttering: `auth.Client`, not `auth.AuthClient`.

## Imports
Group imports: standard library → third-party → first-party. One group per blank-line-separated block. Use absolute imports; avoid relative imports except inside tightly coupled packages where relative imports make refactors easier.

```python
# Standard library
import os
from collections.abc import Callable
from typing import Any

# Third-party
# (intentionally omitted — use whatever the project depends on)

# First-party
from myproject.models import User
from myproject.services import UserService
```

Do not use wildcard imports (`from foo import *`) outside of `__init__.py` re-exports.

## Docstrings (Google style)
Write docstrings for all public modules, classes, functions, and methods. Short functions can use a one-line docstring; complex functions should document `Args`, `Returns`, and `Raises`.

```python
def process_batch(
    items: list[Item],
    max_workers: int = 4,
    on_progress: Callable[[int, int], None] | None = None,
) -> BatchResult:
    """Process items concurrently using a worker pool.

    Args:
        items: The items to process. Must not be empty.
        max_workers: Maximum concurrent workers. Defaults to 4.
        on_progress: Optional callback receiving (completed, total) counts.

    Returns:
        BatchResult with succeeded items and any failures.

    Raises:
        ValueError: If items is empty.
        ProcessingError: If the batch cannot be processed.
    """
    ...
```

## Formatting
- Line length: 120 characters (adjust to repository convention if different).
- Break long argument lists one-per-line with trailing comma.
- For method chains, parenthesize and indent one call per line.
- For long strings, use adjacent string literals inside parentheses rather than backslash continuation.

```python
def create_user(
    email: str,
    name: str,
    role: UserRole = UserRole.MEMBER,
    notify: bool = True,
) -> User:
    ...

error_message = (
    f"Failed to process user {user_id}: "
    f"received status {status} "
    f"with body {body[:100]}"
)
```

## Classes and Data
- Prefer `@dataclass(slots=True)` or `@dataclass(frozen=True)` for plain data containers over hand-written `__init__`.
- Use `enum.Enum` / `enum.StrEnum` for closed sets of values; do not scatter string literals.
- Favor immutability (`frozen=True`, tuples) when the value is not meant to change.

## Packaging Layout
- `src/<package>/` layout keeps tests, tooling, and package code clearly separated.
- Module-level side effects at import time are forbidden — a bare `import mypkg` must be cheap and pure.

## Tooling
- Use a linter/formatter and a static type checker. Configuration and specific tool choice belong in a separate skill, not here.
- Configuration lives in `pyproject.toml` (PEP 621).

## Project Documentation
- Keep a `README.md` with installation, quick start, configuration, and development sections.
- Keep a `CHANGELOG.md` following the "Keep a Changelog" format (`Added`, `Changed`, `Fixed`, `Deprecated`, `Removed`).
- Treat documentation as code — update alongside behavior changes.
