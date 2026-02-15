# Design: BimaSahayak (Insurance Helper)

## Overview

BimaSahayak is an AI-powered mobile-first platform that acts as an insurance "Defense Lawyer" for India's missing middle — 500M+ citizens who struggle with English, complex jargon, and claim rejections. The system uses RAG, OCR, and multilingual NLP to shift the insurance experience from "helping you buy" to "helping you claim."

This document describes the technical architecture, data models, API contracts, and component designs required to implement the six core requirements.

---

## Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐    │
│  │ Android App  │  │   iOS App    │  │  Web App (PWA)     │    │
│  │ (React Native)│  │(React Native)│  │  (Next.js)         │    │
│  └──────┬───────┘  └──────┬───────┘  └────────┬───────────┘    │
│         └──────────────────┼──────────────────┘                  │
│                            │ HTTPS / WebSocket                   │
└────────────────────────────┼─────────────────────────────────────┘
                             │
┌────────────────────────────┼─────────────────────────────────────┐
│                     API GATEWAY (Kong/AWS API Gateway)            │
│            Rate Limiting · Auth · Request Routing                 │
└────────────────────────────┼─────────────────────────────────────┘
                             │
┌────────────────────────────┼─────────────────────────────────────┐
│                     BACKEND SERVICES (Microservices)              │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │  Auth        │  │  Policy       │  │  Document Triage     │    │
│  │  Service     │  │  Analyzer     │  │  Service             │    │
│  │  (FastAPI)   │  │  Service      │  │  (FastAPI)           │    │
│  └─────────────┘  │  (FastAPI)    │  └──────────────────────┘    │
│                    └──────────────┘                                │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │  Voice       │  │  Eligibility  │  │  Claim Assistance    │    │
│  │  Interface   │  │  Service      │  │  Service             │    │
│  │  Service     │  │  (FastAPI)    │  │  (FastAPI)           │    │
│  │  (FastAPI)   │  └──────────────┘  └──────────────────────┘    │
│  └─────────────┘                                                  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │              Notification Service (Celery + Redis)       │     │
│  └─────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────┼─────────────────────────────────────┐
│                        AI / ML LAYER                              │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │  RAG Engine  │  │  OCR Engine   │  │  Translation Layer   │    │
│  │  (LangChain  │  │  (Google      │  │  (Bhashini /         │    │
│  │   + Claude   │  │   Vision API  │  │   AI4Bharat models)  │    │
│  │   /GPT-4o)   │  │   / Tesseract)│  │                      │    │
│  └─────────────┘  └──────────────┘  └──────────────────────┘    │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐                               │
│  │  STT Engine  │  │  TTS Engine   │                               │
│  │  (Bhashini   │  │  (Bhashini    │                               │
│  │   ASR)       │  │   TTS)        │                               │
│  └─────────────┘  └──────────────┘                               │
└──────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────┼─────────────────────────────────────┐
│                        DATA LAYER                                 │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │  PostgreSQL  │  │  Redis        │  │  S3 / MinIO          │    │
│  │  (Primary DB)│  │  (Cache +     │  │  (Document Storage)  │    │
│  │             │  │   Sessions)   │  │                      │    │
│  └─────────────┘  └──────────────┘  └──────────────────────┘    │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐                               │
│  │  Pinecone /  │  │  Elasticsearch│                               │
│  │  ChromaDB    │  │  (Search &    │                               │
│  │  (Vector DB) │  │   Logging)    │                               │
│  └─────────────┘  └──────────────┘                               │
└──────────────────────────────────────────────────────────────────┘
```

### Technology Stack Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Mobile App | React Native | Cross-platform (Android + iOS) from single codebase; large ecosystem |
| Web App | Next.js (PWA) | SEO-friendly, offline-capable, responsive |
| API Gateway | Kong / AWS API Gateway | Rate limiting, auth, routing, analytics |
| Backend | Python FastAPI (microservices) | Async support, Python ML ecosystem, auto-docs (OpenAPI) |
| Task Queue | Celery + Redis | Async document processing, notifications |
| Primary DB | PostgreSQL | Relational data, JSONB for flexible schemas, mature |
| Cache | Redis | Session management, rate limiting, hot data |
| Vector DB | Pinecone / ChromaDB | RAG embeddings storage and similarity search |
| Object Storage | AWS S3 / MinIO | Encrypted document and policy PDF storage |
| Search | Elasticsearch | Full-text search across policies and documents |
| LLM | Claude API (Anthropic) / GPT-4o | RAG responses, policy analysis, red flag detection |
| Embeddings | text-embedding-3-small (OpenAI) / Cohere | Document chunking and vector embeddings |
| OCR | Google Cloud Vision API + Tesseract (fallback) | Printed + handwritten text extraction |
| STT/TTS | Bhashini (AI4Bharat) ASR + TTS | Native Indian language support, government-backed |
| Translation | Bhashini NMT / IndicTrans2 | High-quality Indian language translation |
| CI/CD | GitHub Actions + Docker + Kubernetes | Automated builds, containerized microservices |
| Monitoring | Prometheus + Grafana + Sentry | Metrics, dashboards, error tracking |

---

## Data Models

### Entity Relationship Diagram

```
┌──────────────┐       ┌───────────────────┐       ┌──────────────────┐
│    User       │1─────*│   Policy           │1─────*│  PolicyRedFlag    │
├──────────────┤       ├───────────────────┤       ├──────────────────┤
│ id (UUID)     │       │ id (UUID)          │       │ id (UUID)         │
│ phone         │       │ user_id (FK)       │       │ policy_id (FK)    │
│ name          │       │ insurer_name       │       │ category          │
│ language_pref │       │ policy_number      │       │ severity          │
│ state         │       │ policy_type        │       │ title             │
│ district      │       │ coverage_amount    │       │ description       │
│ created_at    │       │ premium            │       │ original_clause   │
│ tier (enum)   │       │ start_date         │       │ clause_location   │
└──────┬───────┘       │ expiry_date        │       │ created_at        │
       │                │ document_s3_key    │       └──────────────────┘
       │                │ status (enum)      │
       │                │ created_at         │
       │                └───────────────────┘
       │
       │1              ┌───────────────────┐       ┌──────────────────┐
       └──────────────*│  ClaimTriage       │1─────*│  TriageDocument   │
                       ├───────────────────┤       ├──────────────────┤
                       │ id (UUID)          │       │ id (UUID)         │
                       │ user_id (FK)       │       │ triage_id (FK)    │
                       │ policy_id (FK)     │       │ doc_type (enum)   │
                       │ status (enum)      │       │ image_s3_key      │
                       │ overall_result     │       │ ocr_text          │
                       │ created_at         │       │ validation_status │
                       └───────────────────┘       │ issues (JSONB)    │
                                                    │ created_at        │
       ┌───────────────────┐                        └──────────────────┘
       │ EligibilityCheck   │
       ├───────────────────┤       ┌──────────────────┐
       │ id (UUID)          │       │ VoiceSession      │
       │ user_id (FK)       │       ├──────────────────┤
       │ income_bracket     │       │ id (UUID)         │
       │ family_size        │       │ user_id (FK)      │
       │ state              │       │ policy_id (FK)    │
       │ district           │       │ language          │
       │ occupation         │       │ messages (JSONB[])│
       │ is_eligible        │       │ created_at        │
       │ scheme_suggested   │       └──────────────────┘
       │ nearest_csc (JSON) │
       │ created_at         │
       └───────────────────┘

       ┌───────────────────┐
       │ FamilyMember       │
       ├───────────────────┤
       │ id (UUID)          │
       │ user_id (FK)       │
       │ name               │
       │ relation           │
       │ date_of_birth      │
       │ created_at         │
       └───────────────────┘
