# Logging Architecture and Strategy

## 1. Why Logging is Important
Logging is the foundation of software observability. Without comprehensive logs:
- **Root Cause Analysis** of production crashes is nearly impossible.
- **Auditing** user behavior and backend transaction trails becomes speculative.
- **Latency & Reliability Bottlenecks** cannot be measured.
- **Testing and Verification** in developer recruitment pipelines lack automated assessment logs.

By structuring logs, we can track exactly how data flows through our ecosystem, identifying validation errors, API timeouts, and network issues in real time.

---

## 2. Reusable Middleware Design
Instead of hardcoding logger logic into our UI pages and REST controllers, we implemented a decoupled, reusable logging middleware package (`logging_middleware`). This package:
- Implements a single unified `Log()` signature exported across the ecosystem.
- Enforces strict TypeScript validation to prevent malformed log entries.
- Automatically handles token acquisition, expiration detection, and authorization header appending.
- Operates asynchronously, ensuring logging network requests do not block UI renderings or Express response cycles.

---

## 3. Authentication & Token Management
The logging middleware utilizes a secure `TokenManager` class that manages token retrieval and lifecycle validation:
- **Credential Model**: Requires 6 evaluation credentials: `email`, `name`, `rollNo`, `accessCode`, `clientID`, and `clientSecret`.
- **Bearer Token Storage**: Tokens are stored strictly in-memory (no localStorage, sessionStorage, or cookie storage to prevent cross-site scripting/token theft).
- **Auto-Refresh Lifecycle**: Every time a log is sent, the TokenManager verifies the active token. If the token is missing or expired, it automatically calls `POST /evaluation-service/auth` using the configured credentials to refresh the token.
- **Concurrency Protection**: Reuses the token for all subsequent logging API calls until expiration, reducing API authentication overhead.

---

## 4. Validation Mechanism
To guarantee clean data, the middleware performs strict client-side validation on log payloads before sending them over the network. Logs are only processed if they match predefined values:

### Stacks
- `backend`
- `frontend`

### Levels
- `debug`
- `info`
- `warn`
- `error`
- `fatal`

### Backend Packages
- `cache`, `controller`, `cron_job`, `db`, `handler`, `repository`, `route`, `service`

### Frontend Packages
- `api`, `component`, `hook`, `page`, `state`, `style`

### Shared Packages
- `auth`, `config`, `middleware`, `utils`

If validation fails, the middleware immediately rejects the request with a custom `ValidationError`, preventing invalid logs from reaching the evaluation server.

---

## 5. Retry Mechanism
To safeguard against transient network issues (such as temporary Wi-Fi drops, server gateway timeouts, or load balancing delays), the middleware implements a automated retry policy:
- **Retry Threshold**: Attempts to resend a log exactly **once** on connection failure.
- **Execution Flow**: If the initial Axios request throws a network exception or timeout error, the catch block intercepts it, checks the remaining retry count, waits briefly, and invokes the logs API request a second time.
- **Safety Fallback**: If both attempts fail, the logger fails gracefully or throws a `LoggingError` depending on config parameters, ensuring a logging failure never crashes the host application.

---

## 6. Error Handling Strategy
The middleware defines structured, custom error classes inheriting from a base `LoggerBaseError`:
- `ValidationError`: Thrown when a stack, level, or package name violates validation rules.
- `AuthenticationError`: Thrown when credentials fail to authenticate with the evaluation service.
- `LoggingError`: Thrown when the logs API endpoint returns a server error (e.g. 500) or fails after retries.

---

## 7. Example Log Lifecycle

The step-by-step path of a user action triggering a log trace is outlined below:

```
[User Action: Edit Button Clicked]
        │
        ▼
1. Component Code invokes local Hook:
   Log("frontend", "info", "component", "edit button clicked for notification: ID-01")
        │
        ▼
2. Logging Middleware intercepts call & passes to Validation Engine:
   - Check if stack == "frontend" (Passes)
   - Check if level == "info" (Passes)
   - Check if package == "component" (Passes)
        │
        ▼
3. TokenManager checks active JWT token status:
   - Check in-memory JWT cache.
   - If expired/null -> Execute Auth API call (POST /auth) & receive new JWT token.
        │
        ▼
4. Middleware sends HTTP POST to Logging Endpoint:
   - URL: POST http://localhost:3000/evaluation-service/logs
   - Header: Authorization: Bearer <JWT>
   - Payload: { stack, level, packageName, message }
        │
        ▼
5. Logging API returns 200 OK:
   - Log entry saved successfully.
        │
        ▼
6. Promise resolves:
   - Calling component logs success and updates UI.
```
