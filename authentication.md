# Authentication Patterns

Two-factor authentication: session-based (browser) + API keys (programmatic).

## Overview

Every authenticated app should support:
1. **Session cookies** - For browser/interactive use with sliding expiration
2. **API keys** - For programmatic access via Bearer tokens
3. **Remember-me tokens** - For longer-lived browser sessions

## Database Models

```python
"""Auth models using Peewee."""

from datetime import datetime
from peewee import (
    SqliteDatabase, Model, CharField, TextField,
    BooleanField, DateTimeField, ForeignKeyField
)

from .config import DATABASE_PATH


db = SqliteDatabase(str(DATABASE_PATH))


class BaseModel(Model):
    class Meta:
        database = db


class User(BaseModel):
    """User account."""
    id = CharField(primary_key=True, max_length=32)
    username = CharField(max_length=255, unique=True)
    password = CharField(max_length=255)  # bcrypt hash
    is_admin = BooleanField(default=False)
    created_at = DateTimeField(default=datetime.now)


class Session(BaseModel):
    """Active session with sliding expiration."""
    session_id = CharField(primary_key=True, max_length=64)
    user_id = CharField(max_length=32)
    created_at = DateTimeField()
    last_activity = DateTimeField()
    expires_at = DateTimeField()
    ip_address = CharField(max_length=45, null=True)
    user_agent = TextField(null=True)


class RememberMeToken(BaseModel):
    """Long-lived remember-me token."""
    token_id = CharField(primary_key=True, max_length=64)
    user_id = CharField(max_length=32)
    created_at = DateTimeField()
    expires_at = DateTimeField()
    ip_address = CharField(max_length=45, null=True)
    user_agent = TextField(null=True)
    revoked = BooleanField(default=False)


class ApiKey(BaseModel):
    """API key for programmatic access."""
    api_key = CharField(primary_key=True, max_length=64)
    user_id = CharField(max_length=32)
    name = CharField(max_length=255)
    created_at = DateTimeField()
    expires_at = DateTimeField(null=True)
    last_used_at = DateTimeField(null=True)
    revoked = BooleanField(default=False)


def init_db():
    db.connect(reuse_if_open=True)
    db.create_tables([User, Session, RememberMeToken, ApiKey], safe=True)


init_db()
```

## Auth Module (auth.py)

```python
"""
Session-based authentication with sliding expiration and remember-me functionality.

Implements secure HTTP-only session cookies with:
- Sliding expiration (resets on each user interaction)
- ITP-safe sliding window (< 7 days)
- Remember-me tokens for longer-lived authentication
- Secure, HttpOnly, SameSite=Lax cookies
"""

import datetime
import secrets
from typing import Optional, Tuple
from passlib.context import CryptContext

from .models import User, Session, RememberMeToken, ApiKey


# Password hashing with bcrypt
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


# Session configuration (ITP-safe: < 7 days for sliding window)
SESSION_SLIDING_WINDOW_DAYS = 7
SESSION_MAX_AGE_DAYS = 30
REMEMBER_ME_MAX_AGE_DAYS = 90


def generate_secure_token(length: int = 32) -> str:
    """Generate cryptographically secure random token."""
    return secrets.token_urlsafe(length)


def hash_password(password: str) -> str:
    """Hash password using bcrypt."""
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password against bcrypt hash."""
    return pwd_context.verify(plain_password, hashed_password)


class UserManager:
    """Manages user accounts."""

    @staticmethod
    def create_user(username: str, password: str, is_admin: bool = False) -> User:
        """Create user with hashed password."""
        user_id = generate_secure_token(16)
        hashed = hash_password(password)
        return User.create(
            id=user_id,
            username=username,
            password=hashed,
            is_admin=is_admin
        )

    @staticmethod
    def authenticate_user(username: str, password: str) -> Optional[User]:
        """Authenticate by username/password."""
        try:
            user = User.get(User.username == username)
            if verify_password(password, user.password):
                return user
            return None
        except User.DoesNotExist:
            return None

    @staticmethod
    def get_user_by_id(user_id: str) -> Optional[User]:
        """Get user by ID."""
        try:
            return User.get(User.id == user_id)
        except User.DoesNotExist:
            return None


class SessionManager:
    """Manages sessions with sliding expiration."""

    @staticmethod
    def create_session(
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
    def validate_session(session_id: str) -> Optional[Session]:
        """Validate session, return None if expired."""
        try:
            session = Session.get(Session.session_id == session_id)
            if session.expires_at < datetime.datetime.now():
                SessionManager.delete_session(session_id)
                return None
            return session
        except Session.DoesNotExist:
            return None

    @staticmethod
    def should_renew_session(session: Session) -> bool:
        """Check if session should be renewed (ITP-safe sliding window)."""
        now = datetime.datetime.now()
        time_since_activity = now - session.last_activity
        return time_since_activity < datetime.timedelta(days=SESSION_SLIDING_WINDOW_DAYS)

    @staticmethod
    def renew_session(session_id: str) -> Tuple[bool, Optional[datetime.datetime]]:
        """Renew session, return (renewed, new_expires_at)."""
        session = SessionManager.validate_session(session_id)
        if not session or not SessionManager.should_renew_session(session):
            return False, None

        now = datetime.datetime.now()
        new_expires_at = now + datetime.timedelta(days=SESSION_MAX_AGE_DAYS)
        session.last_activity = now
        session.expires_at = new_expires_at
        session.save()
        return True, new_expires_at

    @staticmethod
    def delete_session(session_id: str) -> bool:
        """Delete session."""
        try:
            session = Session.get(Session.session_id == session_id)
            session.delete_instance()
            return True
        except Session.DoesNotExist:
            return False

    @staticmethod
    def create_remember_me_token(user_id: str, **kwargs) -> str:
        """Create remember-me token for longer-lived auth."""
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
    def validate_remember_me_token(token_id: str) -> Optional[RememberMeToken]:
        """Validate remember-me token."""
        try:
            token = RememberMeToken.get(RememberMeToken.token_id == token_id)
            now = datetime.datetime.now()
            if token.expires_at < now or token.revoked:
                return None
            return token
        except RememberMeToken.DoesNotExist:
            return None


class ApiKeyManager:
    """Manages API keys for programmatic access."""

    @staticmethod
    def create_api_key(
        user_id: str,
        name: str,
        expires_at: Optional[datetime.datetime] = None
    ) -> str:
        """Create API key, return the key (show only once!)."""
        api_key = generate_secure_token()
        ApiKey.create(
            api_key=api_key,
            user_id=user_id,
            name=name,
            created_at=datetime.datetime.now(),
            expires_at=expires_at,
            revoked=False
        )
        return api_key

    @staticmethod
    def validate_api_key(api_key: str) -> Optional[ApiKey]:
        """Validate API key, update last_used_at."""
        try:
            key = ApiKey.get(ApiKey.api_key == api_key)
            now = datetime.datetime.now()

            if key.revoked:
                return None
            if key.expires_at and key.expires_at < now:
                return None

            key.last_used_at = now
            key.save()
            return key
        except ApiKey.DoesNotExist:
            return None

    @staticmethod
    def list_user_api_keys(user_id: str) -> list[ApiKey]:
        """List non-revoked API keys for user."""
        return list(ApiKey.select().where(
            (ApiKey.user_id == user_id) & (ApiKey.revoked == False)
        ))

    @staticmethod
    def revoke_api_key(api_key: str) -> bool:
        """Revoke an API key."""
        try:
            key = ApiKey.get(ApiKey.api_key == api_key)
            key.revoked = True
            key.save()
            return True
        except ApiKey.DoesNotExist:
            return False
```