```

### Key Enums

```
PolicyType:       HEALTH | LIFE | MOTOR | TRAVEL | HOME
PolicyStatus:     ACTIVE | EXPIRED | PENDING_RENEWAL
RedFlagSeverity:  HIGH | MEDIUM | LOW
RedFlagCategory:  WAITING_PERIOD | SUB_LIMIT | EXCLUSION | CO_PAYMENT | NETWORK_RESTRICTION | OTHER
TriageStatus:     IN_PROGRESS | COMPLETED | NEEDS_RESUBMISSION
DocType:          DISCHARGE_SUMMARY | BILL | PRESCRIPTION | DIAGNOSTIC_REPORT | CLAIM_FORM | ID_PROOF | OTHER
ValidationStatus: PASS | FAIL | WARNING
UserTier:         FREE | PREMIUM
```

---

## Component Design

### Component 1: Trap Detector (Policy Analyzer Service)

**Implements:** Requirement 1

#### Flow

```
User uploads PDF
       │
       ▼
┌──────────────────┐
│ PDF Ingestion     │ ── Extract text (PyMuPDF for digital, Google Vision for scanned)
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Text Chunking     │ ── Split into ~500 token chunks with overlap
│ + Embedding       │ ── Generate embeddings via text-embedding-3-small
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Vector Store      │ ── Store chunks in Pinecone/ChromaDB per policy_id
│ (per policy)      │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Red Flag RAG      │ ── For each risk category, query vector store with
│ Analysis          │    category-specific prompts:
│                   │    "Find clauses about waiting periods..."
│                   │    "Find clauses about room rent limits..."
│                   │    LLM extracts, classifies severity, and generates
│                   │    plain-language explanation
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Translation       │ ── Translate red flags to user's language via Bhashini
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Store & Return    │ ── Save PolicyRedFlag records, return grouped results
└──────────────────┘
```

#### Key Design Decisions

- **Chunking strategy:** 500 tokens with 50-token overlap to preserve clause boundaries. Headings/section titles prepended to each chunk for context.
- **Multi-pass analysis:** One RAG query per risk category (6 categories) rather than a single monolithic prompt. Increases recall and allows category-specific prompt engineering.
- **Severity classification:** LLM assigns severity based on financial impact heuristics (e.g., sub-limit < 50% of average room rent = HIGH).
- **Original clause linking:** Store chunk index and character offsets to highlight the exact clause in the PDF viewer.

#### API

```
POST /api/v1/policy/upload
  Headers: Authorization: Bearer <token>
  Body: multipart/form-data { file: <pdf>, language?: string }
  Response: { policy_id: string, status: "processing" }

