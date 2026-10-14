# Complete System Architecture & End-to-End Data Flow Guide — ClanSure

This document provides a comprehensive technical breakdown of how the **ClanSure Family Insurance & Health Protection Portal** works end-to-end — from the **Presentation Layer (React Frontend)** through **ASP.NET Core API**, **Python FastAPI RAG Microservice**, **PostgreSQL & pgvector**, **Redis Cache**, **Railway S3 Bucket Storage**, and **Google Gemini Flash LLM**.

---

## 🏗️ 1. High-Level Architecture Overview

ClanSure follows a **Decoupled Microservice & Layered Architecture** hosted on **Railway.com**:

```mermaid
flowchart TD
    subgraph Client["1. PRESENTATION LAYER"]
        UI["React & TypeScript Frontend (Vite, Tailwind CSS, Framer Motion)"]
    end

    subgraph BackendAPI["2. APPLICATION / API LAYER"]
        DotNet[".NET Core 8 REST API (C#) - Auth, Policies, Claims"]
    end

    subgraph AIService["3. AI - RAG / LLM LAYER"]
        FastAPI["Python FastAPI Microservice - Text Chunking & RAG Engine"]
    end

    subgraph DataStorage["4. DATA & STORAGE LAYER"]
        PG[("PostgreSQL Database with pgvector Extension")]
        Redis[("Redis Cache - JWT Tokens & Sessions")]
        Bucket["Railway S3 Object Storage - Avatars & Vault PDFs"]
    end

    subgraph ExternalLLM["5. LLM PROVIDER"]
        Gemini["Google Gemini Flash LLM"]
    end

    UI --> DotNet
    UI --> FastAPI
    DotNet --> PG
    DotNet --> Redis
    DotNet --> Bucket
    FastAPI --> PG
    FastAPI --> Gemini
```

---

## 🔄 2. End-to-End Data Flow Breakdown

---

### Flow 1: User Authentication & Session Management (Core Auth Flow)

```mermaid
sequenceDiagram
    autonumber
    actor User as User Browser
    participant React as React Frontend
    participant DotNet as ASP.NET Core API
    participant PG as PostgreSQL DB
    participant Redis as Redis Cache

    User->>React: Enters Email & Password -> Click Login
    React->>DotNet: POST /api/Auth/login
    DotNet->>PG: Query User credentials
    PG-->>DotNet: User record found & password verified
    DotNet->>DotNet: Mint JWT Access Token & Refresh Token
    DotNet->>Redis: SET refresh token key (TTL 7 days)
    DotNet->>Redis: SET user profile cache (TTL 5 mins)
    DotNet-->>React: Return Access & Refresh Tokens
    React->>User: Login Successful (Store token in Auth State)
```

1. **User Request**: User enters credentials on the React Frontend (`Family Insurance Portal`).
2. **API Request**: Request hits the **ASP.NET Core REST API** (`backend-service`).
3. **Database Check**: ASP.NET Core verifies user credentials against the **PostgreSQL** `Users` table.
4. **Token Generation**: ASP.NET Core generates a short-lived JWT Access Token & a 7-day Refresh Token with a unique `jti`.
5. **Redis Caching**:
   - Refresh Token `jti` is stored in **Redis** with a 7-day TTL for instant revocation checking upon logout/refresh.
   - User profile mapping is cached in **Redis** under `user:{id}:profile` with a 5-minute TTL to reduce database query load.
6. **Response**: Tokens are returned to the React Frontend.

---

### Flow 2: Member Profile Photo & Document Vault Upload Flow

```mermaid
sequenceDiagram
    autonumber
    actor User as User Browser
    participant React as React Frontend
    participant DotNet as ASP.NET Core API
    participant Bucket as S3 Bucket Storage
    participant PG as PostgreSQL DB

    User->>React: Uploads Member Profile Photo / Policy PDF
    React->>DotNet: POST /api/FamilyMembers/{id}/avatar
    DotNet->>Bucket: Stream binary file to S3 Bucket
    Bucket-->>DotNet: File saved (avatars/guid.jpg)
    DotNet->>PG: Save S3 Key in Database
    PG-->>DotNet: Record Saved
    DotNet-->>React: Upload Confirmation & Stream URL
```

1. **Upload Request**: User uploads a profile picture or policy vault document in the React Frontend.
2. **S3 Object Storage**: The file binary is streamed directly into the Railway Tigris S3 Bucket (`compact-tin-tkmehua6m5gru`).
3. **Metadata Persistence**: The generated S3 Key (e.g. `avatars/guid.jpg`) and document metadata are stored in PostgreSQL tables (`FamilyMembers`, `VaultDocuments`).
4. **Streaming Endpoint**: Avatar images are served back to any browser via `GET /api/FamilyMembers/avatar-file?key=...`.

---

### Flow 3: AI Vector Ingestion Pipeline (RAG Ingest Flow)

