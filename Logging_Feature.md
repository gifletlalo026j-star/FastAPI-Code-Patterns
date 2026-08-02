# Part 4 – User-Action Logging Feature

## Introduction

The challenge is to implement a simple logging system that records important user actions, such as logins and administrator actions, in a database.

The goal is to use the same architectural patterns already present in the FastAPI application.

The main patterns used are:

* Repository Pattern
* Service Layer
* Dependency Injection
* FastAPI Endpoints
* Database Models

---

# 1. Feature Requirements

The logging system should record important actions performed by users.

Examples include:

* Successful login
* Failed login
* Viewing the admin user list
* Other administrator actions

Each log record could contain:

* Log ID
* User ID
* Username
* Action
* Timestamp
* Additional details

A simplified model could look like:

```python
class UserActionLog(Base):
    __tablename__ = "user_action_logs"

    id: int
    user_id: int
    action: str
    timestamp: datetime
    details: Optional[str]
```

---

# 2. Architecture

The new feature should follow the same structure as the existing application.

The architecture would be:

```text
FastAPI Endpoint
       ↓
Service Layer
       ↓
Repository Layer
       ↓
Database
```

For authentication-related actions, the flow could be:

```text
User Login
    ↓
Login Endpoint
    ↓
Authenticate User
    ↓
Create Access Token
    ↓
Create Log Entry
    ↓
Log Repository
    ↓
Database
```

For an administrator action:

```text
Admin Request
    ↓
Authentication
    ↓
Authorization
    ↓
Admin Endpoint
    ↓
Perform Action
    ↓
Create Log Entry
    ↓
Log Repository
    ↓
Database
```

---

# 3. Logging Model

The first step is to create a database model representing a user action.

A simplified example is:

```python
class UserActionLog(Base):
    __tablename__ = "user_action_logs"

    id: int
    user_id: int
    action: str
    timestamp: datetime
    details: Optional[str]
```

The model represents one recorded action.

For example:

```text
User ID: 15
Action: "ADMIN_VIEW_USERS"
Timestamp: 2026-08-02 10:30:00
Details: "Viewed list of users"
```

---

# 4. Repository Layer

Following the existing Repository Pattern, a new repository can be created.

Example:

```python
class UserActionLogRepository(Repository[UserActionLog]):
    async def create_log(
        self,
        db: AsyncSession,
        user_id: int,
        action: str,
        details: Optional[str] = None
    ):
        # Create and save log record
        pass
```

The repository is responsible for database operations related to logs.

This keeps database access separate from business logic.

---

# 5. Service Layer

A logging service can be used to handle logging-related business logic.

Example:

```python
class LoggingService:
    def __init__(self, repository: UserActionLogRepository):
        self.repository = repository

    async def log_action(
        self,
        db: AsyncSession,
        user_id: int,
        action: str,
        details: Optional[str] = None
    ):
        return await self.repository.create_log(
            db,
            user_id,
            action,
            details
        )
```

The service provides a simple interface for recording actions.

This means other parts of the application do not need to know how logs are stored in the database.

---

# 6. Dependency Injection

The logging repository or service could be provided using FastAPI dependency injection.

For example:

```python
async def get_logging_service():
    repository = UserActionLogRepository(UserActionLog)
    return LoggingService(repository)
```

An endpoint could then receive the service using:

```python
logging_service: LoggingService = Depends(get_logging_service)
```

The flow would be:

```text
FastAPI Endpoint
       ↓
Depends(get_logging_service)
       ↓
LoggingService
       ↓
UserActionLogRepository
```

This follows the same dependency injection approach used for database sessions and authenticated users.

---

# 7. Logging Successful Login

The login endpoint could record a successful login after authentication.

The flow would be:

```text
POST /token
    ↓
Receive Username + Password
    ↓
Authenticate User
    ↓
Authentication Successful?
   /              \
 No                Yes
 ↓                  ↓
Return 401       Create JWT
                    ↓
                Log Login
                    ↓
                Return Token
```

