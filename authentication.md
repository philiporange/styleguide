# Authentication Patterns

> **Reference documentation** for implementing secure user authentication in web applications.
> These patterns follow security best practices from OWASP guidelines.

This guide covers session-based authentication for browsers and API key authentication for programmatic access.

## Architecture Overview

A well-designed auth system supports multiple access methods:

| Method | Use Case | Storage | Lifetime |
|--------|----------|---------|----------|
| Session cookies | Browser/interactive | Server-side | 30 days (sliding) |
| API keys | Scripts/automation | Server-side | Configurable |
| Remember-me tokens | "Stay logged in" | Server-side | 90 days |

All tokens are generated using `secrets.token_urlsafe()` for cryptographic security.

## Database Models

Store auth data in dedicated tables. The User model stores a bcrypt hash, never plaintext credentials.

```python
"""Models for user accounts and sessions."""

from datetime import datetime
from peewee import (
    SqliteDatabase, Model, CharField, TextField,
    BooleanField, DateTimeField
)

from .config import DATABASE_PATH

db = SqliteDatabase(str(DATABASE_PATH))


class BaseModel(Model):
    class Meta:
        database = db
```

### User Model

The password field stores a bcrypt hash (60 characters). Never store plaintext.

```python
class User(BaseModel):
    """User account."""
    id = CharField(primary_key=True, max_length=32)
    username = CharField(max_length=255, unique=True)
    password = CharField(max_length=255)  # bcrypt hash
    is_admin = BooleanField(default=False)
    created_at = DateTimeField(default=datetime.now)
```

### Session Model

Sessions track active logins with sliding expiration. Store metadata for security auditing.

```python
class Session(BaseModel):
    """Active session with sliding expiration."""
    session_id = CharField(primary_key=True, max_length=64)
    user_id = CharField(max_length=32)
    created_at = DateTimeField()
    last_activity = DateTimeField()
    expires_at = DateTimeField()
    ip_address = CharField(max_length=45, null=True)
    user_agent = TextField(null=True)
```

### Remember-Me Token Model

Long-lived tokens for "stay logged in" functionality. Include revocation support.

```python
class RememberMeToken(BaseModel):
    """Long-lived remember-me token."""
    token_id = CharField(primary_key=True, max_length=64)
    user_id = CharField(max_length=32)
    created_at = DateTimeField()
    expires_at = DateTimeField()
    ip_address = CharField(max_length=45, null=True)
    user_agent = TextField(null=True)
    revoked = BooleanField(default=False)
```

### API Key Model

For programmatic access. Track usage for monitoring.

```python
class ApiKey(BaseModel):
    """API key for programmatic access."""
    api_key = CharField(primary_key=True, max_length=64)
    user_id = CharField(max_length=32)
    name = CharField(max_length=255)
    created_at = DateTimeField()
    expires_at = DateTimeField(null=True)
    last_used_at = DateTimeField(null=True)
    revoked = BooleanField(default=False)
```

### Database Initialization

```python
def init_db():
    db.connect(reuse_if_open=True)
    db.create_tables([User, Session, RememberMeToken, ApiKey], safe=True)

init_db()
```

## Credential Handling

Use bcrypt for secure credential storage. The passlib library provides a clean interface.

```python
from passlib.context import CryptContext

# Configure bcrypt with automatic algorithm upgrades
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_credential(plaintext: str) -> str:
    """Create bcrypt hash of plaintext."""
    return pwd_context.hash(plaintext)


def verify_credential(plaintext: str, hashed: str) -> bool:
    """Verify plaintext against stored hash."""
    return pwd_context.verify(plaintext, hashed)
```

## Token Generation

All tokens use cryptographically secure random generation.

```python
import secrets

def generate_secure_token(length: int = 32) -> str:
    """Generate URL-safe random token."""
    return secrets.token_urlsafe(length)
```

## Session Configuration

Use ITP-safe values (under 7 days) for sliding window to ensure Safari compatibility.

```python
SESSION_SLIDING_WINDOW_DAYS = 7   # ITP-safe: < 7 days
SESSION_MAX_AGE_DAYS = 30
REMEMBER_ME_MAX_AGE_DAYS = 90
```

## User Management

Manager class for user operations.

