# Styleguide

Coding patterns and conventions for Python projects. Use these documents as context when generating new projects.

## Documents

| File | Description |
|------|-------------|
| [python-project-structure.md](python-project-structure.md) | Project layout, package structure, pyproject.toml |
| [fastapi-patterns.md](fastapi-patterns.md) | FastAPI server, routes, static files, schemas |
| [authentication.md](authentication.md) | Session + API key auth, bcrypt, sliding expiration |
| [configuration.md](configuration.md) | Environment vars, .env, settings.yaml |
| [database-patterns.md](database-patterns.md) | SQLite with Peewee ORM |
| [cli-patterns.md](cli-patterns.md) | Click CLI, entry points |
| [client-library.md](client-library.md) | Python client libraries for APIs |
| [frontend-patterns.md](frontend-patterns.md) | Tailwind CSS, vanilla JS, no build tools |
| [documentation.md](documentation.md) | README, USAGE.md, docstrings |
| [code-style.md](code-style.md) | Type hints, error handling, naming |

## Quick Reference

### Stack

- **Backend**: Python 3.10+, FastAPI, Peewee (SQLite)
- **Auth**: Session cookies + API keys, bcrypt
- **Config**: Environment vars → .env → ~/.<project>/settings.yaml
- **Frontend**: Tailwind CSS (CDN), vanilla JS, Font Awesome
- **CLI**: Click

### New Project Checklist

```
project_name/
├── project_name/           # Package (same name as project)
│   ├── __init__.py         # Exports, __version__
│   ├── __main__.py         # python -m entry point
│   ├── config.py           # Configuration loading
│   ├── models.py           # Peewee ORM models
│   ├── auth.py             # Authentication logic
│   ├── server.py           # FastAPI app
│   ├── client.py           # Python client library
│   ├── cli.py              # Click CLI
│   ├── api/                # API routes (larger projects)
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── schemas.py
│   │   ├── dependencies.py
│   │   └── routes/
│   └── static/             # Frontend files
│       ├── index.html
│       ├── style.css
│       └── js/
├── tests/
│   └── test_*.py
├── scripts/                # One-off scripts, migrations
├── pyproject.toml
├── requirements.txt
├── README.md
├── USAGE.md                # Detailed usage guide
└── LICENSE                 # CC0
```

### Key Conventions

| Aspect | Convention |
|--------|------------|
| Package name | Same as project directory |
| Database | SQLite + Peewee, in `/tmp/<project>/` or `~/.<project>/` |
| Config | env vars > .env > settings.yaml > defaults |
| Auth | Session cookies (browser) + API keys (programmatic) |
| Passwords | bcrypt via passlib |
| Frontend | Tailwind CDN, vanilla JS, no npm |
| CLI | Click with entry points in pyproject.toml |
| License | CC0 |
| Author | Philip Orange <git@philiporange.com> |
| Version | Start at 0.1.0 |

### Configuration Locations

```
~/.<project_name>/
├── settings.yaml      # User settings (YAML)
├── database.db        # SQLite database
└── cache/             # Cache files

/tmp/<project_name>/   # Temporary data (default)
```

### Authentication Flow

1. **Browser**: POST /auth/login → session cookie → include credentials
2. **Script**: Create API key → Authorization: Bearer <key>
3. **Remember-me**: Long-lived token → auto-creates new session

### Frontend Stack

```html
<script src="https://cdn.tailwindcss.com"></script>
<link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.2/css/all.min.css" rel="stylesheet">
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

Colors: slate (neutrals), indigo (primary), emerald (success), rose (danger)

## Author

Philip Orange <git@philiporange.com>

## License

CC0
