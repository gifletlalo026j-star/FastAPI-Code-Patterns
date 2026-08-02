# Part 1 – FastAPI Code Analysis

## Introduction

This document explains the main design patterns and advanced concepts used in the FastAPI code provided for this exercise.

The main concepts explored are:

- Repository Pattern
- Generic Repository
- Dependency Injection
- Role-Based Access Control
- Service Layer
- JWT Authentication
- Middleware

---

# 1. Repository Pattern

The code uses a Repository Pattern to separate database operations from the rest of the application.

The generic `Repository[T]` class provides reusable database operations such as:

- Getting a record by ID
- Listing records
- Other CRUD operations

For example:

```python
class Repository(Generic[T]):
    def __init__(self, model: Type[T]):
        self.model = model
```

The repository receives a model and uses it when interacting with the database.

The `UserRepository` then extends the generic repository:

```python
class UserRepository(Repository[User]):
```

It adds functionality specific to users, such as:

```python
async def get_by_username(...)
```

### Why use this pattern?

The Repository Pattern separates database access from business logic.

This means the application can:

- Reuse database operations.
- Keep database queries in one place.
- Make the service layer easier to understand.
- Make the code easier to test and maintain.
- Create specialized repositories for different models.

The repository acts as a layer between the application logic and the database.

---

# 2. Generic[T]

The code defines:

```python
T = TypeVar('T', bound=Base)
```

The repository then uses:

```python
class Repository(Generic[T]):
```

`T` represents a generic model type.

This allows the same repository structure to work with different database models.

For example:

```python
Repository[User]
```

means the repository is working with the `User` model.

### Why is this useful?

Without generics, developers might need to create a completely separate repository class for every model.

With `Generic[T]`, common operations can be reused.

This improves:

- Code reuse
- Maintainability
- Type safety
- Consistency

The specialized `UserRepository` can then add user-specific operations without rewriting the common repository functionality.

---

# 3. Dependency Injection

FastAPI uses dependency injection to provide resources to endpoint functions.

For example:

```python
db: AsyncSession = Depends(get_db)
```

This tells FastAPI to call `get_db()` and provide the resulting database session to the endpoint.

The database dependency is defined as:

```python
async def get_db() -> AsyncSession:
    async with AsyncSession() as session:
        try:
            yield session
        finally:
            await session.close()
```

The session is created for the request and closed afterwards.

---

## Dependency Layers

The application uses several dependency layers.

### Layer 1 – Database Dependency

```text
Endpoint
   ↓
Depends(get_db)
   ↓
AsyncSession
```

The endpoint receives a database session without creating it directly.

### Layer 2 – Authentication Dependency

```text
Endpoint
   ↓
Depends(get_current_user)
   ↓
Depends(oauth2_scheme)
   ↓
JWT Token
   ↓
Authenticated User
```

The `get_current_user` function:

1. Gets the token.
2. Decodes the JWT.
3. Extracts the username.
4. Searches for the user.
5. Returns the authenticated user.

### Benefits

Dependency injection helps:

- Reduce duplicated code.
- Manage resources consistently.
- Separate responsibilities.
- Make components easier to test.
- Reuse common functionality across endpoints.

---

# 4. Role-Based Access Control

The code uses a decorator called:

```python
requires_role()
```

It is used to restrict access based on user roles.

For example:

```python
@requires_role("admin")
```

The decorator checks whether the current user has administrator privileges.

The relevant check is:

```python
if role == "admin" and not current_user.is_superuser:
    raise HTTPException(
        status_code=status.HTTP_403_FORBIDDEN,
        detail="Insufficient permissions"
    )
```

If the user is not a superuser, the application returns a `403 Forbidden` response.

The authorization flow is:

```text
Request
   ↓
Get Current User
   ↓
Verify JWT
   ↓
Find User
   ↓
Check User Role
   ↓
Is User an Admin?
   ├── Yes → Allow Request
   └── No → 403 Forbidden
```

This separates authentication from authorization.

- **Authentication** asks: "Who are you?"
- **Authorization** asks: "Are you allowed to do this?"

---

# 5. Service Layer

The application contains a `UserService` class.

```python
class UserService:
```

The service handles business logic related to users.

For example:

```python
authenticate_user()
```

is responsible for checking whether a username and password are valid.

The service uses the repository to access user data:

```text
API Endpoint
     ↓
UserService
     ↓
UserRepository
     ↓
Database
```

This separation prevents the API endpoint from containing all of the business logic.

---

# 6. JWT Authentication

The application uses JWT tokens for authentication.

The login process works as follows:

```text
User submits username and password
             ↓
       /token endpoint
             ↓
       UserRepository
             ↓
       UserService
             ↓
     Authenticate user
             ↓
       Create JWT token
             ↓
       Return access token
```

The token contains the username:

```python
data={"sub": user.username}
```

The token also has an expiration time.

When a protected endpoint is accessed, the application:

1. Receives the token.
2. Decodes the JWT.
3. Extracts the username.
4. Finds the user.
5. Returns the authenticated user.

---

# 7. Timing Middleware

The application includes a custom middleware:

```python
class TimingMiddleware:
```

The middleware records how long each request takes.

It records the start time:

```python
start_time = datetime.utcnow()
```

It then allows the request to continue:

```python
response = await call_next(request)
```

After the request finishes, it calculates the processing time and adds it to the response:

```python
response.headers["X-Process-Time"] = str(process_time)
```

The result is returned in the HTTP response header.

The flow is:

```text
Request
   ↓
Start Timer
   ↓
Process Request
   ↓
Calculate Duration
   ↓
Add X-Process-Time Header
   ↓
Response
```

This provides a simple way to monitor request processing time.

---

# 8. Lifespan Management

The application uses:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
```

The lifespan function manages application startup and shutdown logic.

The code before:

```python
yield
```

represents startup logic.

The code after:

```python
yield
```

represents shutdown logic.

The simplified flow is:

```text
Application Starts
       ↓
Startup Logic
       ↓
     yield
       ↓
Application Runs
       ↓
Shutdown Logic
       ↓
Application Stops
```

This provides a central place for resources or operations that need to happen when the application starts or stops.

---

# Conclusion

The FastAPI example uses several architectural patterns together.

The main structure can be summarized as:

```text
Client
  ↓
FastAPI Endpoint
  ↓
Dependencies
  ↓
Authentication / Authorization
  ↓
Service Layer
  ↓
Repository Layer
  ↓
Database
```

Middleware operates around the request-processing pipeline, while lifespan management handles application startup and shutdown.

The combination of these patterns helps separate responsibilities and makes the application easier to organize, maintain, and extend.
