# Project Structure and Architectural Separation

This document explains the organization of the workspace directories, detailing Clean Architecture rules and separation of concerns implemented across the codebases for all stages of the Full Stack Evaluation.

---

## 1. Directory Tree Map

```
aditi_project/
│
├── logging_middleware/               # Observability Logging Package
│   ├── src/
│   │   ├── api/
│   │   │   ├── auth.ts               # Authenticates credentials
│   │   │   └── logger.ts             # Logs publishing requests
│   │   ├── services/
│   │   │   ├── AuthService.ts        # Coordinates JWT lifecycles
│   │   │   └── LogService.ts         # Coordinates validation & posting
│   │   ├── types/
│   │   │   ├── LogRequest.ts         # Credentials and request types
│   │   │   └── AuthResponse.ts       # Auth payload structures
│   │   ├── utils/
│   │   │   ├── constants.ts          # Valid stacks, levels, packages
│   │   │   └── errors.ts             # Custom Logger errors
│   │   ├── index.ts                  # Public package entrypoint exports
│   │   └── logger.ts                 # Unified initialization hooks
│   ├── package.json
│   └── tsconfig.json
│
├── notification_app_be/              # REST Backend Service
│   ├── src/
│   │   ├── config/
│   │   │   └── config.ts             # Reads process.env variables
│   │   ├── controllers/
│   │   │   └── notificationController.ts # Translates HTTP payloads
│   │   ├── middleware/
│   │   │   ├── bodyValidator.ts      # Enforces field parameters
│   │   │   ├── errorHandler.ts       # Global catch-all exceptions handler
│   │   │   └── routeHitLogger.ts     # Triggers automatic middleware logs
│   │   ├── models/
│   │   │   └── Notification.ts       # Type definitions for notifications
│   │   ├── repositories/
│   │   │   ├── INotificationRepository.ts # Repository Interface definition
│   │   │   └── InMemoryNotificationRepository.ts # Memory database implementation
│   │   ├── routes/
│   │   │   └── notificationRoutes.ts # Express router declarations
│   │   ├── services/
│   │   │   └── notificationService.ts # Business logic routines
│   │   ├── app.ts                    # Connects router and global middlewares
│   │   └── server.ts                 # Starts Express listener port
│   ├── package.json
│   └── tsconfig.json
│
├── stage6/                           # Stage 6: Standalone Priority Ranking Engine
│   ├── src/
│   │   └── priorityEngine.ts         # Logic sorting Placement > Result > Event
│   ├── package.json
│   ├── tsconfig.json
│   └── .env
│
└── stage7/                           # Stage 7: React Material UI Priority Dashboard
    ├── src/
    │   ├── api/
    │   │   └── notificationApi.ts    # Axios REST client routines
    │   ├── components/
    │   │   ├── NotificationCard.tsx  # Item visualization with custom theme status
    │   │   └── SearchBar.tsx         # Text search and category dropdown filters
    │   ├── hooks/
    │   │   └── useNotifications.ts   # Context retrieval hook
    │   ├── pages/
    │   │   └── Dashboard.tsx         # Top-level metric dashboard layout
    │   ├── state/
    │   │   └── NotificationContext.tsx # Central Provider & action logic
    │   ├── App.tsx                   # Top-level Theme and Provider wrappers
    │   ├── main.tsx                  # Vite DOM mounting entrypoint
    │   └── index.css                 # Custom baseline style overrides
    ├── package.json
    ├── tsconfig.json
    └── vite.config.ts
```

---

## 2. Responsibilities by Package

### A. `logging_middleware/`
An independent utility package linked as a local dependency (`file:../logging_middleware`). It has no knowledge of frontend components or backend database logic. Its only concern is providing a reliable, validated interface to communicate with the evaluation server's auth and logs API.

### B. `notification_app_be/`
Designed strictly around **Clean Architecture Principles**. The backend divides its code into functional rings:
- **Models**: Defines database-agnostic notifications schema.
- **Repository Layer**: Abstraits memory-storage logic behind interfaces. Handles the exact storage array.
- **Service Layer**: Governs core business calculations. It processes requests, checks rules, and fires validation warnings.
- **Controller Layer**: Decodes JSON, routes requests to Services, and formats JSON outputs.
- **Middleware Layer**: Enforces payload schemas, catches server runtime errors, and logs request hits.

### C. `stage6/`
A CLI utility that demonstrates the priority sorting algorithm independently:
- **Sorting Rule**: `Placement` (High Priority) > `Result` (Medium Priority) > `Event` (Low Priority).
- **Secondary Sort**: Sorts matching categories by timestamp in descending order (newest first).
- **Traces**: Logs each step to the central logging server using the `logging_middleware`.

### D. `stage7/`
Organized into reusable React components and decoupled API integration modules:
- **API Client**: Implements client-side Axios methods wrapped in logger middleware calls.
- **State Provider**: Houses the application state, managing the list of notifications, search criteria, and notifications loaded status in a central context.
- **UI Components**: Pure visual layouts that render state and call handler methods provided by context.

---

## 3. Core Architectural Concepts

### Clean Architecture
Clean Architecture ensures that core business logic is not coupled to external libraries, databases, or frameworks. By enforcing the dependency rule (dependencies can only point inward), we protect the service and model layers. If the express library is swapped with NestJS or if in-memory arrays are replaced with PostgreSQL/MongoDB, only the router/repository boundaries are updated.

### Separation of Concerns (SoC)
Separation of Concerns ensures that every module is responsible for a single aspect of the system. For instance:
- `notificationRoutes.ts` only declares routes. It does not validate bodies or save data.
- `bodyValidator.ts` only verifies body parameters. It does not execute storage commands.
- `InMemoryNotificationRepository.ts` only modifies arrays. It does not inspect HTTP response objects.
- `Dashboard.tsx` only renders templates. It delegates API network triggers to the Context state.
This isolation makes the system modular, easy to debug, and simple to test.