## FastAPI Dependencies

```python
"""FastAPI auth dependencies."""

from typing import Optional, Tuple
from fastapi import Request, HTTPException, status, Depends

from .auth import SessionManager, UserManager, ApiKeyManager
from .models import User


SESSION_COOKIE_NAME = "session"
REMEMBER_ME_COOKIE_NAME = "remember_me"


def get_client_info(request: Request) -> Tuple[Optional[str], Optional[str]]:
    """Extract client IP and user agent."""
    ip = request.client.host if request.client else None
    ua = request.headers.get("user-agent")
    return ip, ua


async def get_current_user(request: Request) -> User:
    """
    Get current authenticated user.

    Checks in order:
    1. API key in Authorization header (Bearer token)
    2. Session cookie
    3. Remember-me token (creates new session if valid)
    """
    # 1. Check API key
    auth_header = request.headers.get("authorization")
    if auth_header:
        parts = auth_header.split()
        if len(parts) == 2 and parts[0].lower() == "bearer":
            api_key = parts[1]
            key = ApiKeyManager.validate_api_key(api_key)
            if key:
                user = UserManager.get_user_by_id(key.user_id)
                if user:
                    request.state.user = user
                    request.state.auth_method = "api_key"
                    return user

    # 2. Check session cookie
    session_id = request.cookies.get(SESSION_COOKIE_NAME)
    if session_id:
        session = SessionManager.validate_session(session_id)
        if session:
            user = UserManager.get_user_by_id(session.user_id)
            if user:
                request.state.session = session
                request.state.user = user
                request.state.auth_method = "session"
                return user

    # 3. Check remember-me token
    remember_token = request.cookies.get(REMEMBER_ME_COOKIE_NAME)
    if remember_token:
        token = SessionManager.validate_remember_me_token(remember_token)
        if token:
            user = UserManager.get_user_by_id(token.user_id)
            if user:
                # Create new session from remember-me
                ip, ua = get_client_info(request)
                new_session_id = SessionManager.create_session(
                    user_id=user.id, ip_address=ip, user_agent=ua
                )
                session = SessionManager.validate_session(new_session_id)
                request.state.session = session
                request.state.user = user
                request.state.new_session_id = new_session_id
                request.state.session_from_remember_me = True
                request.state.auth_method = "session"
                return user

    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Not authenticated",
        headers={"WWW-Authenticate": "Bearer"}
    )


async def get_current_admin(user: User = Depends(get_current_user)) -> User:
    """Require admin privileges."""
    if not user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin privileges required"
        )
    return user
```

