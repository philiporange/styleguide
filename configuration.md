# Configuration Patterns

Three-tier configuration: environment variables, .env files, and settings.yaml.

## Configuration Hierarchy

1. **Environment variables** - Highest priority, for secrets and deployment overrides
2. **.env files** - Local development config, MUST NOT be checked into git
3. **settings.yaml** - User-specific settings at `~/.<project_name>/settings.yaml`
4. **Code defaults** - Fallback values in config.py

## Standard config.py Module

```python
"""
Configuration management for project_name.

Configuration hierarchy (highest to lowest priority):
1. Environment variables
2. .env file
3. ~/.<project_name>/settings.yaml
4. Code defaults
"""

import os
from pathlib import Path
from typing import Any, Optional

import yaml
from dotenv import load_dotenv


# Load .env file
def _load_env():
    """Load .env from current dir or parent."""
    for path in [Path(".env"), Path("../.env")]:
        if path.exists():
            load_dotenv(path)
            break


_load_env()


# Project paths
PROJECT_NAME = "project_name"
USER_CONFIG_DIR = Path.home() / f".{PROJECT_NAME}"
SETTINGS_PATH = USER_CONFIG_DIR / "settings.yaml"
DATA_DIR = Path(os.getenv("DATA_DIR", f"/tmp/{PROJECT_NAME}"))
DATABASE_PATH = DATA_DIR / "database.db"


def _load_settings() -> dict:
    """Load user settings from YAML file."""
    if SETTINGS_PATH.exists():
        with open(SETTINGS_PATH) as f:
            return yaml.safe_load(f) or {}
    return {}


def _get(key: str, default: Any = None, cast: type = str) -> Any:
    """
    Get config value with priority: env > settings.yaml > default.

    Args:
        key: Config key (e.g., "HOST", "database.path")
        default: Default value if not found
        cast: Type to cast to (str, int, bool, float)
    """
    # 1. Check environment variable
    env_key = key.upper().replace(".", "_")
    env_val = os.getenv(env_key)
    if env_val is not None:
        if cast == bool:
            return env_val.lower() in ("true", "1", "yes")
        return cast(env_val)

    # 2. Check settings.yaml
    settings = _load_settings()
    parts = key.lower().split(".")
    val = settings
    for part in parts:
        if isinstance(val, dict) and part in val:
            val = val[part]
        else:
            val = None
            break

    if val is not None:
        return cast(val) if not isinstance(val, cast) else val

    # 3. Return default
    return default


# Ensure directories exist
USER_CONFIG_DIR.mkdir(parents=True, exist_ok=True)
DATA_DIR.mkdir(parents=True, exist_ok=True)


# =============================================================================
# Configuration Values
# =============================================================================

# Server
HOST: str = _get("host", "0.0.0.0")
PORT: int = _get("port", 8000, int)
DEBUG: bool = _get("debug", False, bool)

# Security
COOKIE_SECURE: bool = _get("cookie_secure", False, bool)
API_KEY: Optional[str] = os.getenv("API_KEY")  # Never in settings.yaml

# Logging
LOG_LEVEL: str = _get("log_level", "INFO")
LOG_PATH: Path = Path(_get("log_path", str(DATA_DIR / "app.log")))

# Feature flags
FEATURE_X_ENABLED: bool = _get("features.x_enabled", False, bool)


# =============================================================================
# Config Class (alternative pattern for complex configs)
# =============================================================================

class Config:
    """Config class with lazy loading."""

    def __init__(self):
        self._settings = None

    @property
    def settings(self) -> dict:
        if self._settings is None:
            self._settings = _load_settings()
        return self._settings

    # Server
    HOST = _get("host", "0.0.0.0")
    PORT = _get("port", 8000, int)
    DEBUG = _get("debug", False, bool)

    # Security
    COOKIE_SECURE = _get("cookie_secure", False, bool)

    @property
    def API_KEY(self) -> Optional[str]:
        """API key from environment only (never settings.yaml)."""
        return os.getenv("API_KEY")


# Singleton instance
config = Config()
```

## settings.yaml Format

User settings live at `~/.<project_name>/settings.yaml`:

```yaml
# ~/.<project_name>/settings.yaml
# User-specific settings (don't put secrets here)

# Server settings
host: "0.0.0.0"
port: 8000
debug: false

# Logging
log_level: "INFO"
log_path: "/var/log/project_name/app.log"

# Feature flags
features:
  x_enabled: true
  beta_mode: false

# Application-specific
database:
  path: "/data/project_name/db.sqlite"
  pool_size: 5

# Nested settings
server:
  timeout: 30
  max_connections: 100

# List values
allowed_hosts:
  - "localhost"
  - "example.com"
```

## .env File Format

`.env` files MUST NOT be checked into git. Use `.env.example` as a template (see below).

```bash
# .env - Local development config (NOT checked into git)
# Copy from .env.example and fill in your values

# Server
HOST=0.0.0.0
PORT=8000
DEBUG=true

# Security (override in production)
COOKIE_SECURE=false

# External APIs (set in environment, not here)
# API_KEY=<set in environment>

# Paths
DATA_DIR=/tmp/project_name
LOG_PATH=/tmp/project_name/app.log
```