GET /api/v1/policy/{policy_id}/red-flags
  Headers: Authorization: Bearer <token>
  Response: {
    policy_id: string,
    insurer: string,
    policy_number: string,
    red_flags: [
      {
        id: string,
        category: "WAITING_PERIOD" | "SUB_LIMIT" | ...,
        severity: "HIGH" | "MEDIUM" | "LOW",
        title: string,
        description: string,
        original_clause: string,
        clause_page: number,
        clause_location: { start: number, end: number }
      }
    ],
    summary: { high: number, medium: number, low: number },
    processed_at: datetime
  }

GET /api/v1/policy/{policy_id}/red-flags?lang=hi
  (Same response structure, title + description translated)
```

---

### Component 2: Pre-Claim Document Triage Service

**Implements:** Requirement 2

#### Flow

```
User captures/uploads document photos
       │
       ▼
┌──────────────────┐
│ Image Pre-        │ ── Deskew, contrast enhancement, noise reduction
│ Processing        │    (OpenCV / Pillow)
└────────┬─────────┘
         ▼
┌──────────────────┐
│ OCR Engine        │ ── Google Cloud Vision API (primary)
│                   │    Tesseract (fallback for offline/cost)
│                   │    Returns structured text + confidence scores
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Document          │ ── LLM classifies document type from OCR text:
│ Classification    │    DISCHARGE_SUMMARY | BILL | PRESCRIPTION | etc.
│                   │    Extracts key fields: patient_name, policy_number,
│                   │    hospital_name, dates, amounts
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Completeness      │ ── Check mandatory doc types present against
│ Check             │    claim_document_requirements table
│                   │    Flag missing documents with 🛑 alerts
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Consistency       │ ── Cross-verify extracted fields:
│ Validation        │    patient_name match across docs (fuzzy, >90%)
│                   │    policy_number exact match
│                   │    date ranges logical (admission ≤ discharge)
│                   │    hospital in empanelled network (DB lookup)
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Result            │ ── Generate pass/fail per document + overall status
│ Compilation       │    Generate actionable remediation steps for failures
└──────────────────┘
```

#### Key Design Decisions

- **Fuzzy name matching:** Uses Levenshtein distance with threshold of 90% similarity to handle OCR errors and transliteration differences.
- **Document classification:** LLM-based classification (not rule-based) to handle the wide variety of hospital document formats across India.
- **Empanelled network lookup:** Cached hospital network database per insurer, updated weekly. Fallback to "unable to verify" rather than false negatives.

#### API

```
POST /api/v1/triage/start
  Headers: Authorization: Bearer <token>
  Body: { policy_id: string }
  Response: { triage_id: string }

