# API Contract

> **Created by:** Agent_02_TechLead  
> **Phase:** 2  
> **Input:** docs/user_stories.md, project_setup/step_1_project/step_1_project.md

---

## Authentication Endpoints

### POST /api/auth/register
Register a new user account.

**Request Body:**
```json
{
  "email": "string (email format, required)",
  "password": "string (min 8 chars, required)",
  "confirmPassword": "string (must match password, required)"
}
```

**Response (201 Created):**
```json
{
  "userId": "string (UUID)",
  "email": "string",
  "message": "Verification email sent"
}
```

**Error Responses:**
- `400 Bad Request`: Invalid input or email already exists
- `500 Internal Server Error`: Server error

---

### POST /api/auth/login
Authenticate user and create session.

**Request Body:**
```json
{
  "email": "string (required)",
  "password": "string (required)"
}
```

**Response (200 OK):**
```json
{
  "token": "string (JWT)",
  "user": {
    "id": "string (UUID)",
    "email": "string",
    "createdAt": "string (ISO 8601)"
  }
}
```

**Error Responses:**
- `401 Unauthorized`: Invalid credentials
- `403 Forbidden`: Email not verified

---

## User Endpoints

### GET /api/user/dashboard
Get user dashboard data.

**Auth:** Required (Bearer token)

**Response (200 OK):**
```json
{
  "profile": {
    "id": "string",
    "email": "string",
    "createdAt": "string"
  },
  "recentActivity": [
    {
      "id": "string",
      "type": "string",
      "timestamp": "string",
      "description": "string"
    }
  ]
}
```

**Error Responses:**
- `401 Unauthorized`: Invalid or expired token
- `404 Not Found`: User not found
