# Security Best Practices Guide

Comprehensive guide to security in Python applications, including input validation, SQL injection prevention, path traversal prevention, password hashing, and secrets management.

---

## Input Validation

**Always validate and sanitize user input before processing.**

### Using Pydantic for Validation

```python
from pydantic import BaseModel, EmailStr, field_validator, ValidationError
from typing import Annotated

class UserInput(BaseModel):
    username: str
    email: EmailStr
    age: int

    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        """Validate username format."""
        if not v.isalnum() or len(v) < 3:
            raise ValueError("Username must be alphanumeric and at least 3 characters")
        if len(v) > 20:
            raise ValueError("Username must be at most 20 characters")
        return v

    @field_validator("age")
    @classmethod
    def validate_age(cls, v: int) -> int:
        """Validate age range."""
        if not 0 < v < 150:
            raise ValueError("Age must be between 1 and 149")
        return v

# Usage
try:
    user = UserInput(username="alice123", email="alice@example.com", age=30)
    print(f"Valid user: {user}")
except ValidationError as e:
    print(f"Validation error: {e}")

# ❌ INVALID - These will raise ValidationError
# UserInput(username="ab", email="invalid", age=25)  # Username too short, invalid email
# UserInput(username="alice!", email="alice@example.com", age=200)  # Invalid chars, age too high
```

### Custom Validation Rules

```python
from pydantic import BaseModel, field_validator, model_validator
from typing import Annotated
import re

class PasswordInput(BaseModel):
    password: str
    password_confirm: str

    @field_validator("password")
    @classmethod
    def validate_password_strength(cls, v: str) -> str:
        """Ensure password meets security requirements."""
        if len(v) < 12:
            raise ValueError("Password must be at least 12 characters")
        
        if not re.search(r'[A-Z]', v):
            raise ValueError("Password must contain uppercase letter")
        
        if not re.search(r'[0-9]', v):
            raise ValueError("Password must contain number")
        
        if not re.search(r'[!@#$%^&*]', v):
            raise ValueError("Password must contain special character")
        
        return v

    @model_validator(mode="after")
    def passwords_match(self) -> "PasswordInput":
        """Validate passwords match."""
        if self.password != self.password_confirm:
            raise ValueError("Passwords do not match")
        return self

# Usage
try:
    pwd = PasswordInput(
        password="SecurePass123!",
        password_confirm="SecurePass123!"
    )
except ValidationError as e:
    print(f"Password validation failed: {e}")
```

### Allowed Values (Enums)

```python
from enum import Enum
from pydantic import BaseModel, field_validator

class UserRole(str, Enum):
    ADMIN = "admin"
    USER = "user"
    GUEST = "guest"

class UserData(BaseModel):
    username: str
    role: UserRole  # Only accepts defined enum values

# Usage
user = UserData(username="alice", role="admin")  # ✅ Valid

# user = UserData(username="bob", role="superadmin")  # ❌ Invalid - not in enum
```

---

## SQL Injection Prevention

**NEVER concatenate user input into SQL queries.**

### ❌ VULNERABLE - String Concatenation

```python
import sqlite3

def get_user_wrong(username: str):
    """VULNERABLE - DO NOT USE."""
    conn = sqlite3.connect("db.sqlite")
    cursor = conn.cursor()
    # ❌ DANGEROUS - User can inject SQL
    cursor.execute(f"SELECT * FROM users WHERE username = '{username}'")
    return cursor.fetchone()

# Attack example:
# get_user_wrong("alice' OR '1'='1")
# Results in: SELECT * FROM users WHERE username = 'alice' OR '1'='1'
# This returns ALL users, not just alice!
```

### ✅ SAFE - Parameterized Queries

```python
import sqlite3

def get_user_safe(username: str):
    """SAFE - Use parameterized queries."""
    conn = sqlite3.connect("db.sqlite")
    cursor = conn.cursor()
    # ✅ SAFE - Parameters are properly escaped
    cursor.execute("SELECT * FROM users WHERE username = ?", (username,))
    return cursor.fetchone()

# Even if user provides: alice' OR '1'='1
# It's treated as a literal string, not SQL code
```

### ✅ SAFE - SQLAlchemy ORM

```python
from sqlalchemy import select, create_engine
from sqlalchemy.orm import Session, declarative_base
from sqlalchemy import Column, String, Integer

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    username = Column(String)

def get_user_sqlalchemy(session: Session, username: str):
    """SAFE - SQLAlchemy ORM handles parameterization."""
    # ✅ SAFE - ORM handles escaping automatically
    stmt = select(User).where(User.username == username)
    return session.scalar(stmt)
```

### Batch Operations

```python
import sqlite3

def update_users_safe(user_ids: list[int], status: str) -> None:
    """SAFE - Use placeholders for multiple values."""
    conn = sqlite3.connect("db.sqlite")
    cursor = conn.cursor()
    
    # ✅ SAFE - Parameterized for list
    placeholders = ",".join("?" * len(user_ids))
    query = f"UPDATE users SET status = ? WHERE id IN ({placeholders})"
    
    params = [status] + user_ids
    cursor.execute(query, params)
    conn.commit()
```