POST /api/v1/triage/{triage_id}/upload
  Body: multipart/form-data { file: <image>, doc_type_hint?: string }
  Response: {
    document_id: string,
    detected_type: string,
    ocr_confidence: number,
    extracted_fields: { patient_name, policy_number, ... }
  }

GET /api/v1/triage/{triage_id}/validate
  Response: {
    triage_id: string,
    overall_status: "PASS" | "FAIL" | "NEEDS_RESUBMISSION",
    documents: [
      {
        document_id: string,
        doc_type: string,
        status: "PASS" | "FAIL" | "WARNING",
        issues: [ { field: string, message: string, action: string } ]
      }
    ],
    missing_documents: [
      { doc_type: string, alert: string, where_to_get: string }
    ],
    consistency_checks: [
      { check: string, status: "PASS" | "FAIL", details: string }
    ]
  }

GET /api/v1/triage/{triage_id}/checklist.pdf
  Response: PDF binary (downloadable checklist)
```

---

### Component 3: Voice Interface Service

**Implements:** Requirement 3

#### Flow

```
User presses mic button and speaks
       │
       ▼
┌──────────────────┐
│ Audio Capture     │ ── Client-side: WebSocket stream or chunked upload
│ (Client)          │    Format: 16kHz, 16-bit, mono PCM/OGG
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Speech-to-Text    │ ── Bhashini ASR API (AI4Bharat)
│ (STT)             │    Input: audio + language_code
│                   │    Output: transcribed text + confidence
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Intent Detection  │ ── LLM classifies intent:
│ + Entity Extract  │    COVERAGE_CHECK | CLAIM_STATUS | HOSPITAL_SEARCH |
│                   │    PREMIUM_QUERY | GENERAL_QUESTION
│                   │    Extracts entities: treatment_name, hospital_name, etc.
└────────┬─────────┘
         ▼
┌──────────────────┐
│ RAG Query         │ ── Query user's policy vector store with the
│ (Policy Context)  │    transcribed question
│                   │    Retrieve relevant clauses
│                   │    LLM generates answer citing clauses
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Translation       │ ── If LLM response is in English, translate to
│ (if needed)       │    user's language via Bhashini NMT
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Text-to-Speech    │ ── Bhashini TTS API
│ (TTS)             │    Input: text + language_code
│                   │    Output: audio stream (MP3/OGG)
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Response          │ ── Return audio + text transcription + visual cues
│ Delivery          │    to client via WebSocket or HTTP
└──────────────────┘
```

#### Key Design Decisions

- **WebSocket for streaming:** Audio is streamed in real-time for low-latency feel. Fallback to chunked HTTP upload for unstable connections.
- **Session context:** Voice sessions maintain a sliding window of last 5 turns for multi-turn conversation. Stored in Redis with 30-minute TTL.
- **Visual cues:** Each response includes a `visual_cue` field with icon type (check, warning, info) and key data points for on-screen rendering alongside audio.
- **Bandwidth optimization:** Audio compressed to OGG Opus at 24kbps for 3G networks. Client-side VAD (Voice Activity Detection) to minimize upload size.

#### API

```
WebSocket: ws://api/v1/voice/stream
  Client sends: { type: "audio_chunk", data: <base64>, language: "hi" }
  Server sends: { type: "transcript", text: "...", is_final: boolean }
  Server sends: {
    type: "response",
    text: string,
    audio_url: string,
    visual_cue: { icon: string, key_data: string },
    cited_clauses: [ { clause_id, text, page } ],
    session_id: string
  }

