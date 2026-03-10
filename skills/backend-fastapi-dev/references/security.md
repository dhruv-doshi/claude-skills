# Security Reference

Security patterns for Python/FastAPI backends. Read when implementing auth, handling user input, processing file uploads, designing error responses, or threat-modeling a new feature.

## Table of Contents

- [Core Philosophy](#core-philosophy)
- [Least Privilege](#least-privilege)
- [Input Validation and Sanitization](#input-validation-and-sanitization)
- [Whitelist Over Blacklist](#whitelist-over-blacklist)
- [Secure File Handling](#secure-file-handling)
- [Secure Error Handling](#secure-error-handling)
- [Threat Modeling](#threat-modeling)

---

## Core Philosophy

### Secure by Design

Integrate security from the start, not as a retrofit. Before writing code:

1. Document the threat model
2. Identify attack surfaces
3. Define security requirements alongside functional requirements
4. Review design decisions for security implications

### Defense in Depth

Never rely on a single control. Layer multiple defenses so that if one fails, others protect the system: authentication + authorization + input validation + output encoding. Network segmentation + rate limiting + encryption at rest and in transit.

---

## Least Privilege

Users, services, and workflows get exactly the minimum permissions required — nothing more.

### Role-Based Access Control

```python
from enum import Enum

class Permission(str, Enum):
    READ_OWN = "read:own"
    READ_ANY = "read:any"
    WRITE_OWN = "write:own"
    WRITE_ANY = "write:any"
    DELETE_OWN = "delete:own"
    DELETE_ANY = "delete:any"
    ADMIN = "admin"

class Role(str, Enum):
    USER = "user"
    MODERATOR = "moderator"
    ADMIN = "admin"

ROLE_PERMISSIONS: dict[Role, set[Permission]] = {
    Role.USER: {Permission.READ_OWN, Permission.WRITE_OWN, Permission.DELETE_OWN},
    Role.MODERATOR: {Permission.READ_OWN, Permission.READ_ANY, Permission.WRITE_OWN},
    Role.ADMIN: set(Permission),
}
```

### Resource-Level Authorization

Always verify ownership or elevated permission before returning data:

```python
async def get_document(
    document_id: str,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> Document:
    document = await db.get(Document, document_id)
    if not document:
        raise HTTPException(status_code=404)

    if document.owner_id != current_user.id:
        if Permission.READ_ANY not in current_user.permissions:
            raise HTTPException(status_code=403, detail="Access denied")

    return document
```

### Privilege Escalation Auditing

Log every role change with who did it, why, and from where:

```python
audit_log = AuditLog(
    action="PRIVILEGE_ESCALATION",
    target_user_id=user_id,
    previous_role=user.role,
    new_role=new_role,
    reason=reason,
    granted_by_id=admin.id,
    timestamp=datetime.utcnow(),
    ip_address=request.client.host,
)
```

### Service-to-Service Scoping

```python
SERVICE_SCOPES = {
    "user-service": ["users:read", "users:write"],
    "billing-service": ["users:read", "payments:*"],
    "analytics-service": ["events:write"],  # Write-only — cannot read user data
}
```

---

## Input Validation and Sanitization

All input is untrusted. Validate structure, type, length, format. Sanitize to prevent injection.

### Pydantic with Strict Validation

```python
from pydantic import BaseModel, Field, validator, EmailStr

class CreateUserRequest(BaseModel):
    email: EmailStr
    username: str = Field(..., min_length=3, max_length=30, pattern=r"^[a-zA-Z0-9_]+$")
    bio: str | None = Field(default=None, max_length=500)

    @validator("username")
    def username_not_reserved(cls, v):
        reserved = {"admin", "root", "system", "api", "null", "undefined"}
        if v.lower() in reserved:
            raise ValueError("Username is reserved")
        return v

    @validator("bio")
    def sanitize_bio(cls, v):
        if v is None:
            return v
        return bleach.clean(v, tags=[], strip=True)
```

### SQL Injection Prevention

```python
# NEVER — string interpolation
query = f"SELECT * FROM users WHERE id = '{user_id}'"  # VULNERABLE

# ALWAYS — parameterized
result = await db.execute(select(User).where(User.id == user_id))

# Raw SQL when needed — still parameterized
result = await db.execute(
    text("SELECT * FROM users WHERE id = :user_id"),
    {"user_id": user_id},
)
```

### XSS Prevention

```python
import html

def render_user_content(content: str) -> str:
    return html.escape(content)

# Jinja2 auto-escapes by default, but be explicit:
# {{ user_input | e }}
```

### Command Injection Prevention

```python
import subprocess

# NEVER — shell=True with user input
subprocess.run(f"convert {filename}", shell=True)  # VULNERABLE

# ALWAYS — argument list, no shell
subprocess.run(["convert", validated_filename], shell=False, capture_output=True)
```

### Path Traversal Prevention

```python
from pathlib import Path

UPLOAD_DIR = Path("/app/uploads")

def safe_file_path(filename: str) -> Path:
    safe_name = Path(filename).name          # Strip directory components
    full_path = (UPLOAD_DIR / safe_name).resolve()

    if not full_path.is_relative_to(UPLOAD_DIR):
        raise ValueError("Invalid file path")

    return full_path
```

---

## Whitelist Over Blacklist

Explicitly allow known-good values. Blacklists are incomplete by definition.

### Allowed File Types

```python
ALLOWED_EXTENSIONS = {".pdf", ".png", ".jpg", ".jpeg", ".gif", ".txt"}
ALLOWED_MIME_TYPES = {
    "application/pdf",
    "image/png",
    "image/jpeg",
    "image/gif",
    "text/plain",
}

def validate_upload(file: UploadFile) -> bool:
    ext = Path(file.filename).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(400, f"File type {ext} not allowed")
    if file.content_type not in ALLOWED_MIME_TYPES:
        raise HTTPException(400, "Invalid file type")
    return True
```

### Sandboxed Code Execution

```python
SAFE_BUILTINS = {
    "abs", "all", "any", "bool", "dict", "enumerate",
    "filter", "float", "int", "len", "list", "map",
    "max", "min", "print", "range", "round", "set",
    "sorted", "str", "sum", "tuple", "zip",
}

SAFE_MODULES = {"math", "json", "datetime", "collections"}

def create_sandbox():
    safe_globals = {
        "__builtins__": {k: getattr(__builtins__, k) for k in SAFE_BUILTINS},
    }
    for mod in SAFE_MODULES:
        safe_globals[mod] = __import__(mod)
    return safe_globals
```

### Endpoint Allowlisting

```python
ALLOWED_ENDPOINTS = {"/api/v1/users", "/api/v1/products", "/api/v1/orders"}

def validate_proxy_target(path: str) -> bool:
    return path.rstrip("/").lower() in ALLOWED_ENDPOINTS
```

---

## Secure File Handling

File uploads are a significant attack vector. Validate, sanitize, and isolate.

### Comprehensive Upload Validation

```python
import magic
import hashlib

MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB

async def secure_upload(file: UploadFile) -> str:
    # 1. Size check
    contents = await file.read()
    if len(contents) > MAX_FILE_SIZE:
        raise HTTPException(400, "File too large")

    # 2. Extension check
    ext = Path(file.filename).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(400, "File type not allowed")

    # 3. Actual MIME check (don't trust headers)
    detected_mime = magic.from_buffer(contents, mime=True)
    if detected_mime not in ALLOWED_MIME_TYPES:
        raise HTTPException(400, "File content type not allowed")

    # 4. Extension must match content
    if ALLOWED_MIME_TYPES.get(detected_mime) != ext:
        raise HTTPException(400, "Extension does not match content")

    # 5. Generate safe filename — never use user-provided name
    file_hash = hashlib.sha256(contents).hexdigest()[:16]
    safe_filename = f"{file_hash}{ext}"

    # 6. Store outside web root
    (Path("/app/uploads") / safe_filename).write_bytes(contents)
    return safe_filename
```

### Directory Traversal Prevention

```python
def get_user_file(user_id: str, filename: str) -> Path:
    user_dir = UPLOAD_ROOT / str(user_id)
    requested = (user_dir / filename).resolve()

    if not requested.is_relative_to(user_dir):
        raise HTTPException(403, "Access denied")
    if not requested.exists():
        raise HTTPException(404, "File not found")

    return requested
```

### Executable Prevention

```python
DANGEROUS_EXTENSIONS = {
    ".exe", ".dll", ".so", ".sh", ".bat", ".cmd",
    ".ps1", ".vbs", ".js", ".py", ".php", ".asp",
    ".jsp", ".jar", ".war", ".html", ".htm", ".svg",
}

def is_safe_filename(filename: str) -> bool:
    # Check all suffixes — catches double extensions like file.pdf.exe
    return not any(s.lower() in DANGEROUS_EXTENSIONS for s in Path(filename).suffixes)
```

---

## Secure Error Handling

Error messages should help users without helping attackers. Never expose internals.

### Global Exception Handler

```python
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    error_id = str(uuid.uuid4())

    # Full details for internal logs
    logger.error(
        f"Unhandled exception [error_id={error_id}]",
        exc_info=exc,
        extra={"error_id": error_id, "path": request.url.path},
    )

    # Safe message for client
    return JSONResponse(
        status_code=500,
        content={
            "error": {
                "code": "INTERNAL_ERROR",
                "message": "An unexpected error occurred",
                "error_id": error_id,
            }
        },
    )
```

### Custom Exception Classes

```python
class AppException(Exception):
    status_code: int = 500
    error_code: str = "INTERNAL_ERROR"
    public_message: str = "An error occurred"

    def __init__(self, internal_message: str | None = None):
        self.internal_message = internal_message
        super().__init__(internal_message or self.public_message)

class NotFoundError(AppException):
    status_code = 404
    error_code = "NOT_FOUND"
    public_message = "Resource not found"

class AuthenticationError(AppException):
    status_code = 401
    error_code = "AUTHENTICATION_FAILED"
    public_message = "Authentication failed"  # Don't say "wrong password"

class RateLimitError(AppException):
    status_code = 429
    error_code = "RATE_LIMITED"
    public_message = "Too many requests"
```

### Safe vs Unsafe Messages

```python
# UNSAFE — reveals internals
"Database connection failed: PostgreSQL at db.internal:5432 refused"
"User admin@company.com not found in table users"
"SQL syntax error near 'SELECT * FROM users WHERE'"

# SAFE — generic, actionable
"Service temporarily unavailable"
"Invalid credentials"
"Resource not found"
"Invalid request"
```

### Debug Mode Protection

```python
if settings.environment == "production" and settings.debug:
    raise RuntimeError("Debug mode cannot be enabled in production")

def get_error_detail(exc: Exception) -> dict:
    base = {"code": "ERROR", "message": "An error occurred"}
    if settings.debug:
        base["debug"] = {"type": type(exc).__name__, "detail": str(exc)}
    return base
```

---

## Threat Modeling

Document before starting any feature that handles sensitive data, exposes new endpoints, or changes auth flows.

### Template

```markdown
## Threat Model: [Feature Name]

### Assets
What are we protecting? (user data, credentials, system access)

### Trust Boundaries
Where does trusted become untrusted? (API gateway, DB, external services)

### Entry Points
How can attackers interact? (API endpoints, file uploads, webhooks)

### Threats (STRIDE)
- **S**poofing: Can attackers impersonate users/services?
- **T**ampering: Can data be modified in transit/at rest?
- **R**epudiation: Can actions be denied? Do we have audit logs?
- **I**nformation Disclosure: Can sensitive data leak?
- **D**enial of Service: Can the service be overwhelmed?
- **E**levation of Privilege: Can users gain unauthorized access?

### Mitigations
| Threat | Mitigation | Status |
|--------|------------|--------|
| SQL Injection | Parameterized queries | Implemented |
| XSS | Output encoding, CSP | Implemented |

### Residual Risks
Accepted risks with justification.
```