## .env.example Template

`.env.example` should be checked into git as a template for required environment variables.

```bash
# .env.example - Copy to .env and fill in values (this file IS checked into git)

# Server Configuration
HOST=0.0.0.0
PORT=8000
DEBUG=false

# Security
COOKIE_SECURE=true

# API Keys (required)
API_KEY=your_api_key_here
EXTERNAL_SERVICE_KEY=your_key_here

# Paths (optional, has defaults)
# DATA_DIR=/custom/path
# LOG_PATH=/custom/log/path
```

## Environment Variable Conventions

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `HOST` | str | `0.0.0.0` | Server bind address |
| `PORT` | int | `8000` | Server port |
| `DEBUG` | bool | `false` | Enable debug mode |
| `DATA_DIR` | path | `/tmp/project` | Data storage directory |
| `LOG_LEVEL` | str | `INFO` | Logging level |
| `LOG_PATH` | path | (DATA_DIR/app.log) | Log file path |
| `COOKIE_SECURE` | bool | `false` | Require HTTPS for cookies |
| `API_KEY` | str | (none) | External API key |

## Type Conversion Helpers

```python
import os


def get_bool(key: str, default: bool = False) -> bool:
    """Get boolean from environment."""
    val = os.getenv(key, "")
    if not val:
        return default
    return val.lower() in ("true", "1", "yes", "on")


def get_int(key: str, default: int = 0) -> int:
    """Get integer from environment."""
    val = os.getenv(key)
    if val is None:
        return default
    return int(val)


def get_list(key: str, default: list = None, sep: str = ",") -> list:
    """Get list from comma-separated environment variable."""
    val = os.getenv(key)
    if val is None:
        return default or []
    return [item.strip() for item in val.split(sep)]


# Usage
DEBUG = get_bool("DEBUG", False)
PORT = get_int("PORT", 8000)
ALLOWED_HOSTS = get_list("ALLOWED_HOSTS", ["localhost"])
```

## Settings YAML Helper

```python
"""Settings management."""

import yaml
from pathlib import Path
from typing import Any


class Settings:
    """Manage user settings in YAML format."""

    def __init__(self, project_name: str):
        self.project_name = project_name
        self.config_dir = Path.home() / f".{project_name}"
        self.settings_path = self.config_dir / "settings.yaml"
        self._data = None

    def _ensure_dir(self):
        self.config_dir.mkdir(parents=True, exist_ok=True)

    def load(self) -> dict:
        """Load settings from YAML."""
        if self._data is not None:
            return self._data

        if self.settings_path.exists():
            with open(self.settings_path) as f:
                self._data = yaml.safe_load(f) or {}
        else:
            self._data = {}

        return self._data

    def save(self, data: dict = None):
        """Save settings to YAML."""
        self._ensure_dir()
        if data is not None:
            self._data = data
        with open(self.settings_path, "w") as f:
            yaml.dump(self._data, f, default_flow_style=False)

    def get(self, key: str, default: Any = None) -> Any:
        """Get nested setting by dot-notation key."""
        data = self.load()
        parts = key.split(".")
        for part in parts:
            if isinstance(data, dict) and part in data:
                data = data[part]
            else:
                return default
        return data

    def set(self, key: str, value: Any):
        """Set nested setting by dot-notation key."""
        data = self.load()
        parts = key.split(".")

        # Navigate to parent
        current = data
        for part in parts[:-1]:
            if part not in current:
                current[part] = {}
            current = current[part]

        # Set value
        current[parts[-1]] = value
        self.save()


# Usage
settings = Settings("project_name")
port = settings.get("server.port", 8000)
settings.set("server.port", 9000)
```

## CLI Settings Command

```python
"""CLI commands for settings management."""

import click
import yaml

from .config import SETTINGS_PATH, USER_CONFIG_DIR


@click.group()
def settings():
    """Manage project settings."""
    pass


@settings.command()
def show():
    """Show current settings."""
    if SETTINGS_PATH.exists():
        with open(SETTINGS_PATH) as f:
            click.echo(f.read())
    else:
        click.echo(f"No settings file at {SETTINGS_PATH}")


@settings.command()
def edit():
    """Open settings in editor."""
    import subprocess
    import os

    editor = os.environ.get("EDITOR", "nano")

    # Create default if doesn't exist
    if not SETTINGS_PATH.exists():
        USER_CONFIG_DIR.mkdir(parents=True, exist_ok=True)
        with open(SETTINGS_PATH, "w") as f:
            f.write("# Project settings\n\n")

    subprocess.run([editor, str(SETTINGS_PATH)])


@settings.command()
def path():
    """Show settings file path."""
    click.echo(SETTINGS_PATH)
```

## Directory Structure

```
~/.<project_name>/
├── settings.yaml      # User settings
├── database.db        # SQLite database (if not in DATA_DIR)
├── cache/             # Cache files
└── logs/              # Log files

/tmp/<project_name>/   # Temporary data (default DATA_DIR)
├── database.db
├── cache/
└── uploads/
```
