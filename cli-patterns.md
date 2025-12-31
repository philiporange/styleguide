# CLI Patterns

Use Click for all command-line interfaces.

## Basic CLI Structure

```python
"""
Command-line interface for project_name.
"""

import click


@click.group()
@click.version_option(version="0.1.0")
def cli():
    """Project name - short description."""
    pass


@cli.command()
@click.argument("query")
@click.option("-n", "--limit", default=10, help="Maximum results")
@click.option("-v", "--verbose", is_flag=True, help="Verbose output")
def search(query: str, limit: int, verbose: bool):
    """Search for items matching QUERY."""
    results = do_search(query, limit)

    for item in results:
        if verbose:
            click.echo(f"{item.id}: {item.name} - {item.description}")
        else:
            click.echo(f"{item.name}")


@cli.command()
@click.argument("input_file", type=click.Path(exists=True))
@click.argument("output_file", type=click.Path())
@click.option("--force", is_flag=True, help="Overwrite existing output")
def convert(input_file: str, output_file: str, force: bool):
    """Convert INPUT_FILE to OUTPUT_FILE."""
    if Path(output_file).exists() and not force:
        raise click.ClickException(f"Output file exists: {output_file}")

    do_convert(input_file, output_file)
    click.echo(f"Converted to {output_file}")


@cli.command()
def serve():
    """Start the web server."""
    from .server import main
    main()


if __name__ == "__main__":
    cli()
```

## Entry Points in pyproject.toml

```toml
[project.scripts]
project-name = "project_name.cli:cli"
project-server = "project_name.server:main"
```

After installation, users can run:
```bash
project-name search "query"
project-name convert input.txt output.txt
project-server
```

## Common Patterns

### Arguments vs Options

```python
# Arguments: positional, required
@click.argument("filename")

# Options: named, optional (usually)
@click.option("--output", "-o", default="output.txt")
@click.option("--verbose", "-v", is_flag=True)
@click.option("--count", "-n", type=int, default=10)
```

### File Paths

```python
# Input file (must exist)
@click.argument("input_file", type=click.Path(exists=True))

# Output file (may not exist)
@click.argument("output_file", type=click.Path())

# Directory
@click.option("--dir", type=click.Path(file_okay=False, dir_okay=True))
```

### Multiple Values

```python
# Multiple arguments
@click.argument("files", nargs=-1)

# Multiple options
@click.option("--tag", "-t", multiple=True)
```

### Choices

```python
@click.option("--format", type=click.Choice(["json", "csv", "yaml"]))
@click.option("--level", type=click.Choice(["debug", "info", "warn"]), default="info")
```

### Dry Run Pattern

```python
@cli.command()
@click.argument("path")
@click.option("--dry-run", is_flag=True, help="Show what would be done")
def process(path: str, dry_run: bool):
    """Process files in PATH."""
    files = find_files(path)

    for file in files:
        if dry_run:
            click.echo(f"Would process: {file}")
        else:
            click.echo(f"Processing: {file}")
            do_process(file)
```

### Progress Bars

```python
import click

@cli.command()
@click.argument("items", nargs=-1)
def process(items):
    """Process multiple ITEMS."""
    with click.progressbar(items, label="Processing") as bar:
        for item in bar:
            do_work(item)
```

### Confirmation

```python
@cli.command()
@click.option("--yes", "-y", is_flag=True, help="Skip confirmation")
def reset(yes: bool):
    """Reset the database."""
    if not yes:
        click.confirm("This will delete all data. Continue?", abort=True)

    do_reset()
    click.echo("Database reset complete")
```

### Error Handling

```python
@cli.command()
def risky_operation():
    """Do something that might fail."""
    try:
        result = do_something()
        click.echo(f"Success: {result}")
    except ValueError as e:
        raise click.ClickException(str(e))
    except FileNotFoundError as e:
        click.echo(f"Error: {e}", err=True)
        raise SystemExit(1)
```

### Colored Output

```python
@cli.command()
def status():
    """Show status."""
    click.secho("OK", fg="green")
    click.secho("Warning", fg="yellow")
    click.secho("Error", fg="red")
    click.secho("Info", fg="blue", bold=True)
```

## Subcommands

```python
@click.group()
def cli():
    """Main CLI."""
    pass


@cli.group()
def db():
    """Database commands."""
    pass


@db.command()
def init():
    """Initialize database."""
    click.echo("Database initialized")


@db.command()
def migrate():
    """Run migrations."""
    click.echo("Migrations complete")


# Usage: project-name db init
# Usage: project-name db migrate
```

## Full Example

```python
"""CLI for media_search."""

import click
from pathlib import Path

from . import __version__
from .search import search_media
from .server import main as run_server


@click.group()
@click.version_option(version=__version__)
def cli():
    """Media Search - find and manage media files."""
    pass


@cli.command()
@click.argument("query")
@click.option("-n", "--limit", default=10, help="Maximum results")
@click.option("-t", "--type", "media_type",
              type=click.Choice(["movie", "tv", "book"]),
              help="Filter by media type")
@click.option("--json", "output_json", is_flag=True, help="Output as JSON")
def search(query: str, limit: int, media_type: str, output_json: bool):
    """Search for media matching QUERY."""
    results = search_media(query, limit=limit, media_type=media_type)

    if output_json:
        import json
        click.echo(json.dumps([r.dict() for r in results], indent=2))
    else:
        for i, result in enumerate(results, 1):
            click.echo(f"{i}. {result.title} ({result.year})")


@cli.command()
@click.option("--host", default="0.0.0.0", help="Server host")
@click.option("--port", default=8000, type=int, help="Server port")
def serve(host: str, port: int):
    """Start the web server."""
    click.echo(f"Starting server at http://{host}:{port}")
    run_server(host=host, port=port)


@cli.command()
@click.argument("path", type=click.Path(exists=True))
@click.option("--dry-run", is_flag=True, help="Show what would be done")
def scan(path: str, dry_run: bool):
    """Scan PATH for media files."""
    from .scanner import scan_directory

    files = scan_directory(Path(path))

    for file in files:
        if dry_run:
            click.echo(f"Would index: {file}")
        else:
            click.echo(f"Indexed: {file}")


if __name__ == "__main__":
    cli()
```
