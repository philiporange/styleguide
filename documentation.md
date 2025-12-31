# Documentation Patterns

## README.md Structure

```markdown
# Project Name

One-line description of what the project does.

## Features

- Feature 1 description
- Feature 2 description
- Feature 3 description

## Installation

\`\`\`bash
pip install -e .
\`\`\`

## Quick Start

\`\`\`bash
# Basic usage
project-name command argument

# Or Python
python -m project_name
\`\`\`

## Usage

### Command Line

\`\`\`bash
# Search for items
project-name search "query"

# Start server
project-name serve --port 8000
\`\`\`

### Python API

\`\`\`python
from project_name import Client

client = Client()
results = client.search("query")
\`\`\`

## Configuration

Create a `.env` file:

\`\`\`bash
API_KEY=your_key_here
DATA_DIR=/path/to/data
\`\`\`

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/items` | GET | List all items |
| `/api/items` | POST | Create item |
| `/api/items/{id}` | GET | Get item |
| `/api/items/{id}` | DELETE | Delete item |

## Project Structure

\`\`\`
project_name/
├── project_name/
│   ├── __init__.py
│   ├── cli.py
│   └── server.py
├── tests/
├── pyproject.toml
└── README.md
\`\`\`

## License

CC0

## Author

Philip Orange <git@philiporange.com>
```

## USAGE.md Structure

Detailed usage guide with extensive code examples:

```markdown
# Project Name - Usage Guide

## Installation

### Prerequisites

- Python 3.10+
- Required system packages

### Install from source

\`\`\`bash
git clone https://github.com/user/project
cd project
pip install -e .
\`\`\`

## Core Components

### ClassName

The `ClassName` class handles [purpose].

#### `ClassName.method_name(arg1, arg2)`

Description of what the method does.

**Parameters:**
- `arg1` (str): Description of arg1
- `arg2` (int): Description of arg2

**Returns:**
- `ResultType`: Description of return value

**Example:**

\`\`\`python
from project_name import ClassName

obj = ClassName()
result = obj.method_name("value", 10)
print(result)
\`\`\`

**Output:**
\`\`\`
Expected output here
\`\`\`

## Common Patterns

### Pattern 1: Basic Usage

\`\`\`python
# Example code
\`\`\`

### Pattern 2: Advanced Usage

\`\`\`python
# Example code
\`\`\`

## Configuration Reference

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `API_KEY` | str | None | API key for external service |
| `PORT` | int | 8000 | Server port |

## Troubleshooting

### Common Issues

**Problem: Error message**

Solution: How to fix it.

\`\`\`bash
# Fix command
\`\`\`
```

## File-Level Docstrings

Every Python file starts with a docstring describing its purpose:

```python
"""
Database models for project_name.

This module defines the Peewee ORM models for storing items,
categories, and their relationships in SQLite.

Models:
    Item: Main item model with name and description
    Category: Hierarchical category system
    ItemCategory: Many-to-many relationship
"""

from peewee import *
# ... rest of code
```

## Function/Class Docstrings

Use Google-style docstrings:

```python
def search(query: str, limit: int = 10) -> List[Item]:
    """Search for items matching the query.

    Performs fuzzy matching on item names and descriptions.
    Results are sorted by relevance score.

    Args:
        query: Search query string
        limit: Maximum number of results to return

    Returns:
        List of matching Item objects, sorted by relevance

    Raises:
        ValueError: If query is empty
    """
    pass


class SearchEngine:
    """Full-text search engine with fuzzy matching.

    Uses trigram-based indexing for efficient fuzzy searches.
    Supports multiple data sources and customizable scoring.

    Attributes:
        index_path: Path to the search index
        sources: List of registered data sources
    """

    def __init__(self, index_path: Path):
        """Initialize the search engine.

        Args:
            index_path: Directory for storing index files
        """
        pass
```

## Comment Guidelines

1. **Don't comment obvious code** - Code should be self-documenting
2. **No temporary comments** - Don't write "TODO: fix later" or "temporary hack"
3. **Explain why, not what** - Comments explain reasoning, not mechanics
4. **Update comments with code** - Outdated comments are worse than none

```python
# BAD - obvious comment
# Increment counter by 1
counter += 1

# BAD - temporary comment
# TODO: fix this later
result = hacky_workaround()

# GOOD - explains why
# Use 0.85 threshold because lower values cause too many false positives
# in the title matching (see issue #42 for analysis)
MATCH_THRESHOLD = 0.85

# GOOD - explains non-obvious behavior
# Sleep briefly to avoid rate limiting from the API
# (max 10 requests per second)
time.sleep(0.1)
```

## No Emoji in Documentation

Keep README and documentation professional. Don't use emoji:

```markdown
# BAD
## Features

- Fast search
- Easy to use
- Powerful API

# GOOD
## Features

- Fast search
- Easy to use
- Powerful API
```

## License and Author

Always include at the end of README:

```markdown
## License

CC0

## Author

Philip Orange <git@philiporange.com>
```

## Version Information

Include version in `__init__.py`:

```python
__version__ = "0.1.0"
```

Start new projects at version `0.1.0`.