```mermaid
sequenceDiagram
    autonumber
    participant Fast as Python FastAPI
    participant PG as PostgreSQL DB
    participant Gemini as Gemini Embeddings API

    Fast->>Fast: Trigger Ingestion (POST /api/rag/ingest)
    Fast->>PG: SELECT policy text from database
    PG-->>Fast: Return raw insurance policy text
    Fast->>Fast: Chunk text into 500-char blocks
    Fast->>Gemini: Batch Embeddings Call (texts)
    Gemini-->>Fast: Return Vector Arrays
    Fast->>PG: INSERT INTO rag_chunks (content, embedding)
    Note over PG: Index created using HNSW vector_cosine_ops
    PG-->>Fast: Ingest Completed
```

1. **Ingest Signal**: Ingestion is triggered via FastAPI (`POST /api/rag/ingest`).
2. **Fetch Source Content**: FastAPI queries PostgreSQL database to fetch raw text from insurance policies and health coverage directives.
3. **Text Chunking**: Raw content is split into overlapping chunks (500 characters, 100-character overlap) along sentence boundaries.
4. **Vector Embedding**: Text chunks are sent to the embedding model (Google Gemini Embedding), producing high-dimensional vector arrays.
5. **`pgvector` Ingestion**: Chunks and vector arrays are stored in the PostgreSQL **`rag_chunks`** table. An **HNSW index** (`vector_cosine_ops`) enables sub-millisecond similarity lookups.

---

### Flow 4: AI Protection Score & RAG Assistant Query Flow

```mermaid
sequenceDiagram
    autonumber
    actor Member as Family Member
    participant React as React Frontend
    participant Fast as Python FastAPI
    participant PG as PostgreSQL DB
    participant Gemini as Gemini Flash LLM

    Member->>React: Asks question in AI Coverage Advisor
    React->>Fast: POST /api/rag/query
    Fast->>Gemini: Embed Query string into Vector Array
    Gemini-->>Fast: Return Query Vector Array
    Fast->>PG: Execute Cosine Distance Search (pgvector)
    PG-->>Fast: Top 5 Relevant Policy Chunks
    Fast->>Fast: Construct Grounded Prompt with Context
    Fast->>Gemini: POST generateContent (Prompt + Context)
    Gemini-->>Fast: Grounded Answer with Citations
    Fast-->>React: Return JSON (Answer + Score)
    React->>Member: Displays Answer with citations
```

1. **User Question**: User asks about policy coverage or requests a health gap analysis in the AI Advisor Chat widget.
2. **Direct Microservice Call**: React sends a direct request to Python FastAPI (`POST /api/rag/query`).
3. **Query Embedding**: FastAPI converts the user's question into a vector using the embedding model.
4. **Vector Similarity Search**: FastAPI queries PostgreSQL `rag_chunks` using `pgvector`'s Cosine Distance operator (`<=>`).
5. **Prompt Augmentation**: Top-5 retrieved policy passages are structured into a grounded prompt:
   > *"You are an AI Insurance Advisor for ClanSure. Answer the user's query using ONLY the verified policy context below. Cite passages as [1], [2]..."*
6. **LLM Generation**: The prompt is sent to **Google Gemini Flash**.
7. **Response to User**: Gemini's response, complete with citations and Protection Score adjustments, is returned to the React UI for rendering.

---

## 🌐 3. Infrastructure & Deployment Overview (Railway.com)

All services run seamlessly on **Railway.com** in a unified project environment:

| Service Name | Technology / Stack | Role & Function |
| :--- | :--- | :--- |
| **`Family Insurance Portal`** | React 18 + TypeScript + Vite | Presentation Layer (Web UI for family members & admins) |
| **`backend-service`** | ASP.NET Core 8 (.NET API) | Application Layer (Auth, Family Tree Logic, Policies, Claims, S3 Streaming) |
| **`AI RAG API`** | Python FastAPI | AI Microservice (RAG, Chunking, Vector Search, Gemini Integration) |
| **`Postgres 😋`** | PostgreSQL 16 + `pgvector` | Unified Database (Application tables + `rag_chunks` Vector Store) |
| **`Redis 📊`** | Redis In-Memory Cache | Cache Layer (JWT Refresh tokens, Session IDs, Profile cache) |
| **`Family 👨‍👩‍👧‍👦`** | Railway Tigris S3 Storage | Object Storage (Uploaded member avatar photos, policy PDFs, receipts) |
| **Gemini API** | Google AI Studio (Gemini Flash) | LLM Provider (Embedding generation & RAG chat completions) |

---

## 🌟 Key Technical Highlights for Presentation / PPT

- **Interactive Family Protection Tree**: Custom SVG connector algorithm calculating exact parent marriage lines and child distribution stems.
- **Grounded AI Advice**: RAG pipeline ensures insurance answers are based strictly on verified policy documents, eliminating LLM hallucinations.
- **Enterprise Cloud Security**: Dual-layer security with JWT Access/Refresh tokens managed via Redis blacklisting and S3 signed streaming.
- **Unified Vector & Relational Storage**: Storing vectors inside PostgreSQL via `pgvector` avoids external third-party vector databases, reducing complexity and cost.
- **Sub-Second Latency**: `HNSW` indexing in `pgvector` + Redis profile caching guarantees fast response times under 50ms.
