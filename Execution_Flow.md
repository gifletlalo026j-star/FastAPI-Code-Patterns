# Part 2 – Execution Flow

## Overview

This document traces what happens when a client sends a request to the following endpoint:

```http
GET /admin/users/
```

The endpoint is protected by authentication and administrator authorization.

The request passes through middleware, dependencies, authentication, authorization, and finally the endpoint logic.

---

# High-Level Request Flow

```text
Client
   │
   │ GET /admin/users/
   ▼
TimingMiddleware
   │
   ▼
FastAPI Routing
   │
   ▼
Authentication Dependency
   │
   ▼
OAuth2PasswordBearer
   │
   ▼
JWT Token
   │
   ▼
get_current_user()
   │
   ▼
Decode JWT
   │
   ▼
Find User
   │
   ▼
Check Admin Role
   │
   ├── Not Admin → 403 Forbidden
   │
   ▼
Database Dependency
   │
   ▼
get_db()
   │
   ▼
UserRepository.list()
   │
   ▼
Database Query
   │
   ▼
Return Users
   │
   ▼
TimingMiddleware
   │
   ▼
Add X-Process-Time Header
   │
   ▼
HTTP Response
```

---

# Step-by-Step Execution

## Step 1 – Client Sends Request

A client sends a request to:

```http
GET /admin/users/
```

The client must provide a valid authentication token.

The request is intended to retrieve a list of users.

---

## Step 2 – Timing Middleware

The request passes through the application's middleware.

The `TimingMiddleware` records the start time:

```python
start_time = datetime.utcnow()
```

It then passes the request to the next stage:

```python
response = await call_next(request)
```

The middleware waits for the request to finish before continuing.

---

## Step 3 – FastAPI Finds the Endpoint

FastAPI matches the request to:

```python
@app.get("/admin/users/", response_model=List[UserSchema])
```

The endpoint function is:

```python
async def list_users(...)
```

Before executing this function, FastAPI resolves its dependencies.

---

# Step 4 – Authentication

The endpoint requires:

```python
current_user: User = Depends(get_current_user)
```

FastAPI calls:

```python
get_current_user()
```

The function itself depends on:

```python
token: str = Depends(oauth2_scheme)
```

and:

```python
db: AsyncSession = Depends(get_db)
```

---

## Step 5 – OAuth2 Token Extraction

The `OAuth2PasswordBearer` dependency extracts the bearer token from the HTTP Authorization header.

The request is expected to contain something similar to:

```http
Authorization: Bearer <JWT_TOKEN>
```

The token is passed to `get_current_user()`.

---

## Step 6 – JWT Validation

The application attempts to decode the JWT:

```python
payload = jwt.decode(
    token,
    "SECRET_KEY",
    algorithms=["HS256"]
)
```

The username is extracted from the token:

```python
username = payload.get("sub")
```

If the token is invalid or does not contain a username, the application raises:

```text
401 Unauthorized
```

---

## Step 7 – Find the Current User

A `UserRepository` is created:

```python
user_repo = UserRepository(User)
```

The repository searches for the user:

```python
user = await user_repo.get_by_username(db, username)
```

If the user cannot be found, the application returns:

```text
401 Unauthorized
```

If the user exists, the user object is returned.

---

# Step 8 – Authorization

The endpoint is protected by:

```python
@requires_role("admin")
```

The decorator checks the user's permissions.

The code verifies:

```python
current_user.is_superuser
```

If the user is not a superuser, the request is rejected with:

```text
403 Forbidden
```

If the user is an administrator, execution continues.

---

# Step 9 – Database Dependency

The endpoint also requires:

```python
db: AsyncSession = Depends(get_db)
```

FastAPI provides the database session.

The `get_db()` dependency creates a session and ensures that it is closed after the request.

---

# Step 10 – Create User Repository

The endpoint creates:

```python
user_repo = UserRepository(User)
```

This repository provides user-specific database operations.

---

# Step 11 – Retrieve Users

The endpoint calls:

```python
users = await user_repo.list(
    db,
    skip=skip,
    limit=limit
)
```

The repository executes a database query.

The default values are:

```text
skip = 0
limit = 10
```

This means the endpoint returns up to ten users by default.

---

# Step 12 – Return Response

The endpoint returns:

```python
return users
```

FastAPI converts the database objects into the response model:

```python
List[UserSchema]
```

The response contains user information such as:

- ID
- Username
- Email
- Active status

Sensitive information such as the hashed password is not included in `UserSchema`.

---

# Step 13 – Middleware Completes

After the endpoint finishes, control returns to the `TimingMiddleware`.

The middleware calculates the processing time:

```python
process_time = (
    datetime.utcnow() - start_time
).total_seconds() * 1000
```

It adds the processing time to the response header:

```http
X-Process-Time: <processing time>
```

The response is then returned to the client.

---

# Complete Request Flow

```text
┌─────────────────┐
│     Client      │
│ GET /admin/users│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ TimingMiddleware│
│ Start Timer     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ FastAPI Routing │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ OAuth2 Bearer   │
│ Extract Token   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ get_current_user│
│ Decode JWT      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ UserRepository  │
│ Find User       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Admin Check     │
└──────┬─────┬────┘
       │     │
     No│     │Yes
       ▼     ▼
  403 Error  │
             ▼
      ┌──────────────┐
      │ get_db()     │
      │ DB Session   │
      └──────┬───────┘
             │
             ▼
      ┌──────────────┐
      │ UserRepository│
      │ list()       │
      └──────┬───────┘
             │
             ▼
      ┌──────────────┐
      │   Database   │
      └──────┬───────┘
             │
             ▼
      ┌──────────────┐
      │ User List    │
      └──────┬───────┘
             │
             ▼
      ┌──────────────┐
      │ UserSchema   │
      │ Response     │
      └──────┬───────┘
             │
             ▼
      ┌──────────────┐
      │ Timing Header│
      └──────┬───────┘
             │
             ▼
           Client
```

---

# Authentication vs Authorization

It is important to distinguish between these two processes.

## Authentication

Authentication verifies the identity of the user.

```text
JWT Token
    ↓
Decode Token
    ↓
Get Username
    ↓
Find User
```

The result is the authenticated `User`.

---

## Authorization

Authorization determines whether the authenticated user has permission to access the resource.

```text
Authenticated User
        ↓
Check is_superuser
        ↓
      Admin?
     /      \
   Yes       No
   ↓         ↓
Allow      403
```

---

# Potential Failure Points

Several things can cause the request to fail.

### 401 Unauthorized

Possible causes:

- Missing token
- Invalid token
- Expired token
- Missing username in token
- User does not exist

### 403 Forbidden

The user is authenticated but does not have administrator privileges.

### Database Failure

The database query may fail or the database may be unavailable.

### Invalid Request

Invalid query parameters could cause validation errors.

---

# Summary

The `/admin/users/` request passes through several layers before returning a response.

The main flow is:

```text
Client
  ↓
Middleware
  ↓
FastAPI Router
  ↓
Authentication
  ↓
Authorization
  ↓
Database Dependency
  ↓
Repository
  ↓
Database
  ↓
Response Model
  ↓
Middleware
  ↓
Client
```

This layered approach separates concerns and allows authentication, authorization, database access, and request monitoring to be handled independently.