---

## Path Traversal Prevention

**Always validate file paths against a base directory.**

### ❌ VULNERABLE - No Path Validation

```python
from pathlib import Path

def read_user_file_wrong(filename: str) -> str:
    """VULNERABLE - allows path traversal."""
    user_dir = Path("/app/user_files")
    file_path = user_dir / filename
    # ❌ DANGEROUS - User can escape directory with ../
    return file_path.read_text()

# Attack example:
# read_user_file_wrong("../../etc/passwd")
# Results in: /app/user_files/../../etc/passwd = /etc/passwd
# User can read ANY file on the system!
```

### ✅ SAFE - Path Validation

```python
from pathlib import Path

def read_user_file_safe(filename: str, base_dir: Path) -> str:
    """SAFE - Validates path stays within base directory."""
    # Resolve to absolute paths
    file_path = (base_dir / filename).resolve()
    base_dir_resolved = base_dir.resolve()
    
    # Check that file is under base_dir
    if not file_path.is_relative_to(base_dir_resolved):
        raise ValueError(f"Invalid file path - outside allowed directory: {filename}")
    
    if not file_path.exists():
        raise FileNotFoundError(f"File not found: {filename}")
    
    return file_path.read_text()

# Usage
user_dir = Path("/app/user_files")

# ✅ SAFE - Within directory
read_user_file_safe("document.txt", user_dir)

# ❌ BLOCKED - Attempts to escape
try:
    read_user_file_safe("../../etc/passwd", user_dir)
except ValueError as e:
    print(f"Attack prevented: {e}")
```

### Safe File Upload Handler

```python
from pathlib import Path
from fastapi import UploadFile

async def save_uploaded_file_safe(
    upload: UploadFile,
    destination_dir: Path,
    allowed_extensions: set[str] = {".txt", ".pdf", ".jpg"}
) -> Path:
    """Safely save uploaded file."""
    # 1. Validate file extension
    file_ext = Path(upload.filename).suffix.lower()
    if file_ext not in allowed_extensions:
        raise ValueError(f"File type not allowed: {file_ext}")
    
    # 2. Sanitize filename
    safe_filename = Path(upload.filename).name
    
    # 3. Save to validated path
    file_path = (destination_dir / safe_filename).resolve()
    
    # 4. Verify file is within destination
    if not file_path.is_relative_to(destination_dir.resolve()):
        raise ValueError("Invalid file path")
    
    # 5. Write file
    content = await upload.read()
    file_path.write_bytes(content)
    
    return file_path
```

---

## Password Hashing

**NEVER store plaintext passwords. Always use strong hashing algorithms.**

### Using Argon2

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher()

def hash_password(password: str) -> str:
    """Hash password using Argon2."""
    return ph.hash(password)

def verify_password(password: str, hash: str) -> bool:
    """Verify password against hash."""
    try:
        ph.verify(hash, password)
        return True
    except VerifyMismatchError:
        return False

# Usage
# Store hash in database
hashed = hash_password("MySecurePassword123!")
print(f"Hash: {hashed}")

# Later, verify user input
user_input = "MySecurePassword123!"
if verify_password(user_input, hashed):
    print("Password correct - authenticate user")
else:
    print("Password incorrect")

# Add to dependencies:
# dependencies = ["argon2-cffi>=23.1.0"]
```

### Password Requirements

```python
import re
from pydantic import BaseModel, field_validator

class PasswordInput(BaseModel):
    password: str

    @field_validator("password")
    @classmethod
    def validate_password(cls, v: str) -> str:
        """Enforce password requirements."""
        errors = []
        
        if len(v) < 12:
            errors.append("At least 12 characters")
        
        if not re.search(r'[A-Z]', v):
            errors.append("At least one uppercase letter")
        
        if not re.search(r'[a-z]', v):
            errors.append("At least one lowercase letter")
        
        if not re.search(r'[0-9]', v):
            errors.append("At least one number")
        
        if not re.search(r'[!@#$%^&*()_+\-=\[\]{};:"\\|,.<>\/?]', v):
            errors.append("At least one special character")
        
        if errors:
            raise ValueError(f"Password must include: {', '.join(errors)}")
        
        return v
```

---

## Secrets Management

**NEVER hardcode secrets in code. Use environment variables or secret management services.**

### Using Environment Variables

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import Optional

class Settings(BaseSettings):
    """Load configuration from environment variables."""
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    # Required secrets
    database_url: str
    api_key: str
    secret_key: str
    
    # Optional settings with defaults
    debug: bool = False
    log_level: str = "INFO"

# Usage
settings = Settings()

# Access secrets
print(settings.api_key)
print(settings.database_url)

# Add to dependencies:
# dependencies = ["pydantic-settings>=2.0.0"]
```

### .env File

```env
# .env - NEVER commit this to version control

DATABASE_URL=postgresql://user:password@localhost/dbname
API_KEY=your-api-key-here
SECRET_KEY=your-secret-key-here
DEBUG=false
LOG_LEVEL=INFO
```

### .gitignore