POST /api/v1/voice/query (HTTP fallback)
  Body: multipart/form-data {
    audio: <file>,
    language: string,
    policy_id?: string,
    session_id?: string
  }
  Response: {
    transcript: string,
    response_text: string,
    audio_url: string,
    visual_cue: { icon: string, key_data: string },
    cited_clauses: [],
    session_id: string
  }
```

---

### Component 4: Ayushman Eligibility Service

**Implements:** Requirement 4

#### Flow

```
User fills eligibility form / answers voice questions
       │
       ▼
┌──────────────────┐
│ Input Collection  │ ── Collect: income, family_size, state, district,
│                   │    occupation, housing_type, social_category
└────────┬─────────┘
         ▼
┌──────────────────┐
│ SECC Deprivation  │ ── Apply 7 deprivation criteria (D1-D7):
│ Criteria Check    │    D1: One-room kutcha house
│                   │    D2: No adult member aged 16-59
│                   │    D3: Female-headed household, no male 16-59
│                   │    D4: Disabled member, no able-bodied adult
│                   │    D5: SC/ST households
│                   │    D6: No literate adult above 25
│                   │    D7: Landless, major income from manual labor
│                   │    + Automatic inclusion criteria
│                   │    + Automatic exclusion criteria
└────────┬─────────┘
         ▼
┌──────────────────┐
│ State Scheme      │ ── If not PM-JAY eligible, check state schemes:
│ Fallback Check    │    AP: Aarogyasri | TN: CMCHIS | KA: Arogya Karnataka
│                   │    MH: Mahatma Phule | RJ: Chiranjeevi | etc.
└────────┬─────────┘
         ▼
┌──────────────────┐
│ CSC Locator       │ ── Query CSC database by state + district
│                   │    Return top 3 nearest CSCs with:
│                   │    address, distance, hours, phone, Google Maps link
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Enrollment Guide  │ ── Generate personalized document checklist:
│ Generator         │    Aadhaar, Ration Card, Income Certificate, etc.
│                   │    Step-by-step enrollment instructions
│                   │    Estimated processing time
└──────────────────┘
```

#### Key Design Decisions

- **Rule-based eligibility:** Deterministic rules (not ML) for eligibility. Ensures 100% reproducibility and auditability. Rules stored in a versioned config table for easy updates when government criteria change.
- **No persistent storage of sensitive data:** Income, caste, and occupation data processed in-memory only. Only the eligibility result (boolean) and suggested scheme are persisted.
- **CSC data source:** Monthly bulk import from CSC SPV (Special Purpose Vehicle) public directory. Geocoded for distance calculations.

#### API

```
POST /api/v1/eligibility/check
  Body: {
    income_bracket: "below_1L" | "1L_to_2.5L" | "2.5L_to_5L" | "above_5L",
    family_size: number,
    state: string,
    district: string,
    occupation: string,
    housing_type: "kutcha" | "semi_pucca" | "pucca",
    social_category?: "SC" | "ST" | "OBC" | "General",
    has_adult_male_16_59?: boolean,
    has_literate_adult?: boolean,
    has_land?: boolean
  }
  Response: {
    pmjay_eligible: boolean,
    pmjay_reason: string,
    deprivation_criteria_met: string[],
    alternative_schemes: [
      { scheme_name: string, description: string, coverage: string }
    ],
    nearest_cscs: [
      {
        name: string,
        address: string,
        distance_km: number,
        hours: string,
        phone: string,
        maps_url: string
      }
    ],
    enrollment_guide: {
      required_documents: string[],
      steps: string[],
      estimated_days: number
    }
  }

POST /api/v1/eligibility/reminder
  Body: { user_id: string, csc_id: string, reminder_date: date }
  Response: { reminder_id: string, status: "scheduled" }
