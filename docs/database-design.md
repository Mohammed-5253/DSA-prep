# Kairo — Database Design

> MongoDB schema design for **Kairo: AI Conversational Memory Assistant**
> Companion to [architecture.md](file:///Users/mohammed/kairo/docs/architecture.md) and [project-context.md](file:///Users/mohammed/kairo/docs/project-context.md)

**Last updated:** 2026-06-05
**Author:** Mohammed Kutubudin Kapasi

---

## Table of Contents

1. [Why MongoDB?](#1-why-mongodb)
2. [How MongoDB Thinks — Mental Model](#2-how-mongodb-thinks--mental-model)
3. [The Fundamental Design Decision: Embed vs. Reference](#3-the-fundamental-design-decision-embed-vs-reference)
4. [Kairo's Collection Map](#4-kairos-collection-map)
5. [Collection Designs — Deep Dive](#5-collection-designs--deep-dive)
6. [Relationships Between Collections](#6-relationships-between-collections)
7. [Indexing Strategy](#7-indexing-strategy)
8. [Query Patterns — How Data Flows Through Kairo](#8-query-patterns--how-data-flows-through-kairo)
9. [Data Lifecycle — From Upload to Deletion](#9-data-lifecycle--from-upload-to-deletion)
10. [Django + MongoDB Integration](#10-django--mongodb-integration)
11. [Migration & Evolution Strategy](#11-migration--evolution-strategy)
12. [Common Mistakes to Avoid](#12-common-mistakes-to-avoid)
13. [Security at the Database Layer](#13-security-at-the-database-layer)
14. [Scalability Considerations](#14-scalability-considerations)

---

## 1. Why MongoDB?

Before touching any collection design, understand **why** MongoDB is the right fit for Kairo and where it would be the **wrong** choice.

### What MongoDB Is

MongoDB is a **document database**. Instead of rows in tables (like MySQL/PostgreSQL), it stores **documents** — JSON-like objects (technically BSON) that can have nested structures, arrays, and variable shapes.

### Why It Fits Kairo

| Kairo's Data Characteristic | Why MongoDB Handles It Well |
|---|---|
| **Transcripts are variable-length** | A 5-minute conversation has 50 segments; a 2-hour lecture has 2,000. A fixed-column SQL table would need a separate `segments` table with JOINs. MongoDB stores the entire array inside one document. |
| **Summaries have nested structures** | Key points = array of strings. Action items = array of objects with `text`, `deadline`, `completed`. MongoDB handles nested arrays of objects natively. |
| **Embeddings are high-dimensional arrays** | Each embedding is an array of 384 floats. MongoDB stores arrays efficiently. SQL would require a separate table or awkward serialisation. |
| **Schema evolves rapidly** | During development, you'll add fields frequently (e.g., `language`, `speaker_count`, `sentiment_score`). MongoDB doesn't require migrations for new fields — existing documents simply don't have the field until you add it. |
| **Read pattern is document-centric** | "Give me everything about conversation X" is one query in MongoDB. In SQL, it would be 3-4 JOINs across tables. |

### Where MongoDB Would Be the Wrong Choice

| Scenario | Why SQL Would Be Better |
|---|---|
| Complex joins across many tables | MongoDB doesn't support JOINs natively (it has `$lookup`, but it's slower and more limited) |
| Strict data integrity (banking, accounting) | MongoDB has no foreign key constraints — it won't prevent you from storing a `conversation_id` that doesn't exist |
| Heavy aggregation / analytics | SQL's GROUP BY, window functions, and CTEs are more mature |
| ACID transactions across many collections | MongoDB supports multi-document transactions since v4.0, but they're slower than SQL transactions |

**For Kairo:** The data is document-centric (one conversation = one logical unit), the schema is evolving, and the read patterns are "get everything for this conversation." MongoDB is a natural fit.

---

## 2. How MongoDB Thinks — Mental Model

If you've used SQL before, you need to **unlearn some habits**. Here's the mental shift:

### SQL vs. MongoDB — Concept Mapping

| SQL Concept | MongoDB Equivalent | Key Difference |
|---|---|---|
| Database | Database | Same concept — a named container |
| Table | **Collection** | A collection has no fixed schema — each document can have different fields |
| Row | **Document** | A JSON object, can be nested, can have arrays |
| Column | **Field** | Fields can be strings, numbers, arrays, nested objects — not just scalar values |
| Primary Key | `_id` field | Automatically generated as `ObjectId` if you don't set it |
| Foreign Key | Manual reference (ObjectId stored in a field) | **No enforcement** — MongoDB won't check if the referenced document exists |
| JOIN | `$lookup` (aggregation) or application-level join | Much slower than SQL JOINs — this is why you design to **avoid needing them** |
| Schema / Migration | None required | You add fields by just... writing them. Old documents don't break. |

### The Golden Rule of MongoDB Design

> **Design your schema based on how you query, not how you store.**

In SQL, you normalise data to avoid duplication (3NF). In MongoDB, you often **denormalise** (duplicate data) to avoid joins. The question is always:

**"What does the application need to display in a single screen/request?"**

If a single screen needs data from what would be 3 SQL tables, consider putting it all in one MongoDB document.

---

## 3. The Fundamental Design Decision: Embed vs. Reference

Every relationship in MongoDB comes down to this question. For Kairo, we need to make this decision for every piece of data.

### The Two Approaches

```
APPROACH 1: EMBEDDING (data inside the parent document)
─────────────────────────────────────────────────────────

// conversations collection
{
  _id: ObjectId("abc123"),
  title: "DBMS Lecture",
  transcript: {                    ← Embedded directly
    full_text: "Today we...",
    segments: [...]
  },
  summary: {                       ← Embedded directly
    summary_text: "This lecture...",
    key_points: [...],
    action_items: [...]
  }
}

✅ One query gets everything
✅ No joins needed
✅ Atomic updates (one document = one write)
❌ Document can grow very large (MongoDB limit: 16 MB)
❌ Can't query transcripts independently
❌ Updating just the summary means rewriting the whole document


APPROACH 2: REFERENCING (separate collections, linked by ID)
─────────────────────────────────────────────────────────────

// conversations collection
{
  _id: ObjectId("abc123"),
  title: "DBMS Lecture"
}

// transcripts collection
{
  _id: ObjectId("def456"),
  conversation_id: ObjectId("abc123"),   ← Reference
  full_text: "Today we...",
  segments: [...]
}

// summaries collection
{
  _id: ObjectId("ghi789"),
  conversation_id: ObjectId("abc123"),   ← Reference
  summary_text: "This lecture...",
  key_points: [...],
  action_items: [...]
}

✅ Each document stays small
✅ Can query/update each piece independently
✅ No risk of hitting 16 MB limit
✅ Different parts written at different times (pipeline stages)
❌ Need multiple queries (or $lookup) to get full picture
❌ No referential integrity enforcement
```

### Kairo's Decision: **Referenced Collections**

For Kairo, we choose **referencing** (separate collections). Here's why:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   WHY REFERENCING WINS FOR KAIRO                       │
│                                                                        │
│   1. TRANSCRIPT SIZE                                                   │
│      A 2-hour lecture transcript can be 30,000+ words.                │
│      With timestamps per segment, this could approach 5-10 MB.        │
│      Embedding it inside the conversation document risks              │
│      hitting MongoDB's 16 MB document size limit.                     │
│                                                                        │
│   2. PIPELINE WRITES AT DIFFERENT TIMES                               │
│      The conversation is created at upload time.                      │
│      The transcript is written 2 minutes later (after Whisper).       │
│      The summary is written 15 seconds after that (after LLM).       │
│      The embeddings are written last.                                 │
│      Separate collections let each pipeline stage write its own       │
│      document independently, without locking the conversation doc.    │
│                                                                        │
│   3. DASHBOARD DOESN'T NEED FULL TRANSCRIPT                          │
│      The dashboard shows title, date, status, tags — just the        │
│      conversation document. Loading full transcripts for 50           │
│      conversations would waste bandwidth and memory.                  │
│                                                                        │
│   4. EMBEDDINGS ARE MANY-TO-ONE                                       │
│      One conversation produces 10-100+ embedding chunks.              │
│      Embedding an array of 100 × 384-float vectors inside the        │
│      conversation document would make it enormous.                    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**One exception — small, bounded data gets embedded:**
- `tags` array → embedded inside `conversations` (always small, always read with the conversation)
- `segments` array → embedded inside `transcripts` (always read with the transcript, never queried independently)
- `action_items` array → embedded inside `summaries` (always read with the summary)

**The rule of thumb:**

| Condition | Strategy |
|---|---|
| Data is small and bounded (< 100 items) | **Embed** |
| Data is always read with the parent | **Embed** |
| Data can grow unboundedly | **Reference** (separate collection) |
| Data is written at a different time than the parent | **Reference** |
| Data needs to be queried/searched independently | **Reference** |

---

## 4. Kairo's Collection Map

Here is the complete picture of what lives in MongoDB:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KAIRO DATABASE                              │
│                       (MongoDB: "kairo_db")                         │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                     CORE COLLECTIONS                        │   │
│   │                                                             │   │
│   │   ┌──────────┐  ┌────────────────┐  ┌──────────────────┐   │   │
│   │   │  users   │  │ conversations  │  │  transcripts     │   │   │
│   │   │          │  │                │  │                  │   │   │
│   │   │ identity │  │ the central    │  │ Whisper output   │   │   │
│   │   │ & auth   │  │ entity — owns  │  │ (text +          │   │   │
│   │   │          │  │ everything     │  │  timestamps)     │   │   │
│   │   └────┬─────┘  └───────┬────────┘  └────────┬─────────┘   │   │
│   │        │    1:many      │   1:1              │              │   │
│   │        └────────────────┘   ┌────────────────┘              │   │
│   │                             │   1:1                         │   │
│   │                      ┌──────┴──────────┐                    │   │
│   │                      │   summaries     │                    │   │
│   │                      │                 │                    │   │
│   │                      │ LLM output      │                    │   │
│   │                      │ (summary, key   │                    │   │
│   │                      │  points, action │                    │   │
│   │                      │  items)         │                    │   │
│   │                      └──────┬──────────┘                    │   │
│   │                             │                               │   │
│   │                             │   1:many                      │   │
│   │                      ┌──────┴──────────┐                    │   │
│   │                      │  embeddings     │                    │   │
│   │                      │                 │                    │   │
│   │                      │ vector chunks   │                    │   │
│   │                      │ for semantic    │                    │   │
│   │                      │ search          │                    │   │
│   │                      └─────────────────┘                    │   │
│   │                                                             │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                   SUPPORTING DATA                           │   │
│   │                                                             │   │
│   │   Audio files → NOT in MongoDB (filesystem / S3)            │   │
│   │   FAISS index → NOT in MongoDB (in-memory + .faiss file)    │   │
│   │   Task queue  → NOT in MongoDB (Redis)                      │   │
│   │                                                             │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**5 collections. That's it.** A lean design that maps directly to the AI processing pipeline stages.

---

## 5. Collection Designs — Deep Dive

For each collection, I'll explain:
- Every field and **why** it exists
- Data types
- What gets embedded vs. referenced
- Size estimates
- Example document

---

### 5.1 `users` Collection

**Purpose:** Store user identity, credentials, and profile metadata.

**Who writes to it:** Django auth system (on register/login).

**Who reads from it:** Django auth system (on every authenticated request to validate tokens).

```
Document Structure:
──────────────────────────────────────────────────────────

{
  _id:            ObjectId          ← Auto-generated, serves as user_id everywhere
  email:          String            ← Login identifier, must be unique
  username:       String            ← Display name
  password_hash:  String            ← bcrypt hash, NEVER store plaintext
  is_active:      Boolean           ← Soft-disable account without deleting
  created_at:     ISODate           ← Account creation timestamp (UTC)
  last_login:     ISODate | null    ← Updated on each successful login
}
```

**Example document:**
```
{
  _id: ObjectId("665f1a2b3c4d5e6f7a8b9c0d"),
  email: "mohammed@example.com",
  username: "mohammed",
  password_hash: "$2b$12$LJ3m5...hashed...value",
  is_active: true,
  created_at: ISODate("2026-06-05T10:30:00Z"),
  last_login: ISODate("2026-06-05T15:00:00Z")
}
```

**Design decisions:**

| Decision | Reasoning |
|---|---|
| `email` as login, not `username` | Emails are globally unique. Usernames can collide. Users remember their email more reliably. |
| `password_hash` not `password` | **NEVER** store plaintext passwords. bcrypt includes the salt in the hash string itself. |
| `is_active` instead of deleting | Soft-delete pattern. Deleting a user would orphan all their conversations. Setting `is_active: false` preserves data integrity. |
| No `profile_picture`, `bio`, etc. | Keep the MVP lean. Add profile fields later when needed. |
| Timestamps in UTC | Always store times in UTC. Convert to local time in the frontend. This avoids timezone bugs. |

**Estimated size per document:** ~300–500 bytes

---

### 5.2 `conversations` Collection

**Purpose:** The central entity. Represents one uploaded audio recording and its metadata. Acts as the "parent" that links to transcript, summary, and embeddings.

**Who writes to it:** Django (on upload) and Celery (status update when processing completes).

**Who reads from it:** React dashboard (list view), conversation detail page, search results (to show source conversation).

```
Document Structure:
──────────────────────────────────────────────────────────

{
  _id:                    ObjectId        ← Conversation ID used everywhere
  user_id:                ObjectId        ← Reference → users._id (OWNER)
  title:                  String          ← User-provided or auto-generated
  tags:                   [String]        ← User-defined labels, embedded array
  status:                 String          ← "uploading" | "processing" | "ready" | "failed"
  error_message:          String | null   ← Error details if status = "failed"
  audio_file_path:        String          ← Relative path to stored audio file
  audio_file_name:        String          ← Original filename from upload
  audio_file_size_bytes:  Number          ← File size for display and validation
  audio_duration_seconds: Number | null   ← Duration (set after Whisper processes it)
  audio_mime_type:        String          ← "audio/mpeg", "audio/wav", etc.
  created_at:             ISODate         ← Upload timestamp
  updated_at:             ISODate         ← Last status change
}
```

**Example document:**
```
{
  _id: ObjectId("665f1b3c4d5e6f7a8b9c0e1f"),
  user_id: ObjectId("665f1a2b3c4d5e6f7a8b9c0d"),
  title: "DBMS Lecture 5 - Normalisation",
  tags: ["dbms", "lecture", "normalisation", "3nf"],
  status: "ready",
  error_message: null,
  audio_file_path: "audio/665f1a2b/665f1b3c_normalisation.mp3",
  audio_file_name: "dbms_lecture_5.mp3",
  audio_file_size_bytes: 15728640,
  audio_duration_seconds: 1847,
  audio_mime_type: "audio/mpeg",
  created_at: ISODate("2026-06-05T09:00:00Z"),
  updated_at: ISODate("2026-06-05T09:04:23Z")
}
```

**Design decisions:**

| Decision | Reasoning |
|---|---|
| `tags` is **embedded** (not a separate collection) | Tags are always read with the conversation, always small (typically < 10), and don't need independent queries. Embedding avoids a join. |
| `status` as a string enum | Tracks the conversation through the pipeline. The frontend polls this field to show progress. Possible values form a state machine (see §9). |
| `error_message` only when `status = "failed"` | Sparse field. Only populated on errors. MongoDB handles sparse/missing fields naturally. |
| `audio_file_path` not the actual audio bytes | Audio files are large (10–200 MB). Storing binary in MongoDB degrades performance and hits the 16 MB document limit. Store the file on disk; store the **path** in MongoDB. |
| `audio_duration_seconds` is nullable | We don't know the duration at upload time. It's set later when Whisper parses the audio. This is why MongoDB's flexible schema is useful — no ALTER TABLE needed. |
| `user_id` is the **security boundary** | Every query against this collection MUST filter by `user_id`. This prevents User A from seeing User B's conversations (IDOR vulnerability). |

**Estimated size per document:** ~500–800 bytes

---

### 5.3 `transcripts` Collection

**Purpose:** Store the full text output from Whisper, including timestamped segments for navigation.

**Who writes to it:** Celery worker (Stage 1 of AI pipeline — after Whisper finishes).

**Who reads from it:** Conversation detail page (full transcript view), Celery worker (reads it to send to LLM for summarisation and to generate embeddings).

```
Document Structure:
──────────────────────────────────────────────────────────

{
  _id:              ObjectId            ← Transcript document ID
  conversation_id:  ObjectId            ← Reference → conversations._id
  full_text:        String              ← Complete transcript as one string
  segments:         [                   ← Timestamped segments (embedded array)
    {
      start:  Number (float),           ← Start time in seconds (e.g., 0.0)
      end:    Number (float),           ← End time in seconds (e.g., 4.2)
      text:   String                    ← Text spoken in this segment
    }
  ],
  word_count:       Number              ← Total words (useful for UI display)
  language:         String              ← Detected language code ("en")
  whisper_model:    String              ← Which Whisper model was used ("base", "small")
  created_at:       ISODate             ← When transcription completed
}
```

**Example document (abbreviated):**
```
{
  _id: ObjectId("665f1c4d5e6f7a8b9c0e2f3a"),
  conversation_id: ObjectId("665f1b3c4d5e6f7a8b9c0e1f"),
  full_text: "Today we will discuss normalisation. Normalisation is the process of organising data in a database to reduce redundancy...",
  segments: [
    { start: 0.0,  end: 3.8,  text: "Today we will discuss normalisation." },
    { start: 3.8,  end: 9.2,  text: "Normalisation is the process of organising data in a database to reduce redundancy." },
    { start: 9.2,  end: 15.1, text: "There are several normal forms, starting with first normal form." }
    // ... hundreds more segments
  ],
  word_count: 4523,
  language: "en",
  whisper_model: "base",
  created_at: ISODate("2026-06-05T09:02:15Z")
}
```

**Design decisions:**

| Decision | Reasoning |
|---|---|
| `segments` is **embedded** inside the transcript | Segments are always read together with the transcript (for the transcript viewer). They're never queried independently ("find all segments starting at 10.5 seconds" is not a real use case). |
| `full_text` is stored **alongside** segments | Redundant? Yes. But `full_text` is what you send to the LLM for summarisation. Concatenating segments every time would be wasteful. This is a deliberate denormalisation — trade storage for speed. |
| One transcript per conversation | A 1:1 relationship. Enforced by a unique index on `conversation_id`. If you later want to re-transcribe with a different model, you'd replace the document. |
| `whisper_model` recorded | Traceability. If you notice bad transcriptions, you can check which model produced them. Essential for debugging. |

**Size considerations:**
```
┌─────────────────────────────────────────────────────────────┐
│          TRANSCRIPT DOCUMENT SIZE ESTIMATES                  │
│                                                             │
│   Audio Duration    Words     Segments    Est. Doc Size     │
│   ──────────────    ─────     ────────    ─────────────     │
│    5 minutes         750       50         ~20 KB            │
│   15 minutes        2,250      150        ~60 KB            │
│   30 minutes        4,500      300        ~120 KB           │
│    1 hour           9,000      600        ~250 KB           │
│    2 hours          18,000    1,200       ~500 KB           │
│                                                             │
│   MongoDB limit: 16 MB per document                         │
│   Even a 2-hour transcript is well under the limit. ✅      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 5.4 `summaries` Collection

**Purpose:** Store the LLM-generated analysis of each conversation — summary, key points, and action items.

**Who writes to it:** Celery worker (Stage 2 of AI pipeline — after LLM processes the transcript).

**Who reads from it:** Conversation detail page, dashboard (summary preview on cards), action item views.

```
Document Structure:
──────────────────────────────────────────────────────────

{
  _id:              ObjectId            ← Summary document ID
  conversation_id:  ObjectId            ← Reference → conversations._id
  summary_text:     String              ← Concise overview (3-5 sentences)
  key_points:       [String]            ← Bullet-point insights (embedded array)
  action_items:     [                   ← Tasks extracted from conversation (embedded array)
    {
      text:       String,               ← What needs to be done
      deadline:   String | null,         ← Extracted deadline, if mentioned
      completed:  Boolean                ← User can mark as done
    }
  ],
  model_used:       String              ← Which LLM produced this ("gemini-1.5-flash")
  prompt_version:   String              ← Version of the prompt template used ("v1.0")
  created_at:       ISODate             ← When summarisation completed
}
```

**Example document:**
```
{
  _id: ObjectId("665f1d5e6f7a8b9c0e3f4a5b"),
  conversation_id: ObjectId("665f1b3c4d5e6f7a8b9c0e1f"),
  summary_text: "This lecture covered database normalisation, explaining the first three normal forms (1NF, 2NF, 3NF). The professor demonstrated each form using a student grades table, showing how to identify and eliminate repeating groups, partial dependencies, and transitive dependencies.",
  key_points: [
    "1NF eliminates repeating groups — each cell must contain a single atomic value",
    "2NF removes partial dependencies — every non-key attribute must depend on the entire primary key",
    "3NF removes transitive dependencies — non-key attributes must not depend on other non-key attributes",
    "Normalisation reduces data redundancy but can increase query complexity due to more tables",
    "Denormalisation is sometimes used in practice for read-heavy workloads"
  ],
  action_items: [
    {
      text: "Complete normalisation exercise from textbook Chapter 7",
      deadline: "2026-06-10",
      completed: false
    },
    {
      text: "Practice converting an unnormalised table to 3NF for the exam",
      deadline: null,
      completed: false
    }
  ],
  model_used: "gemini-1.5-flash",
  prompt_version: "v1.0",
  created_at: ISODate("2026-06-05T09:02:30Z")
}
```

**Design decisions:**

| Decision | Reasoning |
|---|---|
| `key_points` is embedded as a simple string array | Always read with the summary. Small and bounded (typically 3-10 points). No need for independent queries. |
| `action_items` is embedded as an array of objects | Action items are always displayed on the conversation page. Typically < 10 items. Each has a `completed` flag that can be toggled (update a specific array element with MongoDB's positional operator `$`). |
| `action_items` is **not** a separate collection | You might think: "Shouldn't action items be a separate collection so I can list ALL action items across all conversations?" Good question. For the MVP, we'll query across summaries using MongoDB's array querying. If action items become a major feature (with due dates, assignments, priorities), extract them to a separate collection later. |
| `prompt_version` tracked | If you change your summarisation prompt, you want to know which conversations used which prompt. Enables A/B comparison and debugging of summary quality. |
| `model_used` tracked | Different models produce different quality output. If Gemini gives poor results for long transcripts, you can identify affected summaries. |

**Estimated size per document:** ~1–5 KB

---

### 5.5 `embeddings` Collection

**Purpose:** Store the vector embeddings generated from transcript chunks for semantic search. This collection is the **source of truth** for vectors — FAISS is a speed-optimised copy.

**Who writes to it:** Celery worker (Stage 3 of AI pipeline — after embedding generation).

**Who reads from it:** Search service (to rebuild FAISS if needed), administration/debugging.

```
Document Structure:
──────────────────────────────────────────────────────────

{
  _id:              ObjectId            ← Embedding document ID
  conversation_id:  ObjectId            ← Reference → conversations._id
  user_id:          ObjectId            ← Reference → users._id (denormalised for search filtering)
  chunk_index:      Number              ← Order of this chunk within the transcript (0, 1, 2, ...)
  chunk_text:       String              ← The actual text this embedding represents (~200 words)
  vector:           [Number]            ← 384-dimensional float array (from sentence-transformers)
  start_time:       Number | null       ← Approximate start timestamp in audio (seconds)
  end_time:         Number | null       ← Approximate end timestamp in audio (seconds)
  created_at:       ISODate             ← When embedding was generated
}
```

**Example document:**
```
{
  _id: ObjectId("665f1e6f7a8b9c0e4f5a6b7c"),
  conversation_id: ObjectId("665f1b3c4d5e6f7a8b9c0e1f"),
  user_id: ObjectId("665f1a2b3c4d5e6f7a8b9c0d"),
  chunk_index: 0,
  chunk_text: "Today we will discuss normalisation. Normalisation is the process of organising data in a database to reduce redundancy. There are several normal forms, starting with first normal form which requires that each column contain atomic values...",
  vector: [0.023, -0.118, 0.445, 0.012, -0.089, ...],   // 384 floats
  start_time: 0.0,
  end_time: 45.6,
  created_at: ISODate("2026-06-05T09:02:45Z")
}
```

**Design decisions:**

| Decision | Reasoning |
|---|---|
| **`user_id` is denormalised here** | This is the most important design decision in the embeddings collection. When searching, you MUST filter results to only the current user's data. Without `user_id` here, you'd need to join to `conversations` to check ownership — slow and complex. Denormalising `user_id` means search can filter directly: "find vectors similar to X **where user_id = current_user**". |
| Each chunk is a **separate document** (not embedded in conversations) | One conversation produces 10-100+ chunks. Embedding them as a sub-array would balloon the parent document and exceed the 16 MB limit for long conversations. |
| `chunk_text` stored alongside `vector` | When search returns a match, you need the original text to display. Storing it here avoids a second query to the transcripts collection. |
| `start_time` / `end_time` included | Enables "jump to this moment in the audio" in the UI. Maps search results back to specific points in the recording. |
| `vector` stored in MongoDB, **not only in FAISS** | FAISS is in-memory and volatile. If the process crashes, FAISS loses all vectors. MongoDB is the durable backup. On startup, the search service loads vectors from MongoDB into FAISS. |

**Size considerations:**
```
┌─────────────────────────────────────────────────────────────┐
│          EMBEDDINGS DOCUMENT COUNT ESTIMATES                 │
│                                                             │
│   Audio Duration    Words     Chunks (200-word)   Docs      │
│   ──────────────    ─────     ────────────────    ──────    │
│    5 minutes         750       4-5                 ~5       │
│   15 minutes        2,250      12-15               ~15      │
│   30 minutes        4,500      25-30               ~30      │
│    1 hour           9,000      50-60               ~60      │
│    2 hours          18,000     100-120              ~120     │
│                                                             │
│   Size per document: ~1.6 KB (text) + ~1.5 KB (vector)     │
│                    ≈ 3 KB per chunk                         │
│                                                             │
│   100 conversations × 30 chunks avg = 3,000 documents      │
│   3,000 × 3 KB ≈ 9 MB total                                │
│                                                             │
│   Very manageable. MongoDB handles millions easily. ✅       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Relationships Between Collections

MongoDB doesn't enforce relationships, so **your application code must enforce them**. Here's the complete relationship map:

```
┌──────────┐         ┌────────────────┐         ┌──────────────┐
│  users   │ 1 ───── │ conversations  │ ── 1:1 ─│ transcripts  │
│          │  many   │                │         │              │
│  _id ◄───┼─────────┤ user_id        │         │ conversation │
│          │         │                │ ── 1:1 ─│  _id         │
└──────────┘         │ _id ◄──────────┼─────────┤              │
                     │                │         └──────────────┘
                     │                │
                     │                │         ┌──────────────┐
                     │                │ ── 1:1 ─│  summaries   │
                     │                │         │              │
                     │                │         │ conversation │
                     │                │         │  _id         │
                     │                │         └──────────────┘
                     │                │
                     │                │         ┌──────────────┐
                     │                │ ─ 1:N ──│  embeddings  │
                     │                │         │              │
                     │                │         │ conversation │
                     └────────────────┘         │  _id         │
                                                │ user_id      │
                                                └──────────────┘

RELATIONSHIP SUMMARY:
─────────────────────
users           → conversations    1 : many     "A user owns many conversations"
conversations   → transcripts      1 : 1        "Each conversation has exactly one transcript"
conversations   → summaries        1 : 1        "Each conversation has exactly one summary"
conversations   → embeddings       1 : many     "Each conversation has many embedding chunks"
```

**Critical rule — enforced in application code:**

> Every query to `conversations`, `transcripts`, `summaries`, or `embeddings` that serves a user request **MUST filter by `user_id`** (directly or through the parent conversation's `user_id`).

This is not a technical limitation of MongoDB — it's a **security requirement**. MongoDB won't stop User A from reading User B's transcript. Your Django views must ensure it.

---

## 7. Indexing Strategy

### Why Indexes Matter

Without indexes, MongoDB scans **every document** in a collection to answer a query (called a "collection scan"). For 10 documents, this is fine. For 100,000 documents, it's unacceptably slow.

An index is like the index at the back of a textbook — it lets MongoDB jump directly to the relevant documents instead of reading every page.

### Index Design Principle

> **Create indexes based on your query patterns, not your data structure.**

Ask: "What fields appear in `find()` filters and `sort()` calls?" Those fields need indexes.

### Kairo's Complete Index Plan

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        INDEX PLAN                                       │
│                                                                         │
│  COLLECTION: users                                                      │
│  ──────────────────────────────────────────────────                     │
│  Index 1: { email: 1 }                   UNIQUE                        │
│           ↳ Query: "Find user by email for login"                      │
│           ↳ Also prevents duplicate email registration                 │
│                                                                         │
│  Index 2: { username: 1 }                UNIQUE                        │
│           ↳ Query: "Check if username is taken"                        │
│                                                                         │
│                                                                         │
│  COLLECTION: conversations                                              │
│  ──────────────────────────────────────────────────                     │
│  Index 1: { user_id: 1, created_at: -1 }          COMPOUND             │
│           ↳ Query: "List all conversations for a user, newest first"   │
│           ↳ This is the DASHBOARD query — runs on every page load      │
│           ↳ The -1 means descending (newest first)                     │
│                                                                         │
│  Index 2: { user_id: 1, status: 1 }               COMPOUND             │
│           ↳ Query: "Find all processing/failed conversations for user" │
│           ↳ Used to show "still processing" indicators                 │
│                                                                         │
│  Index 3: { user_id: 1, tags: 1 }                 COMPOUND + MULTIKEY  │
│           ↳ Query: "Find conversations with tag 'dbms' for this user"  │
│           ↳ MongoDB automatically creates a multikey index for arrays  │
│                                                                         │
│                                                                         │
│  COLLECTION: transcripts                                                │
│  ──────────────────────────────────────────────────                     │
│  Index 1: { conversation_id: 1 }                   UNIQUE              │
│           ↳ Query: "Get transcript for conversation X"                 │
│           ↳ Unique enforces 1:1 relationship                           │
│                                                                         │
│                                                                         │
│  COLLECTION: summaries                                                  │
│  ──────────────────────────────────────────────────                     │
│  Index 1: { conversation_id: 1 }                   UNIQUE              │
│           ↳ Query: "Get summary for conversation X"                    │
│           ↳ Unique enforces 1:1 relationship                           │
│                                                                         │
│                                                                         │
│  COLLECTION: embeddings                                                 │
│  ──────────────────────────────────────────────────                     │
│  Index 1: { conversation_id: 1, chunk_index: 1 }   COMPOUND + UNIQUE  │
│           ↳ Query: "Get all chunks for conversation X, in order"       │
│           ↳ Unique prevents duplicate chunks                           │
│                                                                         │
│  Index 2: { user_id: 1 }                                               │
│           ↳ Query: "Load all vectors for user X into FAISS"            │
│           ↳ Used during FAISS index rebuild                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Understanding Index Direction

```
{ user_id: 1, created_at: -1 }
              │                 │
              │                 └── -1 means DESCENDING
              └──── 1 means ASCENDING

Why descending for created_at?
Because the dashboard always shows newest conversations first.
MongoDB can traverse an index in reverse, but a descending index
is more efficient for descending sorts.
```

### How to Verify Your Indexes Are Working

Once you build the application, use MongoDB's `explain()` to verify:

```
// This query should use the { user_id: 1, created_at: -1 } index
db.conversations.find({ user_id: ObjectId("...") }).sort({ created_at: -1 }).explain("executionStats")

// Look for:
//   "winningPlan.stage": "IXSCAN"     ← Good! Using index
//   NOT: "winningPlan.stage": "COLLSCAN"  ← Bad! Scanning every document
```

---

## 8. Query Patterns — How Data Flows Through Kairo

This section maps **every screen in the app** to the MongoDB queries it needs. This is how an experienced engineer thinks about database design — **screen-first, schema-second**.

### 8.1 Dashboard — Conversation List

```
What the user sees:
┌──────────────────────────────────────────────────────┐
│  My Conversations                         [Upload +]  │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │ 🟢 DBMS Lecture 5 - Normalisation               │  │
│  │    Jun 5, 2026 · 30 min · 3 action items        │  │
│  │    #dbms #lecture #normalisation                  │  │
│  └─────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────┐  │
│  │ 🔄 Team Standup - Sprint 12                      │  │
│  │    Jun 4, 2026 · Processing...                   │  │
│  └─────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────┐  │
│  │ 🟢 Client Call - Project Requirements            │  │
│  │    Jun 3, 2026 · 45 min · 5 action items        │  │
│  │    #client #requirements                         │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘

Queries needed:
───────────────
Query 1 (conversations):
  db.conversations.find({ user_id: <current_user> })
                   .sort({ created_at: -1 })
                   .skip(page * limit)
                   .limit(20)

  → Uses index: { user_id: 1, created_at: -1 }
  → Returns: title, status, tags, audio_duration, created_at

Query 2 (summaries — for action item count on cards):
  db.summaries.find({
    conversation_id: { $in: [<ids from Query 1>] }
  })

  → Uses index: { conversation_id: 1 }
  → Returns: only action_items array length (use projection)

Total queries: 2
```

**Why not a single query?** Because conversations and summaries are separate collections. You could use `$lookup` (MongoDB's join equivalent), but two simple queries are often faster and easier to understand than one complex aggregation pipeline. Django makes both queries and combines the results in Python.

### 8.2 Conversation Detail Page

```
What the user sees:
┌──────────────────────────────────────────────────────┐
│  ← Back    DBMS Lecture 5 - Normalisation            │
│                                                       │
│  ┌─────────────────────┐  ┌───────────────────────┐  │
│  │    TRANSCRIPT        │  │    SUMMARY            │  │
│  │                      │  │                       │  │
│  │  [00:00] Today we    │  │  This lecture covered  │  │
│  │  will discuss        │  │  normalisation...     │  │
│  │  normalisation...    │  │                       │  │
│  │                      │  │  KEY POINTS:          │  │
│  │  [00:04] Normalisa-  │  │  • 1NF eliminates...  │  │
│  │  tion is the process │  │  • 2NF removes...     │  │
│  │  of organising...    │  │                       │  │
│  │                      │  │  ACTION ITEMS:        │  │
│  │  [00:09] There are   │  │  ☐ Complete exercise  │  │
│  │  several normal...   │  │  ☐ Practice for exam  │  │
│  └──────────────────────┘  └───────────────────────┘  │
└──────────────────────────────────────────────────────┘

Queries needed:
───────────────
Query 1: db.conversations.findOne({
           _id: <conversation_id>,
           user_id: <current_user>        ← SECURITY: ownership check
         })

Query 2: db.transcripts.findOne({
           conversation_id: <conversation_id>
         })

Query 3: db.summaries.findOne({
           conversation_id: <conversation_id>
         })

Total queries: 3
All use unique indexes — each returns in < 1ms.
```

### 8.3 Semantic Search

```
What the user sees:
┌──────────────────────────────────────────────────────┐
│  🔍  "What was said about normalisation?"             │
│                                                       │
│  Results:                                             │
│  ┌─────────────────────────────────────────────────┐  │
│  │ 📄 DBMS Lecture 5 - Normalisation    Score: 92%  │  │
│  │    "Normalisation is the process of organising   │  │
│  │     data in a database to reduce redundancy..."  │  │
│  │    ⏱ Jump to 0:04                                │  │
│  └─────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────┐  │
│  │ 📄 DBMS Lecture 3 - ER Diagrams      Score: 71%  │  │
│  │    "...before we normalise, we need to           │  │
│  │     understand the entity relationships..."      │  │
│  │    ⏱ Jump to 12:30                               │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘

Query flow (hybrid — FAISS + MongoDB):
──────────────────────────────────────
Step 1: Generate embedding for query text → query_vector
Step 2: FAISS search: find top 10 most similar vectors
        → Returns: [(embedding_id_1, score), (embedding_id_2, score), ...]
Step 3: db.embeddings.find({
          _id: { $in: [<matched embedding IDs>] },
          user_id: <current_user>             ← SECURITY: double-check ownership
        })
        → Returns: chunk_text, conversation_id, start_time, end_time
Step 4: db.conversations.find({
          _id: { $in: [<unique conversation IDs from step 3>] }
        })
        → Returns: title, tags (for display in search results)

Total: 1 FAISS search + 2 MongoDB queries
```

---

## 9. Data Lifecycle — From Upload to Deletion

### Conversation Status State Machine

Every conversation document goes through a defined set of states:

```
                    ┌─────────────┐
                    │  uploading  │  ← File is being received
                    └──────┬──────┘
                           │ File saved successfully
                           ▼
                    ┌─────────────┐
         ┌─────────│ processing  │  ← In the Celery pipeline
         │         └──────┬──────┘
         │                │ All 3 stages complete
         │                ▼
         │         ┌─────────────┐
         │         │    ready    │  ← Fully processed, viewable
         │         └─────────────┘
         │
         │ Any stage fails
         ▼
  ┌─────────────┐
  │   failed    │  ← Error occurred, error_message populated
  └──────┬──────┘
         │ User clicks "Retry"
         ▼
  ┌─────────────┐
  │ processing  │  ← Re-enter the pipeline
  └─────────────┘
```

### Document Creation Timeline

```
TIME ──────────────────────────────────────────────────────────────────────▶

t=0s          t=0.5s         t=120s          t=135s          t=145s
│              │               │               │               │
│ Upload       │ conversations │ transcripts   │ summaries     │ embeddings
│ request      │ doc created   │ doc created   │ doc created   │ docs created
│ arrives      │ status:       │ (Whisper      │ (LLM          │ (sentence-
│              │ "processing"  │  finishes)    │  finishes)    │  transformers
│              │               │               │               │  finishes)
│              │               │               │               │
│              │               │               │               │ conversations
│              │               │               │               │ status → "ready"
```

### Deletion — Cascade Responsibility

When a user deletes a conversation, **all related data must be removed**. MongoDB won't do this for you — there are no cascading deletes.

```
DELETE conversation "abc123":
─────────────────────────────

Your Django code must:

  1. db.embeddings.deleteMany({ conversation_id: "abc123" })
  2. db.summaries.deleteOne({ conversation_id: "abc123" })
  3. db.transcripts.deleteOne({ conversation_id: "abc123" })
  4. Remove audio file from filesystem
  5. Remove vectors from FAISS index
  6. db.conversations.deleteOne({ _id: "abc123", user_id: <current_user> })

ORDER MATTERS:
  Delete children first, parent last.
  If step 6 runs first and step 2 fails, you'd have an
  orphaned summary with no parent conversation — data leak.
```

---

## 10. Django + MongoDB Integration

### The Approach Decision

Django was built for SQL databases (PostgreSQL, MySQL, SQLite). To use MongoDB, you have three options:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DJANGO + MONGODB OPTIONS                           │
│                                                                     │
│  Option 1: djongo                                                   │
│  ───────────────                                                    │
│  What: A SQL-to-MongoDB translator. You write Django models         │
│        normally, and djongo converts ORM queries to MongoDB.        │
│  Pros: Familiar Django patterns, migrations work (mostly)           │
│  Cons: Buggy, poorly maintained, doesn't support all MongoDB       │
│        features, can't use aggregation pipelines                    │
│  Verdict: ❌ Not recommended for new projects                       │
│                                                                     │
│  Option 2: MongoEngine                                              │
│  ───────────────────                                                │
│  What: A dedicated MongoDB ODM (Object-Document Mapper) for        │
│        Python. Replaces Django's ORM entirely.                      │
│  Pros: Mature, well-documented, MongoDB-native, supports           │
│        embedded documents and all MongoDB features                  │
│  Cons: Not compatible with Django's built-in admin, auth,           │
│        or migrations. You lose some Django "batteries."             │
│  Verdict: ✅ Good choice if you're okay managing auth yourself      │
│                                                                     │
│  Option 3: PyMongo (direct driver)                                  │
│  ───────────────────────────────                                    │
│  What: The official MongoDB Python driver. You write queries        │
│        directly — no ORM, no model classes.                         │
│  Pros: Full control, no abstraction leaks, supports everything     │
│  Cons: More code to write, no model validation layer, no ORM       │
│        conveniences                                                 │
│  Verdict: ✅ Good choice for maximum learning and control            │
│                                                                     │
│  RECOMMENDED FOR KAIRO: MongoEngine                                 │
│  ──────────────────────────────────                                  │
│  Why: It gives you model-like classes (familiar from Django),       │
│  built-in validation, and native MongoDB features. You'll learn    │
│  both MongoDB concepts AND good Python patterns.                    │
│                                                                     │
│  Use PyMongo directly for complex aggregation pipelines (e.g.,     │
│  search) where MongoEngine's abstraction gets in the way.          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### How the Layers Fit Together

```
┌──────────────────────────────────────────────────────┐
│           DJANGO + MONGODB ARCHITECTURE               │
│                                                       │
│   DRF Serializers                                     │
│       ↕ validate & transform                         │
│   MongoEngine Documents (your "models")               │
│       ↕ query & persist                              │
│   PyMongo Driver (used by MongoEngine internally)     │
│       ↕ BSON protocol                                │
│   MongoDB Server                                      │
│                                                       │
│   Note: Django's ORM (models.py) is NOT used.        │
│   You define MongoEngine Document classes instead.    │
│   Django's built-in User model is NOT used.           │
│   You build your own auth with JWT.                   │
│                                                       │
└──────────────────────────────────────────────────────┘
```

### Authentication Without Django's Built-in User Model

Since MongoEngine replaces Django's ORM, you can't use `django.contrib.auth.models.User`. You'll build a lightweight auth system yourself:

```
What you'll implement yourself:
─────────────────────────────────
1. User document (MongoEngine) with email, password_hash
2. Registration view — validate input, hash password, save to MongoDB
3. Login view — find user by email, verify hash, issue JWT
4. JWT middleware — extract token from Authorization header, load user
5. Permission check — verify user owns the requested resource

What you'll use libraries for:
─────────────────────────────────
- bcrypt (or passlib) → password hashing
- PyJWT (or djangorestframework-simplejwt) → JWT creation and verification
- DRF's permission classes → attach permission checks to views
```

---

## 11. Migration & Evolution Strategy

### MongoDB Doesn't Have Migrations (Like Django's `makemigrations`)

In SQL, adding a field requires `ALTER TABLE`. In MongoDB, you just... start writing documents with the new field. Old documents still work — they simply don't have that field.

**But you still need a strategy:**

```
SCENARIO: You want to add a "language" field to conversations
──────────────────────────────────────────────────────────────

SQL approach (what you'd do in PostgreSQL):
  ALTER TABLE conversations ADD COLUMN language VARCHAR(5) DEFAULT 'en';
  → Every existing row gets the new column immediately

MongoDB approach:
  1. Update your MongoEngine Document class to include `language`
  2. Set a default value in the class definition
  3. New documents automatically have the field
  4. Old documents DON'T have it — they return the default when queried
  5. Optionally: write a one-time script to backfill old documents

  # Backfill script (run once):
  # db.conversations.updateMany(
  #   { language: { $exists: false } },
  #   { $set: { language: "en" } }
  # )
```

### Schema Version Tracking (Best Practice)

```
APPROACH: Add a schema_version field to documents
──────────────────────────────────────────────────

// Old document (v1):
{
  _id: ObjectId("..."),
  title: "DBMS Lecture",
  status: "ready"
  // no schema_version field
}

// New document (v2):
{
  _id: ObjectId("..."),
  title: "DBMS Lecture",
  status: "ready",
  language: "en",            ← new field
  schema_version: 2          ← tracks which version
}

Your application code handles both:
  if doc.schema_version < 2:
      doc.language = "en"  // apply default
```

---

## 12. Common Mistakes to Avoid

These are mistakes that students and junior developers frequently make with MongoDB. Learn from them before you encounter them.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON MONGODB MISTAKES                              │
│                                                                         │
│  ❌ MISTAKE 1: Storing audio files in MongoDB                           │
│  ─────────────────────────────────────────────                          │
│  Temptation: "I'll use GridFS to store audio in MongoDB"               │
│  Problem: Degrades database performance, makes backups huge,           │
│           and MongoDB isn't optimised for large binary data.            │
│  Fix: Store audio on filesystem. Store the FILE PATH in MongoDB.       │
│                                                                         │
│                                                                         │
│  ❌ MISTAKE 2: Not filtering by user_id                                 │
│  ─────────────────────────────────────────                              │
│  Temptation: db.conversations.findOne({ _id: conversation_id })       │
│  Problem: Any user can access any conversation if they guess the ID.   │
│  Fix: ALWAYS include user_id in the query:                             │
│       db.conversations.findOne({ _id: id, user_id: current_user })    │
│                                                                         │
│                                                                         │
│  ❌ MISTAKE 3: Forgetting to create indexes                             │
│  ─────────────────────────────────────────                              │
│  Symptom: "The dashboard was fast with 10 conversations but is now     │
│           slow with 500"                                                │
│  Problem: Without indexes, MongoDB scans every document.               │
│  Fix: Create indexes BEFORE you have lots of data.                     │
│       Use explain() to verify queries use indexes.                     │
│                                                                         │
│                                                                         │
│  ❌ MISTAKE 4: Embedding unbounded arrays                               │
│  ─────────────────────────────────────────                              │
│  Temptation: Store all embeddings inside the conversation document     │
│  Problem: Array grows with every chunk. 100 chunks × 384 floats =     │
│           153,600 numbers. Document becomes > 1 MB. Slows reads.      │
│  Fix: Use a separate embeddings collection (as designed above).        │
│                                                                         │
│                                                                         │
│  ❌ MISTAKE 5: Not handling orphaned documents                          │
│  ─────────────────────────────────────────                              │
│  Scenario: You delete a conversation but forget to delete its          │
│            transcript, summary, and embeddings.                        │
│  Problem: Orphaned data wastes space and may leak private info.        │
│  Fix: Always delete children before parent (see §9 deletion order).   │
│       Consider writing a cleanup script that finds orphans.            │
│                                                                         │
│                                                                         │
│  ❌ MISTAKE 6: Using string IDs instead of ObjectId                     │
│  ─────────────────────────────────────────────────                      │
│  Temptation: Store user_id as "665f1a2b..." (string)                   │
│  Problem: String comparison is slower than ObjectId comparison.        │
│           Indexes on strings use more memory.                          │
│           Queries with mismatched types return empty results silently. │
│  Fix: Always use ObjectId type for references.                         │
│       Convert strings to ObjectId before querying.                     │
│                                                                         │
│                                                                         │
│  ❌ MISTAKE 7: Not using projections                                    │
│  ─────────────────────────────────────────                              │
│  Temptation: db.transcripts.findOne({ conversation_id: id })          │
│  Problem: Returns the ENTIRE document including the full_text          │
│           (potentially 30,000 words) even if you only need word_count. │
│  Fix: Use projections to fetch only needed fields:                     │
│       db.transcripts.findOne(                                          │
│         { conversation_id: id },                                       │
│         { word_count: 1, language: 1 }   ← only these fields          │
│       )                                                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 13. Security at the Database Layer

### MongoDB Security Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│                  DATABASE SECURITY CHECKLIST                     │
│                                                                 │
│  ☐ 1. ENABLE AUTHENTICATION                                    │
│     MongoDB ships with auth DISABLED by default.                │
│     Always enable it, even in development.                      │
│     Create a dedicated database user for Kairo (not root).     │
│                                                                 │
│  ☐ 2. BIND TO LOCALHOST                                        │
│     Default: MongoDB listens on all interfaces (0.0.0.0).      │
│     Fix: Bind to 127.0.0.1 so only local processes can access. │
│     In mongod.conf: bindIp: 127.0.0.1                          │
│                                                                 │
│  ☐ 3. USE A DEDICATED DATABASE USER                            │
│     Don't use the admin user for application queries.           │
│     Create a user with readWrite role on kairo_db only.         │
│     Principle of least privilege.                               │
│                                                                 │
│  ☐ 4. NEVER BUILD QUERIES FROM RAW USER INPUT                  │
│     MongoDB is vulnerable to NoSQL injection.                   │
│                                                                 │
│     VULNERABLE:                                                 │
│       query = { "email": request.data["email"] }               │
│       → If email = {"$gt": ""}, returns ALL users!             │
│                                                                 │
│     SAFE:                                                       │
│       Validate input type before querying.                      │
│       Use MongoEngine's type-checked fields.                    │
│       Never pass raw dicts from request into query filters.    │
│                                                                 │
│  ☐ 5. HASH PASSWORDS WITH BCRYPT                               │
│     Never store plaintext. Never use MD5 or SHA-256 alone.     │
│     bcrypt includes salt automatically and is slow by design    │
│     (making brute-force attacks impractical).                   │
│                                                                 │
│  ☐ 6. STORE CONNECTION STRINGS IN ENVIRONMENT VARIABLES        │
│     Never hardcode: mongodb://user:password@host:27017/kairo_db │
│     Use: os.environ["MONGODB_URI"]                              │
│     Add .env to .gitignore                                      │
│                                                                 │
│  ☐ 7. BACKUP STRATEGY                                          │
│     Use mongodump for regular backups.                          │
│     For production: enable MongoDB Atlas automatic backups.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 14. Scalability Considerations

### Document Growth Projections

```
SCENARIO: One active user over 6 months, uploading 3 conversations/week
──────────────────────────────────────────────────────────────────────────

  Conversations: 3/week × 26 weeks = 78 documents × ~600 bytes = ~47 KB
  Transcripts:   78 documents × ~120 KB avg = ~9.4 MB
  Summaries:     78 documents × ~3 KB avg = ~234 KB
  Embeddings:    78 × 30 chunks avg = 2,340 documents × ~3 KB = ~7 MB
  Audio files:   78 × 15 MB avg = ~1.17 GB (on filesystem, NOT in MongoDB)

  TOTAL IN MONGODB: ~17 MB
  MongoDB default storage limit: effectively unlimited on disk

  ✅ A single MongoDB instance handles this trivially.


SCENARIO: 100 users over 1 year
────────────────────────────────

  Conversations: 100 × 156 = 15,600 documents
  Embeddings: 100 × 4,680 = 468,000 documents
  Total MongoDB storage: ~1.7 GB

  ✅ Still trivially handled by a single MongoDB instance.
  MongoDB routinely handles millions of documents and terabytes.


WHEN YOU'D NEED TO SCALE:
─────────────────────────
  10,000+ users → Consider MongoDB Atlas (managed) or replica sets
  1M+ embeddings → FAISS index rebuild time increases; consider Milvus
  Heavy concurrent writes → Connection pooling + write concern tuning
```

### Connection Pooling

```
WHY IT MATTERS:
───────────────
Each MongoDB connection uses ~1 MB of memory.
If every Django request opens a new connection, you'll exhaust resources.

SOLUTION:
─────────
Use connection pooling (PyMongo does this by default).
Set maxPoolSize in your connection URI:

  mongodb://user:pass@localhost:27017/kairo_db?maxPoolSize=20

This means up to 20 concurrent connections are reused.
For a university project, the default (100) is more than enough.
```

---

## Summary: What to Do When You Start Building

Here is your step-by-step database setup checklist:

```
STEP 1: Install MongoDB locally
────────────────────────────────
  → Install MongoDB Community Edition for macOS
  → Start the mongod service
  → Verify with mongosh (the MongoDB shell)

STEP 2: Create the database and user
─────────────────────────────────────
  → Create kairo_db database
  → Create a dedicated application user with readWrite role
  → Store connection string in .env file

STEP 3: Install MongoEngine
────────────────────────────
  → Add mongoengine to your Django project's requirements
  → Configure the connection in Django settings

STEP 4: Define your Document classes
─────────────────────────────────────
  → Create MongoEngine Document classes for each collection
  → Define fields with types and validation
  → This document tells you exactly what fields each collection needs

STEP 5: Create indexes
───────────────────────
  → Define indexes in your MongoEngine Document classes (meta = {'indexes': [...]})
  → MongoEngine creates them automatically when the app starts
  → Verify with mongosh: db.conversations.getIndexes()

STEP 6: Test with sample data
──────────────────────────────
  → Insert test documents via mongosh or a Django management command
  → Verify queries work as expected
  → Use explain() to confirm indexes are used

STEP 7: Wire up to Django views
────────────────────────────────
  → Create DRF serializers that work with MongoEngine documents
  → Build your API endpoints
  → Test end-to-end: React → Django → MongoDB → Response
```

---

*This document should be read alongside [architecture.md](file:///Users/mohammed/kairo/docs/architecture.md) which explains how the database fits into the overall system.*

*Last updated: 2026-06-05*
*Author: Mohammed Kutubudin Kapasi*