```python
import datetime
from typing import Optional

class UserManager:
    """Manages user accounts."""

    @staticmethod
    def create_user(username: str, plaintext_pw: str, is_admin: bool = False) -> User:
        """Create user with secure credential storage."""
        user_id = generate_secure_token(16)
        hashed = hash_credential(plaintext_pw)
        return User.create(
            id=user_id,
            username=username,
            password=hashed,
            is_admin=is_admin
        )

    @staticmethod
    def authenticate(username: str, plaintext_pw: str) -> Optional[User]:
        """Verify credentials and return user if valid."""
        try:
            user = User.get(User.username == username)
            if verify_credential(plaintext_pw, user.password):
                return user
            return None
        except User.DoesNotExist:
            return None

    @staticmethod
    def get_by_id(user_id: str) -> Optional[User]:
        """Retrieve user by ID."""
        try:
            return User.get(User.id == user_id)
        except User.DoesNotExist:
            return None
```

## Session Management

Handle session lifecycle with sliding expiration.

```python
from typing import Tuple

class SessionManager:
    """Manages user sessions."""

    @staticmethod
    def create(
        user_id: str,
        ip_address: Optional[str] = None,
        user_agent: Optional[str] = None
    ) -> str:
        """Create new session, return session_id."""
        session_id = generate_secure_token()
        now = datetime.datetime.now()
        expires_at = now + datetime.timedelta(days=SESSION_MAX_AGE_DAYS)

        Session.create(
            session_id=session_id,
            user_id=user_id,
            created_at=now,
            last_activity=now,
            expires_at=expires_at,
            ip_address=ip_address,
            user_agent=user_agent
        )
        return session_id

    @staticmethod
    def validate(session_id: str) -> Optional[Session]:
        """Check if session is valid and not expired."""
        try:
            session = Session.get(Session.session_id == session_id)
            if session.expires_at < datetime.datetime.now():
                SessionManager.delete(session_id)
                return None
            return session
        except Session.DoesNotExist:
            return None

    @staticmethod
    def should_renew(session: Session) -> bool:
        """Check if session is within sliding window for renewal."""
        now = datetime.datetime.now()
        time_since_activity = now - session.last_activity
        return time_since_activity < datetime.timedelta(days=SESSION_SLIDING_WINDOW_DAYS)

    @staticmethod
    def renew(session_id: str) -> Tuple[bool, Optional[datetime.datetime]]:
        """Extend session expiration if within sliding window."""
        session = SessionManager.validate(session_id)
        if not session or not SessionManager.should_renew(session):
            return False, None

        now = datetime.datetime.now()
        new_expires_at = now + datetime.timedelta(days=SESSION_MAX_AGE_DAYS)
        session.last_activity = now
        session.expires_at = new_expires_at
        session.save()
        return True, new_expires_at

    @staticmethod
    def delete(session_id: str) -> bool:
        """Remove session."""
        try:
            session = Session.get(Session.session_id == session_id)
            session.delete_instance()
            return True
        except Session.DoesNotExist:
            return False
```

## Remember-Me Tokens

For "stay logged in" functionality. These create a new session when the user returns.

```python
    @staticmethod
    def create_remember_token(user_id: str, **kwargs) -> str:
        """Create long-lived token for persistent login."""
        token_id = generate_secure_token()
        now = datetime.datetime.now()
        expires_at = now + datetime.timedelta(days=REMEMBER_ME_MAX_AGE_DAYS)

        RememberMeToken.create(
            token_id=token_id,
            user_id=user_id,
            created_at=now,
            expires_at=expires_at,
            revoked=False,
            **kwargs
        )
        return token_id

    @staticmethod
    def validate_remember_token(token_id: str) -> Optional[RememberMeToken]:
        """Check if remember-me token is valid."""
        try:
            token = RememberMeToken.get(RememberMeToken.token_id == token_id)
            now = datetime.datetime.now()
            if token.expires_at < now or token.revoked:
                return None
            return token
        except RememberMeToken.DoesNotExist:
            return None
```

## API Key Management

For scripts and automation that need programmatic access.

```python
class ApiKeyManager:
    """Manages API keys for programmatic access."""

    @staticmethod
    def create(
        user_id: str,
        name: str,
        expires_at: Optional[datetime.datetime] = None
    ) -> str:
        """Create API key. Return value shown only once."""
        key = generate_secure_token()
        ApiKey.create(
            api_key=key,
            user_id=user_id,
            name=name,
            created_at=datetime.datetime.now(),
            expires_at=expires_at,
            revoked=False
        )
        return key

    @staticmethod
    def validate(key: str) -> Optional[ApiKey]:
        """Check if API key is valid, update last used timestamp."""
        try:
            api_key = ApiKey.get(ApiKey.api_key == key)
            now = datetime.datetime.now()

            if api_key.revoked:
                return None
            if api_key.expires_at and api_key.expires_at < now:
                return None

            api_key.last_used_at = now
            api_key.save()
            return api_key
        except ApiKey.DoesNotExist:
            return None

    @staticmethod
    def list_for_user(user_id: str) -> list[ApiKey]:
        """List active API keys for user."""
        return list(ApiKey.select().where(
            (ApiKey.user_id == user_id) & (ApiKey.revoked == False)
        ))

    @staticmethod
    def revoke(key: str) -> bool:
        """Revoke an API key."""
        try:
            api_key = ApiKey.get(ApiKey.api_key == key)
            api_key.revoked = True
            api_key.save()
            return True
        except ApiKey.DoesNotExist:
            return False
```