```

---

### Component 5: Policy Document Management

**Implements:** Requirement 5

#### Design

- **Storage:** Policy PDFs stored in S3 with server-side AES-256 encryption. Each file keyed as `users/{user_id}/policies/{policy_id}/{filename}`.
- **Metadata extraction:** Reuses the PDF text extraction from Component 1. A secondary LLM pass extracts structured metadata (insurer, policy number, coverage, premium, dates) from the first 2 pages.
- **Notifications:** Celery beat scheduler checks for policies expiring within 30 days and 7 days nightly. Sends SMS (via Twilio/MSG91) and push notifications (Firebase Cloud Messaging).
- **Family linking:** `FamilyMember` records linked to `User`. Policies can be tagged with one or more `family_member_id`.
- **Offline sync:** SQLite on-device cache for policy metadata and downloaded PDFs. Synced via background job when connectivity returns.

#### API

```
GET    /api/v1/policies                     ── List all user policies (with filters)
GET    /api/v1/policies/{id}                ── Get policy details + metadata
DELETE /api/v1/policies/{id}                ── Remove policy
GET    /api/v1/policies/{id}/download       ── Download original PDF
PUT    /api/v1/policies/{id}/family-members ── Link family members to policy
GET    /api/v1/family-members               ── List family members
POST   /api/v1/family-members               ── Add family member
```

---

### Component 6: Claim Filing Assistance (Premium)

**Implements:** Requirement 6

#### Design

- **Guided workflow:** Multi-step form wizard on the client. Each step validated server-side before proceeding. Steps: Personal Details → Hospitalization Details → Document Upload → Review → Submit.
- **Claim amount estimator:** Rule engine that applies policy sub-limits, co-payments, and deductibles to the billed amount. Uses extracted policy terms from Component 1.
- **Communication templates:** Pre-built templates stored in DB, personalized with user/policy/claim details via templating engine. Categories: Follow-up, Escalation, Grievance, Appeal.
- **Appeal assistance:** For rejected claims, LLM analyzes rejection letter (uploaded as image/PDF) against policy terms and generates counter-arguments citing specific policy clauses.

---

## Cross-Cutting Concerns

### Authentication & Authorization

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│ Phone + OTP  │────▶│ Auth Service  │────▶│ JWT Token     │
│ (MSG91/      │     │ (FastAPI)     │     │ (Access +     │
│  Firebase)   │     │               │     │  Refresh)     │
└─────────────┘     └──────────────┘     └──────────────┘
```

- **OTP-based login:** Phone number + 6-digit OTP (via MSG91 or Firebase Auth). No password required — matches target user behavior.
- **JWT tokens:** Access token (15 min TTL) + Refresh token (30 day TTL). Stored in secure HTTP-only cookies (web) or secure storage (mobile).
- **Tier enforcement:** Middleware checks `user.tier` against endpoint requirements. Free-tier endpoints are rate-limited (e.g., 2 policy uploads/month).

### Translation Pipeline

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Source Text   │────▶│ Bhashini NMT │────▶│ Translated    │
│ (English)     │     │ (IndicTrans2) │     │ Text          │
└──────────────┘     └──────────────┘     └──────────────┘
                              │
                     ┌────────▼────────┐
                     │ Translation      │
                     │ Cache (Redis)    │ ── Cache common phrases; TTL 7 days
                     └─────────────────┘