A successful login could be recorded as:

```python
await logging_service.log_action(
    db,
    user.id,
    "LOGIN_SUCCESS"
)
```

---

# 8. Logging Failed Login

Failed login attempts can also be recorded.

For example:

```python
if not user:
    # Log failed login attempt
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Incorrect username or password"
    )
```

The action could be recorded as:

```text
LOGIN_FAILED
```

However, the application would need to decide how to associate a failed login with a user if the username does not exist.

This is an important design question that should be discussed with the development team.

---

# 9. Logging Administrator Actions

The `/admin/users/` endpoint is an example of an administrator action.

After the user passes authentication and authorization, the application could record:

```python
await logging_service.log_action(
    db,
    current_user.id,
    "ADMIN_VIEW_USERS"
)
```

The complete flow would be:

```text
GET /admin/users/
       ↓
Authenticate User
       ↓
Check Admin Permission
       ↓
Admin Authorized
       ↓
Log ADMIN_VIEW_USERS
       ↓
Retrieve Users
       ↓
Return Response
```

---

# 10. Why Use the Existing Patterns?

Using the existing architecture provides several benefits.

## Consistency

The new feature follows the same structure as the rest of the application.

## Maintainability

Database operations remain in the repository.

Business logic remains in the service layer.

## Reusability

The logging service can be used by multiple endpoints.

## Testability

Each layer can be tested independently.

For example:

* Repository tests can test database operations.
* Service tests can test logging logic.
* Endpoint tests can test the complete request flow.

---

# 11. Implementation Plan

The implementation could be completed in the following order:

### Step 1

Create the `UserActionLog` database model.

### Step 2

Create a database migration for the new table.

### Step 3

Create `UserActionLogRepository`.

### Step 4

Create `LoggingService`.

### Step 5

Create a FastAPI dependency for the logging service.

### Step 6

Add logging to successful login.

### Step 7

Add logging to failed login attempts.

### Step 8

Add logging to administrator actions.

### Step 9

Test the logging functionality.

### Step 10

Review the implementation for security and performance.

---

# 12. Example Logging Flow

The final architecture could look like:

```text
                 ┌─────────────────┐
                 │  FastAPI Route  │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ LoggingService  │
                 └────────┬────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │ UserActionLogRepository│
              └───────────┬───────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │    Database     │
                 └─────────────────┘
```

The feature follows the existing architecture rather than introducing a completely different approach.

---

# 13. Important Security Considerations

The logging system should not store sensitive information.

For example, it should **not** record:

* Plain-text passwords
* Password hashes
* JWT secret keys
* Full authentication tokens

Logs should contain only the information needed to understand what action occurred.

The system should also consider access control so that only authorized users can view sensitive logs.

---

# 14. Testing

The new feature should be tested to ensure it works correctly.

Important test cases include:

### Successful Login

Verify that a successful login creates a `LOGIN_SUCCESS` record.

### Failed Login

Verify that a failed login creates a `LOGIN_FAILED` record where appropriate.

### Admin Action

Verify that an administrator action creates the correct log record.

### Unauthorized User

Verify that unauthorized users cannot perform admin actions or create misleading admin logs.

### Database Failure

Verify that the application handles database errors appropriately.

---

# Conclusion

The user-action logging feature can be added without changing the overall architecture of the application.

The new feature would use:

```text
FastAPI
   ↓
Dependency Injection
   ↓
Logging Service
   ↓
Logging Repository
   ↓
Database
```

The most important lesson is that new features should follow the existing architectural patterns whenever possible.

By using the Repository Pattern, Service Layer, and Dependency Injection, the logging system remains consistent with the rest of the application and is easier to maintain and test.

This implementation exercise helped demonstrate how understanding an existing architecture makes it easier to extend an application with new functionality.
