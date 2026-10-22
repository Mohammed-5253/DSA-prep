# Kairo — Backend Design

> Backend architecture, API contracts, and integration strategies for **Kairo: AI Conversational Memory Assistant**
> Companion to [architecture.md](file:///Users/mohammed/kairo/docs/architecture.md) and [database-design.md](file:///Users/mohammed/kairo/docs/database-design.md)

**Last updated:** 2026-06-07
**Author:** Mohammed Kutubudin Kapasi

---

## Table of Contents

1. [How a Senior Engineer Thinks About Backend Design](#1-how-a-senior-engineer-thinks-about-backend-design)
2. [Django Project Structure](#2-django-project-structure)
3. [Django Apps — Responsibilities & Boundaries](#3-django-apps--responsibilities--boundaries)
4. [API Contract — Every Endpoint](#4-api-contract--every-endpoint)
5. [Authentication Strategy](#5-authentication-strategy)
6. [File Upload Strategy](#6-file-upload-strategy)
7. [Background Processing Strategy (Celery)](#7-background-processing-strategy-celery)
8. [AI Integration Strategy](#8-ai-integration-strategy)
9. [Database Interaction Patterns](#9-database-interaction-patterns)
10. [Error Handling Strategy](#10-error-handling-strategy)
11. [Settings & Configuration Strategy](#11-settings--configuration-strategy)
12. [Build Order — What to Implement First](#12-build-order--what-to-implement-first)

---

## 1. How a Senior Engineer Thinks About Backend Design

Before diving into specifics, understand the **thinking process** an experienced engineer follows. This is more valuable than any code.

### The Three Questions

Every backend design decision starts with these three questions:

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   1. "What does each API endpoint need to DO?"                     │
│       → Define the inputs, outputs, and side effects               │
│       → This gives you your API contract                           │
│                                                                     │
│   2. "What can go wrong?"                                          │
│       → Invalid input, missing data, failed AI processing,        │
│         unauthorized access, server overload                       │
│       → This gives you your error handling strategy                │
│                                                                     │
│   3. "What should NOT happen in the request/response cycle?"       │
│       → Anything that takes > 1 second                             │
│       → This gives you your async processing strategy              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### The Single Responsibility Principle for Apps

Django "apps" are not just folders — they represent **bounded contexts**. Each app owns a specific domain and should be able to answer: *"What one thing am I responsible for?"*

If an app has two answers, it should probably be split.

---

## 2. Django Project Structure

### Complete Directory Layout

```
kairo_backend/
│
├── manage.py                          ← Django entry point
│
├── config/                            ← Project-level configuration
│   ├── __init__.py
│   ├── settings/                      ← Split settings by environment
│   │   ├── __init__.py
│   │   ├── base.py                    ← Shared settings (all environments)
│   │   ├── development.py             ← Dev overrides (DEBUG=True, etc.)
│   │   └── production.py              ← Prod overrides (security hardening)
│   ├── urls.py                        ← Root URL router
│   ├── wsgi.py                        ← WSGI entry point (production)
│   └── celery.py                      ← Celery app initialisation
│
├── apps/                              ← All Django apps live here
│   ├── __init__.py
│   │
│   ├── accounts/                      ← User identity & authentication
│   │   ├── __init__.py
│   │   ├── documents.py               ← MongoEngine User document
│   │   ├── serializers.py             ← DRF serializers for auth
│   │   ├── views.py                   ← Register, login, refresh endpoints
│   │   ├── authentication.py          ← Custom JWT auth backend for DRF
│   │   ├── urls.py                    ← /api/auth/... routes
│   │   └── tests.py
│   │
│   ├── conversations/                 ← Core business logic
│   │   ├── __init__.py
│   │   ├── documents.py               ← MongoEngine Conversation, Transcript, Summary docs
│   │   ├── serializers.py             ← Request/response validation
│   │   ├── views.py                   ← CRUD + upload + status endpoints
│   │   ├── tasks.py                   ← Celery task definitions
│   │   ├── services.py                ← Business logic (separate from views)
│   │   ├── urls.py                    ← /api/conversations/... routes
│   │   └── tests.py
│   │
│   ├── search/                        ← Semantic search
│   │   ├── __init__.py
│   │   ├── documents.py               ← MongoEngine Embedding document
│   │   ├── services.py                ← FAISS index management + query logic
│   │   ├── serializers.py             ← Search request/response shapes
│   │   ├── views.py                   ← Search endpoint
│   │   ├── urls.py                    ← /api/search/... routes
│   │   └── tests.py
│   │
│   └── ai/                            ← AI model integrations (NEW — separated from conversations)
│       ├── __init__.py
│       ├── transcription.py           ← Whisper wrapper
│       ├── summarisation.py           ← LLM wrapper (Gemini/OpenAI)
│       ├── embeddings.py              ← Sentence-transformer wrapper
│       └── prompts.py                 ← All LLM prompt templates
│
├── common/                            ← Shared cross-cutting utilities
│   ├── __init__.py
│   ├── permissions.py                 ← Custom DRF permission classes
│   ├── exceptions.py                  ← Centralised error handling
│   ├── validators.py                  ← File validation, input sanitisation
│   ├── pagination.py                  ← Custom pagination classes
│   └── middleware.py                  ← Request logging, error formatting
│
├── media/                             ← Uploaded audio files (local dev)
│   └── audio/                         ← Organised by user_id subdirectories
│
├── requirements/                      ← Dependency management
│   ├── base.txt                       ← Shared dependencies
│   ├── development.txt                ← Dev-only (debugger, test tools)
│   └── production.txt                 ← Prod-only (gunicorn, sentry)
│
├── .env.example                       ← Template for environment variables
├── .env                               ← Actual secrets (NEVER committed to Git)
└── .gitignore                         ← Must include .env, media/, *.faiss
```

### Why This Structure?

| Decision | Reasoning | Alternative Considered |
|---|---|---|
| **`config/settings/` split** | `development.py` has `DEBUG=True`; `production.py` has `DEBUG=False`, secure cookies, HTTPS. One `settings.py` leads to `if DEBUG:` conditionals everywhere — messy and error-prone. | Single `settings.py` |
| **`documents.py` instead of `models.py`** | We're using MongoEngine, not Django's ORM. Naming the file `documents.py` makes it immediately clear these are MongoDB document classes, not SQL models. Prevents confusion. | Keep `models.py` name |
| **Separate `ai/` app** | AI integrations (Whisper, Gemini, sentence-transformers) are **shared infrastructure**, not owned by any single app. Both `conversations/tasks.py` and `search/services.py` need embeddings. Centralising AI wrappers in one place avoids duplication and makes swapping providers easy. | Put AI code in `conversations/` |
| **`services.py` layer** | Views should be thin — they handle HTTP concerns (request parsing, response formatting). Business logic (validation rules, orchestration, multi-step operations) goes in services. This separation makes business logic testable without HTTP. | Put everything in views |
| **`requirements/` split** | You don't need `gunicorn` in development or `django-debug-toolbar` in production. Split files keep environments clean. | Single `requirements.txt` |

---

## 3. Django Apps — Responsibilities & Boundaries

### The Four Apps

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        APP RESPONSIBILITY MAP                           │
│                                                                         │
│   ┌──────────────┐   ┌──────────────────┐   ┌──────────┐   ┌───────┐  │
│   │  accounts    │   │  conversations   │   │  search  │   │  ai   │  │
│   │              │   │                  │   │          │   │       │  │
│   │  "Who are    │   │  "What happened  │   │  "Find   │   │  "Run │  │
│   │   you?"      │   │   in the audio?" │   │   it"    │   │   the │  │
│   │              │   │                  │   │          │   │   AI" │  │
│   └──────┬───────┘   └────────┬─────────┘   └────┬─────┘   └───┬───┘  │
│          │                    │                   │             │       │
│          │  depends on        │   depends on      │  depends on │       │
│          │  nothing           │   accounts (user) │  ai (embed) │       │
│          │                    │   ai (tasks)      │             │       │
│          │                    │                   │             │       │
│          ▼                    ▼                   ▼             ▼       │
│   ┌──────────────────────────────────────────────────────────────────┐ │
│   │                        common/                                   │ │
│   │          permissions, validators, exceptions, pagination         │ │
│   └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### App 1: `accounts` — Identity & Authentication

```
Responsibility:
  "I know WHO the user is. I handle registration, login, and token management.
   I know nothing about conversations, audio, or AI."

Owns:
  • users collection in MongoDB
  • JWT token lifecycle (issue, refresh, invalidate)
  • Password hashing and verification
  • Custom DRF authentication backend

Exposes to other apps:
  • The current authenticated user (via request.user in DRF views)
  • User document class (for user_id references)

Does NOT:
  • Know about conversations, transcripts, or summaries
  • Handle file uploads
  • Interact with AI services
```

**Internal structure — what each file does:**

| File | Responsibility |
|---|---|
| `documents.py` | MongoEngine `User` document class (email, password_hash, is_active, timestamps) |
| `serializers.py` | `RegisterSerializer` — validates email uniqueness, password strength. `LoginSerializer` — validates credentials. `TokenSerializer` — shapes the JWT response. |
| `views.py` | `RegisterView` — create user. `LoginView` — verify credentials, return tokens. `TokenRefreshView` — exchange refresh token for new access token. `MeView` — return current user profile. |
| `authentication.py` | Custom DRF authentication class. Reads the `Authorization: Bearer <token>` header, decodes the JWT, loads the User document from MongoDB, sets `request.user`. This is what makes `request.user` work in every other app. |
| `urls.py` | Routes: `/api/auth/register`, `/api/auth/login`, `/api/auth/refresh`, `/api/auth/me` |

---

### App 2: `conversations` — Core Business Logic

```
Responsibility:
  "I manage the lifecycle of a conversation — from audio upload to processed results.
   I orchestrate the AI pipeline but don't run AI models myself."

Owns:
  • conversations collection
  • transcripts collection
  • summaries collection
  • Audio file storage management
  • Celery task orchestration (dispatch and status tracking)

Exposes to other apps:
  • Conversation document class (for search to reference)
  • Conversation status (for frontend polling)

Does NOT:
  • Run Whisper, call Gemini, or generate embeddings directly
    (delegates to ai/ app via Celery tasks)
  • Handle semantic search (that's the search app)
  • Manage user authentication (that's accounts)
```

**Internal structure — what each file does:**

| File | Responsibility |
|---|---|
| `documents.py` | MongoEngine `Conversation`, `Transcript`, `Summary` document classes with field definitions, indexes, and validation rules |
| `serializers.py` | `UploadSerializer` — validates file type, size, title, tags. `ConversationListSerializer` — shapes data for dashboard cards. `ConversationDetailSerializer` — shapes full data for detail page. `ActionItemUpdateSerializer` — validates action item toggle. |
| `views.py` | `UploadView` — receive file, validate, store, enqueue task, return 202. `ConversationListView` — paginated list for dashboard. `ConversationDetailView` — full conversation with transcript + summary. `ConversationDeleteView` — cascade delete all related data. `ActionItemToggleView` — mark action item as complete/incomplete. |
| `services.py` | `ConversationService` — business logic layer. `upload_conversation()` — orchestrates file storage + document creation + task dispatch. `delete_conversation()` — orchestrates cascade deletion (embeddings → summary → transcript → audio file → conversation). `get_dashboard_data()` — combines conversation list with summary previews. |
| `tasks.py` | `process_conversation(conversation_id)` — the Celery task that chains: transcribe → summarise → embed. This file defines the task; the AI work is done by the `ai/` app. |
| `urls.py` | Routes: `/api/conversations/`, `/api/conversations/upload/`, `/api/conversations/{id}/`, `/api/conversations/{id}/action-items/{index}/` |

---

### App 3: `search` — Semantic Search

```
Responsibility:
  "I find relevant information across conversations using natural language queries.
   I manage the FAISS vector index and return ranked search results."

Owns:
  • embeddings collection
  • FAISS index lifecycle (load, search, rebuild)

Exposes to other apps:
  • Embedding document class (for the AI pipeline to write to)
  • add_to_index() method (called by the embedding task)

Does NOT:
  • Generate embeddings (the ai/ app does this)
  • Know about audio files or transcription
  • Manage user authentication
```

**Internal structure:**

| File | Responsibility |
|---|---|
| `documents.py` | MongoEngine `Embedding` document class |
| `services.py` | `SearchService` — the core of semantic search. `initialize_index()` — load or rebuild FAISS from MongoDB. `add_vectors(conversation_id, vectors)` — add new vectors to FAISS + MongoDB. `search(query_text, user_id, top_k=5)` — generate query embedding, search FAISS, filter by user, enrich with metadata from MongoDB. `remove_vectors(conversation_id)` — remove vectors when a conversation is deleted. |
| `serializers.py` | `SearchRequestSerializer` — validates query text length. `SearchResultSerializer` — shapes search results for the frontend. |
| `views.py` | `SearchView` — receive query, call SearchService, return ranked results |
| `urls.py` | Route: `/api/search/` |

---

### App 4: `ai` — AI Model Integrations

```
Responsibility:
  "I am the interface between Kairo and external AI systems.
   I wrap Whisper, Gemini/OpenAI, and sentence-transformers behind
   clean Python interfaces so the rest of the codebase never touches
   AI libraries directly."

Owns:
  • AI model configuration (which model to use, API keys)
  • Prompt templates for LLM interactions
  • Model loading and lifecycle

Exposes to other apps:
  • transcribe(audio_file_path) → transcript text + segments
  • summarise(transcript_text) → summary + key_points + action_items
  • generate_embeddings(text_chunks) → list of vectors

Does NOT:
  • Know about MongoDB collections
  • Know about HTTP/APIs
  • Handle authentication or authorization
  • Decide WHEN to run — it's called by tasks.py in the conversations app
```

**Why a separate `ai/` app?**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WHY SEPARATE THE AI LAYER?                           │
│                                                                         │
│   PROBLEM WITHOUT SEPARATION:                                           │
│   ────────────────────────────                                          │
│   conversations/tasks.py:                                               │
│     import whisper                                                      │
│     import google.generativeai                                          │
│     from sentence_transformers import SentenceTransformer               │
│                                                                         │
│     def process_conversation(conversation_id):                          │
│         model = whisper.load_model("base")   ← 3 different AI libs     │
│         result = model.transcribe(...)       ← mixed with business     │
│         genai.configure(api_key=...)         ← logic and MongoDB       │
│         response = genai.generate(...)       ← operations              │
│         embedder = SentenceTransformer(...)                             │
│         vectors = embedder.encode(...)                                  │
│                                                                         │
│   This file now has 5 responsibilities and imports 3 AI libraries.     │
│   If you switch from Gemini to OpenAI, you edit business logic files.  │
│   If Whisper has a bug, you're debugging inside task orchestration.     │
│                                                                         │
│   SOLUTION WITH SEPARATION:                                             │
│   ─────────────────────────                                             │
│   conversations/tasks.py:                                               │
│     from apps.ai.transcription import transcribe                        │
│     from apps.ai.summarisation import summarise                         │
│     from apps.ai.embeddings import generate_embeddings                  │
│                                                                         │
│     def process_conversation(conversation_id):                          │
│         transcript = transcribe(audio_path)                             │
│         summary = summarise(transcript.full_text)                       │
│         vectors = generate_embeddings(chunks)                           │
│                                                                         │
│   Now tasks.py is ONLY about orchestration. Each AI integration         │
│   is isolated. Swapping Gemini for OpenAI means editing ONE file.      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Internal structure:**

| File | Responsibility |
|---|---|
| `transcription.py` | `transcribe(audio_file_path, model_name="base")` → returns `{ full_text, segments[], word_count, language }`. Wraps the Whisper library. Handles model loading, error handling, and output formatting. |
| `summarisation.py` | `summarise(transcript_text, model="gemini-1.5-flash")` → returns `{ summary_text, key_points[], action_items[] }`. Wraps the Gemini/OpenAI API call. Handles prompt construction, API call, response parsing, and JSON validation. |
| `embeddings.py` | `generate_embeddings(text_chunks)` → returns `list[list[float]]`. Wraps the sentence-transformers library. Handles model loading and batch encoding. |
| `prompts.py` | Stores all LLM prompt templates as constants. Separating prompts from code makes them easy to iterate on, version, and test without changing Python logic. |

---

### Dependency Rules Between Apps

```
                    ┌──────────┐
                    │ accounts │
                    └──────┬───┘
                           │ provides: request.user
                           ▼
                  ┌─────────────────┐
                  │  conversations  │
                  └──┬──────────┬───┘
                     │          │
          imports    │          │  calls in tasks.py
                     │          │
                     ▼          ▼
               ┌──────────┐  ┌──────┐
               │  search   │  │  ai  │
               └──────────┘  └──────┘

RULES:
  ✅ conversations can import from accounts (to get user_id)
  ✅ conversations can import from ai (to run AI in tasks)
  ✅ conversations can import from search (to add/remove vectors)
  ✅ search can import from ai (to generate query embeddings)
  ❌ accounts NEVER imports from conversations (no circular deps)
  ❌ ai NEVER imports from conversations (ai is a pure service layer)
  ❌ search NEVER imports from conversations (avoids coupling)

  common/ can be imported by ANY app (it's shared infrastructure)
```

---

## 4. API Contract — Every Endpoint

This is the **complete API specification** for Kairo's MVP. Every endpoint that the React frontend will call.

### URL Structure

```
/api/
├── auth/
│   ├── register/          POST     Create new user account
│   ├── login/             POST     Authenticate and get tokens
│   ├── token/refresh/     POST     Exchange refresh token for new access token
│   └── me/                GET      Get current user profile
│
├── conversations/
│   ├── /                  GET      List user's conversations (dashboard)
│   ├── upload/            POST     Upload audio file
│   ├── {id}/              GET      Get conversation details + transcript + summary
│   ├── {id}/              DELETE   Delete conversation and all related data
│   ├── {id}/              PATCH    Update conversation metadata (title, tags)
│   ├── {id}/retry/        POST     Retry failed processing
│   └── {id}/action-items/{index}/  PATCH    Toggle action item completion
│
└── search/
    └── /                  POST     Semantic search across conversations
```

---

### 4.1 Authentication Endpoints

#### `POST /api/auth/register/`

```
PURPOSE: Create a new user account.

REQUEST:
  Headers: Content-Type: application/json
  Body:
  {
    "email": "mohammed@example.com",       ← Required, must be valid email, unique
    "username": "mohammed",                ← Required, 3-30 chars, alphanumeric
    "password": "securePassword123!",      ← Required, min 8 chars
    "password_confirm": "securePassword123!"  ← Required, must match password
  }

RESPONSE (Success — 201 Created):
  {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "email": "mohammed@example.com",
    "username": "mohammed",
    "created_at": "2026-06-07T10:30:00Z"
  }

RESPONSE (Validation Error — 400 Bad Request):
  {
    "errors": {
      "email": ["A user with this email already exists."],
      "password": ["Password must be at least 8 characters."]
    }
  }

WHAT HAPPENS INSIDE:
  1. Validate all fields (serializer)
  2. Check email uniqueness (query MongoDB users collection)
  3. Hash password with bcrypt
  4. Create user document in MongoDB
  5. Return user data (WITHOUT password hash)

SECURITY NOTES:
  • Never return the password hash in any response
  • Rate limit this endpoint (prevent mass account creation)
  • Email normalisation: convert to lowercase before storing
```

#### `POST /api/auth/login/`

```
PURPOSE: Authenticate user and issue JWT tokens.

REQUEST:
  Headers: Content-Type: application/json
  Body:
  {
    "email": "mohammed@example.com",
    "password": "securePassword123!"
  }

RESPONSE (Success — 200 OK):
  {
    "access_token": "eyJhbGciOi...",      ← Short-lived (15-30 minutes)
    "refresh_token": "eyJhbGciOi...",      ← Long-lived (7 days)
    "user": {
      "id": "665f1a2b3c4d5e6f7a8b9c0d",
      "email": "mohammed@example.com",
      "username": "mohammed"
    }
  }

RESPONSE (Invalid Credentials — 401 Unauthorized):
  {
    "error": "Invalid email or password."
  }

WHAT HAPPENS INSIDE:
  1. Find user by email in MongoDB
  2. If not found → 401 (DON'T say "email not found" — leaks info)
  3. Verify password against bcrypt hash
  4. If mismatch → 401 (same message as above)
  5. Update last_login timestamp
  6. Generate JWT access + refresh tokens
  7. Return tokens + user info

SECURITY NOTES:
  • Use the SAME error message for "email not found" and "wrong password"
    → Prevents attackers from knowing which emails are registered
  • Rate limit: max 5 attempts per email per 15 minutes
  • Never log passwords (not even in error logs)
```

#### `POST /api/auth/token/refresh/`

```
PURPOSE: Get a new access token using a valid refresh token.
         Called automatically by the React app when the access token expires.

REQUEST:
  Headers: Content-Type: application/json
  Body:
  {
    "refresh_token": "eyJhbGciOi..."
  }

RESPONSE (Success — 200 OK):
  {
    "access_token": "eyJhbGciOi..."     ← New access token
  }

RESPONSE (Expired/Invalid Token — 401 Unauthorized):
  {
    "error": "Refresh token is invalid or expired."
  }
```

#### `GET /api/auth/me/`

```
PURPOSE: Get the currently authenticated user's profile.
         Used by the React app on page load to verify the session.

REQUEST:
  Headers: Authorization: Bearer <access_token>

RESPONSE (Success — 200 OK):
  {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "email": "mohammed@example.com",
    "username": "mohammed",
    "created_at": "2026-06-07T10:30:00Z",
    "last_login": "2026-06-07T15:00:00Z"
  }

RESPONSE (Not Authenticated — 401 Unauthorized):
  {
    "error": "Authentication credentials were not provided."
  }
```

---

### 4.2 Conversation Endpoints

#### `GET /api/conversations/`

```
PURPOSE: List all conversations for the authenticated user (dashboard data).

REQUEST:
  Headers: Authorization: Bearer <access_token>
  Query Params:
    ?page=1              ← Pagination (default: 1)
    ?page_size=20        ← Items per page (default: 20, max: 50)
    ?status=ready        ← Optional filter: "processing", "ready", "failed"
    ?tag=dbms            ← Optional filter by tag
    ?ordering=-created_at ← Sort: "-created_at" (newest first, default) or "created_at"

RESPONSE (Success — 200 OK):
  {
    "count": 42,
    "next": "/api/conversations/?page=2",
    "previous": null,
    "results": [
      {
        "id": "665f1b3c4d5e6f7a8b9c0e1f",
        "title": "DBMS Lecture 5 - Normalisation",
        "tags": ["dbms", "lecture", "normalisation"],
        "status": "ready",
        "audio_duration_seconds": 1847,
        "audio_file_name": "dbms_lecture_5.mp3",
        "created_at": "2026-06-07T09:00:00Z",
        "summary_preview": "This lecture covered database normalisation...",
        "action_item_count": 2
      },
      {
        "id": "665f2c4d5e6f7a8b9c0f1e2d",
        "title": "Team Standup",
        "tags": ["meeting"],
        "status": "processing",
        "audio_duration_seconds": null,
        "audio_file_name": "standup_june7.mp3",
        "created_at": "2026-06-07T10:15:00Z",
        "summary_preview": null,
        "action_item_count": 0
      }
    ]
  }

WHAT HAPPENS INSIDE:
  1. Get current user from JWT (request.user)
  2. Query conversations collection: filter by user_id + any query params
  3. Sort by created_at descending
  4. Paginate results
  5. For each conversation in the page, fetch summary_preview and action_item_count
     from summaries collection (batch query with $in)
  6. Combine and return

SECURITY:
  • Conversations are ALWAYS filtered by user_id — never return another user's data
```

#### `POST /api/conversations/upload/`

```
PURPOSE: Upload an audio file and start the AI processing pipeline.

REQUEST:
  Headers:
    Authorization: Bearer <access_token>
    Content-Type: multipart/form-data
  Body (form data):
    audio_file: <binary file>             ← Required, max 100 MB
    title: "DBMS Lecture 5"               ← Optional (auto-generated from filename if empty)
    tags: "dbms,lecture,normalisation"     ← Optional, comma-separated

RESPONSE (Success — 202 Accepted):       ← NOTE: 202, not 201
  {
    "id": "665f1b3c4d5e6f7a8b9c0e1f",
    "title": "DBMS Lecture 5",
    "tags": ["dbms", "lecture", "normalisation"],
    "status": "processing",
    "created_at": "2026-06-07T09:00:00Z",
    "message": "Audio uploaded successfully. Processing has begun."
  }

RESPONSE (Validation Error — 400 Bad Request):
  {
    "errors": {
      "audio_file": ["File type 'application/pdf' is not supported. Allowed: audio/mpeg, audio/wav, audio/mp4, audio/webm."]
    }
  }

RESPONSE (File Too Large — 413 Request Entity Too Large):
  {
    "error": "File size exceeds the 100 MB limit."
  }

WHAT HAPPENS INSIDE (see §6 for detail):
  1. Validate file type (MIME type check)
  2. Validate file size (reject > 100 MB)
  3. Generate safe filename (never use the user's original filename for storage)
  4. Save file to media/audio/{user_id}/{uuid}_{safe_name}.mp3
  5. Create conversation document in MongoDB (status: "processing")
  6. Dispatch Celery task: process_conversation.delay(conversation_id)
  7. Return 202 Accepted immediately

WHY 202 AND NOT 201?
  201 means "resource is fully created and ready."
  202 means "request accepted, processing in background."
  The conversation exists but isn't complete yet — 202 is semantically correct.
```

#### `GET /api/conversations/{id}/`

```
PURPOSE: Get full conversation details — metadata + transcript + summary.

REQUEST:
  Headers: Authorization: Bearer <access_token>

RESPONSE (Success — 200 OK):
  {
    "id": "665f1b3c4d5e6f7a8b9c0e1f",
    "title": "DBMS Lecture 5 - Normalisation",
    "tags": ["dbms", "lecture", "normalisation"],
    "status": "ready",
    "audio_file_name": "dbms_lecture_5.mp3",
    "audio_duration_seconds": 1847,
    "created_at": "2026-06-07T09:00:00Z",
    "updated_at": "2026-06-07T09:04:23Z",
    "transcript": {                           ← null if still processing
      "full_text": "Today we will discuss...",
      "segments": [
        {"start": 0.0, "end": 3.8, "text": "Today we will discuss normalisation."},
        {"start": 3.8, "end": 9.2, "text": "Normalisation is the process of..."}
      ],
      "word_count": 4523,
      "language": "en"
    },
    "summary": {                              ← null if still processing
      "summary_text": "This lecture covered...",
      "key_points": [
        "1NF eliminates repeating groups",
        "2NF removes partial dependencies",
        "3NF removes transitive dependencies"
      ],
      "action_items": [
        {"text": "Complete normalisation exercise", "deadline": "2026-06-10", "completed": false, "index": 0},
        {"text": "Practice for exam", "deadline": null, "completed": false, "index": 1}
      ]
    },
    "error_message": null                     ← Populated only if status = "failed"
  }

RESPONSE (Not Found or Not Owner — 404 Not Found):
  {
    "error": "Conversation not found."
  }

SECURITY:
  • Query MUST include user_id filter
  • Return 404 (not 403) if the conversation exists but belongs to another user
    → 403 reveals that the resource EXISTS, which is an information leak
```

#### `DELETE /api/conversations/{id}/`

```
PURPOSE: Delete a conversation and ALL associated data.

REQUEST:
  Headers: Authorization: Bearer <access_token>

RESPONSE (Success — 204 No Content):
  (empty body)

WHAT HAPPENS INSIDE (cascade delete):
  1. Verify conversation exists AND belongs to current user
  2. Delete all embeddings for this conversation (MongoDB + FAISS)
  3. Delete summary document
  4. Delete transcript document
  5. Delete audio file from filesystem
  6. Delete conversation document
  7. Return 204

ORDERING: Children first, parent last (see database-design.md §9)
```

#### `PATCH /api/conversations/{id}/`

```
PURPOSE: Update conversation metadata (title, tags).

REQUEST:
  Headers: Authorization: Bearer <access_token>
  Body:
  {
    "title": "DBMS Lecture 5 - Normalisation (Updated)",
    "tags": ["dbms", "lecture", "normalisation", "3nf"]
  }

RESPONSE (Success — 200 OK):
  {
    "id": "665f1b3c4d5e6f7a8b9c0e1f",
    "title": "DBMS Lecture 5 - Normalisation (Updated)",
    "tags": ["dbms", "lecture", "normalisation", "3nf"],
    "updated_at": "2026-06-07T12:00:00Z"
  }

NOTE: PATCH means partial update — only the fields included in the body are changed.
```

#### `POST /api/conversations/{id}/retry/`

```
PURPOSE: Retry processing a failed conversation.

REQUEST:
  Headers: Authorization: Bearer <access_token>

RESPONSE (Success — 202 Accepted):
  {
    "id": "665f1b3c4d5e6f7a8b9c0e1f",
    "status": "processing",
    "message": "Processing has been restarted."
  }

RESPONSE (Not Failed — 400 Bad Request):
  {
    "error": "Only conversations with status 'failed' can be retried."
  }

WHAT HAPPENS INSIDE:
  1. Verify conversation exists and belongs to user
  2. Verify status is "failed"
  3. Clean up any partial data (incomplete transcript/summary)
  4. Reset status to "processing", clear error_message
  5. Dispatch Celery task again
  6. Return 202
```

#### `PATCH /api/conversations/{id}/action-items/{index}/`

```
PURPOSE: Toggle an action item's completed status.

REQUEST:
  Headers: Authorization: Bearer <access_token>
  Body:
  {
    "completed": true
  }

RESPONSE (Success — 200 OK):
  {
    "index": 0,
    "text": "Complete normalisation exercise",
    "deadline": "2026-06-10",
    "completed": true
  }

WHAT HAPPENS INSIDE:
  1. Find summary by conversation_id
  2. Verify ownership (conversation.user_id == current user)
  3. Update action_items[index].completed using MongoDB positional update
  4. Return updated action item
```

---

### 4.3 Search Endpoints

#### `POST /api/search/`

```
PURPOSE: Search across all conversations using natural language.

REQUEST:
  Headers: Authorization: Bearer <access_token>
  Body:
  {
    "query": "What was said about normalisation?",
    "top_k": 5                                      ← Optional, default 5, max 20
  }

RESPONSE (Success — 200 OK):
  {
    "query": "What was said about normalisation?",
    "results": [
      {
        "conversation_id": "665f1b3c4d5e6f7a8b9c0e1f",
        "conversation_title": "DBMS Lecture 5 - Normalisation",
        "chunk_text": "Normalisation is the process of organising data in a database to reduce redundancy...",
        "relevance_score": 0.92,
        "start_time": 3.8,
        "end_time": 45.6
      },
      {
        "conversation_id": "665f2d4e6f7a8b9c0f1e3d4e",
        "conversation_title": "DBMS Lecture 3 - ER Diagrams",
        "chunk_text": "...before we normalise, we need to understand the entity relationships...",
        "relevance_score": 0.71,
        "start_time": 750.2,
        "end_time": 795.0
      }
    ],
    "total_results": 2
  }

WHAT HAPPENS INSIDE:
  1. Validate query (min 3 chars, max 500 chars)
  2. Generate embedding vector for query text (ai/embeddings.py)
  3. Search FAISS index for top_k similar vectors
  4. Filter results to only current user's embeddings (user_id check)
  5. Fetch chunk_text and conversation metadata from MongoDB
  6. Sort by relevance_score descending
  7. Return enriched results

SECURITY:
  • FAISS returns vectors from ALL users. You MUST filter by user_id
    after the FAISS search. Never return another user's conversation data.
```

---

## 5. Authentication Strategy

### The JWT Lifecycle

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        JWT TOKEN LIFECYCLE                               │
│                                                                          │
│   LOGIN                                                                  │
│   ─────                                                                  │
│   User sends email + password                                            │
│       ↓                                                                  │
│   Server validates credentials                                           │
│       ↓                                                                  │
│   Server creates TWO tokens:                                             │
│                                                                          │
│   ┌─────────────────────────────┐   ┌─────────────────────────────────┐  │
│   │      ACCESS TOKEN           │   │        REFRESH TOKEN            │  │
│   │                             │   │                                 │  │
│   │  Payload:                   │   │  Payload:                       │  │
│   │    user_id: "abc123"        │   │    user_id: "abc123"            │  │
│   │    type: "access"           │   │    type: "refresh"              │  │
│   │    exp: now + 30 min        │   │    exp: now + 7 days            │  │
│   │                             │   │                                 │  │
│   │  Used for: every API call   │   │  Used for: getting a new        │  │
│   │  Sent in: Authorization     │   │  access token when the old      │  │
│   │  header on every request    │   │  one expires                    │  │
│   │                             │   │                                 │  │
│   │  Lifetime: SHORT (30 min)   │   │  Lifetime: LONG (7 days)       │  │
│   └─────────────────────────────┘   └─────────────────────────────────┘  │
│                                                                          │
│                                                                          │
│   USING THE ACCESS TOKEN                                                 │
│   ──────────────────────                                                 │
│   React app stores both tokens in memory (JavaScript variable)           │
│   Every API request includes:                                            │
│     Authorization: Bearer eyJhbGciOi...                                  │
│                                                                          │
│   Django middleware:                                                      │
│     1. Extracts token from Authorization header                          │
│     2. Decodes and verifies JWT signature                                │
│     3. Checks expiration                                                 │
│     4. Loads user from MongoDB using user_id from token payload          │
│     5. Sets request.user = <User document>                               │
│                                                                          │
│                                                                          │
│   TOKEN REFRESH FLOW                                                     │
│   ──────────────────                                                     │
│   When access token expires (30 min):                                    │
│     1. API returns 401                                                   │
│     2. React interceptor catches the 401                                 │
│     3. React sends refresh token to /api/auth/token/refresh/             │
│     4. Server validates refresh token, issues new access token           │
│     5. React retries the original request with new access token          │
│     6. User never notices anything happened                              │
│                                                                          │
│   When refresh token expires (7 days):                                   │
│     → User must log in again                                             │
│                                                                          │
│                                                                          │
│   TOKEN STORAGE — THE CRITICAL DECISION                                  │
│   ──────────────────────────────────────                                  │
│                                                                          │
│   ❌ localStorage  → Vulnerable to XSS. Any script on the page          │
│                      can read localStorage and steal the token.          │
│                                                                          │
│   ❌ Cookies       → Vulnerable to CSRF unless configured carefully.     │
│                      Adds complexity with SameSite, HttpOnly, Secure.    │
│                                                                          │
│   ✅ In-memory     → Stored in a JavaScript variable (React state/      │
│      variable        context). Lost on page refresh — which means        │
│                      the user must re-login on refresh. This is          │
│                      the MOST SECURE option for an SPA.                  │
│                                                                          │
│   TRADE-OFF: Security vs. convenience. For a university project,        │
│   in-memory is the right choice. In production apps, you'd use          │
│   HttpOnly cookies with CSRF protection for a better UX.                │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### How the Custom Auth Backend Works in DRF

```
REQUEST FLOW THROUGH DJANGO:
─────────────────────────────

  HTTP Request arrives
       │
       ▼
  Django Middleware Stack
       │
       ▼
  DRF Authentication Backend (your custom class in accounts/authentication.py)
       │
       ├── 1. Look for "Authorization: Bearer <token>" header
       │       → If missing: request.user = None (anonymous)
       │
       ├── 2. Decode JWT token
       │       → If invalid signature: raise 401
       │       → If expired: raise 401
       │
       ├── 3. Extract user_id from token payload
       │
       ├── 4. Query MongoDB: users.findOne({ _id: user_id, is_active: true })
       │       → If not found or inactive: raise 401
       │
       └── 5. Set request.user = User document
               → Now every view can access request.user

  DRF View (your view function)
       │
       ├── permission_classes = [IsAuthenticated]
       │       → If request.user is None: return 401 automatically
       │
       └── Your code: user_id = request.user.id
                       → Safe to use — guaranteed to be authenticated
```

---

## 6. File Upload Strategy

### The Upload Process — Step by Step

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        FILE UPLOAD PIPELINE                             │
│                                                                         │
│  Step 1: CLIENT-SIDE VALIDATION (React)                                │
│  ───────────────────────────────────────                                │
│  Before sending the request:                                            │
│    ✓ Check file extension (.mp3, .wav, .m4a, .webm)                    │
│    ✓ Check file size (< 100 MB)                                        │
│    ✓ Show immediate error to user if invalid                           │
│                                                                         │
│  Purpose: Quick feedback. Don't waste bandwidth uploading a 200 MB     │
│  file only to have the server reject it.                               │
│                                                                         │
│  NOTE: Client-side validation is for UX only. NEVER trust it for       │
│  security. Attackers bypass the browser entirely.                      │
│                                                                         │
│                                                                         │
│  Step 2: SERVER-SIDE VALIDATION (Django)                                │
│  ───────────────────────────────────────                                │
│  After receiving the file:                                              │
│    ✓ Check MIME type via file header (not just extension)               │
│      → Read the first few bytes of the file ("magic bytes")            │
│      → Verify it matches a known audio format                          │
│      → A file named "lecture.mp3" could actually be a PHP script       │
│    ✓ Check file size server-side (don't trust Content-Length header)    │
│    ✓ Validate title and tags                                           │
│                                                                         │
│                                                                         │
│  Step 3: SAFE FILE NAMING                                               │
│  ────────────────────────                                               │
│  NEVER use the user's original filename for storage.                   │
│                                                                         │
│    User uploads: "../../etc/passwd.mp3"                                 │
│    If you store as-is → PATH TRAVERSAL ATTACK                          │
│                                                                         │
│    Instead, generate a safe path:                                       │
│      media/audio/{user_id}/{uuid4}.mp3                                  │
│                                                                         │
│    Example:                                                             │
│      media/audio/665f1a2b/a3b4c5d6-e7f8-9012-3456-7890abcdef12.mp3    │
│                                                                         │
│    Store the ORIGINAL filename in MongoDB (audio_file_name field)      │
│    for display purposes only.                                           │
│                                                                         │
│                                                                         │
│  Step 4: FILE STORAGE                                                   │
│  ────────────────────                                                   │
│    Write file to disk in the generated safe path.                      │
│    Organise by user_id subdirectory for:                                │
│      → Easy per-user disk usage tracking                               │
│      → Simplified cleanup when a user is deleted                       │
│      → Avoid hitting filesystem limits (too many files in one dir)     │
│                                                                         │
│                                                                         │
│  Step 5: DATABASE RECORD                                                │
│  ───────────────────────                                                │
│    Create conversation document in MongoDB with:                       │
│      → status: "processing"                                            │
│      → audio_file_path: relative path to stored file                   │
│      → audio_file_name: original user filename                         │
│      → audio_file_size_bytes: actual file size                         │
│                                                                         │
│                                                                         │
│  Step 6: DISPATCH CELERY TASK                                           │
│  ────────────────────────────                                           │
│    process_conversation.delay(conversation_id=str(conversation.id))     │
│    → This sends a message to Redis                                     │
│    → A Celery worker picks it up asynchronously                        │
│    → Django does NOT wait for this to complete                         │
│                                                                         │
│                                                                         │
│  Step 7: RETURN 202 ACCEPTED                                            │
│  ───────────────────────────                                            │
│    Return conversation_id and status: "processing"                     │
│    Total time: < 2 seconds (file write + MongoDB insert)               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Allowed MIME Types

| MIME Type | Extension | Notes |
|---|---|---|
| `audio/mpeg` | .mp3 | Most common format |
| `audio/wav` | .wav | Uncompressed, large files |
| `audio/x-wav` | .wav | Alternative MIME type |
| `audio/mp4` | .m4a | Apple's audio format |
| `audio/x-m4a` | .m4a | Alternative |
| `audio/webm` | .webm | Browser recording format |

---

## 7. Background Processing Strategy (Celery)

### How Celery Works — Mental Model

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     CELERY MENTAL MODEL                                 │
│                                                                         │
│   Think of Celery as a RESTAURANT KITCHEN:                             │
│                                                                         │
│   Django (Waiter)         Redis (Order Queue)      Celery Worker (Chef) │
│   ──────────────          ─────────────────        ────────────────────  │
│   Takes the order         Holds the ticket         Cooks the meal       │
│   from the customer       in the queue             when ready           │
│   (HTTP request)          until a chef              (runs AI models)    │
│                           is available                                  │
│   Returns immediately     FIFO: first in,          Reports back when   │
│   with "order placed"     first out                done (updates DB)   │
│                                                                         │
│   The waiter does NOT     The queue ensures        Multiple chefs can  │
│   cook the meal.          no order is lost         work simultaneously │
│   The waiter does NOT     even if all chefs                            │
│   wait for the meal.      are busy                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Task Chain Design

The AI pipeline is a **chain** — each task triggers the next:

```
process_conversation(conversation_id)
│
├── 1. transcribe_audio(conversation_id)
│       ├── Load audio file from disk
│       ├── Call ai/transcription.py → transcribe()
│       ├── Save Transcript document to MongoDB
│       ├── Update conversation.audio_duration_seconds
│       └── On success → chain to step 2
│
├── 2. summarise_conversation(conversation_id)
│       ├── Load transcript.full_text from MongoDB
│       ├── Call ai/summarisation.py → summarise()
│       ├── Save Summary document to MongoDB
│       └── On success → chain to step 3
│
└── 3. generate_embeddings(conversation_id)
        ├── Load transcript.full_text from MongoDB
        ├── Split text into ~200-word chunks with 50-word overlap
        ├── Call ai/embeddings.py → generate_embeddings()
        ├── Save Embedding documents to MongoDB
        ├── Add vectors to FAISS index
        ├── Update conversation.status → "ready"
        └── DONE
```

### Error Handling in the Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TASK ERROR HANDLING                                   │
│                                                                         │
│   SCENARIO 1: Whisper fails (corrupted audio)                          │
│   ─────────────────────────────────────────────                         │
│   → Task raises an exception                                           │
│   → Celery retries automatically (max 2 retries with exponential       │
│     backoff: wait 30s, then 120s)                                      │
│   → If all retries fail:                                               │
│       → Update conversation.status = "failed"                          │
│       → Set conversation.error_message = "Transcription failed: ..."   │
│       → DO NOT cascade to summarisation or embedding                   │
│                                                                         │
│   SCENARIO 2: Gemini API rate limited                                  │
│   ──────────────────────────────────                                    │
│   → API returns 429 Too Many Requests                                  │
│   → Task raises a retryable exception                                  │
│   → Celery retries with exponential backoff                            │
│   → Usually succeeds on retry                                          │
│                                                                         │
│   SCENARIO 3: Embedding generation fails                               │
│   ──────────────────────────────────────                                │
│   → Transcript and summary are already saved (they don't get lost)     │
│   → Only the embeddings step failed                                    │
│   → Set status = "failed" with specific error                          │
│   → On retry, the task checks what already exists and skips            │
│     completed steps (idempotency)                                      │
│                                                                         │
│   KEY PRINCIPLE — IDEMPOTENCY:                                         │
│   ────────────────────────────                                          │
│   Every task must be safe to run MULTIPLE TIMES with the same input.   │
│   If a transcript already exists for this conversation, skip           │
│   transcription. If a summary exists, skip summarisation.              │
│   This prevents duplicate data on retries.                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Celery Configuration Decisions

| Setting | Value | Why |
|---|---|---|
| Broker | Redis | Lightweight, fast, easy to install. Perfect for a single-server setup. |
| Result backend | Not needed | We store results directly in MongoDB, not in Celery's result backend. Tasks update MongoDB themselves. |
| Concurrency | 2 workers | Whisper is CPU-heavy. 2 workers on a dev machine prevents system overload. |
| Max retries | 2 | 3 total attempts (1 original + 2 retries). More retries delay error reporting. |
| Retry backoff | Exponential (30s, 120s) | Gives transient issues (API rate limits, network blips) time to resolve. |
| Task time limit | 600 seconds (10 min) | Kill the task if it hangs. A 2-hour audio file on the `base` Whisper model should finish in ~5 min. |
| Task serializer | JSON | Default. Don't use pickle — it's a security risk (arbitrary code execution via crafted pickle payload). |

---

## 8. AI Integration Strategy

### The Adapter Pattern — Designing for Swappability

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      AI ADAPTER PATTERN                                 │
│                                                                         │
│   The ai/ app wraps each AI service behind a CONSISTENT INTERFACE.     │
│   The rest of the codebase never imports whisper, google.generativeai, │
│   or sentence_transformers directly. They only call ai/ functions.      │
│                                                                         │
│   This means you can swap AI providers by editing ONE file:            │
│                                                                         │
│   ┌──────────────────────────────────┐                                 │
│   │  conversations/tasks.py          │                                 │
│   │                                  │                                 │
│   │  # These imports NEVER change:   │                                 │
│   │  from apps.ai.transcription      │                                 │
│   │       import transcribe          │──────► ai/transcription.py      │
│   │  from apps.ai.summarisation      │        (change internals here   │
│   │       import summarise           │──────► ai/summarisation.py       │
│   │  from apps.ai.embeddings         │        without touching          │
│   │       import generate_embeddings │──────► ai/embeddings.py          │
│   │                                  │        tasks.py)                 │
│   └──────────────────────────────────┘                                 │
│                                                                         │
│   SWAP EXAMPLES:                                                        │
│   ──────────────                                                        │
│   Gemini → OpenAI:  Edit ai/summarisation.py only                      │
│   Whisper → Google STT: Edit ai/transcription.py only                  │
│   MiniLM → Ada-002: Edit ai/embeddings.py only                        │
│                                                                         │
│   EVERYTHING ELSE stays the same — views, tasks, serializers, etc.     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Transcription Integration (Whisper)

```
ai/transcription.py — Interface Design:
────────────────────────────────────────

FUNCTION: transcribe(audio_file_path, model_name=None)

INPUT:
  audio_file_path: str    ← Absolute path to the audio file on disk
  model_name: str         ← "tiny", "base", "small", "medium", "large"
                             Default from settings: WHISPER_MODEL_NAME

OUTPUT:
  {
    "full_text": "Today we will discuss...",
    "segments": [
      {"start": 0.0, "end": 3.8, "text": "Today we will discuss normalisation."},
      ...
    ],
    "word_count": 4523,
    "language": "en",
    "duration_seconds": 1847.5
  }

INTERNAL STEPS:
  1. Load Whisper model (cache it — don't reload for every call)
  2. Call model.transcribe(audio_file_path)
  3. Extract segments with timestamps
  4. Count words
  5. Format into standard output structure
  6. Return

ERROR CASES:
  • File not found → raise FileNotFoundError
  • Corrupted audio → Whisper may raise RuntimeError → catch and re-raise
    as a descriptive TranscriptionError
  • Model not available → raise ConfigurationError

PERFORMANCE:
  Model loading takes 5-10 seconds. Load ONCE when the Celery worker
  starts (module-level or lazy singleton). Not on every task execution.
```

### Summarisation Integration (Gemini / OpenAI)

```
ai/summarisation.py — Interface Design:
─────────────────────────────────────────

FUNCTION: summarise(transcript_text, model=None)

INPUT:
  transcript_text: str    ← Full transcript text
  model: str              ← "gemini-1.5-flash", "gpt-4o-mini", etc.
                             Default from settings: LLM_MODEL_NAME

OUTPUT:
  {
    "summary_text": "This lecture covered...",
    "key_points": ["Point 1", "Point 2", ...],
    "action_items": [
      {"text": "Complete exercise", "deadline": "2026-06-10"},
      {"text": "Practice for exam", "deadline": null}
    ],
    "model_used": "gemini-1.5-flash"
  }

INTERNAL STEPS:
  1. Load prompt template from prompts.py
  2. Insert transcript into prompt
  3. Call LLM API with structured output request (JSON mode)
  4. Parse and validate JSON response
  5. Handle malformed responses gracefully (retry once, or return
     partial results with a warning)
  6. Return structured output

HANDLING LONG TRANSCRIPTS:
  LLMs have token limits. A 2-hour transcript may exceed the limit.
  Strategy:
    • If transcript < token limit: send as-is
    • If transcript > token limit: split into sections, summarise
      each section, then summarise the summaries (hierarchical summarisation)
    • Track which approach was used in the output for debugging
```

### Prompt Management

```
ai/prompts.py — Why Separate Prompts from Code:
─────────────────────────────────────────────────

PROBLEM:
  Prompts are the "instructions" you give the LLM. They need to be:
    • Iterated on frequently (small wording changes → big quality changes)
    • Versioned (track which prompt produced which summary)
    • Readable (long prompts buried in Python logic are hard to review)

SOLUTION:
  Store prompts as constants in a dedicated file.

EXAMPLE STRUCTURE:

  SUMMARISATION_PROMPT_V1 = """
  You are an expert at analysing conversations and extracting structured insights.

  Given the following transcript of a conversation, produce a JSON response with:
  1. "summary_text": A concise summary in 3-5 sentences
  2. "key_points": A list of the most important points discussed (3-10 items)
  3. "action_items": A list of tasks or commitments mentioned, each with:
     - "text": description of the action
     - "deadline": deadline if mentioned, null if not

  IMPORTANT RULES:
  - Only include action items that are explicitly stated or clearly implied
  - Do not invent information not present in the transcript
  - Keep the summary factual and objective
  - Respond ONLY with valid JSON, no markdown or extra text

  TRANSCRIPT:
  {transcript_text}
  """

  CURRENT_SUMMARISATION_PROMPT = SUMMARISATION_PROMPT_V1

  When you improve the prompt, create V2 and point CURRENT to it.
  This gives you a history of prompt evolution.
```

### Embedding Integration

```
ai/embeddings.py — Interface Design:
──────────────────────────────────────

FUNCTION: generate_embeddings(text_chunks)

INPUT:
  text_chunks: list[str]    ← List of text chunks (~200 words each)

OUTPUT:
  list[list[float]]         ← List of 384-dimensional vectors

INTERNAL STEPS:
  1. Load sentence-transformer model (cached singleton)
  2. Batch encode all chunks (much faster than one-by-one)
  3. Return list of vectors

FUNCTION: generate_single_embedding(text)

INPUT:
  text: str                 ← Single text (e.g., a search query)

OUTPUT:
  list[float]               ← Single 384-dimensional vector

WHY TWO FUNCTIONS:
  generate_embeddings() is for batch processing during the pipeline.
  generate_single_embedding() is for real-time search query embedding.
  They use the same model but the batch version is optimised for throughput.
```

---

## 9. Database Interaction Patterns

### The Services Layer Pattern

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   VIEW → SERVICE → DOCUMENT PATTERN                    │
│                                                                         │
│   ANTI-PATTERN (fat views):                                            │
│   ─────────────────────────                                             │
│   views.py:                                                             │
│     class UploadView(APIView):                                         │
│         def post(self, request):                                       │
│             # validate file (15 lines)                                 │
│             # save to disk (10 lines)                                  │
│             # create conversation doc (5 lines)                        │
│             # dispatch celery task (3 lines)                           │
│             # format response (5 lines)                                │
│             # 38 lines of mixed concerns!                              │
│                                                                         │
│   PREFERRED PATTERN (thin views + services):                           │
│   ──────────────────────────────────────────                            │
│   views.py:                                                             │
│     class UploadView(APIView):                                         │
│         def post(self, request):                                       │
│             serializer = UploadSerializer(data=request.data)           │
│             serializer.is_valid(raise_exception=True)                  │
│             conversation = ConversationService.upload(                  │
│                 user=request.user,                                      │
│                 audio_file=request.FILES["audio_file"],                │
│                 **serializer.validated_data                             │
│             )                                                          │
│             return Response(                                           │
│                 ConversationSerializer(conversation).data,             │
│                 status=202                                             │
│             )                                                          │
│             # 10 lines! View only does: validate → delegate → respond  │
│                                                                         │
│   services.py:                                                          │
│     class ConversationService:                                         │
│         @staticmethod                                                  │
│         def upload(user, audio_file, title, tags):                     │
│             # validate file type and size                              │
│             # generate safe file path                                  │
│             # save file to disk                                        │
│             # create Conversation document                             │
│             # dispatch Celery task                                     │
│             # return conversation                                      │
│                                                                         │
│   WHY THIS IS BETTER:                                                  │
│     ✅ Views are thin — easy to read and test                          │
│     ✅ Business logic is reusable (services can be called from         │
│        views, management commands, or Celery tasks)                    │
│     ✅ Easy to unit test services without HTTP overhead                 │
│     ✅ Clear separation: views handle HTTP, services handle logic      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Query Security — The Ownership Filter

Every database query serving a user request follows the same pattern:

```
THE UNIVERSAL QUERY PATTERN:
────────────────────────────

ALWAYS:
  conversation = Conversation.objects.get(
      id=conversation_id,
      user_id=request.user.id       ← THIS LINE IS NON-NEGOTIABLE
  )

NEVER:
  conversation = Conversation.objects.get(id=conversation_id)
  # ← Missing user_id filter = ANY user can access ANY conversation

This should be enforced at the SERVICE LAYER, not in every view.
The service always receives the user object and always includes
user_id in every query. Views never query MongoDB directly.
```

### How Each Endpoint Touches the Database

```
┌────────────────────────────────┬──────────────────────────────────────────────┐
│ Endpoint                       │ Database Operations                          │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ POST /auth/register            │ WRITE users                                 │
│ POST /auth/login               │ READ users (find by email)                  │
│                                │ WRITE users (update last_login)             │
│ GET  /auth/me                  │ READ users (by _id from JWT)                │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ GET  /conversations            │ READ conversations (filter user_id, paginate)│
│                                │ READ summaries (batch, for preview + count) │
│ POST /conversations/upload     │ WRITE conversations (create)                │
│ GET  /conversations/{id}       │ READ conversations (by _id + user_id)       │
│                                │ READ transcripts (by conversation_id)       │
│                                │ READ summaries (by conversation_id)         │
│ DELETE /conversations/{id}     │ DELETE embeddings (by conversation_id)      │
│                                │ DELETE summaries (by conversation_id)       │
│                                │ DELETE transcripts (by conversation_id)     │
│                                │ DELETE conversations (by _id + user_id)     │
│ PATCH /conversations/{id}      │ WRITE conversations (update title/tags)     │
│ POST  /conversations/{id}/retry│ WRITE conversations (reset status)          │
│                                │ DELETE partial transcript/summary/embeddings│
│ PATCH /.../action-items/{idx}  │ WRITE summaries (update array element)      │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ POST /search                   │ READ embeddings (FAISS search + MongoDB     │
│                                │      metadata lookup)                       │
│                                │ READ conversations (titles for results)     │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ Celery: transcribe             │ READ conversations (get audio_file_path)    │
│                                │ WRITE transcripts (create)                  │
│                                │ WRITE conversations (update duration)       │
│ Celery: summarise              │ READ transcripts (get full_text)            │
│                                │ WRITE summaries (create)                    │
│ Celery: embed                  │ READ transcripts (get full_text)            │
│                                │ WRITE embeddings (create many)              │
│                                │ WRITE conversations (status → "ready")      │
└────────────────────────────────┴──────────────────────────────────────────────┘
```

---

## 10. Error Handling Strategy

### Centralised Error Responses

Every error from Kairo's API should follow a **consistent format**:

```
SUCCESS RESPONSES:
  → Return the data directly (no wrapper)
  → HTTP status codes: 200, 201, 202, 204

ERROR RESPONSES:
  → Always return JSON with "error" or "errors" key
  → HTTP status codes: 400, 401, 403, 404, 413, 429, 500

SINGLE ERROR:
  {
    "error": "Conversation not found."
  }

VALIDATION ERRORS (multiple fields):
  {
    "errors": {
      "email": ["This field is required."],
      "password": ["Password must be at least 8 characters.", "Password must contain a number."]
    }
  }

SERVER ERROR:
  {
    "error": "An unexpected error occurred. Please try again later."
  }

  NOTE: NEVER expose stack traces or internal error details in production.
  Log them server-side; show a generic message to the user.
```

### Error Codes and When to Use Them

| HTTP Status | When to Use | Example |
|---|---|---|
| **400** Bad Request | Input validation failed | Missing required field, invalid file type |
| **401** Unauthorized | Authentication required or failed | Missing/invalid/expired JWT token |
| **403** Forbidden | Authenticated but not permitted | Trying to access another user's resource (though we return 404 instead — see note) |
| **404** Not Found | Resource doesn't exist OR user doesn't own it | Conversation ID doesn't exist or belongs to another user |
| **413** Payload Too Large | File exceeds size limit | Audio file > 100 MB |
| **429** Too Many Requests | Rate limit exceeded | Too many login attempts |
| **500** Internal Server Error | Unexpected server failure | Database connection lost, uncaught exception |

**Why 404 instead of 403 for "wrong owner"?**
If you return 403, the attacker knows the resource EXISTS but they can't access it. They can enumerate valid IDs. Returning 404 reveals nothing — the resource either doesn't exist or the user can't see it. Either way, the response is the same.

---

## 11. Settings & Configuration Strategy

### Environment Variables

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    .env FILE (NEVER COMMIT TO GIT)                      │
│                                                                         │
│  # Django                                                               │
│  DJANGO_SECRET_KEY=your-random-secret-key-here                         │
│  DJANGO_DEBUG=True                                                      │
│  DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1                              │
│                                                                         │
│  # MongoDB                                                              │
│  MONGODB_URI=mongodb://kairo_user:password@localhost:27017/kairo_db     │
│  MONGODB_DB_NAME=kairo_db                                               │
│                                                                         │
│  # Redis (Celery broker)                                                │
│  REDIS_URL=redis://localhost:6379/0                                     │
│                                                                         │
│  # AI Services                                                          │
│  WHISPER_MODEL_NAME=base                                                │
│  GEMINI_API_KEY=your-gemini-api-key-here                               │
│  LLM_MODEL_NAME=gemini-1.5-flash                                      │
│  EMBEDDING_MODEL_NAME=all-MiniLM-L6-v2                                 │
│                                                                         │
│  # JWT                                                                  │
│  JWT_SECRET_KEY=your-jwt-secret-here                                   │
│  JWT_ACCESS_TOKEN_LIFETIME_MINUTES=30                                  │
│  JWT_REFRESH_TOKEN_LIFETIME_DAYS=7                                     │
│                                                                         │
│  # File Upload                                                          │
│  MAX_UPLOAD_SIZE_MB=100                                                 │
│  MEDIA_ROOT=./media                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Why Split Settings?

```
config/settings/base.py        ← Shared: installed apps, middleware, REST framework config
config/settings/development.py ← DEBUG=True, verbose logging, relaxed CORS
config/settings/production.py  ← DEBUG=False, strict CORS, HTTPS enforcement

Usage:
  Development: DJANGO_SETTINGS_MODULE=config.settings.development
  Production:  DJANGO_SETTINGS_MODULE=config.settings.production
```

---

## 12. Build Order — What to Implement First

Each step builds on the previous one and produces a **testable, demonstrable result**.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     IMPLEMENTATION PHASES                                │
│                                                                         │
│  PHASE 1: Foundation                                    [Week 1-2]     │
│  ─────────────────────                                                  │
│  □ Django project setup (config/, settings split, .env)                │
│  □ MongoDB connection (MongoEngine config in settings)                 │
│  □ accounts app (User document, register, login, JWT)                  │
│  □ Custom DRF auth backend (authentication.py)                         │
│  □ Test: Register → Login → Call /api/auth/me → Get user data         │
│                                                                         │
│  MILESTONE: You can register, login, and authenticate API requests.   │
│                                                                         │
│                                                                         │
│  PHASE 2: Upload & Storage                              [Week 2-3]     │
│  ─────────────────────────                                              │
│  □ Conversation document class                                         │
│  □ File upload view with validation                                    │
│  □ Safe filename generation                                            │
│  □ Conversation list endpoint (dashboard data)                         │
│  □ Conversation detail endpoint                                        │
│  □ Test: Upload audio → See it in list → View detail (no AI yet)      │
│                                                                         │
│  MILESTONE: Users can upload audio and see their conversations.       │
│  Status stays "processing" forever (no workers yet).                   │
│                                                                         │
│                                                                         │
│  PHASE 3: AI Pipeline                                   [Week 3-5]     │
│  ────────────────────                                                   │
│  □ Celery + Redis setup (config/celery.py, worker command)             │
│  □ ai/transcription.py (Whisper wrapper)                               │
│  □ ai/summarisation.py (Gemini wrapper)                                │
│  □ ai/embeddings.py (sentence-transformers wrapper)                    │
│  □ ai/prompts.py (summarisation prompt)                                │
│  □ Transcript + Summary document classes                               │
│  □ conversations/tasks.py (pipeline chain)                              │
│  □ Test: Upload audio → Worker processes it → Transcript + Summary    │
│          appear in conversation detail                                  │
│                                                                         │
│  MILESTONE: The full pipeline works end-to-end. This is the core      │
│  value of Kairo — upload audio, get structured insights.              │
│                                                                         │
│                                                                         │
│  PHASE 4: Search                                        [Week 5-6]     │
│  ───────────────                                                        │
│  □ Embedding document class                                            │
│  □ search/services.py (FAISS index management)                         │
│  □ Embedding generation added to pipeline (Stage 3)                    │
│  □ Search endpoint                                                     │
│  □ Test: Upload 3 conversations → Search "normalisation" →            │
│          Get relevant results                                          │
│                                                                         │
│  MILESTONE: Semantic search works across conversations.               │
│                                                                         │
│                                                                         │
│  PHASE 5: Polish                                        [Week 6-7]     │
│  ───────────────                                                        │
│  □ Delete conversation (cascade)                                       │
│  □ Update conversation metadata (PATCH)                                │
│  □ Retry failed processing                                             │
│  □ Action item toggle                                                  │
│  □ Error handling (centralised exception handler)                      │
│  □ Pagination for conversation list                                    │
│  □ Test: Full CRUD lifecycle + error scenarios                         │
│                                                                         │
│  MILESTONE: Backend is feature-complete for MVP.                       │
│                                                                         │
│                                                                         │
│  PHASE 6: React Integration                             [Week 7-9]     │
│  ──────────────────────────                                             │
│  □ CORS configuration for React dev server                             │
│  □ React app setup                                                     │
│  □ Connect React to all API endpoints                                  │
│  □ End-to-end testing through the browser                              │
│                                                                         │
│  MILESTONE: Full application works through the browser.               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

*This document should be read alongside [architecture.md](file:///Users/mohammed/kairo/docs/architecture.md) (system-level design) and [database-design.md](file:///Users/mohammed/kairo/docs/database-design.md) (MongoDB schema details).*

*Last updated: 2026-06-07*
*Author: Mohammed Kutubudin Kapasi*
