# FastAPI Patterns

## Basic Server Structure

```python
"""
FastAPI server for project_name.

Provides REST API endpoints for [purpose].
"""

import os
from pathlib import Path
from fastapi import FastAPI, HTTPException
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse
import uvicorn

from .config import HOST, PORT


app = FastAPI(
    title="Project Name",
    description="API description",
    version="0.1.0"
)

# Mount static files
STATIC_DIR = Path(__file__).parent / "static"
if STATIC_DIR.exists():
    app.mount("/static", StaticFiles(directory=STATIC_DIR), name="static")


@app.get("/")
async def root():
    """Serve the main page."""
    index_path = STATIC_DIR / "index.html"
    if index_path.exists():
        return FileResponse(index_path)
    return {"message": "API running"}


@app.get("/health")
async def health():
    """Health check endpoint."""
    return {"status": "ok"}


def main():
    """Run the server."""
    uvicorn.run(
        "project_name.server:app",
        host=HOST,
        port=PORT,
        reload=os.getenv("DEV_MODE") == "1"
    )


if __name__ == "__main__":
    main()
```

## Route Organization

For larger APIs, organize routes in a subdirectory:

```
project_name/
├── api/
│   ├── __init__.py
│   ├── main.py           # FastAPI app creation
│   ├── schemas.py        # Pydantic models
│   ├── dependencies.py   # Shared dependencies
│   └── routes/
│       ├── __init__.py
│       ├── items.py
│       ├── users.py
│       └── admin.py
```

### api/main.py

```python
"""FastAPI application factory."""

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from pathlib import Path

from .routes import items, users, admin


def create_app() -> FastAPI:
    app = FastAPI(title="Project Name", version="0.1.0")

    # Include routers
    app.include_router(items.router, prefix="/items", tags=["items"])
    app.include_router(users.router, prefix="/users", tags=["users"])
    app.include_router(admin.router, prefix="/admin", tags=["admin"])

    # Static files
    static_dir = Path(__file__).parent.parent / "static"
    if static_dir.exists():
        app.mount("/static", StaticFiles(directory=static_dir), name="static")

    return app


app = create_app()
```

### api/routes/items.py

```python
"""Item management routes."""

from fastapi import APIRouter, HTTPException
from typing import List

from ..schemas import Item, ItemCreate


router = APIRouter()


@router.get("/", response_model=List[Item])
async def list_items():
    """List all items."""
    return []


@router.get("/{item_id}", response_model=Item)
async def get_item(item_id: str):
    """Get a specific item."""
    raise HTTPException(status_code=404, detail="Item not found")


@router.post("/", response_model=Item)
async def create_item(item: ItemCreate):
    """Create a new item."""
    return item
```

### api/schemas.py

```python
"""Pydantic models for API request/response."""

from pydantic import BaseModel
from typing import Optional, List
from datetime import datetime


class ItemBase(BaseModel):
    name: str
    description: Optional[str] = None


class ItemCreate(ItemBase):
    pass


class Item(ItemBase):
    id: str
    created_at: datetime

    class Config:
        from_attributes = True  # For ORM model conversion


class SearchRequest(BaseModel):
    query: str
    limit: int = 10


class SearchResult(BaseModel):
    results: List[Item]
    total: int
    query: str
```

## Static File Serving

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse
from pathlib import Path

app = FastAPI()

# Static directory relative to this file
STATIC_DIR = Path(__file__).parent / "static"

# Mount static files at /static
app.mount("/static", StaticFiles(directory=STATIC_DIR), name="static")


@app.get("/")
async def serve_index():
    """Serve main HTML page."""
    return FileResponse(STATIC_DIR / "index.html")


@app.get("/{path:path}")
async def serve_file(path: str):
    """Serve any static file."""
    file_path = STATIC_DIR / path
    if file_path.exists() and file_path.is_file():
        return FileResponse(file_path)
    # Fall back to index for SPA routing
    return FileResponse(STATIC_DIR / "index.html")
```

## Error Handling

```python
from fastapi import HTTPException
from fastapi.responses import JSONResponse


# Custom exception
class ItemNotFoundError(Exception):
    def __init__(self, item_id: str):
        self.item_id = item_id


# Exception handler
@app.exception_handler(ItemNotFoundError)
async def item_not_found_handler(request, exc: ItemNotFoundError):
    return JSONResponse(
        status_code=404,
        content={"error": f"Item {exc.item_id} not found"}
    )


# Using HTTPException (preferred for simple cases)
@app.get("/items/{item_id}")
async def get_item(item_id: str):
    item = find_item(item_id)
    if not item:
        raise HTTPException(status_code=404, detail="Item not found")
    return item
```

## Background Tasks

```python
from fastapi import BackgroundTasks


def send_notification(email: str, message: str):
    """Background task to send notification."""
    # Long-running operation
    pass


@app.post("/items/")
async def create_item(item: ItemCreate, background_tasks: BackgroundTasks):
    """Create item and send notification in background."""
    new_item = save_item(item)
    background_tasks.add_task(send_notification, "admin@example.com", f"New item: {item.name}")
    return new_item
```

## File Upload

```python
from fastapi import UploadFile, File


@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    """Handle file upload."""
    content = await file.read()

    # Save to disk
    save_path = Path("/tmp/uploads") / file.filename
    save_path.parent.mkdir(parents=True, exist_ok=True)
    save_path.write_bytes(content)

    return {
        "filename": file.filename,
        "size": len(content),
        "content_type": file.content_type
    }
```

## Server Entry Point

```python
# In server.py or __main__.py
def main():
    import uvicorn
    from .config import HOST, PORT

    uvicorn.run(
        "project_name.server:app",
        host=HOST,
        port=PORT,
        reload=False
    )


if __name__ == "__main__":
    main()
```

## Convenience Run Script

Create `run.py` at project root:

```python
#!/usr/bin/env python3
"""Convenience script to run the server."""

import uvicorn


if __name__ == "__main__":
    uvicorn.run(
        "project_name.server:app",
        host="0.0.0.0",
        port=8000,
        reload=True
    )
```