## FastAPI Integration

### Authentication Dependency

Create a dependency that checks multiple auth methods in priority order.

```python
from fastapi import Request, HTTPException, status, Depends
from typing import Tuple

SESSION_COOKIE_NAME = "session"
REMEMBER_ME_COOKIE_NAME = "remember_me"


def get_client_info(request: Request) -> Tuple[Optional[str], Optional[str]]:
    """Extract client IP and user agent for logging."""
    ip = request.client.host if request.client else None
    ua = request.headers.get("user-agent")
    return ip, ua


async def get_current_user(request: Request) -> User:
    """
    Authenticate request and return user.

    Checks in order:
    1. Bearer token in Authorization header
    2. Session cookie
    3. Remember-me cookie (creates new session)
    """
    # Check Authorization header
    auth_header = request.headers.get("authorization")
    if auth_header:
        parts = auth_header.split()
        if len(parts) == 2 and parts[0].lower() == "bearer":
            key = ApiKeyManager.validate(parts[1])
            if key:
                user = UserManager.get_by_id(key.user_id)
                if user:
                    request.state.user = user
                    request.state.auth_method = "api_key"
                    return user

    # Check session cookie
    session_id = request.cookies.get(SESSION_COOKIE_NAME)
    if session_id:
        session = SessionManager.validate(session_id)
        if session:
            user = UserManager.get_by_id(session.user_id)
            if user:
                request.state.session = session
                request.state.user = user
                request.state.auth_method = "session"
                return user

    # Check remember-me cookie
    remember_token = request.cookies.get(REMEMBER_ME_COOKIE_NAME)
    if remember_token:
        token = SessionManager.validate_remember_token(remember_token)
        if token:
            user = UserManager.get_by_id(token.user_id)
            if user:
                ip, ua = get_client_info(request)
                new_session_id = SessionManager.create(
                    user_id=user.id, ip_address=ip, user_agent=ua
                )
                session = SessionManager.validate(new_session_id)
                request.state.session = session
                request.state.user = user
                request.state.new_session_id = new_session_id
                request.state.auth_method = "session"
                return user

    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Not authenticated",
        headers={"WWW-Authenticate": "Bearer"}
    )


async def require_admin(user: User = Depends(get_current_user)) -> User:
    """Require admin privileges."""
    if not user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin privileges required"
        )
    return user
```

### Auth Routes

Standard routes for login, logout, and registration.

```python
from fastapi import APIRouter, Response
from pydantic import BaseModel

router = APIRouter(tags=["auth"])


class LoginRequest(BaseModel):
    username: str
    password: str
    remember_me: bool = False


class RegisterRequest(BaseModel):
    username: str
    password: str
```

### Cookie Configuration

Set secure cookie attributes. Use `httponly` to prevent XSS access, `samesite` to prevent CSRF.

```python
from .config import COOKIE_SECURE  # True in production (HTTPS)


def set_session_cookie(response: Response, session_id: str, expires_at: datetime.datetime):
    """Set session cookie with security attributes."""
    response.set_cookie(
        key=SESSION_COOKIE_NAME,
        value=session_id,
        httponly=True,
        secure=COOKIE_SECURE,
        samesite="lax",
        expires=expires_at.strftime("%a, %d %b %Y %H:%M:%S GMT"),
        path="/"
    )
```

### Login Route

```python
@router.post("/auth/login")
async def login(request: LoginRequest, req: Request, response: Response):
    """Authenticate user and create session."""
    user = UserManager.authenticate(request.username, request.password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    ip, ua = get_client_info(req)
    session_id = SessionManager.create(user.id, ip, ua)
    session = SessionManager.validate(session_id)
    set_session_cookie(response, session_id, session.expires_at)

    if request.remember_me:
        token_id = SessionManager.create_remember_token(user.id, ip_address=ip, user_agent=ua)
        token = SessionManager.validate_remember_token(token_id)
        response.set_cookie(
            key=REMEMBER_ME_COOKIE_NAME,
            value=token_id,
            httponly=True,
            secure=COOKIE_SECURE,
            samesite="lax",
            expires=token.expires_at.strftime("%a, %d %b %Y %H:%M:%S GMT"),
            path="/"
        )

    return {"message": "Login successful", "user_id": user.id, "username": user.username}
```

