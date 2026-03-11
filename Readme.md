# Resume Chatbot — AI-Powered Candidate Search System

> **An enterprise-grade agentic AI system that enables HR teams to search 20,000+ resumes using natural language, built with FastAPI, PostgreSQL, and GPT-4.**

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [The Agentic SQL Engine](#the-agentic-sql-engine)
4. [Context Management System](#context-management-system)
5. [Token Optimization Strategies](#token-optimization-strategies)
6. [Resume Parsing Pipeline](#resume-parsing-pipeline)
7. [Data Ingestion at Scale](#data-ingestion-at-scale)
8. [API Design](#api-design)
9. [Database Design](#database-design)
10. [Frontend Integration](#frontend-integration)
11. [Technical Highlights](#technical-highlights)

---

## Project Overview

### The Problem
HR and Talent Acquisition teams receive hundreds of resumes daily via email. Finding the right candidate means manually searching through thousands of documents — a time-consuming and error-prone process.

### The Solution
I built an **Agentic AI Chatbot** that allows users to search resumes using natural language:

```
User: "Find Python developers with 5+ years experience who worked on fintech projects"

Bot: Found 12 candidates matching your criteria:
     1. John Doe — 7 years — Python, Django, AWS — Fintech background
     2. Jane Smith — 5 years — Python, FastAPI, Docker — Payment systems
     ...
```

### Key Capabilities
- **Natural Language to SQL**: Converts plain English queries into optimized PostgreSQL queries
- **Context-Aware Conversations**: Remembers previous queries for intelligent follow-ups
- **Multi-Intent Classification**: Handles search, detailed profiles, Excel exports, and general chat
- **Resume PDF Analysis**: Deep-dives into resume content using GPT-4 when structured data isn't enough
- **Excel Export**: Bulk download query results with custom column selection
- **Real-time Sync**: Automatically ingests resumes from Microsoft 365 email

### Scale
- **20,000+ resumes** stored and searchable
- **~500 unique skills** indexed from candidate profiles
- **Sub-second query response** for most searches
- **85% cost reduction** through incremental sync and token optimization

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND                                        │
│                       Next.js + TypeScript                                  │
│                     (Chat UI, Authentication)                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                              REST API Calls
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          FASTAPI BACKEND                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │Auth API  │  │Chat API  │  │Resume API│  │Sync API  │  │Export API│      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
│       │             │             │             │             │              │
│       └─────────────┴──────┬──────┴─────────────┴─────────────┘              │
│                            │                                                 │
│  ┌─────────────────────────▼─────────────────────────────────────────────┐  │
│  │                     AGENTIC SQL ENGINE                                 │  │
│  │                                                                        │  │
│  │   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐   │  │
│  │   │  Intent    │   │    SQL     │   │  Query     │   │  Response  │   │  │
│  │   │  Detector  │──▶│  Generator │──▶│  Executor  │──▶│  Formatter │   │  │
│  │   └────────────┘   └────────────┘   └────────────┘   └────────────┘   │  │
│  │         │                 │                                │           │  │
│  │         └────────────────────────Context Manager───────────┘           │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                      ▼
     ┌────────────────┐    ┌────────────────┐    ┌────────────────┐
     │   PostgreSQL   │    │   OpenAI API   │    │  MS Graph API  │
     │   (20K+ rows)  │    │   (GPT-4o)     │    │  (Email Sync)  │
     └────────────────┘    └────────────────┘    └────────────────┘
```

### Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Backend** | FastAPI + Python 3.11 | Async REST API, high performance |
| **AI/LLM** | OpenAI GPT-4o | Intent detection, SQL generation, response formatting |
| **Database** | PostgreSQL + JSONB | Structured resume data with flexible schema |
| **ORM** | SQLAlchemy 2.0 (async) | Type-safe database operations |
| **Email Integration** | MS Graph API | OAuth2 client credentials flow for email access |
| **Document Parsing** | pypdf + python-docx | Extract text from PDF/Word resumes |
| **Export** | openpyxl | Generate styled Excel spreadsheets |
| **Frontend** | Next.js + TypeScript | Modern React-based chat interface |

---

## The Agentic SQL Engine

The heart of this system is the **Agentic SQL Engine** — a multi-step AI pipeline that converts natural language into database queries.

### Why "Agentic"?

Unlike simple prompt-to-SQL approaches, this system:
1. **Classifies intent** before choosing a processing path
2. **Maintains conversation state** across multiple exchanges
3. **Retries with self-correction** when SQL execution fails
4. **Decides dynamically** whether to use structured data or read full resume PDFs

### Intent Classification (5 Types)

```python
# The agent first classifies every user message into one of 5 intents:

INTENTS = {
    "resume_query":   "Search/filter/analyze resumes → Generate SQL",
    "person_detail":  "Deep-dive into ONE candidate → Use structured data or PDF",
    "excel_export":   "Bulk download request → Generate Excel file",
    "conversation":   "General chat → Use conversation context",
    "out_of_scope":   "Unrelated questions → Polite redirect"
}
```

### Query Processing Flow

```
User: "Find Python developers with 5+ years experience"
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    STEP 1: DETECT INTENT                             │
│  Input: User message + Last 4 conversation pairs                     │
│  Output: "resume_query"                                              │
│  Model: GPT-4o (temperature=0)                                       │
└─────────────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    STEP 2: GENERATE SQL                              │
│  Input: User request + Schema + Query rules + Distinct values        │
│  Context: Previous SQL (if follow-up detected)                       │
│  Output:                                                             │
│    SELECT id, name, email, skills, experience_years                  │
│    FROM resumes                                                      │
│    WHERE skills @> '["Python"]'::jsonb                               │
│      AND experience_years >= 5                                       │
│    ORDER BY experience_years DESC                                    │
│    LIMIT 100                                                         │
└─────────────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    STEP 3: EXECUTE SQL                               │
│  - Validates SELECT-only (blocks INSERT/UPDATE/DELETE)               │
│  - Executes against PostgreSQL                                       │
│  - Filters binary columns (pdf_data)                                 │
│  - Returns: {results: [...], row_count: N}                           │
└─────────────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    STEP 4: FORMAT RESPONSE                           │
│  - ≤10 rows: Markdown table with key details                         │
│  - >10 rows: Summary ("Found 42 Python developers...")               │
│  - Appends Excel download link if user requested                     │
└─────────────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    STEP 5: UPDATE CONTEXT                            │
│  - Store conversation pair (user_msg, ai_response, sql, summary)     │
│  - If single result: Set as "active candidate" for follow-ups        │
│  - If multiple results: Clear active candidate                       │
└─────────────────────────────────────────────────────────────────────┘
```

### SQL Retry Logic with Self-Correction

```python
async def _process_resume_query(self, user_message: str, ...):
    """
    Retry up to 3 times if SQL execution fails.
    Each retry includes the error message so LLM can self-correct.
    """
    max_sql_retries = 3
    last_error = None
    
    for attempt in range(max_sql_retries):
        context = ""
        if last_error:
            # Feed error back to LLM for self-correction
            context = f"\n\nPrevious attempt failed: {last_error}\nPlease fix the SQL."
        
        sql_result = await self.tools.generate_sql(user_message + context)
        exec_result = await self.tools.execute_sql(sql_result["sql"])
        
        if exec_result["status"] == "success":
            return await self.tools.format_response(...)
        
        last_error = exec_result.get("error")
    
    return f"Query failed after {max_sql_retries} attempts: {last_error}"
```

### LLM-Based Follow-up Detection

Instead of brittle keyword matching ("show me", "these", "them"), the system uses **GPT-4 to intelligently detect follow-ups**:

```python
# Context passed to SQL generation LLM:
"""
PREVIOUS QUERY CONTEXT:
Previous user request: "how many HR resumes do we have?"
Previous SQL: SELECT COUNT(*) FROM resumes WHERE department ILIKE '%HR%'
Previous result: Found 330 HR resumes

Current request: "can you show me?"

→ LLM decides: This CONTINUES the previous query
→ Reuses WHERE clause, changes SELECT to return actual rows
"""
```

| Previous Query | Current Request | LLM Decision |
|---------------|-----------------|--------------|
| COUNT HR resumes | "can you show me?" | **Follow-up**: Reuse WHERE, change SELECT |
| Python developers | "export to Excel" | **Follow-up**: Reuse WHERE, add Excel export |
| Python developers | "find Java devs" | **Fresh query**: Different topic |
| Show all resumes | "filter by 5+ years" | **Refine**: Add to existing WHERE |

---

## Context Management System

The chatbot maintains a **two-level context system** for intelligent conversation handling.

### Level 1: Conversation Context (Rolling Window)

Stores the last **4 conversation pairs** for continuity:

```python
# Stored per chat session
conversation_context = [
    {
        "user_msg": "How many Python developers?",
        "ai_response": "Found 127 Python developers in the database.",
        "sql": "SELECT COUNT(*) FROM resumes WHERE skills @> '[\"Python\"]'::jsonb",
        "result_summary": "Found 127 results"
    },
    {
        "user_msg": "Show me the top 5 by experience",
        "ai_response": "Here are the top 5 Python developers...",
        "sql": "SELECT * FROM resumes WHERE skills @> ... ORDER BY experience_years DESC LIMIT 5",
        "result_summary": "Found 5 results: John, Jane, Bob..."
    }
    # ... up to 4 pairs
]
```

### Level 2: Active Candidate (Single Focus)

Tracks the **currently discussed candidate** for pronoun resolution:

```python
# When a single candidate is returned or asked about
active_candidate = {
    "email": "john.doe@example.com",  # Unique identifier
    "name": "John Doe",
    "data": { ... full candidate record ... },
    "download_shown": False  # Prevents duplicate download links
}
```

### Context Update Rules

| Event | Conversation Context | Active Candidate |
|-------|---------------------|------------------|
| Any query | Store (user_msg, response, sql, summary) | — |
| Single result returned | Store | **SET** to that candidate |
| `person_detail` query | Store | **SET/REPLACE** |
| Multiple results returned | Store | **CLEAR** |
| New person mentioned by name | Store | **REPLACE** |

### Pronoun Resolution

```
User: "Find John Doe"
Bot: Found John Doe — 5 years experience, Python, React...
     [Sets active_candidate = John Doe]

User: "What certifications does he have?"
      ↓
      Detects pronoun "he" → resolves to active_candidate (John Doe)
      ↓
      Queries John's certifications from DB or PDF
```

---

## Token Optimization Strategies

Managing token usage is critical when making multiple LLM calls per query. Here's how I reduced costs by **~85%**:

### 1. Skills Limit (Top 500)

Instead of sending all 2000+ unique skills to the LLM, I fetch only the **top 500 by frequency**:

```python
async def get_distinct_values(self) -> Dict:
    # Fetch top 500 skills by frequency (covers 95%+ of queries)
    SELECT skill, COUNT(*) as freq 
    FROM resumes, jsonb_array_elements_text(skills) AS skill 
    GROUP BY skill 
    ORDER BY freq DESC 
    LIMIT 500
```

### 2. No Duplicate Schema

The database schema is embedded **once** in the system prompt — not repeated in user messages.

### 3. Context Truncation

```python
# AI responses truncated to 500 chars in context storage
ai_response = response[:500] if len(response) > 500 else response

# Only 4 conversation pairs retained (rolling window)
if len(context) > MAX_CONTEXT_PAIRS:
    context = context[-MAX_CONTEXT_PAIRS:]
```

### 4. Structured Data First, PDF Fallback

For `person_detail` queries, the system first tries to answer from **indexed database fields** (fast, no LLM call needed). Only if insufficient, it falls back to **reading the full resume PDF**:

```python
async def _process_person_detail(self, ...):
    # Step 1: Try structured data (experience, skills, summary columns)
    structured_result = await self.tools.answer_from_structured_data(
        user_question, candidate_data
    )
    
    if structured_result["status"] == "answered":
        return structured_result["response"]  # Fast path!
    
    # Step 2: Fallback to PDF parsing
    return await self.tools.answer_from_resume_pdf(email, name)
```

### Token Usage Per Query Type

| Query Type | LLM Calls | Approximate Tokens |
|------------|-----------|-------------------|
| `resume_query` | 3 (intent + sql + format) | ~2,000 |
| `person_detail` (from DB) | 2 (intent + answer) | ~1,500 |
| `person_detail` (from PDF) | 3 (intent + answer + pdf) | ~4,000 |
| `excel_export` | 4 (intent + columns + sql + exec) | ~2,500 |
| `conversation` | 2 (intent + respond) | ~1,000 |

---

## Resume Parsing Pipeline

When a new resume arrives, it goes through a multi-stage parsing pipeline:

### Extraction Flow

```
PDF/Word Document
        │
        ▼
┌───────────────────────────────────────┐
│          TEXT EXTRACTION              │
│  pypdf (PDF) / python-docx (Word)     │
│  Handles tables, multi-page docs      │
└───────────────────────────────────────┘
        │
        ▼ (text, max 15,000 chars)
┌───────────────────────────────────────┐
│        STRUCTURED EXTRACTION          │
│           OpenAI GPT-4o               │
│                                       │
│  Extracts:                            │
│  • name, email, phone                 │
│  • department (inferred from skills)  │
│  • experience_years (estimated)       │
│  • experience (full work history)     │
│  • skills (array)                     │
│  • project_metadata (array of objects)│
│  • specialization_keywords (domains)  │
│  • summary (AI-generated)             │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│         VALIDATION & STORAGE          │
│  • Skip if no name AND no email       │
│  • Deduplicate by email               │
│  • Store PDF binary + parsed fields   │
└───────────────────────────────────────┘
```

### Intelligent Field Extraction

The LLM not only extracts explicit information but also **infers implicit data**:

```python
# Example: Department inference from skills/experience
"""
Resume mentions: "Built microservices using Go and Kubernetes"
Skills extracted: ["Go", "Kubernetes", "Docker", "gRPC"]

→ LLM infers department: "Backend Engineering" or "DevOps"
"""

# Example: Specialization keywords extraction
"""
Resume mentions: "Developed trading algorithm for hedge fund"
                 "Built payment gateway integration"

→ specialization_keywords: ["Fintech", "Trading", "Payments"]
"""
```

---

## Data Ingestion at Scale

The system has ingested **20,000+ resumes** from 8 months of company email history.

### MS Graph API Integration

```python
class HistoricalEmailFetcher:
    """
    Fetches emails with resume attachments from Microsoft 365.
    Uses client credentials OAuth2 flow (daemon app).
    """
    
    async def fetch_emails_with_attachments(
        self,
        start_date: datetime,
        end_date: datetime
    ) -> AsyncGenerator[EmailWithAttachment, None]:
        # Filters: hasAttachments=true, receivedDateTime in range
        # Pagination: Handles 1000s of emails via @odata.nextLink
        ...
```

### Incremental Sync Strategy

To minimize API calls and OpenAI costs, the system uses **incremental sync**:

```python
# Instead of re-fetching all emails every time:
last_sync = await get_max_received_at()  # Most recent resume in DB
new_emails = await fetch_emails(since=last_sync)  # Only new ones

# Result: 85% reduction in API and LLM costs
```

| Sync Strategy | Monthly Emails Processed | OpenAI Calls | Cost |
|--------------|-------------------------|--------------|------|
| Full sync daily | 1,500 | 1,500 | High |
| Incremental | ~200 (new only) | 200 | **85% less** |

### Deduplication

Each resume is stored with a unique key (message_id + attachment_id) to prevent duplicates:

```python
# Check before processing
existing = await db.query(Resume).filter_by(source_email_id=unique_key).first()
if existing:
    logger.info("Skipping duplicate resume")
    continue
```

---

## API Design

The backend exposes a clean RESTful API built with FastAPI.

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/auth/login` | POST | Authenticate user |
| `/chats` | GET | List user's chat sessions |
| `/chats` | POST | Create new chat |
| `/chats/{id}/messages` | POST | **Send message → AI response** |
| `/resumes` | GET | List all resumes (paginated) |
| `/resumes/{id}` | GET | Get resume details |
| `/resumes/{id}/pdf` | GET | Download original PDF |
| `/exports/{filename}` | GET | Download generated Excel |
| `/sync/manual` | POST | Trigger email sync |

### Chat Message Flow

```python
@router.post("/{chat_id}/messages")
async def send_message(
    chat_id: int,
    request: ChatMessageRequest,
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user)
):
    # 1. Validate chat belongs to user
    chat = await get_chat_or_404(chat_id, user.id)
    
    # 2. Save user message
    user_message = Message(chat_id=chat_id, role="user", content=request.content)
    db.add(user_message)
    
    # 3. Process through Agentic SQL Engine
    agent = SQLAgent()
    ai_response = await agent.process_query(
        user_message=request.content,
        conversation_history=chat.messages,
        chat_id=chat_id
    )
    
    # 4. Save AI response
    assistant_message = Message(chat_id=chat_id, role="assistant", content=ai_response)
    db.add(assistant_message)
    await db.commit()
    
    return ChatMessageResponse(user=user_message, assistant=assistant_message)
```

### Async All The Way

The entire stack is async for maximum concurrency:

```python
# Database: SQLAlchemy 2.0 async
async with async_session() as db:
    result = await db.execute(select(Resume).where(...))

# HTTP Client: httpx async
async with httpx.AsyncClient() as client:
    response = await client.get(graph_api_url)

# OpenAI: AsyncOpenAI client
response = await openai_client.chat.completions.create(...)
```

---

## Database Design

### Resume Table Schema

```sql
CREATE TABLE resumes (
    id              SERIAL PRIMARY KEY,
    
    -- Original document (stored for download)
    pdf_data        BYTEA,
    pdf_filename    VARCHAR(500),
    
    -- Core extracted fields (indexed for search)
    name            VARCHAR(255),       -- INDEX
    email           VARCHAR(255),       -- INDEX (unique identifier)
    phone           VARCHAR(50),
    department      VARCHAR(255),       -- INDEX
    experience_years INTEGER,           -- INDEX
    
    -- Rich text content
    experience      TEXT,               -- Full work history
    summary         TEXT,               -- AI-generated summary
    
    -- JSONB for flexible arrays (using PostgreSQL-specific operators)
    skills          JSONB DEFAULT '[]', -- ["Python", "React", "AWS"]
    project_metadata JSONB DEFAULT '[]', -- [{title, keywords, description}]
    specialization_keywords JSONB DEFAULT '[]', -- ["Fintech", "Healthcare"]
    
    -- Source tracking
    source_email_id VARCHAR(500) UNIQUE, -- Deduplication key
    source_type     VARCHAR(50),         -- 'direct' or 'portal'
    sender_email    VARCHAR(255),
    email_subject   VARCHAR(500),
    
    -- Timestamps
    received_at     TIMESTAMP WITH TIME ZONE, -- When email was received
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### JSONB Query Patterns

PostgreSQL's JSONB operators enable efficient skill-based filtering:

```sql
-- Find candidates with Python skill (exact match)
SELECT * FROM resumes WHERE skills @> '["Python"]'::jsonb;

-- Find candidates with ANY of these skills
SELECT * FROM resumes WHERE skills ?| array['Python', 'Java', 'Go'];

-- Find candidates with ALL of these skills
SELECT * FROM resumes WHERE skills ?& array['Python', 'React'];

-- Text search within skills array (for fuzzy matching)
SELECT * FROM resumes WHERE skills::text ILIKE '%mobile%';
```

---

## Frontend Integration

The frontend is built with **Next.js + TypeScript** and provides a clean chat interface.

### Component Architecture

```
src/
├── app/
│   ├── page.tsx          # Login page
│   └── chat/page.tsx     # Main chat interface
├── components/
│   ├── ChatWindow.tsx    # Message display + input
│   ├── Sidebar.tsx       # Chat list navigation
│   ├── MessageBubble.tsx # Individual message rendering
│   └── TypingIndicator.tsx # Loading state
└── lib/
    ├── api.ts            # API client (fetch wrapper)
    └── store.ts          # Zustand state management
```

### Real-time Feel

While not WebSocket-based, the UI provides a responsive experience:
- Optimistic UI updates
- Typing indicator while waiting for AI
- Markdown rendering for formatted responses
- Clickable download links for Excel/PDF

---

## Technical Highlights

### What Makes This System Impressive

| Feature | Implementation | Why It Matters |
|---------|---------------|----------------|
| **Agentic Architecture** | Multi-step LLM pipeline with intent routing | Handles diverse queries without hardcoded rules |
| **Context Memory** | 2-level system (conversation + active candidate) | Natural follow-up conversations |
| **Self-Correcting SQL** | 3-retry loop with error feedback | Recovers from LLM mistakes automatically |
| **Token Optimization** | Skills limit, context truncation, structured-first | 85% cost reduction vs naive approach |
| **20K+ Scale** | Incremental sync, deduplication, JSONB indexing | Production-ready data volume |
| **Async Everything** | FastAPI + SQLAlchemy async + httpx | High concurrency, no blocking |
| **Security** | SELECT-only validation, binary data filtering | Safe SQL execution |

### Challenges Solved

1. **Pronoun Resolution**: "What about his experience?" → Resolves to active candidate
2. **Follow-up Detection**: LLM-based instead of brittle regex patterns
3. **Cost Control**: Token optimization kept OpenAI costs predictable
4. **Duplicate Prevention**: Unique keys from email message + attachment IDs
5. **Schema Evolution**: JSONB columns handle varying resume structures

### Production Considerations

- **Error Handling**: Graceful degradation with user-friendly messages
- **Logging**: Structured logging for debugging and monitoring
- **Validation**: Pydantic schemas for all API inputs/outputs
- **Security**: SQL injection prevention through parameterized queries

---

## Code Quality

- **Type Hints**: Full Python type annotations throughout
- **Async/Await**: Native async for all I/O operations
- **Separation of Concerns**: Agent → Tools → DB Manager → Prompts
- **Configuration**: Environment-based settings via Pydantic
- **Documentation**: Detailed docstrings and architecture docs

---

## Summary

This Resume Chatbot demonstrates my ability to:

✅ **Design complex AI systems** with agentic architectures  
✅ **Build production-grade APIs** with FastAPI and PostgreSQL  
✅ **Optimize LLM usage** for cost-effective deployment  
✅ **Handle scale** (20,000+ documents, complex queries)  
✅ **Integrate external services** (OpenAI, MS Graph API)  
✅ **Write clean, maintainable code** with proper separation of concerns  

---

*Built by me as an internal tool for HR automation. This document showcases the technical depth without exposing proprietary code.*
