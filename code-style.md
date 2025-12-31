# Code Style

## Type Hints

Use type hints where they add clarity, but don't over-annotate:

```python
# Good - clarifies function signature
def search(query: str, limit: int = 10) -> List[Item]:
    pass

# Good - complex return type
def get_stats() -> Dict[str, int]:
    pass

# Good - optional values
def find_item(name: str) -> Optional[Item]:
    pass

# Unnecessary - obvious from context
items: List[Item] = []  # Just use: items = []
count: int = 0          # Just use: count = 0
```

### Common Type Patterns

```python
from typing import Optional, List, Dict, Tuple, Any, Union
from pathlib import Path

def process_file(path: Path) -> bytes:
    pass

def search(query: str, limit: int = 10) -> List[Dict[str, Any]]:
    pass

def get_config(key: str) -> Optional[str]:
    pass

def parse(data: Union[str, bytes]) -> Dict:
    pass
```

## Dataclasses

Use dataclasses for structured data:

```python
from dataclasses import dataclass, field
from typing import Optional, List
from datetime import datetime


@dataclass
class SearchResult:
    """A single search result."""
    id: str
    title: str
    score: float
    metadata: Dict[str, Any] = field(default_factory=dict)


@dataclass
class SearchResponse:
    """Search API response."""
    query: str
    results: List[SearchResult]
    total: int
    elapsed_ms: float
```

## Error Handling

### Custom Exceptions

Define in `exceptions.py`:

```python
"""Custom exceptions for project_name."""


class ProjectError(Exception):
    """Base exception for project errors."""
    pass


class NotFoundError(ProjectError):
    """Resource not found."""
    pass


class ValidationError(ProjectError):
    """Invalid input data."""
    pass


class ConfigurationError(ProjectError):
    """Configuration error."""
    pass
```

### Using Exceptions

```python
from .exceptions import NotFoundError, ValidationError


def get_item(item_id: str) -> Item:
    if not item_id:
        raise ValidationError("Item ID cannot be empty")

    item = Item.get_or_none(Item.id == item_id)
    if not item:
        raise NotFoundError(f"Item not found: {item_id}")

    return item


# In CLI or API
try:
    item = get_item(item_id)
except NotFoundError as e:
    click.echo(f"Error: {e}", err=True)
    raise SystemExit(1)
```

## Logging

Use loguru or standard logging:

```python
# With loguru (preferred)
from loguru import logger

logger.info("Processing item: {}", item.name)
logger.warning("Rate limit approaching")
logger.error("Failed to connect: {}", str(e))

# With standard logging
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.info("Processing item: %s", item.name)
```

## Context Managers

Use `with` statements for resource management:

```python
# File operations
with open(path, 'r') as f:
    data = f.read()

# Database connections
with db.atomic():
    item.save()
    log_action(item)

# Custom context managers
from contextlib import contextmanager

@contextmanager
def timer(name: str):
    start = time.time()
    try:
        yield
    finally:
        elapsed = time.time() - start
        logger.info(f"{name} took {elapsed:.2f}s")


with timer("search"):
    results = search(query)
```

## Lazy Loading

For optional or heavy dependencies:

```python
# Module-level lazy loading
_redis_client = None


def get_redis():
    global _redis_client
    if _redis_client is None:
        import redis
        from .config import REDIS_URL
        _redis_client = redis.from_url(REDIS_URL)
    return _redis_client


# Class-level lazy loading
class Searcher:
    def __init__(self):
        self._index = None

    @property
    def index(self):
        if self._index is None:
            self._index = self._load_index()
        return self._index

    def _load_index(self):
        # Expensive operation
        pass
```

## String Formatting

Use f-strings:

```python
# Good
message = f"Found {count} items matching '{query}'"
path = f"/api/items/{item_id}"

# Avoid
message = "Found {} items matching '{}'".format(count, query)
message = "Found %d items matching '%s'" % (count, query)
```

## Path Handling

Always use `pathlib.Path`:

```python
from pathlib import Path

# Build paths
config_dir = Path.home() / ".config" / "project"
data_file = config_dir / "data.json"

# Check existence
if data_file.exists():
    content = data_file.read_text()

# Create directories
config_dir.mkdir(parents=True, exist_ok=True)

# Iterate files
for file in config_dir.glob("*.json"):
    print(file.name)
```

## Imports

Order imports:
1. Standard library
2. Third-party packages
3. Local imports

```python
# Standard library
import os
import json
from pathlib import Path
from typing import Optional, List
from datetime import datetime

# Third-party
import click
from fastapi import FastAPI
from peewee import SqliteDatabase

# Local
from .config import DATA_DIR
from .models import Item
from .exceptions import NotFoundError
```

## Naming Conventions

```python
# Modules: lowercase_with_underscores
# config.py, data_loader.py

# Classes: PascalCase
class SearchEngine:
    pass

# Functions/methods: lowercase_with_underscores
def search_items(query: str) -> List[Item]:
    pass

# Constants: UPPERCASE_WITH_UNDERSCORES
MAX_RESULTS = 100
DEFAULT_TIMEOUT = 30

# Variables: lowercase_with_underscores
item_count = 0
search_results = []

# Private: prefix with underscore
def _internal_helper():
    pass

class MyClass:
    def _private_method(self):
        pass
```

## Avoid

1. **No try/except imports** - List all deps in requirements.txt
2. **No backwards compatibility hacks** - Just change the code
3. **No temporary comments** - Code is permanent infrastructure
4. **No over-engineering** - Solve the current problem, not hypothetical ones
5. **No unused code** - Delete it, don't comment it out
6. **No magic numbers** - Use named constants

```python
# BAD
try:
    import optional_lib
except ImportError:
    optional_lib = None

# BAD
# Old implementation (keeping for reference)
# def old_function():
#     pass

# BAD
if count > 42:  # Magic number

# GOOD
MAX_ITEMS = 42
if count > MAX_ITEMS:
    pass
```
