# Python Project Structure

## Directory Layout

```
project_name/
├── project_name/           # Main package (MUST match project name)
│   ├── __init__.py         # Package exports
│   ├── __main__.py         # Entry point for python -m
│   ├── config.py           # Configuration
│   ├── cli.py              # CLI commands (Click)
│   ├── server.py           # FastAPI server
│   ├── client.py           # Python client library
│   ├── models.py           # Peewee ORM models
│   └── utils.py            # Shared utilities
├── tests/
│   ├── __init__.py
│   ├── test_core.py
│   └── test_*.py
├── scripts/                # One-off scripts, migrations
├── static/                 # Web frontend files
│   ├── index.html
│   ├── app.js
│   └── styles.css
├── examples/               # Usage examples
├── pyproject.toml          # Package configuration
├── requirements.txt        # Dependencies
├── README.md               # Project documentation
├── USAGE.md                # Detailed usage guide
├── LICENSE                 # CC0 license
└── run.py                  # Convenience startup script (optional)
```

## Package Structure Rules

1. **Package name = project directory name** - The inner package directory must have the same name as the outer project directory

2. **No code at root level** - All Python code lives inside the package directory, not at the project root

3. **Single responsibility modules** - Each module has one clear purpose:
   - `config.py` - Configuration loading
   - `models.py` - Database models
   - `server.py` - Web server
   - `cli.py` - Command-line interface
   - `client.py` - Client library for the API

## __init__.py Exports

Export key classes and functions for easy importing:

```python
"""
Project description in one sentence.

More details about what the project does and how.
"""

from .core import MainClass, main_function
from .models import Model1, Model2
from .config import CONFIG_VAR

__version__ = "0.1.0"
__all__ = ["MainClass", "main_function", "Model1", "Model2"]
```

## __main__.py Entry Point

Enable `python -m project_name`:

```python
"""Entry point for python -m project_name."""

from .cli import cli

if __name__ == "__main__":
    cli()
```

## pyproject.toml Template

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "project-name"
version = "0.1.0"
description = "Short description"
readme = "README.md"
license = {text = "CC0-1.0"}
authors = [{name = "Philip Orange", email = "git@philiporange.com"}]
requires-python = ">=3.10"
dependencies = [
    "fastapi>=0.100.0",
    "uvicorn>=0.23.0",
    "peewee>=3.16.0",
    "click>=8.1.0",
    "python-dotenv>=1.0.0",
]

[project.optional-dependencies]
dev = ["pytest>=7.0.0", "black", "ruff"]

[project.scripts]
project-name = "project_name.cli:cli"
project-name-server = "project_name.server:main"

[project.urls]
Homepage = "https://github.com/philiporange/project-name"

[tool.setuptools.packages.find]
where = ["."]
include = ["project_name*"]
```

## requirements.txt

List direct dependencies only (no version pinning unless necessary):

```
fastapi
uvicorn
peewee
click
python-dotenv
requests
```

## Data Storage Locations

- **Temporary/cache data**: `/tmp/<project_name>/`
- **Persistent user data**: `~/.project_name/` or `~/.config/project_name/`
- **Project-specific data**: Check environment variable first, then default

```python
from pathlib import Path
import os

DATA_DIR = Path(os.getenv("PROJECT_DATA_DIR", "/tmp/project_name"))
DATA_DIR.mkdir(parents=True, exist_ok=True)
```

## Import Conventions

- Never use try/except for imports
- List all dependencies in requirements.txt
- Use lazy loading for optional heavy dependencies:

```python
# Lazy loading for optional dependencies
_heavy_lib = None

def get_heavy_lib():
    global _heavy_lib
    if _heavy_lib is None:
        import heavy_library
        _heavy_lib = heavy_library
    return _heavy_lib
```