### Logout Route

```python
@router.post("/auth/logout")
async def logout(request: Request, response: Response, user=Depends(get_current_user)):
    """End session and clear cookies."""
    session_id = request.cookies.get(SESSION_COOKIE_NAME)
    if session_id:
        SessionManager.delete(session_id)
        response.delete_cookie(SESSION_COOKIE_NAME, path="/")

    remember_token = request.cookies.get(REMEMBER_ME_COOKIE_NAME)
    if remember_token:
        # Revoke the remember-me token
        response.delete_cookie(REMEMBER_ME_COOKIE_NAME, path="/")

    return {"message": "Logout successful"}
```

### User Info Route

```python
@router.get("/auth/me")
async def get_me(request: Request, user=Depends(get_current_user)):
    """Get current user info."""
    return {
        "user_id": user.id,
        "username": user.username,
        "is_admin": user.is_admin,
        "auth_method": getattr(request.state, "auth_method", "unknown")
    }
```

### Registration Route

```python
@router.post("/auth/register")
async def register(request: RegisterRequest):
    """Register new user."""
    try:
        user = UserManager.create_user(request.username, request.password)
        return {"message": "User registered", "user_id": user.id, "username": user.username}
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
```

### API Key Routes

```python
class CreateApiKeyRequest(BaseModel):
    name: str
    expires_days: int | None = None


@router.post("/auth/api-keys")
async def create_api_key(request: CreateApiKeyRequest, user=Depends(get_current_user)):
    """Create new API key."""
    expires_at = None
    if request.expires_days:
        expires_at = datetime.datetime.now() + datetime.timedelta(days=request.expires_days)

    key = ApiKeyManager.create(user.id, request.name, expires_at)

    return {
        "api_key": key,
        "name": request.name,
        "expires_at": expires_at.isoformat() if expires_at else None,
        "warning": "Store this key securely. It will not be shown again."
    }


@router.get("/auth/api-keys")
async def list_api_keys(user=Depends(get_current_user)):
    """List API keys (values masked)."""
    keys = ApiKeyManager.list_for_user(user.id)
    return [
        {
            "prefix": key.api_key[:8],
            "name": key.name,
            "created_at": key.created_at.isoformat(),
            "last_used_at": key.last_used_at.isoformat() if key.last_used_at else None,
            "expires_at": key.expires_at.isoformat() if key.expires_at else None
        }
        for key in keys
    ]


@router.delete("/auth/api-keys/{key_prefix}")
async def revoke_api_key(key_prefix: str, user=Depends(get_current_user)):
    """Revoke API key by prefix."""
    keys = ApiKeyManager.list_for_user(user.id)
    for key in keys:
        if key.api_key.startswith(key_prefix):
            ApiKeyManager.revoke(key.api_key)
            return {"message": "API key revoked", "name": key.name}

    raise HTTPException(status_code=404, detail="API key not found")
```

## Configuration

```python
# config.py
import os

# Set True in production (HTTPS), False for local dev (HTTP)
COOKIE_SECURE = os.getenv("COOKIE_SECURE", "false").lower() == "true"
```

## Frontend Integration

JavaScript helpers for browser-based auth.

```javascript
// API client with automatic cookie handling
async function apiRequest(endpoint, options = {}) {
    const response = await fetch(`${API_BASE}${endpoint}`, {
        ...options,
        credentials: 'include',  // Send cookies
        headers: {
            'Content-Type': 'application/json',
            ...options.headers
        }
    });

    if (response.status === 401) {
        window.location.href = '/login';
        throw new Error('Session expired');
    }

    if (!response.ok) {
        const error = await response.json();
        throw new Error(error.detail || 'Request failed');
    }

    return response.json();
}

// Check if user is authenticated
async function checkAuth() {
    try {
        return await apiRequest('/auth/me');
    } catch {
        window.location.href = '/login';
        return null;
    }
}

// Login helper
async function login(username, pw, rememberMe = false) {
    return await apiRequest('/auth/login', {
        method: 'POST',
        body: JSON.stringify({ username, password: pw, remember_me: rememberMe })
    });
}

// Logout helper
async function logout() {
    await apiRequest('/auth/logout', { method: 'POST' });
    window.location.href = '/login';
}
```
