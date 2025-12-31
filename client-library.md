# Client Library Patterns

Every API project should have a Python client library for programmatic access.

## Client Module Structure

```python
"""
Python client for project_name API.

Provides a convenient interface for interacting with the project_name server.
Supports both session-based and API key authentication.
"""

import requests
from typing import Optional, List, Dict, Any
from dataclasses import dataclass


@dataclass
class Item:
    """Item returned from API."""
    id: str
    name: str
    description: str
    created_at: str

    @classmethod
    def from_dict(cls, data: dict) -> "Item":
        return cls(
            id=data["id"],
            name=data["name"],
            description=data.get("description", ""),
            created_at=data.get("created_at", "")
        )


class ProjectClient:
    """
    Client for the Project Name API.

    Supports two authentication methods:
    - API key (recommended for scripts)
    - Session-based (for interactive use)

    Example:
        # With API key
        client = ProjectClient(api_key="your_api_key")
        items = client.list_items()

        # With session login
        client = ProjectClient()
        client.login("username", "password")
        items = client.list_items()
    """

    def __init__(
        self,
        base_url: str = "http://localhost:8000",
        api_key: Optional[str] = None,
        timeout: int = 30
    ):
        """
        Initialize the client.

        Args:
            base_url: API server URL
            api_key: API key for authentication (optional)
            timeout: Request timeout in seconds
        """
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.timeout = timeout
        self.session = requests.Session()

        # Set API key header if provided
        if api_key:
            self.session.headers["Authorization"] = f"Bearer {api_key}"

    def _request(
        self,
        method: str,
        endpoint: str,
        **kwargs
    ) -> Dict[str, Any]:
        """Make HTTP request to API."""
        url = f"{self.base_url}{endpoint}"

        # Set default headers
        headers = kwargs.pop("headers", {})
        if "Content-Type" not in headers and method in ("POST", "PUT", "PATCH"):
            headers["Content-Type"] = "application/json"

        response = self.session.request(
            method,
            url,
            headers=headers,
            timeout=self.timeout,
            **kwargs
        )

        # Handle errors
        if response.status_code == 401:
            raise AuthenticationError("Authentication required")
        elif response.status_code == 403:
            raise AuthorizationError("Permission denied")
        elif response.status_code == 404:
            raise NotFoundError(f"Resource not found: {endpoint}")
        elif not response.ok:
            try:
                detail = response.json().get("detail", response.text)
            except:
                detail = response.text
            raise APIError(f"API error {response.status_code}: {detail}")

        # Return JSON if available
        if response.headers.get("Content-Type", "").startswith("application/json"):
            return response.json()
        return {"content": response.content}

    # =========================================================================
    # Authentication
    # =========================================================================

    def login(self, username: str, password: str, remember_me: bool = False) -> Dict:
        """
        Login with username and password.

        Creates a session cookie for subsequent requests.

        Args:
            username: Account username
            password: Account password
            remember_me: Create long-lived session

        Returns:
            User info dict
        """
        return self._request("POST", "/auth/login", json={
            "username": username,
            "password": password,
            "remember_me": remember_me
        })

    def logout(self) -> Dict:
        """Logout and clear session."""
        return self._request("POST", "/auth/logout")

    def register(self, username: str, password: str) -> Dict:
        """Register new user account."""
        return self._request("POST", "/auth/register", json={
            "username": username,
            "password": password
        })

    def get_me(self) -> Dict:
        """Get current authenticated user info."""
        return self._request("GET", "/auth/me")

    # =========================================================================
    # API Key Management
    # =========================================================================

    def create_api_key(self, name: str, expires_days: Optional[int] = None) -> Dict:
        """
        Create new API key.

        Args:
            name: Descriptive name for the key
            expires_days: Days until expiration (None = never)

        Returns:
            Dict with api_key (save this, shown only once!)
        """
        return self._request("POST", "/auth/api-keys", json={
            "name": name,
            "expires_days": expires_days
        })

    def list_api_keys(self) -> List[Dict]:
        """List API keys for current user (values masked)."""
        return self._request("GET", "/auth/api-keys")

    def revoke_api_key(self, key_prefix: str) -> Dict:
        """Revoke API key by prefix."""
        return self._request("DELETE", f"/auth/api-keys/{key_prefix}")

    # =========================================================================
    # Items (example resource)
    # =========================================================================

    def list_items(self, limit: int = 100, offset: int = 0) -> List[Item]:
        """
        List all items.

        Args:
            limit: Maximum items to return
            offset: Skip first N items

        Returns:
            List of Item objects
        """
        data = self._request("GET", "/api/items", params={
            "limit": limit,
            "offset": offset
        })
        return [Item.from_dict(item) for item in data]

    def get_item(self, item_id: str) -> Item:
        """Get item by ID."""
        data = self._request("GET", f"/api/items/{item_id}")
        return Item.from_dict(data)

    def create_item(self, name: str, description: str = "") -> Item:
        """Create new item."""
        data = self._request("POST", "/api/items", json={
            "name": name,
            "description": description
        })
        return Item.from_dict(data)

    def update_item(self, item_id: str, **kwargs) -> Item:
        """Update item fields."""
        data = self._request("PATCH", f"/api/items/{item_id}", json=kwargs)
        return Item.from_dict(data)

    def delete_item(self, item_id: str) -> Dict:
        """Delete item."""
        return self._request("DELETE", f"/api/items/{item_id}")

    def search_items(self, query: str, limit: int = 10) -> List[Item]:
        """Search items by query."""
        data = self._request("GET", "/api/items/search", params={
            "q": query,
            "limit": limit
        })
        return [Item.from_dict(item) for item in data]


# =============================================================================
# Exceptions
# =============================================================================

class APIError(Exception):
    """Base API error."""
    pass


class AuthenticationError(APIError):
    """Authentication required or failed."""
    pass


class AuthorizationError(APIError):
    """Permission denied."""
    pass


class NotFoundError(APIError):
    """Resource not found."""
    pass
```

