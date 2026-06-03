# API Documentation & Postman Collection Spec

This document details the REST API specifications for the **Notification Backend Service** (`notification_app_be`) and provides a import-ready **Postman Collection v2.1.0** schema.

---

## 1. Notification APIs Spec

### A. POST /notifications
* **Purpose**: Creates a new notification in the database.
* **Status Code**: `210 Created` (custom status code defined by the evaluation specifications) or `400 Bad Request` on validation failure.
* **Headers**: `Content-Type: application/json`

#### Validation Rules:
- `title`: String, Required, 1-100 characters.
- `message`: String, Required, 1-500 characters.
- `type`: String, Required, Must be one of: `info`, `warning`, `error`, `success`.
- `status`: String, Optional, Must be: `active`, `inactive`. Defaults to `active`.

#### Request Body Example:
```json
{
  "title": "Database Optimization Complete",
  "message": "Index cleanups and cache resizing completed successfully.",
  "type": "success",
  "status": "active"
}
```

#### Response Example (210 Created):
```json
{
  "success": true,
  "data": {
    "id": "fe98b50e-dc99-43ef-b387-052637738f61",
    "title": "Database Optimization Complete",
    "message": "Index cleanups and cache resizing completed successfully.",
    "type": "success",
    "status": "active",
    "createdAt": "2026-06-03T10:45:00.000Z"
  }
}
```

#### Error Example (400 Bad Request):
```json
{
  "success": false,
  "error": {
    "message": "Validation Error",
    "details": "\"title\" is required, \"type\" must be one of [info, warning, error, success]"
  }
}
```

---

### B. GET /notifications
* **Purpose**: Retrieves all notifications.
* **Status Code**: `200 OK`.
* **Headers**: `Content-Type: application/json`

#### Request Body: None

#### Response Example (200 OK):
```json
{
  "success": true,
  "data": [
    {
      "id": "fe98b50e-dc99-43ef-b387-052637738f61",
      "title": "Database Optimization Complete",
      "message": "Index cleanups and cache resizing completed successfully.",
      "type": "success",
      "status": "active",
      "createdAt": "2026-06-03T10:45:00.000Z"
    }
  ]
}
```

---

### C. PUT /notifications/:id
* **Purpose**: Updates an existing notification by ID.
* **Status Code**: `200 OK` on success, `404 Not Found` if the ID is missing, `400 Bad Request` on validation failure.
* **Headers**: `Content-Type: application/json`

#### Validation Rules:
- `title`: String, Optional, 1-100 characters.
- `message`: String, Optional, 1-500 characters.
- `type`: String, Optional, Must be: `info`, `warning`, `error`, `success`.
- `status`: String, Optional, Must be: `active`, `inactive`.

#### Request Body Example:
```json
{
  "title": "Database Optimization Update",
  "status": "inactive"
}
```

#### Response Example (200 OK):
```json
{
  "success": true,
  "data": {
    "id": "fe98b50e-dc99-43ef-b387-052637738f61",
    "title": "Database Optimization Update",
    "message": "Index cleanups and cache resizing completed successfully.",
    "type": "success",
    "status": "inactive",
    "createdAt": "2026-06-03T10:45:00.000Z"
  }
}
```

#### Error Example (404 Not Found):
```json
{
  "success": false,
  "error": {
    "message": "Notification not found"
  }
}
```

---

### D. DELETE /notifications/:id
* **Purpose**: Deletes a specific notification by ID.
* **Status Code**: `200 OK` on success, `404 Not Found` if missing.
* **Headers**: `Content-Type: application/json`

#### Request Body: None

#### Response Example (200 OK):
```json
{
  "success": true,
  "message": "Notification deleted successfully"
}
```

---

## 2. Postman Collection Import JSON

To import these APIs directly into Postman, copy the JSON block below and save it as `notifications_api_collection.json`, then select **Import** in Postman.

```json
{
  "info": {
    "_postman_id": "8b51d8b9-e1cc-4fa0-bca4-d4b998a44977",
    "name": "Notification System & Logging APIs",
    "description": "Standardized evaluation endpoints for Notification CRUD operations and central Logging Service.",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Logging Service",
      "item": [
        {
          "name": "Client Authentication",
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"email\": \"candidate@evaluation.com\",\n  \"name\": \"Candidate Name\",\n  \"rollNo\": \"ROLL-12345\",\n  \"accessCode\": \"CODE-XYZ\",\n  \"clientID\": \"client-id-abc\",\n  \"clientSecret\": \"client-secret-123\"\n}"
            },
            "url": {
              "raw": "http://localhost:3000/evaluation-service/auth",
              "protocol": "http",
              "host": ["localhost"],
              "port": "3000",
              "path": ["evaluation-service", "auth"]
            }
          },
          "response": []
        },
        {
          "name": "Publish Log Entry",
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              },
              {
                "key": "Authorization",
                "value": "Bearer {{jwt_token}}"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"stack\": \"backend\",\n  \"level\": \"info\",\n  \"packageName\": \"repository\",\n  \"message\": \"database query executed: create\"\n}"
            },
            "url": {
              "raw": "http://localhost:3000/evaluation-service/logs",
              "protocol": "http",
              "host": ["localhost"],
              "port": "3000",
              "path": ["evaluation-service", "logs"]
            }
          },
          "response": []
        }
      ]
    },
    {
      "name": "Notification APIs",
      "item": [
        {
          "name": "Get All Notifications",
          "request": {
            "method": "GET",
            "header": [],
            "url": {
              "raw": "http://localhost:8080/notifications",
              "protocol": "http",
              "host": ["localhost"],
              "port": "8080",
              "path": ["notifications"]
            }
          },
          "response": []
        },
        {
          "name": "Create Notification",
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"title\": \"System Core Startup\",\n  \"message\": \"Notification backend booted on port 8080\",\n  \"type\": \"info\",\n  \"status\": \"active\"\n}"
            },
            "url": {
              "raw": "http://localhost:8080/notifications",
              "protocol": "http",
              "host": ["localhost"],
              "port": "8080",
              "path": ["notifications"]
            }
          },
          "response": []
        },
        {
          "name": "Update Notification by ID",
          "request": {
            "method": "PUT",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"title\": \"Updated Core Startup Message\",\n  \"status\": \"inactive\"\n}"
            },
            "url": {
              "raw": "http://localhost:8080/notifications/{{notification_id}}",
              "protocol": "http",
              "host": ["localhost"],
              "port": "8080",
              "path": ["notifications", "{{notification_id}}"]
            }
          },
          "response": []
        },
        {
          "name": "Delete Notification by ID",
          "request": {
            "method": "DELETE",
            "header": [],
            "url": {
              "raw": "http://localhost:8080/notifications/{{notification_id}}",
              "protocol": "http",
              "host": ["localhost"],
              "port": "8080",
              "path": ["notifications", "{{notification_id}}"]
            }
          },
          "response": []
        }
      ]
    }
  ]
}
```
