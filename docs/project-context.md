# Kairo — Project Context

> **AI Conversational Memory Assistant**
> A web-based intelligent recall system that transforms spoken conversations into structured, searchable, and actionable knowledge.

**Author:** Mohammed Kutubudin Kapasi (BCA23745)
**Program:** Bachelor of Computer Applications, University of Mysore
**Year:** 2026
**Status:** Pre-development — Architecture & Planning Phase

---

## 1. Project Vision

Kairo is a **digital extension of human memory**. It captures the knowledge embedded in everyday conversations — meetings, lectures, consultations, brainstorming sessions — and makes that knowledge **permanent, structured, and instantly retrievable**.

The name *Kairo* (from the Greek *kairos*, "the opportune moment") reflects the product's core promise: **the right information, recalled at the right time**.

### Vision Statement

> *"Never lose a conversation again. Kairo turns what was said into what you can search, summarise, and act on."*

### Design Principles

| Principle | Meaning |
|---|---|
| **Memory-first** | Every feature serves the goal of augmenting human recall |
| **Conversations as data** | Audio is the input; structured knowledge is the output |
| **Ask, don't browse** | Users retrieve information through natural language queries, not manual navigation |
| **Progressive intelligence** | The system gets more useful as more conversations are stored |

---

## 2. Problem Statement

### The Problem

In academic and professional settings, critical information is exchanged through spoken conversations — lectures, meetings, discussions, consultations. However:

1. **Human memory is unreliable.** Studies show people forget ~50% of new information within an hour and ~70% within 24 hours (Ebbinghaus forgetting curve).
2. **Manual note-taking is lossy.** It splits attention between listening and writing, producing incomplete records.
3. **Raw audio recordings are unusable at scale.** A 60-minute meeting produces a monolithic audio file that is impossible to search, skim, or extract action items from.
4. **Existing tools solve only part of the problem.** Transcription services produce text walls without structure. Meeting assistants are tied to specific platforms (Zoom, Teams). No affordable, general-purpose tool combines transcription + summarisation + semantic search + task extraction in a single workflow.

### The Gap Kairo Fills

| Existing Approach | Limitation | Kairo's Solution |
|---|---|---|
| Manual notes | Incomplete, attention-splitting | Automated transcription from uploaded audio |
| Voice recorders | Unstructured, unsearchable | AI-generated summaries, key points, and action items |
| Transcription services (Otter, Rev) | Text-only output, no semantic understanding | NLP-powered insight extraction and semantic search |
| Meeting bots (Fireflies, Recall.ai) | Platform-locked, subscription-heavy | Platform-agnostic upload; open-source AI stack |

---

## 3. Target Users

### Primary Personas

#### 👨‍🎓 Students & Self-Learners
- Upload lecture recordings to generate study notes
- Query past lectures before exams: *"What did the professor say about normalisation in DBMS?"*
- Extract assignment deadlines and action items automatically

#### 👩‍💼 Professionals & Knowledge Workers
- Process meeting recordings into structured summaries
- Search across weeks of meetings: *"What decisions were made about the Q3 budget?"*
- Never miss an action item buried in a long discussion

#### 👥 Freelancers & Small Teams
- Document client calls without manual note-taking
- Track commitments and deliverables across multiple projects
- Build a searchable archive of project discussions

### User Needs Matrix

| Need | Priority | Feature Mapping |
|---|---|---|
| Convert audio to text accurately | **P0 — Critical** | Speech-to-text pipeline (Whisper) |
| Get a quick summary of a conversation | **P0 — Critical** | LLM-powered summarisation |
| Find specific information from past conversations | **P0 — Critical** | Semantic search with vector embeddings |
| See action items and deadlines | **P1 — High** | NLP-based task extraction |
| Organise conversations by topic/project | **P1 — High** | Tagging and folder system |
| Secure, private storage of conversations | **P1 — High** | Authentication + encrypted storage |
| Share conversation summaries with others | **P2 — Medium** | Export / share functionality (future) |

---

## 4. Core Features

### MVP Features (v1.0)

#### 4.1 Audio Upload & Management
- Upload audio files (MP3, WAV, M4A, WebM) via a web interface
- Associate uploads with a user account and optional metadata (title, date, tags)
- View upload history with status indicators (processing, ready, failed)

#### 4.2 AI Transcription
- Automatic speech-to-text conversion using OpenAI Whisper (or equivalent)
- Support for English language (multi-language as future scope)
- Timestamped transcript output for navigation

#### 4.3 Intelligent Summarisation
- Generate concise summaries of each conversation using an LLM
- Extract structured sections: **Summary**, **Key Points**, **Action Items**
- Allow users to regenerate or refine summaries

#### 4.4 Semantic Search & Query
- Natural language query interface: *"What was discussed about the project deadline?"*
- Vector embedding-based search across all stored conversations
- Return relevant transcript segments with source attribution

#### 4.5 Conversation Dashboard
- Central hub for all uploaded and processed conversations
- Filter and sort by date, tags, or status
- Quick-view cards showing title, date, summary preview, and action item count

#### 4.6 User Authentication
- Secure user registration and login
- Each user's data is isolated and private
- Session management with secure token handling

### Future Features (Post-MVP)

| Feature | Description |
|---|---|
| Real-time transcription | Live microphone input with streaming transcription |
| Speaker diarisation | Identify and label different speakers in a conversation |
| Calendar integration | Sync action items and deadlines to Google Calendar |
| Mobile app | React Native or PWA for on-the-go access |
| Team workspaces | Shared conversation libraries with role-based access |
| Communication analytics | Track talk-time ratios, topic frequency, sentiment trends |