## Usage Examples

### Basic Usage with API Key

```python
from project_name import ProjectClient

# Initialize with API key
client = ProjectClient(
    base_url="http://localhost:8000",
    api_key="your_api_key_here"
)

# List items
items = client.list_items()
for item in items:
    print(f"{item.id}: {item.name}")

# Create item
new_item = client.create_item("My Item", "Description")
print(f"Created: {new_item.id}")

# Search
results = client.search_items("query")
```

### Session-Based Authentication

```python
from project_name import ProjectClient

client = ProjectClient("http://localhost:8000")

# Login (creates session cookie)
client.login("username", "password", remember_me=True)

# Make authenticated requests
user = client.get_me()
print(f"Logged in as: {user['username']}")

items = client.list_items()

# Logout when done
client.logout()
```

### Creating API Keys Programmatically

```python
from project_name import ProjectClient

# Login first
client = ProjectClient()
client.login("admin", "password")

# Create API key
result = client.create_api_key("My Script", expires_days=90)
api_key = result["api_key"]
print(f"New API key: {api_key}")
print("Save this key - it won't be shown again!")

# Now use the API key
client2 = ProjectClient(api_key=api_key)
items = client2.list_items()
```

### Error Handling

```python
from project_name import (
    ProjectClient,
    APIError,
    AuthenticationError,
    NotFoundError
)

client = ProjectClient(api_key="your_key")

try:
    item = client.get_item("nonexistent")
except NotFoundError:
    print("Item not found")
except AuthenticationError:
    print("Invalid API key")
except APIError as e:
    print(f"API error: {e}")
```

## File Upload Support

```python
def upload_file(self, file_path: str) -> Dict:
    """Upload file to API."""
    with open(file_path, "rb") as f:
        # Don't set Content-Type - requests handles multipart
        response = self.session.post(
            f"{self.base_url}/api/upload",
            files={"file": f},
            timeout=self.timeout
        )

    if not response.ok:
        raise APIError(f"Upload failed: {response.text}")

    return response.json()


def download_file(self, item_id: str, output_path: str) -> str:
    """Download file from API."""
    response = self.session.get(
        f"{self.base_url}/api/items/{item_id}/download",
        stream=True,
        timeout=self.timeout
    )

    if not response.ok:
        raise APIError(f"Download failed: {response.status_code}")

    with open(output_path, "wb") as f:
        for chunk in response.iter_content(chunk_size=8192):
            f.write(chunk)

    return output_path
```

## Async Client (Optional)

```python
"""Async client using httpx."""

import httpx
from typing import Optional, List, Dict, Any


class AsyncProjectClient:
    """Async client for Project API."""

    def __init__(
        self,
        base_url: str = "http://localhost:8000",
        api_key: Optional[str] = None,
        timeout: int = 30
    ):
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout

        headers = {}
        if api_key:
            headers["Authorization"] = f"Bearer {api_key}"

        self.client = httpx.AsyncClient(
            base_url=self.base_url,
            headers=headers,
            timeout=timeout
        )

    async def __aenter__(self):
        return self

    async def __aexit__(self, *args):
        await self.client.aclose()

    async def _request(self, method: str, endpoint: str, **kwargs) -> Dict:
        response = await self.client.request(method, endpoint, **kwargs)

        if response.status_code == 401:
            raise AuthenticationError("Authentication required")
        elif not response.is_success:
            raise APIError(f"API error {response.status_code}")

        return response.json()

    async def list_items(self) -> List[Dict]:
        return await self._request("GET", "/api/items")

    async def create_item(self, name: str, description: str = "") -> Dict:
        return await self._request("POST", "/api/items", json={
            "name": name,
            "description": description
        })


# Usage
async def main():
    async with AsyncProjectClient(api_key="key") as client:
        items = await client.list_items()
        print(items)
```

## CLI Integration

The client should be usable from CLI:

```python
# cli.py
import click
from .client import ProjectClient


@click.group()
@click.option("--api-key", envvar="PROJECT_API_KEY")
@click.option("--base-url", default="http://localhost:8000")
@click.pass_context
def cli(ctx, api_key, base_url):
    """Project Name CLI."""
    ctx.obj = ProjectClient(base_url=base_url, api_key=api_key)


@cli.command()
@click.pass_obj
def list_items(client):
    """List all items."""
    items = client.list_items()
    for item in items:
        click.echo(f"{item.id}: {item.name}")


@cli.command()
@click.argument("name")
@click.option("--description", "-d", default="")
@click.pass_obj
def create(client, name, description):
    """Create new item."""
    item = client.create_item(name, description)
    click.echo(f"Created: {item.id}")
```