## Auth Routes

```python
"""Auth routes."""

import datetime
from fastapi import APIRouter, Request, Response, HTTPException, Depends, status
from pydantic import BaseModel

from .auth import SessionManager, UserManager, ApiKeyManager
from .dependencies import get_current_user, get_client_info
from .config import COOKIE_SECURE


router = APIRouter(tags=["auth"])

SESSION_COOKIE_NAME = "session"
REMEMBER_ME_COOKIE_NAME = "remember_me"


class LoginRequest(BaseModel):
    username: str
    password: str
    remember_me: bool = False


class RegisterRequest(BaseModel):
    username: str
    password: str


class CreateApiKeyRequest(BaseModel):
    name: str
    expires_days: int | None = None


def set_session_cookie(response: Response, session_id: str, expires_at: datetime.datetime):
    """Set session cookie with secure attributes."""
    response.set_cookie(
        key=SESSION_COOKIE_NAME,
        value=session_id,
        httponly=True,
        secure=COOKIE_SECURE,
        samesite="lax",
        expires=expires_at.strftime("%a, %d %b %Y %H:%M:%S GMT"),
        path="/"
    )


@router.post("/auth/register")
async def register(request: RegisterRequest):
    """Register new user."""
    try:
        user = UserManager.create_user(request.username, request.password)
        return {"message": "User registered", "user_id": user.id, "username": user.username}
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.post("/auth/login")
async def login(request: LoginRequest, req: Request, response: Response):
    """Login and create session."""
    user = UserManager.authenticate_user(request.username, request.password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    ip, ua = get_client_info(req)
    session_id = SessionManager.create_session(user.id, ip, ua)
    session = SessionManager.validate_session(session_id)
    set_session_cookie(response, session_id, session.expires_at)

    if request.remember_me:
        token_id = SessionManager.create_remember_me_token(user.id, ip_address=ip, user_agent=ua)
        token = SessionManager.validate_remember_me_token(token_id)
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


@router.post("/auth/logout")
async def logout(request: Request, response: Response, user=Depends(get_current_user)):
    """Logout - delete session and cookies."""
    session_id = request.cookies.get(SESSION_COOKIE_NAME)
    if session_id:
        SessionManager.delete_session(session_id)
        response.delete_cookie(SESSION_COOKIE_NAME, path="/")

    remember_token = request.cookies.get(REMEMBER_ME_COOKIE_NAME)
    if remember_token:
        SessionManager.revoke_remember_me_token(remember_token)
        response.delete_cookie(REMEMBER_ME_COOKIE_NAME, path="/")

    return {"message": "Logout successful"}


@router.get("/auth/me")
async def get_me(request: Request, user=Depends(get_current_user)):
    """Get current user info."""
    return {
        "user_id": user.id,
        "username": user.username,
        "is_admin": user.is_admin,
        "auth_method": getattr(request.state, "auth_method", "unknown")
    }


@router.post("/auth/api-keys")
async def create_api_key(request: CreateApiKeyRequest, user=Depends(get_current_user)):
    """Create new API key."""
    expires_at = None
    if request.expires_days:
        expires_at = datetime.datetime.now() + datetime.timedelta(days=request.expires_days)

    api_key = ApiKeyManager.create_api_key(user.id, request.name, expires_at)

    return {
        "api_key": api_key,
        "name": request.name,
        "expires_at": expires_at.isoformat() if expires_at else None,
        "warning": "Store this key securely. It will not be shown again."
    }


@router.get("/auth/api-keys")
async def list_api_keys(user=Depends(get_current_user)):
    """List API keys (values masked)."""
    keys = ApiKeyManager.list_user_api_keys(user.id)
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
    keys = ApiKeyManager.list_user_api_keys(user.id)
    for key in keys:
        if key.api_key.startswith(key_prefix):
            ApiKeyManager.revoke_api_key(key.api_key)
            return {"message": "API key revoked", "name": key.name}

    raise HTTPException(status_code=404, detail="API key not found")
```

## Cookie Configuration

```python
# In config.py
import os

# Cookie security - False for local HTTP dev, True for production HTTPS
COOKIE_SECURE = os.getenv("COOKIE_SECURE", "false").lower() == "true"
```

## Frontend Auth (JavaScript)

```javascript
// Check auth on page load
async function checkAuth() {
    try {
        const user = await apiRequest('/auth/me');
        return user;
    } catch {
        window.location.href = '/login';
        return null;
    }
}

// Login
async function login(username, password, rememberMe = false) {
    return await apiRequest('/auth/login', {
        method: 'POST',
        body: JSON.stringify({ username, password, remember_me: rememberMe })
    });
}

// Logout
async function logout() {
    await apiRequest('/auth/logout', { method: 'POST' });
    window.location.href = '/login';
}

// API requests automatically include credentials
async function apiRequest(endpoint, options = {}) {
    const response = await fetch(`${API_BASE}${endpoint}`, {
        ...options,
        credentials: 'include',  // Include cookies
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
```