---

## 5. Technical Stack

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT (Browser)                      │
│                                                         │
│   React.js SPA                                          │
│   ├── Audio Upload UI                                   │
│   ├── Conversation Dashboard                            │
│   ├── Transcript Viewer                                 │
│   ├── Search / Query Interface                          │
│   └── Auth (Login / Register)                           │
└─────────────────┬───────────────────────────────────────┘
                  │ HTTP / REST API
                  ▼
┌─────────────────────────────────────────────────────────┐
│                 BACKEND (Django / Python)                │
│                                                         │
│   Django REST Framework                                 │
│   ├── Auth Module (JWT / Session)                       │
│   ├── Upload Handler (file validation, storage)         │
│   ├── Transcription Service (Whisper integration)       │
│   ├── NLP Pipeline (summarisation, extraction)          │
│   ├── Search Service (embedding + vector query)         │
│   └── REST API Endpoints                                │
└────────┬────────────────┬───────────────┬───────────────┘
         │                │               │
         ▼                ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
│   MongoDB    │ │ File Storage │ │  Vector Database  │
│              │ │              │ │                   │
│ • Users      │ │ • Audio files│ │ • Conversation    │
│ • Transcripts│ │ • Processed  │ │   embeddings      │
│ • Summaries  │ │   exports    │ │ • Semantic index   │
│ • Action     │ │              │ │                   │
│   items      │ │              │ │ (FAISS / Pinecone)│
└──────────────┘ └──────────────┘ └──────────────────┘
```

### Stack Decisions

| Layer | Technology | Rationale |
|---|---|---|
| **Frontend** | React.js | Component-based UI; rich ecosystem; team familiarity |
| **Backend** | Django (Python) + Django REST Framework | Python-native AI/ML library access; robust ORM; built-in admin; mature auth system |
| **Transcription** | OpenAI Whisper (open-source) | State-of-the-art accuracy; runs locally; no API cost for inference |
| **Summarisation / NLP** | LLM (e.g., OpenAI API, or local model via Ollama) | High-quality natural language understanding and generation |
| **Database** | MongoDB | Flexible document schema for transcripts, summaries, and variable-length conversation data |
| **Vector Search** | FAISS (local) or Pinecone (managed) | Efficient similarity search over conversation embeddings for semantic queries |
| **Auth** | Django auth + JWT (djangorestframework-simplejwt) | Industry-standard token-based authentication for SPA clients |
| **File Storage** | Local filesystem (dev) / S3-compatible (prod) | Simple for development; scalable path for production |

### Key Technical Considerations

- **Whisper model size:** Use `base` or `small` for development speed; `medium` for production accuracy. The `large` model provides the best results but requires significant compute.
- **LLM strategy:** Start with OpenAI API for rapid prototyping. Evaluate local alternatives (Ollama + Llama/Mistral) for cost control and privacy.
- **Embedding model:** Use `sentence-transformers/all-MiniLM-L6-v2` or OpenAI `text-embedding-ada-002` for generating conversation embeddings.
- **Async processing:** Audio transcription and NLP processing are long-running tasks. Use Celery + Redis (or Django background tasks) to handle them asynchronously.

---

## 6. Current Status

### Phase: Pre-Development — Architecture & Planning

| Milestone | Status |
|---|---|
| Project synopsis approved | ✅ Complete |
| Project vision and scope defined | ✅ Complete |
| Repository initialised | ✅ Complete |
| Project context documented | ✅ Complete (this document) |
| Technical stack finalised | ✅ Complete |
| System architecture designed | ✅ Complete ([architecture.md](file:///Users/mohammed/kairo/docs/architecture.md)) |
| Database schema designed | ✅ Complete ([database-design.md](file:///Users/mohammed/kairo/docs/database-design.md)) |
| API contract defined | ✅ Complete ([backend-design.md](file:///Users/mohammed/kairo/docs/backend-design.md)) |
| Frontend wireframes / UI design | 🔲 Not started |
| Development environment setup | 🔲 Not started |
| MVP implementation | 🔲 Not started |
| Testing and validation | 🔲 Not started |
| Documentation and submission | 🔲 Not started |

### Immediate Next Steps

1. **System architecture document** — Define component boundaries, data flow diagrams, and API contracts
2. **Database schema design** — Model entities (User, Conversation, Transcript, Summary, ActionItem, Embedding)
3. **Development environment setup** — Initialise Django project, React app, MongoDB connection, and Whisper integration
4. **UI wireframes** — Design the conversation dashboard, upload flow, transcript viewer, and search interface

---

## Appendix: Glossary

| Term | Definition |
|---|---|
| **Transcription** | Converting spoken audio into written text |
| **Summarisation** | Generating a concise overview of a longer text |
| **Action item** | A task or commitment identified from a conversation |
| **Semantic search** | Finding information based on meaning rather than exact keyword matching |
| **Embedding** | A numerical vector representation of text that captures its semantic meaning |
| **Vector database** | A database optimised for storing and querying high-dimensional vectors |
| **Diarisation** | Identifying and separating different speakers in an audio recording |
| **LLM** | Large Language Model — an AI model trained on large text corpora for language tasks |

---

*Last updated: 2026-06-04*
*Document owner: Mohammed Kutubudin Kapasi*
