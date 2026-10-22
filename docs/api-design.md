# Kairo — REST API Reference

> Complete API specification for **Kairo: AI Conversational Memory Assistant**
> This is the definitive reference for every endpoint the React frontend will consume.
> Companion to [backend-design.md](file:///Users/mohammed/kairo/docs/backend-design.md)

**Base URL:** `http://localhost:8000/api`
**Content Type:** `application/json` (unless specified otherwise)
**Authentication:** JWT Bearer Token (see [§1](#1-api-conventions))

**Last updated:** 2026-06-07
**Author:** Mohammed Kutubudin Kapasi

---

## Table of Contents

1. [API Conventions](#1-api-conventions)
2. [Authentication APIs](#2-authentication-apis)
3. [Conversation APIs](#3-conversation-apis)
4. [Search APIs](#4-search-apis)
5. [Error Reference](#5-error-reference)
6. [Rate Limiting](#6-rate-limiting)
7. [CORS Configuration](#7-cors-configuration)
8. [API Design Decisions — Why It's Done This Way](#8-api-design-decisions--why-its-done-this-way)

---

## 1. API Conventions

Before reading any endpoint, understand the rules that apply **everywhere**.

### 1.1 URL Structure

```
All URLs follow this pattern:

  /api/{app}/{resource}/                    ← Collection (list, create)
  /api/{app}/{resource}/{id}/               ← Single item (retrieve, update, delete)
  /api/{app}/{resource}/{id}/{action}/      ← Custom action on an item

Trailing slashes are REQUIRED (Django convention).
Without the trailing slash, Django returns a 301 redirect → wastes a round trip.
```

### 1.2 HTTP Methods — When to Use Each

```
┌────────────┬───────────────────────────────────────────────────────────────┐
│  METHOD    │  MEANING                                                     │
├────────────┼───────────────────────────────────────────────────────────────┤
│  GET       │  Read data. NEVER changes anything on the server.            │
│            │  Safe to retry. Safe to cache. No request body.              │
│            │                                                              │
│  POST      │  Create something new OR trigger an action.                  │
│            │  NOT safe to retry blindly (could create duplicates).        │
│            │  Has a request body.                                         │
│            │                                                              │
│  PATCH     │  Partially update an existing resource.                      │
│            │  Only the fields you send are changed.                       │
│            │  Fields you DON'T send remain untouched.                     │
│            │                                                              │
│  PUT       │  Replace an entire resource. (Not used in Kairo's MVP.)      │
│            │  Every field must be sent — missing fields are set to null.  │
│            │                                                              │
│  DELETE    │  Remove a resource permanently.                              │
│            │  Returns 204 No Content (empty body) on success.             │
│            │                                                              │
└────────────┴───────────────────────────────────────────────────────────────┘

WHY PATCH AND NOT PUT?
  When a user edits a conversation title, they send:
    PATCH { "title": "New Title" }

  With PUT, they'd have to send EVERY field:
    PUT { "title": "New Title", "tags": [...], "status": "ready", ... }

  If they forget a field, it gets erased. PATCH is safer for partial updates.
```

### 1.3 Request Headers

Every authenticated request must include:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

For file uploads, use:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: multipart/form-data
```

### 1.4 Response Format

**Successful single resource:**
```json
{
  "id": "665f1a2b3c4d5e6f7a8b9c0d",
  "email": "mohammed@example.com",
  "username": "mohammed",
  "created_at": "2026-06-07T10:30:00Z"
}
```

**Successful list (paginated):**
```json
{
  "count": 42,
  "next": "http://localhost:8000/api/conversations/?page=2",
  "previous": null,
  "results": [...]
}
```

**Error (single):**
```json
{
  "error": "Conversation not found."
}
```

**Error (validation — multiple fields):**
```json
{
  "errors": {
    "email": ["This field is required.", "Enter a valid email address."],
    "password": ["Password must be at least 8 characters."]
  }
}
```

### 1.5 Date/Time Format

All timestamps use **ISO 8601 format in UTC**:

```
"2026-06-07T10:30:00Z"
         │         │
         │         └── Z means UTC (Zulu time)
         └── T separates date and time

The frontend is responsible for converting UTC to the user's local timezone.
NEVER store or return local times from the API.
```

### 1.6 ID Format

All IDs are **MongoDB ObjectId strings** — 24-character hexadecimal:

```
"665f1a2b3c4d5e6f7a8b9c0d"

Validate format before querying: must be exactly 24 hex characters.
Invalid IDs should return 400, not 500 (don't let invalid ObjectId crash the server).
```

---

## 2. Authentication APIs

### Overview

| # | Endpoint | Method | Auth Required | Purpose |
|---|---|---|---|---|
| A1 | `/api/auth/register/` | POST | ❌ No | Create a new user account |
| A2 | `/api/auth/login/` | POST | ❌ No | Authenticate and receive JWT tokens |
| A3 | `/api/auth/token/refresh/` | POST | ❌ No | Exchange refresh token for new access token |
| A4 | `/api/auth/me/` | GET | ✅ Yes | Get current authenticated user's profile |

---

### A1. Register

```
POST /api/auth/register/
```

**Purpose:** Create a new user account.

**Auth:** None required (public endpoint).

#### Request

```
Headers:
  Content-Type: application/json

Body:
{
  "email": "mohammed@example.com",
  "username": "mohammed",
  "password": "SecurePass123!",
  "password_confirm": "SecurePass123!"
}
```

#### Validation Rules

| Field | Type | Required | Rules |
|---|---|---|---|
| `email` | string | ✅ | Must be a valid email format. Normalised to lowercase before storage. Must be **unique** across all users. Max 254 characters (RFC 5321). |
| `username` | string | ✅ | 3–30 characters. Alphanumeric + underscores only (`^[a-zA-Z0-9_]+$`). Must be **unique**. |
| `password` | string | ✅ | Minimum 8 characters. Must contain at least 1 letter and 1 number. Cannot be entirely numeric. Cannot be a commonly used password (e.g., "password123"). |
| `password_confirm` | string | ✅ | Must exactly match `password`. |

#### Responses

**201 Created** — Account created successfully:
```json
{
  "id": "665f1a2b3c4d5e6f7a8b9c0d",
  "email": "mohammed@example.com",
  "username": "mohammed",
  "created_at": "2026-06-07T10:30:00Z"
}
```

**400 Bad Request** — Validation failed:
```json
{
  "errors": {
    "email": ["A user with this email already exists."],
    "password": ["Password must be at least 8 characters."],
    "password_confirm": ["Passwords do not match."]
  }
}
```

**429 Too Many Requests** — Rate limit exceeded:
```json
{
  "error": "Too many registration attempts. Please try again later."
}
```

#### What Happens Inside

```
1. Deserialise request body using RegisterSerializer
2. Run validation:
   a. Email format check (Django's EmailValidator)
   b. Email uniqueness check (query users collection)
   c. Username format check (regex)
   d. Username uniqueness check
   e. Password strength check
   f. password == password_confirm check
3. If any validation fails → return 400 with all errors
4. Normalise email to lowercase
5. Hash password using bcrypt (12 rounds)
6. Create User document in MongoDB
7. Return user data (WITHOUT password_hash, NEVER)
```

#### Security Considerations

```
┌──────────────────────────────────────────────────────────────────┐
│  SECURITY NOTES FOR REGISTER                                     │
│                                                                  │
│  1. NEVER return password_hash in the response                  │
│  2. NEVER log the plaintext password                            │
│  3. Email normalisation prevents "User@Email.COM" and            │
│     "user@email.com" from being treated as different accounts   │
│  4. Rate limit: 5 registrations per IP per hour                 │
│  5. bcrypt cost factor of 12 means ~250ms per hash —            │
│     slow enough to resist brute force, fast enough for UX       │
│  6. Return ALL validation errors at once, not one at a time     │
│     (better UX: user fixes all issues in one attempt)           │
└──────────────────────────────────────────────────────────────────┘
```

---

### A2. Login

```
POST /api/auth/login/
```

**Purpose:** Verify credentials and issue JWT access + refresh tokens.

**Auth:** None required.

#### Request

```
Headers:
  Content-Type: application/json

Body:
{
  "email": "mohammed@example.com",
  "password": "SecurePass123!"
}
```

#### Validation Rules

| Field | Type | Required | Rules |
|---|---|---|---|
| `email` | string | ✅ | Must be valid email format. Normalised to lowercase before lookup. |
| `password` | string | ✅ | Non-empty string. |

#### Responses

**200 OK** — Login successful:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiNjY1ZjFhMmIzYzRkNWU2ZjdhOGI5YzBkIiwidHlwZSI6ImFjY2VzcyIsImV4cCI6MTcxODAxMjYwMH0.signature",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiNjY1ZjFhMmIzYzRkNWU2ZjdhOGI5YzBkIiwidHlwZSI6InJlZnJlc2giLCJleHAiOjE3MTg2MTI2MDB9.signature",
  "user": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "email": "mohammed@example.com",
    "username": "mohammed"
  }
}
```

**401 Unauthorized** — Invalid credentials:
```json
{
  "error": "Invalid email or password."
}
```

**429 Too Many Requests** — Too many login attempts:
```json
{
  "error": "Too many login attempts. Please try again in 15 minutes."
}
```

#### What Happens Inside

```
1. Deserialise request body using LoginSerializer
2. Normalise email to lowercase
3. Query users collection: find by email
4. If user not found → return 401 "Invalid email or password."
5. If user.is_active == false → return 401 "Invalid email or password."
6. Verify password against stored bcrypt hash
7. If mismatch → return 401 "Invalid email or password."
8. Update user.last_login to current timestamp
9. Generate JWT tokens:
   a. Access token: { user_id, type: "access", exp: now + 30min }
   b. Refresh token: { user_id, type: "refresh", exp: now + 7days }
10. Sign both tokens with JWT_SECRET_KEY
11. Return tokens + user info
```

#### Security Considerations

```
┌──────────────────────────────────────────────────────────────────┐
│  SECURITY NOTES FOR LOGIN                                        │
│                                                                  │
│  1. SAME ERROR MESSAGE for "email not found" and "wrong pass"   │
│     → "Invalid email or password."                               │
│     → Prevents user enumeration attacks                         │
│     → Attacker cannot discover which emails are registered       │
│                                                                  │
│  2. Rate limit: 5 attempts per email per 15 minutes             │
│     → Prevents brute force password guessing                    │
│                                                                  │
│  3. Never log the password — even in debug mode                 │
│     → Logs may be read by other developers or stored insecurely │
│                                                                  │
│  4. Check is_active BEFORE verifying password                    │
│     → If the account is disabled, don't waste CPU on bcrypt     │
│     → But return the SAME error message (don't reveal status)   │
│                                                                  │
│  5. Timing attack mitigation:                                    │
│     → Even if user not found, still run bcrypt on a dummy hash  │
│     → This ensures the response time is similar whether the     │
│       user exists or not (prevents timing-based enumeration)    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

#### JWT Token Structure (Decoded)

```
ACCESS TOKEN PAYLOAD:
{
  "user_id": "665f1a2b3c4d5e6f7a8b9c0d",
  "type": "access",
  "iat": 1718010800,           ← Issued at (Unix timestamp)
  "exp": 1718012600            ← Expires at (30 minutes later)
}

REFRESH TOKEN PAYLOAD:
{
  "user_id": "665f1a2b3c4d5e6f7a8b9c0d",
  "type": "refresh",
  "iat": 1718010800,
  "exp": 1718615600            ← Expires at (7 days later)
}

WHY TWO TOKENS?
─────────────────
The access token is sent with EVERY request. If it's stolen, the
damage is limited to 30 minutes. The refresh token is sent rarely
(only to /token/refresh/) and lasts longer, reducing how often
the user must log in.

If the refresh token is stolen, the attacker has 7 days of access.
This is why tokens should be stored in memory, not localStorage.
```

---

### A3. Token Refresh

```
POST /api/auth/token/refresh/
```

**Purpose:** Exchange a valid refresh token for a new access token. Called automatically by the React app's Axios interceptor when a request returns 401.

**Auth:** None required (the refresh token itself is the credential).

#### Request

```
Headers:
  Content-Type: application/json

Body:
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

#### Validation Rules

| Field | Type | Required | Rules |
|---|---|---|---|
| `refresh_token` | string | ✅ | Must be a valid JWT. Must have `type: "refresh"` in payload. Must not be expired. |

#### Responses

**200 OK** — New access token issued:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**401 Unauthorized** — Refresh token invalid or expired:
```json
{
  "error": "Refresh token is invalid or expired. Please log in again."
}
```

#### What Happens Inside

```
1. Decode refresh token using JWT_SECRET_KEY
2. If decode fails (invalid signature, malformed) → 401
3. Check token.type == "refresh" (reject access tokens used here)
4. Check token.exp > current time (not expired)
5. Verify user still exists and is_active in MongoDB
6. Generate new access token with fresh 30-minute expiration
7. Return new access token (DO NOT issue a new refresh token)
```

#### How React Uses This Automatically

```
React Axios Interceptor (conceptual flow):
────────────────────────────────────────────

1. User makes an API call → /api/conversations/
2. Server returns 401 (access token expired)
3. Axios interceptor catches the 401
4. Interceptor calls POST /api/auth/token/refresh/
   with the stored refresh token
5. If successful → store new access token
6. Retry the ORIGINAL request with the new token
7. User sees the data — never knows anything happened

8. If refresh also fails (expired) → redirect to login page
```

---

### A4. Get Current User

```
GET /api/auth/me/
```

**Purpose:** Return the authenticated user's profile. Used by React on app load to verify the session and display user info.

**Auth:** ✅ Required.

#### Request

```
Headers:
  Authorization: Bearer <access_token>
```

No request body (GET requests never have a body).

#### Responses

**200 OK:**
```json
{
  "id": "665f1a2b3c4d5e6f7a8b9c0d",
  "email": "mohammed@example.com",
  "username": "mohammed",
  "created_at": "2026-06-07T10:30:00Z",
  "last_login": "2026-06-07T15:00:00Z"
}
```

**401 Unauthorized:**
```json
{
  "error": "Authentication credentials were not provided."
}
```

---

## 3. Conversation APIs

### Overview

| # | Endpoint | Method | Auth | Purpose |
|---|---|---|---|---|
| C1 | `/api/conversations/` | GET | ✅ | List conversations (dashboard) |
| C2 | `/api/conversations/upload/` | POST | ✅ | Upload audio file |
| C3 | `/api/conversations/{id}/` | GET | ✅ | Get conversation detail |
| C4 | `/api/conversations/{id}/` | PATCH | ✅ | Update title/tags |
| C5 | `/api/conversations/{id}/` | DELETE | ✅ | Delete conversation |
| C6 | `/api/conversations/{id}/retry/` | POST | ✅ | Retry failed processing |
| C7 | `/api/conversations/{id}/action-items/{index}/` | PATCH | ✅ | Toggle action item |

---

### C1. List Conversations

```
GET /api/conversations/
```

**Purpose:** Paginated list of the authenticated user's conversations for the dashboard.

**Auth:** ✅ Required.

#### Request

```
Headers:
  Authorization: Bearer <access_token>

Query Parameters (all optional):
  ?page=1                    ← Page number (default: 1)
  ?page_size=20              ← Results per page (default: 20, max: 50)
  ?status=ready              ← Filter: "uploading", "processing", "ready", "failed"
  ?tag=dbms                  ← Filter by tag (exact match)
  ?ordering=-created_at      ← Sort: "-created_at" (newest, default) or "created_at" (oldest)
  ?search=normalisation      ← Basic text search in title (NOT semantic search)
```

#### Validation Rules

| Parameter | Type | Rules |
|---|---|---|
| `page` | integer | ≥ 1. Invalid values default to 1. |
| `page_size` | integer | 1–50. Values > 50 are clamped to 50. Default: 20. |
| `status` | string | Must be one of: `"uploading"`, `"processing"`, `"ready"`, `"failed"`. Invalid values are ignored (no filter applied). |
| `tag` | string | Case-insensitive match. Trimmed of whitespace. |
| `ordering` | string | Only `"created_at"` or `"-created_at"` allowed. Invalid values default to `"-created_at"`. |
| `search` | string | Min 2 chars. Matched against `title` field (case-insensitive substring). |

#### Response

**200 OK:**
```json
{
  "count": 42,
  "next": "http://localhost:8000/api/conversations/?page=2",
  "previous": null,
  "results": [
    {
      "id": "665f1b3c4d5e6f7a8b9c0e1f",
      "title": "DBMS Lecture 5 - Normalisation",
      "tags": ["dbms", "lecture", "normalisation"],
      "status": "ready",
      "audio_file_name": "dbms_lecture_5.mp3",
      "audio_duration_seconds": 1847,
      "created_at": "2026-06-07T09:00:00Z",
      "summary_preview": "This lecture covered database normalisation, explaining 1NF, 2NF, and 3NF...",
      "action_item_count": 2
    },
    {
      "id": "665f2c4d5e6f7a8b9c0f1e2d",
      "title": "Team Standup - Sprint 12",
      "tags": ["meeting", "standup"],
      "status": "processing",
      "audio_file_name": "standup_june7.mp3",
      "audio_duration_seconds": null,
      "created_at": "2026-06-07T10:15:00Z",
      "summary_preview": null,
      "action_item_count": 0
    }
  ]
}
```

**Response field explanations:**

| Field | Notes |
|---|---|
| `count` | Total number of conversations (across all pages) for the current filters |
| `next` / `previous` | Full URL for next/previous page, or `null` if there is none |
| `summary_preview` | First 150 characters of the summary text. `null` if not yet processed. |
| `action_item_count` | Number of action items in the summary. `0` if not yet processed. |
| `audio_duration_seconds` | `null` until Whisper processes the audio (duration is extracted during transcription). |

#### What Happens Inside

```
1. Extract user_id from JWT (request.user.id)
2. Build MongoDB query:
   base = { user_id: ObjectId(request.user.id) }
   + status filter (if provided)
   + tag filter (if provided)
   + title search (if provided): { title: { $regex: search_term, $options: "i" } }
3. Sort by ordering parameter (default: { created_at: -1 })
4. Apply pagination: skip + limit
5. Execute query → list of conversation documents
6. Batch fetch summaries for this page:
   db.summaries.find({ conversation_id: { $in: [page_ids] } },
                     { summary_text: 1, action_items: 1 })
   → For each: extract first 150 chars as preview, count action items
7. Merge conversation data + summary data
8. Build paginated response with count, next, previous
```

#### Indexes Used

```
{ user_id: 1, created_at: -1 }   ← Dashboard listing (primary query)
{ user_id: 1, status: 1 }        ← When filtering by status
{ user_id: 1, tags: 1 }          ← When filtering by tag
{ conversation_id: 1 }           ← Summary batch lookup
```

---

### C2. Upload Audio

```
POST /api/conversations/upload/
```

**Purpose:** Upload an audio file and start the AI processing pipeline.

**Auth:** ✅ Required.

#### Request

```
Headers:
  Authorization: Bearer <access_token>
  Content-Type: multipart/form-data           ← NOT application/json

Body (form data):
  audio_file: <binary file>                   ← The audio file
  title: "DBMS Lecture 5 - Normalisation"     ← Optional
  tags: "dbms,lecture,normalisation"           ← Optional, comma-separated
```

#### Validation Rules

| Field | Type | Required | Rules |
|---|---|---|---|
| `audio_file` | file | ✅ | **MIME type** must be one of: `audio/mpeg`, `audio/wav`, `audio/x-wav`, `audio/mp4`, `audio/x-m4a`, `audio/webm`. **Size** must be ≤ 100 MB. Must not be empty (0 bytes). MIME check uses file header bytes, NOT the file extension. |
| `title` | string | ❌ | 1–200 characters. If not provided, auto-generated from original filename (without extension). Stripped of leading/trailing whitespace. |
| `tags` | string | ❌ | Comma-separated. Each tag: 1–50 characters, alphanumeric + hyphens. Max 20 tags. Normalised to lowercase. Duplicates removed. |

#### Responses

**202 Accepted** — Upload successful, processing started:
```json
{
  "id": "665f1b3c4d5e6f7a8b9c0e1f",
  "title": "DBMS Lecture 5 - Normalisation",
  "tags": ["dbms", "lecture", "normalisation"],
  "status": "processing",
  "audio_file_name": "dbms_lecture_5.mp3",
  "audio_file_size_bytes": 15728640,
  "created_at": "2026-06-07T09:00:00Z",
  "message": "Audio uploaded successfully. Processing has begun."
}
```

**400 Bad Request** — Validation failed:
```json
{
  "errors": {
    "audio_file": [
      "File type 'application/pdf' is not supported. Allowed types: audio/mpeg, audio/wav, audio/mp4, audio/webm."
    ]
  }
}
```

**413 Payload Too Large** — File exceeds size limit:
```json
{
  "error": "File size exceeds the 100 MB limit."
}
```

#### What Happens Inside

```
 1. DRF parses multipart form data → request.FILES["audio_file"] + request.data
 2. UploadSerializer validates:
    a. File MIME type (read magic bytes from file header, NOT extension)
    b. File size ≤ 100 MB
    c. Title length and format
    d. Tags parsing and validation
 3. If validation fails → return 400 with errors
 4. Generate safe storage path:
    media/audio/{user_id}/{uuid4}.{original_ext}
    NEVER use the user's original filename for storage.
 5. Write file to disk (streaming, not loading entire file into memory)
 6. Create Conversation document in MongoDB:
    {
      user_id: request.user.id,
      title: validated_title,
      tags: validated_tags,
      status: "processing",
      audio_file_path: "audio/{user_id}/{uuid4}.mp3",
      audio_file_name: "dbms_lecture_5.mp3",    ← original name (for display only)
      audio_file_size_bytes: 15728640,
      audio_mime_type: "audio/mpeg",
      audio_duration_seconds: null,              ← set later by Whisper
      created_at: now(),
      updated_at: now()
    }
 7. Dispatch Celery task:
    process_conversation.delay(str(conversation.id))
 8. Return 202 Accepted immediately (Django does NOT wait for processing)
```

#### Security Considerations

```
┌──────────────────────────────────────────────────────────────────────┐
│  FILE UPLOAD SECURITY                                                │
│                                                                      │
│  THREAT 1: Fake file extension                                      │
│    Attack: User uploads "malware.php" renamed to "lecture.mp3"      │
│    Defence: Check MIME type from file HEADER BYTES, not extension   │
│                                                                      │
│  THREAT 2: Path traversal                                           │
│    Attack: Filename "../../etc/passwd.mp3"                          │
│    Defence: NEVER use user-provided filename for storage.           │
│    Generate UUID-based filename. Store original name separately.    │
│                                                                      │
│  THREAT 3: Disk exhaustion                                          │
│    Attack: Upload hundreds of 100 MB files                          │
│    Defence: Rate limit uploads. Consider per-user storage quotas.   │
│                                                                      │
│  THREAT 4: Memory exhaustion                                        │
│    Attack: Upload huge file that gets loaded into memory            │
│    Defence: Django's UploadedFile uses streaming for files > 2.5 MB │
│    Also enforce limit at Nginx level (client_max_body_size 100m)    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

### C3. Get Conversation Detail

```
GET /api/conversations/{id}/
```

**Purpose:** Retrieve full conversation data including transcript and summary.

**Auth:** ✅ Required. User must own the conversation.

#### Request

```
Headers:
  Authorization: Bearer <access_token>

URL Parameters:
  id: string (24-character hex ObjectId)
```

#### Validation Rules

| Parameter | Rules |
|---|---|
| `id` | Must be a valid 24-character hexadecimal string. If invalid format → 400. If valid but not found (or not owned by user) → 404. |

#### Responses

**200 OK** — Full conversation data:
```json
{
  "id": "665f1b3c4d5e6f7a8b9c0e1f",
  "title": "DBMS Lecture 5 - Normalisation",
  "tags": ["dbms", "lecture", "normalisation"],
  "status": "ready",
  "error_message": null,
  "audio_file_name": "dbms_lecture_5.mp3",
  "audio_file_size_bytes": 15728640,
  "audio_duration_seconds": 1847,
  "audio_mime_type": "audio/mpeg",
  "created_at": "2026-06-07T09:00:00Z",
  "updated_at": "2026-06-07T09:04:23Z",

  "transcript": {
    "full_text": "Today we will discuss normalisation. Normalisation is the process of organising data...",
    "segments": [
      { "start": 0.0, "end": 3.8, "text": "Today we will discuss normalisation." },
      { "start": 3.8, "end": 9.2, "text": "Normalisation is the process of organising data in a database to reduce redundancy." },
      { "start": 9.2, "end": 15.1, "text": "There are several normal forms, starting with first normal form." }
    ],
    "word_count": 4523,
    "language": "en",
    "whisper_model": "base"
  },

  "summary": {
    "summary_text": "This lecture covered database normalisation, explaining the first three normal forms (1NF, 2NF, 3NF). The professor demonstrated each form using a student grades table.",
    "key_points": [
      "1NF eliminates repeating groups — each cell must contain a single atomic value",
      "2NF removes partial dependencies — every non-key attribute must depend on the entire primary key",
      "3NF removes transitive dependencies — non-key attributes must not depend on other non-key attributes"
    ],
    "action_items": [
      { "index": 0, "text": "Complete normalisation exercise from Chapter 7", "deadline": "2026-06-10", "completed": false },
      { "index": 1, "text": "Practice converting unnormalised tables to 3NF", "deadline": null, "completed": true }
    ],
    "model_used": "gemini-1.5-flash"
  }
}
```

**When status is "processing"** — transcript and summary are null:
```json
{
  "id": "665f2c4d5e6f7a8b9c0f1e2d",
  "title": "Team Standup",
  "status": "processing",
  "transcript": null,
  "summary": null,
  "error_message": null,
  "..."
}
```

**When status is "failed"** — error_message is populated:
```json
{
  "id": "665f3d5e6f7a8b9c0f2e3d4e",
  "title": "Client Call",
  "status": "failed",
  "transcript": null,
  "summary": null,
  "error_message": "Transcription failed: Audio file appears to be corrupted or contains no speech."
}
```

**400 Bad Request** — Invalid ID format:
```json
{
  "error": "Invalid conversation ID format."
}
```

**404 Not Found** — Conversation doesn't exist or doesn't belong to user:
```json
{
  "error": "Conversation not found."
}
```

#### What Happens Inside

```
1. Validate {id} is a valid ObjectId format → 400 if not
2. Query: db.conversations.findOne({ _id: ObjectId(id), user_id: request.user.id })
   ← BOTH conditions MUST be present (security)
3. If not found → 404 (same response whether it doesn't exist or belongs to another user)
4. If status == "ready":
   a. Fetch transcript: db.transcripts.findOne({ conversation_id: id })
   b. Fetch summary: db.summaries.findOne({ conversation_id: id })
5. If status == "processing" or "failed":
   a. Set transcript = null, summary = null
6. Combine and return
```

---

### C4. Update Conversation

```
PATCH /api/conversations/{id}/
```

**Purpose:** Update conversation metadata (title and/or tags).

**Auth:** ✅ Required. User must own the conversation.

#### Request

```
Headers:
  Authorization: Bearer <access_token>
  Content-Type: application/json

Body (partial — only include fields you want to change):
{
  "title": "DBMS Lecture 5 - Normalisation (Updated)",
  "tags": ["dbms", "lecture", "normalisation", "3nf"]
}
```

#### Validation Rules

| Field | Type | Required | Rules |
|---|---|---|---|
| `title` | string | ❌ | 1–200 characters. Stripped of whitespace. |
| `tags` | array[string] | ❌ | Each tag: 1–50 chars. Max 20 tags. Normalised to lowercase. |

**Fields that CANNOT be updated via this endpoint:** `status`, `audio_file_path`, `user_id`, `created_at`. These are system-managed.

#### Responses

**200 OK:**
```json
{
  "id": "665f1b3c4d5e6f7a8b9c0e1f",
  "title": "DBMS Lecture 5 - Normalisation (Updated)",
  "tags": ["dbms", "lecture", "normalisation", "3nf"],
  "updated_at": "2026-06-07T12:30:00Z"
}
```

**400 Bad Request:**
```json
{
  "errors": {
    "title": ["Title must be between 1 and 200 characters."]
  }
}
```

**404 Not Found:**
```json
{
  "error": "Conversation not found."
}
```

---

### C5. Delete Conversation

```
DELETE /api/conversations/{id}/
```

**Purpose:** Permanently delete a conversation and **all** associated data.

**Auth:** ✅ Required. User must own the conversation.

#### Request

```
Headers:
  Authorization: Bearer <access_token>

No request body.
```

#### Responses

**204 No Content** — Deletion successful:
```
(empty body)
```

**404 Not Found:**
```json
{
  "error": "Conversation not found."
}
```

#### What Happens Inside (Cascade Delete)

```
THIS IS THE MOST DANGEROUS OPERATION IN KAIRO.
Order matters. Children first, parent last.

1. Verify ownership: conversations.findOne({ _id: id, user_id: user_id })
2. If not found → 404
3. Delete embeddings:    db.embeddings.deleteMany({ conversation_id: id })
4. Remove from FAISS:    search_service.remove_vectors(conversation_id)
5. Delete summary:       db.summaries.deleteOne({ conversation_id: id })
6. Delete transcript:    db.transcripts.deleteOne({ conversation_id: id })
7. Delete audio file:    os.remove(conversation.audio_file_path)
8. Delete conversation:  db.conversations.deleteOne({ _id: id })
9. Return 204

If any step fails after step 3, you have orphaned data.
In production: wrap in a try/catch and log failures for cleanup.
```

---

### C6. Retry Failed Processing

```
POST /api/conversations/{id}/retry/
```

**Purpose:** Restart the AI processing pipeline for a conversation that failed.

**Auth:** ✅ Required. User must own the conversation.

#### Request

```
Headers:
  Authorization: Bearer <access_token>

No request body.
```

#### Validation Rules

| Condition | Error |
|---|---|
| Conversation not found / not owned | 404 |
| Conversation status is NOT `"failed"` | 400: "Only failed conversations can be retried." |

#### Responses

**202 Accepted:**
```json
{
  "id": "665f1b3c4d5e6f7a8b9c0e1f",
  "status": "processing",
  "message": "Processing has been restarted."
}
```

**400 Bad Request:**
```json
{
  "error": "Only conversations with status 'failed' can be retried."
}
```

#### What Happens Inside

```
1. Verify ownership
2. Check status == "failed" → 400 if not
3. Clean up partial data from previous attempt:
   a. Delete transcript if exists (may be incomplete)
   b. Delete summary if exists
   c. Delete embeddings if any
   d. Remove vectors from FAISS
4. Update conversation: status → "processing", error_message → null
5. Dispatch Celery task: process_conversation.delay(conversation_id)
6. Return 202
```

---

### C7. Toggle Action Item

```
PATCH /api/conversations/{id}/action-items/{index}/
```

**Purpose:** Mark an action item as completed or uncompleted.

**Auth:** ✅ Required. User must own the conversation.

#### Request

```
Headers:
  Authorization: Bearer <access_token>
  Content-Type: application/json

URL Parameters:
  id: string       ← Conversation ID (24-char hex)
  index: integer   ← Action item index within the summary's action_items array (0-based)

Body:
{
  "completed": true
}
```

#### Validation Rules

| Field | Type | Required | Rules |
|---|---|---|---|
| `id` | string (URL) | ✅ | Valid ObjectId. User must own the conversation. |
| `index` | integer (URL) | ✅ | ≥ 0. Must be < length of action_items array. |
| `completed` | boolean (body) | ✅ | Must be `true` or `false`. |

#### Responses

**200 OK:**
```json
{
  "index": 0,
  "text": "Complete normalisation exercise from Chapter 7",
  "deadline": "2026-06-10",
  "completed": true
}
```

**400 Bad Request** — Invalid index:
```json
{
  "error": "Action item index 5 is out of range. This conversation has 2 action items (indices 0-1)."
}
```

**404 Not Found:**
```json
{
  "error": "Conversation not found."
}
```

#### What Happens Inside

```
1. Verify conversation ownership
2. Fetch summary: db.summaries.findOne({ conversation_id: id })
3. If no summary → 404 (conversation hasn't been processed yet)
4. If index >= len(summary.action_items) → 400
5. Update the specific array element using MongoDB positional update:
   db.summaries.updateOne(
     { conversation_id: id },
     { $set: { "action_items.{index}.completed": completed } }
   )
6. Return the updated action item
```

---

## 4. Search APIs

### Overview

| # | Endpoint | Method | Auth | Purpose |
|---|---|---|---|---|
| S1 | `/api/search/` | POST | ✅ | Semantic search across all conversations |

---

### S1. Semantic Search

```
POST /api/search/
```

**Purpose:** Search across all of the user's conversations using natural language. Uses vector embeddings + FAISS for semantic similarity matching.

**Auth:** ✅ Required.

#### Request

```
Headers:
  Authorization: Bearer <access_token>
  Content-Type: application/json

Body:
{
  "query": "What was said about normalisation?",
  "top_k": 5
}
```

#### Validation Rules

| Field | Type | Required | Rules |
|---|---|---|---|
| `query` | string | ✅ | 3–500 characters. Stripped of leading/trailing whitespace. Must contain at least one non-whitespace word after trimming. |
| `top_k` | integer | ❌ | 1–20. Default: 5. Clamped to range if out of bounds. |

#### Responses

**200 OK** — Results found:
```json
{
  "query": "What was said about normalisation?",
  "results": [
    {
      "conversation_id": "665f1b3c4d5e6f7a8b9c0e1f",
      "conversation_title": "DBMS Lecture 5 - Normalisation",
      "chunk_text": "Normalisation is the process of organising data in a database to reduce redundancy. There are several normal forms, starting with first normal form which requires that each column contain atomic values...",
      "relevance_score": 0.92,
      "start_time": 3.8,
      "end_time": 45.6
    },
    {
      "conversation_id": "665f2d4e6f7a8b9c0f1e3d4e",
      "conversation_title": "DBMS Lecture 3 - ER Diagrams",
      "chunk_text": "...before we normalise the data, we need to understand the entity relationships that exist in the system...",
      "relevance_score": 0.71,
      "start_time": 750.2,
      "end_time": 795.0
    }
  ],
  "total_results": 2
}
```

**200 OK** — No results (not an error):
```json
{
  "query": "quantum physics",
  "results": [],
  "total_results": 0
}
```

**400 Bad Request:**
```json
{
  "errors": {
    "query": ["Query must be between 3 and 500 characters."]
  }
}
```

#### What Happens Inside

```
1. Validate query text (length, non-empty after trim)
2. Generate embedding vector for query:
   query_vector = ai.embeddings.generate_single_embedding(query_text)
   → Returns a 384-dimensional float vector
3. Search FAISS index:
   matches = faiss_index.search(query_vector, top_k * 3)
   → Returns: [(embedding_doc_id, distance_score), ...]
   → We request 3x top_k to allow for user_id filtering
4. Fetch matched embedding documents from MongoDB:
   db.embeddings.find({ _id: { $in: matched_ids }, user_id: request.user.id })
   ← CRITICAL: Filter by user_id! FAISS doesn't know about users.
5. Keep only results belonging to the current user
6. Fetch conversation metadata for display:
   db.conversations.find({ _id: { $in: unique_conversation_ids } })
7. Combine: chunk_text + conversation_title + timestamps
8. Sort by relevance_score descending
9. Trim to top_k results
10. Return

WHY POST AND NOT GET?
  GET requests put parameters in the URL: /api/search/?query=What+was+said...
  Long queries get truncated by URL length limits (2048 chars in some browsers).
  POST allows the query in the request body with no length constraint.
  This is a common REST convention for search endpoints.
```

#### Security: The User Isolation Problem

```
┌──────────────────────────────────────────────────────────────────────┐
│  THE MOST CRITICAL SECURITY ISSUE IN SEARCH                         │
│                                                                      │
│  FAISS is a DUMB vector index. It knows nothing about users.        │
│  When you search FAISS, it returns the K most similar vectors       │
│  across ALL users.                                                   │
│                                                                      │
│  WITHOUT user filtering:                                             │
│    User A searches "project budget"                                  │
│    FAISS returns chunks from User B's confidential client call!     │
│                                                                      │
│  WITH user filtering (what we do):                                   │
│    1. FAISS returns top matches (all users)                          │
│    2. We filter: keep only user_id == current_user                  │
│    3. This may reduce results below top_k                           │
│    4. That's why we request 3x top_k from FAISS — to have          │
│       enough results after filtering                                 │
│                                                                      │
│  ALTERNATIVE (future optimisation):                                  │
│    Build per-user FAISS indexes. More memory, but no post-filter.   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Error Reference

### Complete Error Catalogue

Every error Kairo can return, organised by HTTP status code:

```
┌────────┬──────────────────────────────────┬─────────────────────────────────────┐
│ Status │ Error Message                    │ When It Occurs                      │
├────────┼──────────────────────────────────┼─────────────────────────────────────┤
│        │                                  │                                     │
│  400   │ VALIDATION ERRORS                │                                     │
│        │                                  │                                     │
│        │ "This field is required."        │ Required field missing from body    │
│        │ "Enter a valid email address."   │ Malformed email format              │
│        │ "A user with this email already  │ Duplicate email on register         │
│        │  exists."                        │                                     │
│        │ "Username must be 3-30 chars,    │ Invalid username format             │
│        │  alphanumeric and underscores."  │                                     │
│        │ "Password must be at least 8     │ Password too short                  │
│        │  characters."                    │                                     │
│        │ "Passwords do not match."        │ password != password_confirm        │
│        │ "File type 'X' is not supported."│ Wrong MIME type on upload           │
│        │ "Title must be between 1-200     │ Title too long or empty             │
│        │  characters."                    │                                     │
│        │ "Maximum 20 tags allowed."       │ Too many tags                       │
│        │ "Query must be between 3-500     │ Search query too short/long         │
│        │  characters."                    │                                     │
│        │ "Invalid conversation ID format."│ URL param is not valid ObjectId     │
│        │ "Only conversations with status  │ Retry on non-failed conversation    │
│        │  'failed' can be retried."       │                                     │
│        │ "Action item index X out of      │ Toggle with invalid array index     │
│        │  range."                         │                                     │
│        │                                  │                                     │
├────────┼──────────────────────────────────┼─────────────────────────────────────┤
│        │                                  │                                     │
│  401   │ AUTHENTICATION ERRORS            │                                     │
│        │                                  │                                     │
│        │ "Authentication credentials were │ No Authorization header provided    │
│        │  not provided."                  │                                     │
│        │ "Invalid or expired token."      │ JWT decode failed or expired        │
│        │ "Invalid email or password."     │ Wrong credentials on login          │
│        │ "Refresh token is invalid or     │ Bad refresh token                   │
│        │  expired. Please log in again."  │                                     │
│        │                                  │                                     │
├────────┼──────────────────────────────────┼─────────────────────────────────────┤
│        │                                  │                                     │
│  404   │ NOT FOUND ERRORS                 │                                     │
│        │                                  │                                     │
│        │ "Conversation not found."        │ ID doesn't exist OR user doesn't   │
│        │                                  │ own it (SAME message for both)      │
│        │                                  │                                     │
├────────┼──────────────────────────────────┼─────────────────────────────────────┤
│        │                                  │                                     │
│  413   │ PAYLOAD ERRORS                   │                                     │
│        │                                  │                                     │
│        │ "File size exceeds the 100 MB    │ Audio file too large               │
│        │  limit."                         │                                     │
│        │                                  │                                     │
├────────┼──────────────────────────────────┼─────────────────────────────────────┤
│        │                                  │                                     │
│  429   │ RATE LIMIT ERRORS                │                                     │
│        │                                  │                                     │
│        │ "Too many registration attempts."│ Register rate limit hit             │
│        │ "Too many login attempts. Try    │ Login rate limit hit                │
│        │  again in 15 minutes."           │                                     │
│        │ "Too many requests. Please slow  │ General API rate limit              │
│        │  down."                          │                                     │
│        │                                  │                                     │
├────────┼──────────────────────────────────┼─────────────────────────────────────┤
│        │                                  │                                     │
│  500   │ SERVER ERRORS                    │                                     │
│        │                                  │                                     │
│        │ "An unexpected error occurred.   │ Unhandled exception, DB down,      │
│        │  Please try again later."        │ any uncaught error                  │
│        │                                  │                                     │
│        │ NEVER expose stack traces, file  │                                     │
│        │ paths, or internal error details │                                     │
│        │ to the client in production.     │                                     │
│        │                                  │                                     │
└────────┴──────────────────────────────────┴─────────────────────────────────────┘
```

### DRF Exception Handler

```
CONCEPT: Django REST Framework has a centralised exception handler
that intercepts all errors before they reach the client. You customise
it to ensure EVERY error follows the same JSON format.

WHAT IT DOES:
  → Catches DRF validation errors → formats as { "errors": { field: [msgs] } }
  → Catches DRF auth errors → formats as { "error": "message" }
  → Catches unhandled Python exceptions → formats as { "error": "generic message" }
  → Logs the full traceback server-side (for debugging)
  → NEVER sends the traceback to the client

WHERE: common/exceptions.py → registered in settings as DEFAULT_EXCEPTION_HANDLER
```

---

## 6. Rate Limiting

### Rate Limits by Endpoint

| Endpoint | Limit | Window | Scope |
|---|---|---|---|
| `POST /auth/register/` | 5 requests | per hour | per IP address |
| `POST /auth/login/` | 5 requests | per 15 min | per email + IP |
| `POST /auth/token/refresh/` | 30 requests | per hour | per user |
| `POST /conversations/upload/` | 10 requests | per hour | per user |
| `GET /conversations/` | 60 requests | per minute | per user |
| `GET /conversations/{id}/` | 60 requests | per minute | per user |
| `POST /search/` | 30 requests | per minute | per user |
| All other endpoints | 120 requests | per minute | per user |

### How Rate Limiting Works

```
APPROACH: Use DRF's built-in throttling classes.

DRF provides:
  → AnonRateThrottle: limits by IP (for unauthenticated endpoints)
  → UserRateThrottle: limits by user ID (for authenticated endpoints)

Custom throttle classes for specific endpoints (e.g., login).

When limit is hit:
  → Return 429 Too Many Requests
  → Include Retry-After header (seconds until the limit resets)
  → Response body: { "error": "Too many requests. Try again in X seconds." }
```

---

## 7. CORS Configuration

### What Is CORS and Why Does It Matter?

```
PROBLEM:
  React runs on http://localhost:3000 (dev server)
  Django runs on http://localhost:8000

  These are DIFFERENT ORIGINS (different ports = different origin).
  By default, browsers BLOCK cross-origin requests for security.
  Without CORS configuration, React cannot call Django's API.

SOLUTION:
  Django must include Access-Control-Allow-Origin headers
  telling the browser "requests from localhost:3000 are allowed."

CONFIGURATION:
  Development:
    CORS_ALLOWED_ORIGINS = ["http://localhost:3000"]

  Production:
    CORS_ALLOWED_ORIGINS = ["https://kairo.yourdomain.com"]

  NEVER use CORS_ALLOW_ALL_ORIGINS = True in production.
  It allows ANY website to call your API — an attacker's site
  could make requests using your user's cookies/tokens.
```

---

## 8. API Design Decisions — Why It's Done This Way

### Decision Log

| # | Decision | Chosen | Alternative | Why |
|---|---|---|---|---|
| 1 | **Login returns user data** | Return tokens + user info | Return only tokens (client calls /me separately) | Saves an extra API call on login. React has everything it needs in one response. |
| 2 | **Search uses POST** | POST with body | GET with query params | Long search queries would get truncated in URLs. POST allows arbitrary-length queries in the body. |
| 3 | **Upload returns 202** | 202 Accepted | 201 Created | The conversation isn't "created" yet — it's being processed. 202 correctly signals asynchronous processing. |
| 4 | **Delete returns 204** | 204 No Content | 200 with deleted resource | The resource no longer exists — there's nothing meaningful to return. 204 is the REST convention. |
| 5 | **404 for wrong owner** | 404 Not Found | 403 Forbidden | 403 confirms the resource EXISTS but is forbidden. 404 reveals nothing — same response for "doesn't exist" and "not yours." |
| 6 | **Action items by index** | URL param /{index}/ | Body { "action_item_id": "..." } | Action items are embedded in the summary — they don't have their own IDs. Array index is the natural identifier. |
| 7 | **Tags as comma string in upload** | Comma-separated string | JSON array | multipart/form-data doesn't support nested JSON. Comma-separated is the simplest encoding for a flat list in form data. The PATCH endpoint accepts a proper JSON array since it uses application/json. |
| 8 | **Pagination style** | Page number pagination | Cursor-based pagination | Simpler to implement. Cursor-based is better for infinite scroll (future), but page numbers work well for dashboard with page buttons. |
| 9 | **Same error for email not found / wrong password** | One generic message | Separate messages | Separate messages enable user enumeration attacks ("this email is registered"). Security over UX. |
| 10 | **No PATCH for password** | Not in MVP | PATCH /auth/password/ | Password change requires careful security (verify old password, rate limit, invalidate sessions). Defer to post-MVP. |

### Endpoint Count Summary

```
┌────────────────────┬──────────────────┐
│ Domain             │ Endpoint Count   │
├────────────────────┼──────────────────┤
│ Authentication     │ 4 endpoints      │
│ Conversations      │ 7 endpoints      │
│ Search             │ 1 endpoint       │
├────────────────────┼──────────────────┤
│ TOTAL MVP          │ 12 endpoints     │
└────────────────────┴──────────────────┘

12 endpoints is a lean, focused API. Each one has a clear purpose.
Adding endpoints is easy later; removing them breaks clients.
Start small, expand as needed.
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KAIRO API — QUICK REFERENCE                              │
│                                                                             │
│  AUTH                                                                       │
│  ────                                                                       │
│  POST /api/auth/register/              → 201  { id, email, username }      │
│  POST /api/auth/login/                 → 200  { access_token, refresh, user}│
│  POST /api/auth/token/refresh/         → 200  { access_token }             │
│  GET  /api/auth/me/                    → 200  { id, email, username, ... } │
│                                                                             │
│  CONVERSATIONS                                                              │
│  ─────────────                                                              │
│  GET    /api/conversations/            → 200  { count, results: [...] }    │
│  POST   /api/conversations/upload/     → 202  { id, status: processing }   │
│  GET    /api/conversations/{id}/       → 200  { id, transcript, summary }  │
│  PATCH  /api/conversations/{id}/       → 200  { id, title, tags }          │
│  DELETE /api/conversations/{id}/       → 204  (empty)                      │
│  POST   /api/conversations/{id}/retry/ → 202  { id, status: processing }   │
│  PATCH  .../action-items/{index}/      → 200  { index, text, completed }   │
│                                                                             │
│  SEARCH                                                                     │
│  ──────                                                                     │
│  POST /api/search/                     → 200  { query, results: [...] }    │
│                                                                             │
│  ALL AUTHENTICATED ENDPOINTS REQUIRE:                                       │
│    Authorization: Bearer <access_token>                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*This document should be read alongside [backend-design.md](file:///Users/mohammed/kairo/docs/backend-design.md) which explains the architectural reasoning behind these APIs.*

*Last updated: 2026-06-07*
*Author: Mohammed Kutubudin Kapasi*