```
# Never commit sensitive files
.env
.env.local
.env.*.local
secrets.yaml
```

### Rotating Secrets

```python
from datetime import datetime, timedelta
from pydantic import BaseModel

class SecretRotation(BaseModel):
    """Track secret rotation for compliance."""
    secret_name: str
    created_at: datetime
    rotated_at: datetime | None = None
    expires_at: datetime | None = None
    
    def should_rotate(self) -> bool:
        """Check if secret should be rotated."""
        if self.expires_at is None:
            return False
        return datetime.now() >= self.expires_at
    
    def days_until_expiry(self) -> int:
        """Days until secret expires."""
        if self.expires_at is None:
            return -1
        delta = self.expires_at - datetime.now()
        return delta.days

def rotate_secret(old_secret: str, new_secret: str) -> None:
    """Rotate secrets with minimal downtime."""
    # 1. Update new secret in secret store
    # 2. Update application to accept both old and new
    # 3. Wait for all instances to load new config
    # 4. Remove old secret
    # 5. Verify no errors
    pass
```

---

## HTTPS and Transport Security

**Always use HTTPS for external APIs and web services.**

```python
import httpx
from ssl import create_default_context

# ✅ GOOD - HTTPS is enforced
async with httpx.AsyncClient() as client:
    response = await client.get("https://api.example.com/data")

# ✅ GOOD - Custom SSL context with certificate pinning
import ssl

context = ssl.create_default_context()
context.check_hostname = True
context.verify_mode = ssl.CERT_REQUIRED

async with httpx.AsyncClient(verify=context) as client:
    response = await client.get("https://api.example.com/data")

# ❌ BAD - Disabling SSL verification
# async with httpx.AsyncClient(verify=False) as client:  # DO NOT DO THIS
#     response = await client.get("https://api.example.com/data")

# ❌ BAD - Using HTTP for sensitive data
# response = await client.get("http://api.example.com/data")  # DO NOT DO THIS
```

---

## Authentication and Authorization

**Implement proper authentication and authorization checks.**

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthCredentials
from typing import Annotated
import jwt

security = HTTPBearer()

def get_current_user(credentials: Annotated[HTTPAuthCredentials, Depends(security)]):
    """Verify JWT token and extract user."""
    try:
        payload = jwt.decode(
            credentials.credentials,
            "secret-key",  # Should be from settings
            algorithms=["HS256"]
        )
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
        return user_id
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

def require_admin(user_id: str = Depends(get_current_user)) -> str:
    """Check user has admin role."""
    # Verify user is admin in database
    user = get_user_from_db(user_id)
    if user.role != "admin":
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN)
    return user_id

# Usage in endpoint
@app.post("/admin/users")
async def create_user(user_id: str = Depends(require_admin)):
    """Only admin users can create users."""
    pass
```

---

## Security Rules Summary

**Critical Rules:**
- ✅ ALWAYS validate and sanitize user input
- ✅ ALWAYS use parameterized queries for SQL
- ✅ ALWAYS validate file paths stay within allowed directories
- ✅ ALWAYS hash passwords with Argon2 or bcrypt
- ✅ ALWAYS use HTTPS for external APIs
- ✅ NEVER log sensitive data
- ✅ NEVER hardcode secrets in code
- ✅ NEVER disable SSL verification
- ✅ NEVER trust user input

**Best Practices:**
- Use Pydantic for input validation
- Use environment variables for secrets
- Use strong password hashing (Argon2)
- Implement proper authentication/authorization
- Keep dependencies updated for security patches
- Use HTTPS everywhere
- Rotate secrets regularly
- Log security events
- Use security headers in HTTP responses
- Validate all inputs, even "trusted" sources

---

## Security Testing

```python
import pytest
from pydantic import ValidationError

def test_sql_injection_prevention():
    """Test that SQL injection attempts are blocked."""
    # If using parameterized queries, this should be safe
    user = get_user_safe("alice' OR '1'='1'")
    assert user is None or user.username == "alice' OR '1'='1'"

def test_path_traversal_prevention():
    """Test that path traversal is blocked."""
    from pathlib import Path
    
    with pytest.raises(ValueError):
        read_user_file_safe("../../etc/passwd", Path("/app/files"))

def test_password_validation():
    """Test strong password requirements."""
    from pydantic import ValidationError
    
    # Weak password should fail
    with pytest.raises(ValidationError):
        PasswordInput(password="short")
    
    # Strong password should pass
    pwd = PasswordInput(password="SecurePass123!")
    assert pwd.password == "SecurePass123!"

def test_input_validation():
    """Test input sanitization."""
    with pytest.raises(ValidationError):
        UserInput(
            username="alice!@#",  # Invalid characters
            email="not-an-email",  # Invalid email
            age=200  # Out of range
        )
```

---

## Additional Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Cheat Sheets](https://cheatsheetseries.owasp.org/)
- [Python Security Best Practices](https://python.readthedocs.io/en/latest/library/security_warnings.html)
- [Pydantic Validation](https://docs.pydantic.dev/)
- [Argon2 Documentation](https://argon2-cffi.readthedocs.io/)