```

- All user-facing AI-generated text passes through the translation pipeline when `language != "en"`.
- Common red flag descriptions and UI strings are pre-translated and cached.
- Domain-specific insurance terms have a curated glossary per language to ensure accuracy.

### Error Handling Strategy

| Error Type | Handling |
|-----------|----------|
| OCR low confidence (<60%) | Return result with warning; suggest re-upload with better lighting |
| LLM timeout/failure | Return cached/partial results; queue for retry; show "processing" state |
| Translation failure | Fall back to English with "Translation unavailable" notice |
| STT failure | Show "Could not understand" prompt; suggest text input fallback |
| Document upload failure | Client-side retry (3 attempts); show offline queue option |
| Rate limit exceeded | Return 429 with upgrade prompt for free-tier users |

### Monitoring & Observability

| Tool | Purpose |
|------|---------|
| Prometheus + Grafana | System metrics (latency, throughput, error rates per service) |
| Sentry | Application error tracking and alerting |
| Elasticsearch + Kibana | Centralized logging, request tracing |
| OpenTelemetry | Distributed tracing across microservices |
| Custom dashboards | Business metrics: uploads/day, red flags detected, eligibility checks, claim triage pass rate |

---

## Security Design

### Data Encryption

| Data State | Method |
|-----------|--------|
| At rest (S3) | AES-256 server-side encryption (SSE-S3) |
| At rest (PostgreSQL) | Column-level encryption for PII (pgcrypto) |
| In transit | TLS 1.3 for all API communication |
| Client storage | Android Keystore / iOS Keychain for tokens; encrypted SQLite for offline data |

### Data Privacy

- **Minimal data collection:** Only collect what's needed for each feature.
- **Eligibility data:** Processed in-memory; not persisted unless user opts in.
- **Document retention:** User-uploaded documents retained while account is active. Deleted within 30 days of account deletion.
- **Audit logging:** All data access logged with user_id, action, timestamp. Logs retained for 1 year.
- **DPDPA compliance:** Consent management for data collection, right to erasure, data portability export.

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────┐
│                 AWS Cloud (ap-south-1, Mumbai)    │
│                                                   │
│  ┌──────────────┐     ┌──────────────────────┐   │
│  │ Route 53      │────▶│ CloudFront CDN        │   │
│  │ (DNS)         │     │ (Static assets + API) │   │
│  └──────────────┘     └──────────┬───────────┘   │
│                                   │                │
│  ┌──────────────────────────────────────────────┐│
│  │  EKS (Kubernetes)                             ││
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌───────┐ ││
│  │  │Auth    │ │Policy  │ │Triage  │ │Voice  │ ││
│  │  │Service │ │Analyzer│ │Service │ │Service│ ││
│  │  │(2 pods)│ │(4 pods)│ │(3 pods)│ │(3 pods)│ ││
│  │  └────────┘ └────────┘ └────────┘ └───────┘ ││
│  │  ┌────────┐ ┌────────┐ ┌────────────────┐   ││
│  │  │Elig.   │ │Claim   │ │Notification    │   ││
│  │  │Service │ │Service │ │Service (2 pods)│   ││
│  │  │(2 pods)│ │(2 pods)│ └────────────────┘   ││
│  │  └────────┘ └────────┘                        ││
│  └──────────────────────────────────────────────┘│
│                                                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ RDS       │ │ ElastiCa │ │ S3               │ │
│  │ PostgreSQL│ │ che Redis│ │ (Documents)      │ │
│  │ (Multi-AZ)│ │ (Cluster)│ │                  │ │
│  └──────────┘ └──────────┘ └──────────────────┘ │
└─────────────────────────────────────────────────┘
```

- **Region:** ap-south-1 (Mumbai) for low latency to Indian users.
- **Multi-AZ:** PostgreSQL and Redis deployed across 2 availability zones.
- **Auto-scaling:** HPA (Horizontal Pod Autoscaler) on CPU/memory. Policy Analyzer scales up during peak hours (evenings/weekends).
- **Blue-green deployments:** Zero-downtime deploys via Kubernetes rolling updates.

---

## MVP Scope (Phase 1 — 12 Weeks)

| Week | Deliverable |
|------|-------------|
| 1–2 | Project setup: CI/CD, DB schema, auth service, S3 bucket, base API framework |
| 3–4 | Trap Detector: PDF ingestion, RAG pipeline, red flag detection, basic UI |
| 5–6 | Pre-Claim Triage: Camera capture, OCR pipeline, document classification, validation engine |
| 7–8 | Voice Interface: Bhashini STT/TTS integration, intent detection, conversational RAG |
| 9–10 | Ayushman Eligibility: Rule engine, CSC locator, enrollment guide generator |
| 11 | Policy Management: Storage, metadata extraction, notifications |
| 12 | Integration testing, performance tuning, security audit, beta launch |

---

**Document Version:** 1.0
**Last Updated:** February 15, 2026
