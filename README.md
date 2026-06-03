# Full Stack Evaluation Assignment: Auditable Notification Portal

A production-grade, multi-stage architecture designed to satisfy the official Full Stack Evaluation assignment. This repository implements a high-scale priority-ranked notification portal integrated with a custom token-based JWT logging middleware for observability.

---

## 1. Project Overview & Stage Mapping

This project is structured around the 7 Stages of the evaluation rubric:

*   **Stage 1: REST API Design & SSE Flow**: Standardized REST endpoints for notification CRUD, supporting real-time Server-Sent Events (SSE) subscriptions.
*   **Stage 2: Database Schema & Query Optimization**: Relational schema (PostgreSQL) optimized with targeted composite indexes (`idx_notif_type_created`) to prevent sequential scans.
*   **Stage 3: Performance Analysis & Index Costing**: Deep profiling of slow notification retrieval query plans (Seq Scan vs. Index Scan).
*   **Stage 4: Caching & Feed Strategies**: Redis caching strategies (Cache-Aside, Write-Through), cursor-based pagination, and push vs. pull notification flows.
*   **Stage 5: Asynchronous Notify-All Engine**: Message-queue queue design (RabbitMQ/Kafka) featuring idempotency checks, retry limits, and Dead Letter Queue (DLQ) ingestion.
*   **Stage 6: Priority Ranking Engine**: Independent TypeScript engine that ranks notifications by category priority (`placement` > `result` > `event`) and sorts by creation timestamp.
*   **Stage 7: Priority Inbox Dashboard**: A stunning Material UI React dashboard running on `http://localhost:3000` consuming the Express API, prioritizing feeds, and sending traces to the logging server.

---

## 2. Key Modules & Folder Structure

```
aditi_project/
├── logging_middleware/        # Reusable token-based logging middleware package
├── notification_app_be/       # Clean architecture Express REST API backend
├── stage6/                    # Stage 6: Standalone Priority Ranking Engine (TypeScript CLI)
├── stage7/                    # Stage 7: React + Vite + Material UI Dashboard (Port 3000)
└── notification_system_design.md  # Comprehensive Stages 1-5 System Design Specification
```

For a detailed folder breakdown, see [project_structure.md](./project_structure.md).

---

## 3. Tech Stack

- **Logging Middleware**: TypeScript, Axios, JWT TokenManager
- **Express Backend**: Node.js, Express, TypeScript, Joi (validation), Ts-Node
- **Priority Engine (Stage 6)**: TypeScript CLI
- **Frontend Dashboard (Stage 7)**: React 19, TypeScript, Material UI (v6), Axios, Context API, Vite

---

## 4. Setup & Installation

### A. Environment Configuration

Create the required `.env` files in their respective directories:

#### Backend Config (`notification_app_be/.env`)
```env
PORT=8080
LOGGING_BASE_URL=http://localhost:3000
LOGGING_EMAIL=candidate@evaluation.com
LOGGING_NAME=Candidate Name
LOGGING_ROLL_NO=ROLL-12345
LOGGING_ACCESS_CODE=CODE-XYZ
LOGGING_CLIENT_ID=client-id-abc
LOGGING_CLIENT_SECRET=client-secret-123
```

#### Stage 6 Config (`stage6/.env`)
```env
API_BASE_URL=http://localhost:8080
LOGGING_BASE_URL=http://localhost:3000
LOGGING_EMAIL=candidate@evaluation.com
LOGGING_NAME=Candidate Name
LOGGING_ROLL_NO=ROLL-12345
LOGGING_ACCESS_CODE=CODE-XYZ
LOGGING_CLIENT_ID=client-id-abc
LOGGING_CLIENT_SECRET=client-secret-123
```

#### Stage 7 Config (`stage7/.env`)
```env
VITE_API_BASE_URL=http://localhost:8080
VITE_LOGGING_BASE_URL=http://localhost:3000
VITE_LOGGING_EMAIL=candidate@evaluation.com
VITE_LOGGING_NAME=Candidate Name
VITE_LOGGING_ROLL_NO=ROLL-12345
VITE_LOGGING_ACCESS_CODE=CODE-XYZ
VITE_LOGGING_CLIENT_ID=client-id-abc
VITE_LOGGING_CLIENT_SECRET=client-secret-123
```

---

## 5. Running the System

Follow these steps in order to start and run the evaluation modules:

### Step 1: Build Logging Middleware
First, initialize and compile the shared logging dependency:
```bash
cd logging_middleware
npm install
npm run build
```

### Step 2: Start Express Backend
Run the backend server:
```bash
cd ../notification_app_be
npm install
npm run dev
```
*The REST API will launch on `http://localhost:8080`.*

### Step 3: Run Stage 6 Priority Ranking CLI
Execute the ranking engine:
```bash
cd ../stage6
npm install
npm run start
```
*This compiles the TypeScript code and executes the priority ranking logic, printing sorted arrays to the console and logging steps to the log server.*

### Step 4: Run Stage 7 Priority Dashboard
Launch the frontend dashboard:
```bash
cd ../stage7
npm install
npm run dev
```
*The dashboard will run on `http://localhost:3000` (or as configured in Vite).*

---

## 6. Logging Middleware Usage

Initialize once in your entrypoint:
```typescript
import { initLogger, Log } from 'logging-middleware';

initLogger({
  baseUrl: 'http://localhost:3000',
  credentials: {
    email: 'candidate@evaluation.com',
    name: 'Candidate',
    rollNo: 'ROLL-123',
    accessCode: 'CODE-XYZ',
    clientID: 'client-123',
    clientSecret: 'secret-123'
  }
});
```

Instrument events asynchronously:
```typescript
await Log('frontend', 'info', 'component', 'Priority Inbox filtered by Placement type');
```

---

## 7. Submission Deliverables & Documentation
- **Stages 1-5 Architectural Specifications**: [notification_system_design.md](./notification_system_design.md)
- **API Spec & Payloads**: [api_documentation.md](./api_documentation.md)
- **Security & Logging Architecture**: [logging_architecture.md](./logging_architecture.md)
- **Folder Breakdown**: [project_structure.md](./project_structure.md)
- **Submission Checklist**: [submission_checklist.md](./submission_checklist.md)

---

## Author
* **Developer**: Campus Hiring Evaluation Candidate
* **Architecture Advisor**: Antigravity AI (Google DeepMind)
