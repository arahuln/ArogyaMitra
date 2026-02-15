# Design: ArogyaMitra (Health Companion)

## Overview

ArogyaMitra is a voice-first, AI-powered multilingual health assistant for India. It solves three critical problems:

1. **Language barrier:** Health information is in English with complex medical jargon — ArogyaMitra's AI bot is the primary interface, letting users manage their entire health via voice in 12+ Indian languages. No English literacy required.
2. **Complex chronic disease tracking:** Patients with diabetes, hypertension, thyroid, and multi-condition combos have 5-8 medications with non-standard schedules (alternate days, every 3 days, tapering doses, cyclical patterns). Normal reminder apps can't handle this — ArogyaMitra's smart scheduler can.
3. **Scattered medical records:** Prescriptions in drawers, lab reports on WhatsApp, discharge summaries lost. When visiting a new doctor, the patient can't recall their history. ArogyaMitra centralizes everything into a single, doctor-ready health portfolio accessible via QR code.

The platform uses RAG for medical Q&A, Bhashini (AI4Bharat) for multilingual voice/text, and OCR for prescription digitization. It focuses on preventive health and informed doctor visits rather than diagnosis.

This document describes the technical architecture, data models, API contracts, and component designs required to implement the core requirements.

---

## Architecture

### High-Level System Architecture

```
+------------------------------------------------------------------+
|                         CLIENT LAYER                              |
|  +----------------+  +----------------+  +--------------------+  |
|  |  Android App   |  |   iOS App      |  |  Web App (PWA)     |  |
|  |  (React Native)|  | (React Native) |  |  (Next.js)         |  |
|  +-------+--------+  +-------+--------+  +---------+----------+  |
|          +--------------------+-----------------------+           |
|                               | HTTPS / WebSocket                |
+-------------------------------+----------------------------------+
                                |
+-------------------------------+----------------------------------+
|                    API GATEWAY (AWS API Gateway)                  |
|           Rate Limiting - Auth - Request Routing                  |
+-------------------------------+----------------------------------+
                                |
+-------------------------------+----------------------------------+
|                    BACKEND SERVICES (Microservices)               |
|                                                                   |
|  +--------------+  +-----------------+  +---------------------+  |
|  |  Auth         |  |  Onboarding     |  |  Health Tracking    |  |
|  |  Service      |  |  Service        |  |  Service            |  |
|  |  (FastAPI)    |  |  (FastAPI)      |  |  (FastAPI)          |  |
|  +--------------+  +-----------------+  +---------------------+  |
|                                                                   |
|  +--------------+  +-----------------+  +---------------------+  |
|  |  AI Assistant |  |  Report         |  |  Notification       |  |
|  |  Service      |  |  Service        |  |  Service            |  |
|  |  (FastAPI)    |  |  (FastAPI)      |  |  (Celery + Redis)   |  |
|  +--------------+  +-----------------+  +---------------------+  |
|                                                                   |
|  +--------------+                                                 |
|  |  Doctor Rec.  |                                                |
|  |  Service      |                                                |
|  |  (FastAPI)    |                                                |
|  +--------------+                                                 |
+------------------------------------------------------------------+
                                |
+-------------------------------+----------------------------------+
|                         AI / ML LAYER                             |
|                                                                   |
|  +--------------+  +-----------------+  +---------------------+  |
|  |  RAG Engine   |  |  OCR Engine     |  |  Translation Layer  |  |
|  |  (LangChain   |  |  (Google Vision |  |  (Bhashini /        |  |
|  |   + Claude)   |  |   / Tesseract)  |  |   AI4Bharat models) |  |
|  +--------------+  +-----------------+  +---------------------+  |
|                                                                   |
|  +--------------+  +-----------------+                            |
|  |  STT Engine   |  |  TTS Engine     |                           |
|  |  (Bhashini    |  |  (Bhashini      |                           |
|  |   ASR)        |  |   TTS)          |                           |
|  +--------------+  +-----------------+                            |
+------------------------------------------------------------------+
                                |
+-------------------------------+----------------------------------+
|                         DATA LAYER                                |
|                                                                   |
|  +--------------+  +-----------------+  +---------------------+  |
|  |  PostgreSQL   |  |  Redis          |  |  S3 / MinIO         |  |
|  |  (Primary DB) |  |  (Cache +       |  |  (Prescriptions &   |  |
|  |               |  |   Sessions +    |  |   Report PDFs)      |  |
|  |               |  |   Task Queue)   |  |                     |  |
|  +--------------+  +-----------------+  +---------------------+  |
|                                                                   |
|  +--------------+  +-----------------+                            |
|  |  Pinecone /   |  |  Elasticsearch  |                           |
|  |  ChromaDB     |  |  (Search &      |                           |
|  |  (Vector DB)  |  |   Logging)      |                           |
|  +--------------+  +-----------------+                            |
+------------------------------------------------------------------+
```

### Technology Stack Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Mobile App | React Native | Cross-platform (Android + iOS) from single codebase |
| Web App | Next.js (PWA) | SEO-friendly, offline-capable, responsive |
| API Gateway | AWS API Gateway | Rate limiting, auth, routing, analytics |
| Backend | Python FastAPI (microservices) | Async support, Python ML ecosystem, auto-docs (OpenAPI) |
| Task Queue | Celery + Redis | Async report generation, scheduled notifications |
| Primary DB | PostgreSQL | Relational data, JSONB for flexible schemas |
| Cache | Redis | Session management, rate limiting, hot data |
| Vector DB | Pinecone / ChromaDB | RAG embeddings for medical knowledge base |
| Object Storage | AWS S3 | Encrypted prescription images and report PDFs |
| Search | Elasticsearch | Full-text search across health records and articles |
| LLM | Claude API (Anthropic) | RAG responses, symptom analysis, doctor recommendations |
| Embeddings | text-embedding-3-small (OpenAI) | Medical knowledge chunking and vector embeddings |
| OCR | Google Cloud Vision API + Tesseract (fallback) | Prescription text extraction |
| STT/TTS | Bhashini (AI4Bharat) ASR + TTS | Native Indian language voice support |
| Translation | Bhashini NMT / IndicTrans2 | Indian language translation |
| Push Notifications | Firebase Cloud Messaging (FCM) | Cross-platform push notifications |
| SMS | MSG91 | OTP and critical health alerts |
| CI/CD | GitHub Actions + Docker + Kubernetes | Automated builds, containerized microservices |
| Monitoring | Prometheus + Grafana + Sentry | Metrics, dashboards, error tracking |

---

## Data Models

### Entity Relationship Diagram

```
+------------------+        +----------------------+
|    User           |1------*|   HealthProfile       |
+------------------+        +----------------------+
| id (UUID)         |        | id (UUID)             |
| phone             |        | user_id (FK)          |
| name              |        | age                   |
| language_pref     |        | gender                |
| state             |        | height_cm             |
| district          |        | weight_kg             |
| created_at        |        | bmi                   |
| tier (enum)       |        | blood_group           |
+--------+---------+        | allergies (TEXT[])     |
         |                   | chronic_conditions[]   |
         |                   | dietary_pref (enum)    |
         |                   | activity_level (enum)  |
         |                   | smoking (boolean)      |
         |                   | alcohol (boolean)      |
         |                   | updated_at             |
         |                   +----------------------+
         |
         |1       +----------------------+
         +-------*|   CaregiverLink       |
         |        +----------------------+
         |        | id (UUID)             |
         |        | patient_id (FK)       |  -- the patient being monitored
         |        | caregiver_phone       |  -- caregiver's phone number
         |        | caregiver_name        |
         |        | alert_on_missed_dose  |  -- boolean
         |        | alert_on_critical     |  -- boolean (critical vital readings)
         |        | alert_method (enum)   |  -- PUSH | SMS | BOTH
         |        | quiet_hours_start     |  -- e.g., 22:00 (no alerts)
         |        | quiet_hours_end       |  -- e.g., 07:00
         |        | consent_given_at      |  -- patient's explicit consent timestamp
         |        | is_active (boolean)   |
         |        +----------------------+
         |
         |1       +----------------------+
         +-------*|   VitalLog            |
         |        +----------------------+
         |        | id (UUID)             |
         |        | user_id (FK)          |
         |        | type (enum)           |
         |        | value (DECIMAL)       |
         |        | value_secondary       |  -- for BP: diastolic
         |        | context (enum)        |  -- FASTING | POST_MEAL | RESTING | etc.
         |        | unit (string)         |
         |        | input_method (enum)   |  -- VOICE | MANUAL | DEVICE
         |        | recorded_at           |
         |        | notes (TEXT)          |
         |        +----------------------+
         |
         |1       +----------------------+        +------------------------+
         +-------*|   Medication          |1------*|  MedicationSchedule     |
         |        +----------------------+        +------------------------+
         |        | id (UUID)             |        | id (UUID)               |
         |        | user_id (FK)          |        | medication_id (FK)      |
         |        | name                  |        | schedule_type (enum)    |
         |        | dosage                |        | schedule_config (JSONB) |
         |        | food_instruction(enum)|        |   -- times, days, taper |
         |        | prescribed_by         |        |      rule, cycle rule,  |
         |        | start_date            |        |      split doses        |
         |        | end_date              |        | is_active (boolean)     |
         |        | course_days           |        +------------------------+
         |        | is_active (boolean)   |
         |        | prescription_s3_key   |        +------------------------+
         |        | created_at            |        |  DoseLog                |
         |        +----------------------+        +------------------------+
         |                                         | id (UUID)               |
         |                                         | medication_id (FK)      |
         |                                         | scheduled_time          |
         |                                         | scheduled_dose          |
         |                                         | status (enum)           |
         |                                         |   TAKEN|SKIPPED|MISSED  |
         |                                         | confirmed_via (enum)    |
         |                                         |   VOICE|TAP|AUTO_MISSED |
         |                                         | confirmed_at            |
         |                                         | caregiver_alerted       |
         |                                         +------------------------+
         |
         |1       +----------------------+
         +-------*|   SymptomLog          |
         |        +----------------------+
         |        | id (UUID)             |
         |        | user_id (FK)          |
         |        | symptoms (JSONB)      |
         |        | follow_up_qa (JSONB[])| -- [{q: "...", a: "..."}]
         |        | severity (enum)       |
         |        | ai_specialty_rec      |
         |        | ai_alternatives[]     |
         |        | ai_urgency (enum)     |
         |        | ai_home_care (TEXT)    |
         |        | is_emergency (boolean)|
         |        | logged_at             |
         |        +----------------------+
         |
         |1       +----------------------+
         +-------*|   WeeklyReport        |
         |        +----------------------+
         |        | id (UUID)             |
         |        | user_id (FK)          |
         |        | week_start (DATE)     |
         |        | week_end (DATE)       |
         |        | health_score (INT)    |
         |        | vital_summary (JSONB) |
         |        | med_adherence_pct     |
         |        | symptom_summary(JSONB)|
         |        | action_items (JSONB)  |
         |        | report_s3_key         |
         |        | generated_at          |
         |        +----------------------+
         |
         |1       +----------------------+
         +-------*|   ChatSession         |
         |        +----------------------+
         |        | id (UUID)             |
         |        | user_id (FK)          |
         |        | language              |
         |        | messages (JSONB[])    |
         |        | created_at            |
         |        +----------------------+
         |
         |1       +----------------------+
         +-------*|   MedicalEvent        |
                  +----------------------+
                  | id (UUID)             |
                  | user_id (FK)          |
                  | type (enum)           |
                  | title                 |
                  | description           |
                  | doctor_name           |
                  | hospital_name         |
                  | event_date            |
                  | documents_s3_keys[]   |
                  | created_at            |
                  +----------------------+

+----------------------+
|   HealthTip           |
+----------------------+
| id (UUID)             |
| category (enum)       |
| title                 |
| content (TEXT)        |
| target_conditions[]   |
| target_age_range      |
| target_diet (enum)    |
| target_activity(enum) |
| season (enum)         |
| region (TEXT[])       |
| language_versions{}   |
| is_active (boolean)   |
| created_at            |
+----------------------+
```

### Key Enums

```
VitalType:          BLOOD_PRESSURE | BLOOD_SUGAR | TEMPERATURE | WEIGHT | HEART_RATE | SPO2
VitalContext:       FASTING | POST_MEAL | RANDOM | RESTING | POST_EXERCISE
ScheduleType:       DAILY | ALTERNATE_DAY | EVERY_N_DAYS | SPECIFIC_DAYS | TAPERING | CYCLICAL | SPLIT_DOSE | FOR_X_DAYS
FoodInstruction:    BEFORE_FOOD | AFTER_FOOD | EMPTY_STOMACH | WITH_FOOD | BEDTIME | ANY
DoseStatus:         TAKEN | SKIPPED | MISSED
ConfirmedVia:       VOICE | TAP | AUTO_MISSED
AlertMethod:        PUSH | SMS | BOTH
Severity:           MILD | MODERATE | SEVERE
Urgency:            ROUTINE | SOON | URGENT | EMERGENCY
DietaryPref:        VEG | NON_VEG | VEGAN | EGGETARIAN
ActivityLevel:      SEDENTARY | LIGHT | MODERATE | ACTIVE
InputMethod:        VOICE | MANUAL | DEVICE
MedicalEventType:   SURGERY | HOSPITALIZATION | LAB_TEST | VACCINATION | OTHER
DocumentType:       PRESCRIPTION | LAB_REPORT | DISCHARGE_SUMMARY | SCAN_REPORT | XRAY | VACCINATION | INSURANCE | OTHER
TipCategory:        FOOD | EXERCISE | HYDRATION | SLEEP | SEASONAL | MENTAL_WELLNESS
Season:             SUMMER | MONSOON | WINTER | ALL_YEAR
UserTier:           FREE | PREMIUM
```

---

## Component Design

### Component 1: Voice-First Multilingual AI Bot (Core Interface)

**Implements:** Requirement 1

The AI bot is NOT just a Q&A feature — it is the **primary interface** for the entire app. Users can do everything via voice: log vitals ("mera sugar aaj 140 hai"), set reminders ("dawai ka reminder lagao alternate day"), ask health questions, and get reports read aloud. This removes the English literacy barrier entirely.

#### Flow

```
User speaks into mic or types (voice is primary)
       |
       v
+--------------------+
| Input Processing    | -- Voice: Bhashini ASR -> transcribed text
|                     | -- Handles code-switching (Hindi-English mix:
|                     |    "mera BP check karo")
|                     | -- Text: pass through
+----------+---------+
           v
+--------------------+
| Language Detection  | -- Auto-detect input language (or use user pref)
| + Translation       | -- Translate to English for LLM processing
+----------+---------+
           v
+--------------------+
| Intent Detection    | -- LLM classifies intent:
| + Action Router     |    SYMPTOM_QUERY | MEDICATION_QUERY | DIET_QUERY |
|                     |    LOG_VITAL | SET_REMINDER | CHECK_REPORT |
|                     |    EXERCISE_QUERY | GENERAL_HEALTH | EMERGENCY
|                     | -- Action intents (LOG_VITAL, SET_REMINDER) route
|                     |    directly to the relevant service API
+----------+---------+
           v
+--------------------+
| RAG Query           | -- Query medical knowledge vector store
| (Medical KB)        | -- Sources: WHO guidelines, ICMR, curated medical DB
|                     | -- Retrieve relevant passages
|                     | -- LLM generates response citing sources
|                     | -- Explains medical terms in everyday language
|                     |    (e.g., "BP high hai" not "hypertensive readings")
+----------+---------+
           v
+--------------------+
| Personalization     | -- Inject user's health profile context
|                     | -- Tailor response to conditions, age, medications
|                     | -- A diabetic asking about diet gets diabetes-specific advice
+----------+---------+
           v
+--------------------+
| Safety Filter       | -- Add disclaimers
|                     | -- Detect emergency symptoms -> show helpline
|                     | -- Block diagnosis/prescription language
+----------+---------+
           v
+--------------------+
| Translation + TTS   | -- Translate response to user's language
|                     | -- Generate audio via Bhashini TTS
|                     | -- Audio is the PRIMARY response, text is secondary
+--------------------+
```

#### Key Design Decisions

- **Voice as primary, not secondary:** Unlike most health apps that bolt on voice as an afterthought, voice is the default input/output. Text UI exists as a fallback. Every feature has a voice command equivalent.
- **Action routing:** The bot isn't just for questions — it routes action intents to backend services. "Mera sugar 140 hai" triggers the vital logging API. "Kal ki dawai skip kar di" updates medication adherence. This makes the bot a universal controller for the entire app.
- **Code-switching support:** Indian users naturally mix Hindi-English (Hinglish) or regional-English. The STT and intent detection handle mixed-language input gracefully.
- **Medical jargon translation:** LLM prompt includes instructions to always explain medical terms in 8th-grade equivalent local language. No jargon in user-facing responses.
- **Medical knowledge base:** Curated corpus from WHO, ICMR, AIIMS patient education materials, and government health scheme documents. NOT sourced from unverified internet content.
- **Safety-first responses:** Every health response passes through a safety filter that blocks diagnostic language, adds disclaimers, and escalates emergency symptoms.
- **Session context:** Chat sessions maintain a sliding window of last 8 turns stored in Redis (30-minute TTL) for multi-turn conversations.

#### API

```
WebSocket: ws://api/v1/assistant/stream
  Client sends: { type: "text", text: "...", language: "hi" }
  Client sends: { type: "audio_chunk", data: <base64>, language: "hi" }
  Server sends: { type: "transcript", text: "...", is_final: boolean }
  Server sends: {
    type: "response",
    text: string,
    audio_url: string,
    disclaimer: string,
    sources: [ { title, url } ],
    is_emergency: boolean,
    emergency_numbers?: [ { label, number } ],
    session_id: string
  }

POST /api/v1/assistant/query (HTTP fallback)
  Body: {
    text?: string,
    audio?: <file>,
    language: string,
    session_id?: string
  }
  Response: {
    transcript?: string,
    response_text: string,
    audio_url: string,
    disclaimer: string,
    sources: [],
    is_emergency: boolean,
    session_id: string
  }
```

---

### Component 2: Health Onboarding & Symptom Profiling Service

**Implements:** Requirement 2

#### Flow

```
New user signs up (Phone + OTP)
       |
       v
+--------------------+
| Step 1: Basics      | -- Age, gender, height, weight -> auto-calculate BMI
+----------+---------+
           v
+--------------------+
| Step 2: Medical     | -- Blood group, known allergies, chronic conditions
| History             |    (multi-select from common list + free text)
+----------+---------+
           v
+--------------------+
| Step 3: Medications | -- Current medications (name, dosage, frequency)
|                     | -- Option: photograph prescription -> OCR extract
+----------+---------+
           v
+--------------------+
| Step 4: Lifestyle   | -- Dietary preference, activity level,
|                     |    smoking/alcohol habits
+----------+---------+
           v
+--------------------+
| Step 5: Current     | -- "What's bothering you right now?" (optional)
| Symptoms            | -- Free text or guided body-map picker
+----------+---------+
           v
+--------------------+
| Profile Compilation | -- Store HealthProfile record
|                     | -- Calculate initial health score
|                     | -- Configure initial notification preferences
+--------------------+
```

#### Key Design Decisions

- **Progressive onboarding:** Steps 1-2 are required (< 2 minutes). Steps 3-5 can be skipped and completed later. This reduces drop-off.
- **Voice-enabled onboarding:** Every step can be completed via voice. The AI assistant guides the user through questions conversationally.
- **OCR for prescriptions:** Users can photograph current prescriptions instead of manually typing medication names. OCR extracts drug names and dosages.
- **Monthly check-ins:** Celery beat scheduler triggers a monthly push notification asking users to review/update their profile.

#### API

```
POST /api/v1/onboarding/profile
  Body: {
    age: number,
    gender: "male" | "female" | "other",
    height_cm: number,
    weight_kg: number,
    blood_group?: string,
    allergies?: string[],
    chronic_conditions?: string[],
    dietary_pref: "VEG" | "NON_VEG" | "VEGAN" | "EGGETARIAN",
    activity_level: "SEDENTARY" | "LIGHT" | "MODERATE" | "ACTIVE",
    smoking?: boolean,
    alcohol?: boolean
  }
  Response: { profile_id: string, bmi: number, health_score: number }

PUT /api/v1/onboarding/profile
  Body: { ...partial fields to update... }
  Response: { profile_id: string, updated_fields: string[] }

POST /api/v1/onboarding/symptoms
  Body: {
    symptoms: [ { area: string, description: string, severity: "MILD"|"MODERATE"|"SEVERE" } ],
    language?: string
  }
  Response: {
    symptom_log_id: string,
    suggested_specialty: string,
    urgency: "ROUTINE" | "SOON" | "URGENT" | "EMERGENCY",
    home_care_tips: string[]
  }
```

---

### Component 3: Chronic Disease Tracker & Smart Medication Scheduler

**Implements:** Requirement 3

The core differentiator: chronic disease patients have **complex medication schedules** that normal reminder apps can't handle. A diabetic with hypertension and thyroid might take 6 medications — Metformin twice daily, Thyronorm on empty stomach, Amlodipine once daily, Methotrexate every 7 days, Prednisolone on a tapering schedule, and insulin with varying doses. ArogyaMitra's scheduler handles ALL of these patterns.

#### Supported Schedule Patterns

```
DAILY:            Every day at fixed times (most medications)
ALTERNATE_DAY:    Every other day (steroids, some antibiotics)
EVERY_N_DAYS:     Every 2, 3, 5, 7 days (Methotrexate, B12 injections)
SPECIFIC_DAYS:    Mon/Wed/Fri only (dialysis meds, physiotherapy)
TAPERING:         40mg week 1 -> 35mg week 2 -> 30mg week 3 -> ...
                  (auto-calculates dose per day based on taper rule)
CYCLICAL:         21 days on, 7 days off (cancer meds, hormonal therapy)
SPLIT_DOSE:       Different doses at different times
                  (Insulin: 10u morning, 6u night)
FOR_X_DAYS:       Short course: "Take for 5 days" (antibiotics, pain meds)
                  Auto-stops after X days, moves to past medications
BEFORE_FOOD:      30 min before food (Thyronorm, some antacids)
AFTER_FOOD:       After food (Metformin, most NSAIDs)
```

#### Flow

```
User adds medication (voice: "Methotrexate 15mg har Sunday" OR form OR prescription OCR)
       |
       v
+--------------------+
| Schedule Parser     | -- NLP/rule engine parses frequency into schedule_type
|                     | -- Voice: "alternate day" -> ALTERNATE_DAY
|                     | -- Voice: "dose kam karte jao" -> TAPERING (ask details)
|                     | -- OCR: "1-0-1" -> twice daily (morning + night)
|                     | -- OCR: "OD" -> once daily, "BD" -> twice daily
+----------+---------+
           v
+--------------------+
| Schedule Generator  | -- Generate concrete reminder instances:
|                     |    For TAPERING: calculate dose for each day
|                     |    For CYCLICAL: calculate on/off windows
|                     |    For EVERY_N_DAYS: project next 30 days
|                     | -- Store in MedicationSchedule table
+----------+---------+
           v
+--------------------+
| Daily Schedule      | -- Each morning: compile today's medications
| Compiler            |    into a unified daily schedule card:
|                     |    "8 AM: Thyronorm 50mcg (empty stomach)
|                     |     8:30 AM: Metformin 500mg (after breakfast)
|                     |     2 PM: Amlodipine 5mg
|                     |     10 PM: Metformin 500mg, Insulin 6u"
+----------+---------+
           v
+--------------------+
| Reminder Engine     | -- Push notification at each scheduled time
| (Celery)            | -- Shows specific dose for TODAY (critical for tapers)
|                     | -- Voice confirmation: user says "le li" (taken) or
|                     |    "nahi li" (skipped). Also: one-tap buttons
|                     | -- Missed dose: follow-up reminder after 30 min
|                     | -- Still no response after 60 min: log as MISSED
+----------+---------+
           v
+--------------------+
| Caregiver Alert     | -- If dose status = MISSED and patient has linked
| Engine              |    caregiver with alert_on_missed_dose = true:
|                     | -- Send push/SMS to caregiver:
|                     |    "Papa ne subah ki dawai nahi li (Metformin + Amlodipine)"
|                     | -- Respects quiet hours (no alerts 10 PM - 7 AM)
|                     | -- Smart alerting: only after 2nd missed follow-up
|                     |    (avoids alert fatigue)
|                     | -- Critical vital alert: also sent to caregiver
|                     |    if alert_on_critical = true
+--------------------+

Vital Logging Flow:
+--------------------+
| Input (voice/form)  | -- Voice: "mera sugar aaj 140 hai" -> parsed
|                     | -- Form: manual entry with type selector
+----------+---------+
           v
+--------------------+
| Input Validation    | -- Validate ranges (BP: 60-250, Sugar: 30-600, etc.)
|                     | -- Flag out-of-range values with confirmation prompt
+----------+---------+
           v
+--------------------+
| Storage + Trends    | -- Store VitalLog record
|                     | -- Update 7-day and 30-day rolling averages (Redis)
|                     | -- Compare against user's baseline + normal ranges
+----------+---------+
           v
+--------------------+
| Alert Generation    | -- Critical: BP > 180/120, Sugar > 400, SpO2 < 90
| (if needed)         | -- Push notification + "consult doctor" message
+--------------------+

Prescription Upload Flow:
+--------------------+
| Image Upload        | -- Deskew, enhance, noise reduction
+----------+---------+
           v
+--------------------+
| OCR Engine          | -- Google Vision API -> extract text
+----------+---------+
           v
+--------------------+
| LLM Extraction      | -- Extract: drug names, dosages, frequency,
|                     |    doctor name, date
|                     | -- Parse Indian prescription shorthand:
|                     |    "1-0-1" = morning + night
|                     |    "OD/BD/TDS" = 1x/2x/3x daily
|                     |    "SOS" = as needed
|                     | -- Match drug names against CDSCO database
+----------+---------+
           v
+--------------------+
| Medication Records  | -- Create Medication records
|                     | -- Auto-detect schedule pattern from frequency
|                     | -- Set up smart reminders
+--------------------+
```

#### Key Design Decisions

- **Schedule pattern engine:** Core differentiator. Implements a finite-state schedule engine that pre-computes reminder instances for the next 30 days. On day 1 of each month, Celery regenerates the next month's schedule. Handles all 10 schedule patterns including FOR_X_DAYS courses that auto-expire.
- **Voice dose confirmation:** User can confirm doses by voice ("le li" = taken, "nahi li" = skipped) which is critical for elderly/low-literacy users who struggle with buttons. Voice confirmation is parsed by Bhashini STT and routed to the acknowledge API.
- **Caregiver alert system:** If a patient misses a dose (no response after 2 follow-ups / 60 min), their linked caregiver gets a push/SMS alert. Smart alerting avoids fatigue: only triggers after the 2nd missed follow-up, respects quiet hours (10 PM - 7 AM), and offers a daily summary mode instead of per-dose alerts. Caregiver linking requires explicit patient consent (timestamp stored).
- **Indian prescription parsing:** Indian doctors use shorthand like "1-0-1" (morning-afternoon-night), "OD" (once daily), "BD" (twice daily), "HS" (at bedtime), "AC" (before food), "PC" (after food). LLM + rules parse these into structured schedules.
- **Unified daily card:** Instead of 6 separate reminders for 6 medications, the user sees one consolidated "Today's Schedule" card showing all medications organized by time slot with food instructions.
- **Vital range validation:** Hard-coded physiological ranges prevent typos. Out-of-range values prompt re-entry.
- **Running averages:** 7-day and 30-day rolling averages cached in Redis for instant trend display.
- **Drug database:** Matched against India's CDSCO drug database. Fuzzy matching handles OCR errors and brand/generic name differences.

#### API

```
POST /api/v1/health/vitals
  Body: {
    type: "BLOOD_PRESSURE" | "BLOOD_SUGAR" | "TEMPERATURE" | "WEIGHT" | "HEART_RATE" | "SPO2",
    value: number,
    value_secondary?: number,  // for BP: diastolic
    sugar_context?: "FASTING" | "POST_MEAL" | "RANDOM",  // for blood sugar
    unit: string,
    notes?: string,
    recorded_at?: datetime
  }
  Response: {
    vital_id: string,
    status: "NORMAL" | "ELEVATED" | "HIGH" | "CRITICAL",
    trend: "IMPROVING" | "STABLE" | "WORSENING",
    message?: string
  }

GET /api/v1/health/vitals?type=BLOOD_SUGAR&from=2026-01-01&to=2026-02-15
  Response: {
    vitals: [ { id, type, value, unit, recorded_at, status } ],
    stats: { avg_7d, avg_30d, min, max, count }
  }

POST /api/v1/health/medications
  Body: {
    name: string,
    dosage: string,
    schedule_type: "DAILY" | "ALTERNATE_DAY" | "EVERY_N_DAYS" | "SPECIFIC_DAYS"
                   | "TAPERING" | "CYCLICAL" | "SPLIT_DOSE",
    schedule_config: {
      times: ["08:00", "22:00"],
      days_of_week?: [1,3,5],          // for SPECIFIC_DAYS
      interval_days?: 7,               // for EVERY_N_DAYS
      taper_rule?: {                    // for TAPERING
        start_dose: 40, step: -5, step_interval_days: 7, min_dose: 5
      },
      cycle_rule?: {                    // for CYCLICAL
        on_days: 21, off_days: 7
      },
      split_doses?: [                   // for SPLIT_DOSE
        { time: "08:00", dose: "10 units" },
        { time: "22:00", dose: "6 units" }
      ]
    },
    food_instruction: "BEFORE_FOOD" | "AFTER_FOOD" | "EMPTY_STOMACH" | "WITH_FOOD" | "ANY",
    prescribed_by?: string,
    start_date: date,
    end_date?: date
  }
  Response: { medication_id: string, reminders_generated: number, next_dose: datetime }

GET /api/v1/health/medications/today
  Response: {
    date: date,
    schedule: [
      {
        time: "08:00",
        medications: [
          { id, name, dose_today: "50mcg", food: "EMPTY_STOMACH", status: "PENDING"|"TAKEN"|"SKIPPED" }
        ]
      },
      { time: "08:30", medications: [...] },
      { time: "22:00", medications: [...] }
    ],
    adherence_today: { taken: 3, pending: 2, skipped: 0 }
  }

POST /api/v1/health/medications/{id}/acknowledge
  Body: { action: "TAKEN" | "SKIPPED", time?: datetime }
  Response: { updated: true, adherence_pct_week: number }

POST /api/v1/health/prescriptions/upload
  Body: multipart/form-data { image: <file>, language?: string }
  Response: {
    prescription_id: string,
    extracted_medications: [
      { name, dosage, frequency_raw: "1-0-1", schedule_type: "DAILY",
        food_instruction: "AFTER_FOOD", confidence: number }
    ],
    doctor_name?: string,
    date?: string
  }

GET /api/v1/health/medications?active=true
  Response: {
    medications: [ { id, name, dosage, schedule_type, next_dose, is_active } ]
  }

GET /api/v1/health/timeline
  Response: {
    events: [
      { date, type: "vital"|"medication"|"symptom"|"medical_event"|"document", summary, id }
    ]
  }

GET /api/v1/health/export.pdf
  Response: PDF binary (complete health record)

--- Caregiver APIs ---

POST /api/v1/caregivers/link
  Body: {
    caregiver_phone: string,
    caregiver_name: string,
    alert_on_missed_dose: boolean,
    alert_on_critical: boolean,
    alert_method: "PUSH" | "SMS" | "BOTH",
    quiet_hours?: { start: "22:00", end: "07:00" }
  }
  Response: { link_id: string, consent_recorded_at: datetime }

GET /api/v1/caregivers
  Response: {
    caregivers: [ { link_id, name, phone, alert_settings, is_active } ]
  }

DELETE /api/v1/caregivers/{link_id}
  Response: { revoked: true }

GET /api/v1/caregivers/dashboard  (caregiver views patient)
  Response: {
    patient_name: string,
    today_schedule: [...],
    adherence_today: { taken, pending, missed },
    adherence_week: number,
    recent_vitals: { bp: {...}, sugar: {...} },
    missed_doses_today: [ { medication, scheduled_time } ]
  }
```

---

### Component 3A: Centralized Health Record (Doctor-Ready Portfolio)

**Implements:** Requirement 3A

The "health wallet" — all medical documents in one place, organized and instantly shareable with any doctor.

#### Flow

```
User uploads document (photo/PDF/voice: "mera blood test upload karo")
       |
       v
+--------------------+
| Document Ingestion  | -- Accept: camera photo, gallery image, PDF, WhatsApp forward
|                     | -- Deskew, enhance for OCR
+----------+---------+
           v
+--------------------+
| OCR + AI Extraction | -- Google Vision API for text extraction
|                     | -- LLM extracts structured data:
|                     |    Prescription: drug names, doses, doctor, date
|                     |    Lab report: test names, values, normal ranges, flags
|                     |    Discharge summary: diagnosis, procedures, follow-up
|                     |    Vaccination: vaccine name, dose number, date
+----------+---------+
           v
+--------------------+
| Auto-Categorization | -- Classify document type:
|                     |    PRESCRIPTION | LAB_REPORT | DISCHARGE_SUMMARY |
|                     |    SCAN_REPORT | XRAY | VACCINATION | INSURANCE | OTHER
|                     | -- Auto-tag by condition if detectable
+----------+---------+
           v
+--------------------+
| Storage + Index     | -- Store original in S3 (AES-256 encrypted)
|                     | -- Store extracted data in PostgreSQL
|                     | -- Index in Elasticsearch for full-text search
|                     | -- Add to chronological health timeline
+--------------------+

Doctor Visit Summary Generation:
+--------------------+
| Compile Summary     | -- Active medications with current dosages
|                     | -- Last 3 months of vitals with trends
|                     | -- Known conditions and allergies
|                     | -- Recent lab results with flags
|                     | -- Recent medical events
+----------+---------+
           v
+--------------------+
| Generate Output     | -- PDF: formatted one-page summary
|                     | -- QR Code: time-limited web link (24hr expiry)
|                     |    Doctor scans -> sees read-only health record
|                     | -- WhatsApp share: deep link to summary
+--------------------+
```

#### Key Design Decisions

- **QR code sharing:** Home screen displays a QR code. Doctor scans it with any phone camera, opens a read-only web page showing the patient's health summary. Link expires in 24 hours for security. No app install required for the doctor.
- **Auto-categorization:** LLM classifies document type from OCR text (>95% accuracy for common Indian hospital documents). User can correct if wrong.
- **Indian lab report parsing:** Understands Indian lab report formats: CBC, LFT, KFT, HbA1c, Lipid Profile, Thyroid Panel, etc. Extracts values and flags abnormals.
- **WhatsApp integration:** Many Indians share medical documents via WhatsApp. The app registers as a share target so users can forward documents directly from WhatsApp.
- **Offline access:** All previously synced documents available offline via encrypted SQLite cache.

#### API

```
POST /api/v1/records/upload
  Body: multipart/form-data { file: <image/pdf>, category_hint?: string }
  Response: {
    record_id: string,
    detected_type: string,
    extracted_data: { ... },
    confidence: number
  }

GET /api/v1/records?category=LAB_REPORT&condition=Diabetes&from=2025-01-01
  Response: {
    records: [ { id, type, title, date, extracted_summary, thumbnail_url } ]
  }

GET /api/v1/records/{id}
  Response: { id, type, original_url, extracted_data, tags, uploaded_at }

GET /api/v1/records/doctor-summary
  Response: {
    active_medications: [...],
    recent_vitals: { bp: {...}, sugar: {...}, ... },
    conditions: [...],
    allergies: [...],
    recent_labs: [...],
    recent_events: [...],
    pdf_url: string,
    qr_code_url: string,
    share_link: string  // 24hr expiry
  }

GET /api/v1/records/doctor-summary/qr
  Response: { qr_image_base64: string, share_url: string, expires_at: datetime }

GET /api/v1/records/export.pdf
  Response: PDF binary (full health record)
```

---

### Component 4: Weekly Health Report Service

**Implements:** Requirement 4

#### Flow

```
Celery Beat triggers every Monday 6:00 AM IST
       |
       v
+--------------------+
| Fetch User Data     | -- Query vitals, medications, symptoms for past 7 days
|                     | -- Fetch health profile for context
+----------+---------+
           v
+--------------------+
| Vital Analysis      | -- Calculate avg, min, max for each vital type
|                     | -- Compare against normal ranges for user's age/gender
|                     | -- Identify trends (improving/worsening/stable)
+----------+---------+
           v
+--------------------+
| Medication          | -- Calculate adherence: (reminders acknowledged /
| Adherence Calc      |    total reminders) * 100
+----------+---------+
           v
+--------------------+
| Health Score        | -- Weighted formula:
| Calculation         |    Vitals in range: 40%
|                     |    Medication adherence: 25%
|                     |    Activity level: 20%
|                     |    Symptom frequency: 15%
|                     | -- Score: 0-100 (Poor/Fair/Good/Excellent)
+----------+---------+
           v
+--------------------+
| LLM Summary         | -- Generate natural language summary
| Generation          | -- Highlight positive trends and concerns
|                     | -- Generate 2-3 personalized action items
+----------+---------+
           v
+--------------------+
| Translation         | -- Translate report to user's language via Bhashini
+----------+---------+
           v
+--------------------+
| PDF Generation      | -- Generate formatted PDF report
| + Delivery          | -- Store in S3
|                     | -- Send push notification with in-app deep link
+--------------------+
```

#### Key Design Decisions

- **Health score formula:** Weighted composite score. Weights are tuned per chronic condition (e.g., for diabetics, blood sugar control gets higher weight). Score is informational, not diagnostic.
- **Batch processing:** Reports generated in batches via Celery workers. Staggered delivery (6 AM - 9 AM) to avoid notification spam and server spikes.
- **No-data handling:** If user logged fewer than 3 data points in a week, report is replaced with a gentle nudge notification encouraging them to resume tracking.
- **Historical comparison:** Each report includes comparison to previous week to show week-over-week progress.

#### API

```
GET /api/v1/reports/weekly/latest
  Response: {
    report_id: string,
    week_start: date,
    week_end: date,
    health_score: number,
    health_score_change: number,
    vital_summary: {
      blood_pressure: { avg, trend, status },
      blood_sugar: { avg, trend, status },
      ...
    },
    medication_adherence_pct: number,
    symptom_summary: [ { symptom, count, trend } ],
    highlights: [ string ],
    concerns: [ string ],
    action_items: [ string ],
    report_pdf_url: string
  }

GET /api/v1/reports/weekly?from=2026-01-01&to=2026-02-15
  Response: {
    reports: [ { report_id, week_start, health_score, summary } ]
  }

GET /api/v1/reports/weekly/{report_id}/pdf
  Response: PDF binary
```

---

### Component 5: Smart Health Notification Service

**Implements:** Requirement 5

#### Flow

```
Celery Beat triggers notification scheduler (daily at 5 AM)
       |
       v
+--------------------+
| User Selection      | -- Select users eligible for notification today
|                     | -- Respect frequency settings (2-3x/week)
|                     | -- Track last notification date per user
+----------+---------+
           v
+--------------------+
| Tip Selection       | -- Query HealthTip table filtered by:
|                     |    user's conditions, dietary_pref, age,
|                     |    activity_level, current season, region
|                     | -- Exclude recently sent tips (no repeats in 30 days)
|                     | -- Prioritize: seasonal alerts > condition-specific > general
+----------+---------+
           v
+--------------------+
| Personalization     | -- LLM personalizes tip text with user's name and context
|                     | -- e.g., "Rahul, as a diabetic, try replacing white rice
|                     |    with brown rice. It has a lower glycemic index."
+----------+---------+
           v
+--------------------+
| Translation         | -- Translate to user's language via Bhashini
+----------+---------+
           v
+--------------------+
| Delivery            | -- Push notification via FCM
|                     | -- Schedule at user's preferred time slot
|                     | -- Log delivery for analytics
+--------------------+
```

#### Key Design Decisions

- **Not daily:** Users explicitly said they don't want daily notifications. Default is 3x/week (Mon, Wed, Fri). Configurable to 2x/week.
- **Curated content:** Health tips are expert-written and reviewed, stored in the HealthTip table. NOT AI-generated on-the-fly (to avoid misinformation).
- **Seasonal awareness:** Tips database includes seasonal tags. During monsoon: dengue/malaria prevention. During winter: joint care, immunity. During summer: hydration, heatstroke.
- **Regional food awareness:** Food suggestions reference regional staples (e.g., ragi for Karnataka, dal-chawal for North India) rather than generic Western suggestions.

#### API

```
GET /api/v1/notifications/preferences
  Response: {
    frequency: "2x_week" | "3x_week",
    preferred_time: "morning" | "afternoon" | "evening",
    categories_enabled: [ "FOOD", "EXERCISE", "HYDRATION", "SLEEP", "SEASONAL" ],
    is_enabled: boolean
  }

PUT /api/v1/notifications/preferences
  Body: { frequency?, preferred_time?, categories_enabled?, is_enabled? }
  Response: { updated: true }

GET /api/v1/tips?category=FOOD&limit=10
  Response: {
    tips: [ { id, category, title, content, target_conditions } ]
  }

GET /api/v1/tips/{tip_id}
  Response: { id, category, title, content, language_versions: {} }
```

---

### Component 6: Symptom-to-Specialty Guidance Service

**Implements:** Requirement 5 (FR-5)

Unlike a simple symptom checker, this component has a **conversational follow-up flow** — the bot asks 2-3 clarifying questions before making a recommendation, just like a triage nurse would.

#### Flow

```
User describes symptoms via voice
  "Mujhe 2 din se chakkar aa rahe hain aur haath mein jhunjhuni hai"
       |
       v
+--------------------+
| Symptom Parsing     | -- Bhashini STT -> text -> translate to English
|                     | -- NLP extracts: dizziness (2 days), tingling (hands)
|                     | -- Standardize against symptom ontology
+----------+---------+
           v
+--------------------+
| Emergency Check     | -- Check against emergency symptom list:
| (IMMEDIATE)         |    chest pain, breathing difficulty, stroke signs
|                     |    (face droop, arm weakness, speech slur),
|                     |    severe bleeding, unconsciousness, seizure
|                     | -- If match: SKIP all follow-up questions
|                     |    IMMEDIATELY return emergency response
|                     |    with 108/112 call button
+----------+---------+
           v (no emergency)
+--------------------+
| Follow-Up Questions | -- LLM generates 2-3 contextual clarifying questions:
| (Conversational)    |
|                     |    For dizziness + tingling:
|                     |    Q1: "Kya yeh ek taraf hai ya dono taraf?"
|                     |        (one side or both?)
|                     |    Q2: "Kya bolne mein dikkat ho rahi hai ya
|                     |         chehra ek taraf jhuk raha hai?"
|                     |        (speech/face droop? -> stroke red flag)
|                     |    Q3: "Kab se ho raha hai? Achanak hua ya
|                     |         dheere dheere?"
|                     |        (sudden vs gradual onset)
|                     |
|                     | -- Questions are generated from a curated
|                     |    question bank per symptom cluster
|                     | -- User answers via voice; answers stored
|                     |    in SymptomLog.follow_up_qa
|                     | -- If any answer reveals red flag -> emergency
+----------+---------+
           v
+--------------------+
| Specialty Mapping   | -- Uses symptom + follow-up answers + user profile
|                     |    (age, gender, existing conditions)
|                     | -- Rule-based for common patterns (fast, reliable)
|                     | -- LLM for complex/multi-symptom cases with RAG
|                     |    over medical guidelines
|                     | -- Condition-aware: diabetic with tingling may
|                     |    suggest Endocrinologist alongside Neurologist
+----------+---------+
           v
+--------------------+
| Home Care Tips      | -- For non-urgent: basic things to do RIGHT NOW
|                     |    rest, hydration, ice/heat, safe OTC options
|                     | -- Condition-aware: diabetic with fever ->
|                     |    "sugar zyada check karein"
|                     | -- Sourced from RAG over medical knowledge base
|                     | -- Always includes disclaimers
+----------+---------+
           v
+--------------------+
| Response Assembly   | -- Primary specialty + up to 2 alternatives with reasons
|                     | -- Urgency level: ROUTINE / SOON / URGENT / EMERGENCY
|                     | -- Home care tips
|                     | -- Red flag warning: "If X happens, call 108 immediately"
|                     | -- Disclaimer
|                     | -- Translate + TTS in user's language
|                     | -- Save to SymptomLog (visible in health timeline)
+--------------------+
```

#### Follow-Up Question Bank (Examples)

| Symptom Cluster | Follow-Up Questions | Red Flag Check |
|----------------|--------------------|--------------------|
| Dizziness + tingling | One side or both? Speech difficulty? Sudden onset? | Stroke signs → EMERGENCY |
| Chest discomfort | Pain or pressure? Radiating to arm/jaw? Breathless? | Heart attack signs → EMERGENCY |
| Headache | Location? Visual changes? Neck stiffness? Worst ever? | Meningitis/aneurysm → EMERGENCY |
| Abdominal pain | Location (upper/lower/left/right)? Fever? Blood in stool? | Appendicitis signs → URGENT |
| Joint pain | One joint or many? Swelling? Morning stiffness? Recent injury? | — |
| Skin rash | Itchy? Spreading? Fever? New medication started? | — |

#### Key Design Decisions

- **Conversational triage, not just mapping:** The 2-3 follow-up questions dramatically improve specialty accuracy. "Dizziness" alone could be ENT (vertigo), Neurologist (nerve), or Cardiologist (low BP). Follow-ups narrow it down.
- **Red flag detection at every step:** Emergency check happens on initial input AND after each follow-up answer. If stroke signs emerge mid-conversation, bot immediately stops and shows emergency screen.
- **Emergency-first:** Emergency symptoms bypass ALL follow-ups — immediate 108/112 with one-tap call.
- **No diagnosis:** System explicitly maps symptoms to doctor specialties, NOT diseases. Responses never say "you might have X disease" — only "these symptoms are typically addressed by a [Specialty] doctor."
- **Condition-aware recommendations:** A diabetic with tingling hands gets Endocrinologist suggested alongside Neurologist (diabetic neuropathy is common). User's existing conditions from HealthProfile are injected into the mapping prompt.
- **Symptom log for doctors:** Every symptom session is saved with the full Q&A trail, so when the user visits the doctor, they can show exactly what they reported and when.

#### API

```
POST /api/v1/symptoms/start
  Body: {
    description: string,   // initial symptom description
    language?: string
  }
  Response: {
    session_id: string,
    is_emergency: boolean,
    emergency_numbers?: [ { label: "Ambulance", number: "108" } ],
    follow_up_questions?: [
      { id: string, question: string, question_audio_url: string }
    ]
  }

POST /api/v1/symptoms/{session_id}/answer
  Body: {
    question_id: string,
    answer: string
  }
  Response: {
    is_emergency: boolean,
    emergency_numbers?: [...],
    more_questions?: [...],          // if more clarification needed
    recommendation?: {               // if enough info to recommend
      primary_specialty: { name: string, reason: string },
      alternative_specialties: [ { name: string, reason: string } ],
      urgency: "ROUTINE" | "SOON" | "URGENT" | "EMERGENCY",
      urgency_description: string,
      home_care_tips: [ string ],
      red_flag_warning: string,
      disclaimer: string,
      symptom_log_id: string
    }
  }

GET /api/v1/symptoms/log/{symptom_log_id}
  Response: {
    id: string,
    symptoms: [...],
    follow_up_qa: [ { question, answer } ],
    recommendation: { specialty, urgency, home_care },
    logged_at: datetime
  }
```

---

## Cross-Cutting Concerns

### Authentication & Authorization

```
+---------------+     +----------------+     +----------------+
| Phone + OTP    |---->| Auth Service    |---->| JWT Token       |
| (MSG91 /       |     | (FastAPI)       |     | (Access +       |
|  Firebase)     |     |                 |     |  Refresh)       |
+---------------+     +----------------+     +----------------+
```

- **OTP-based login:** Phone number + 6-digit OTP. No password required.
- **JWT tokens:** Access token (15 min TTL) + Refresh token (30 day TTL).
- **Tier enforcement:** Middleware checks `user.tier` against endpoint limits. Free-tier: 5 AI queries/day, basic tracking only.

### Translation Pipeline

```
+----------------+     +----------------+     +----------------+
| Source Text     |---->| Bhashini NMT   |---->| Translated      |
| (English)       |     | (IndicTrans2)   |     | Text            |
+----------------+     +----------------+     +----------------+
                               |
                      +--------v---------+
                      | Translation       |
                      | Cache (Redis)     | -- Cache common phrases; TTL 7 days
                      +------------------+
```

- All user-facing AI-generated text passes through translation when `language != "en"`.
- Health tips and common responses are pre-translated and cached.
- Medical terms have a curated glossary per language to ensure accuracy.

### Error Handling Strategy

| Error Type | Handling |
|-----------|----------|
| OCR low confidence (<60%) | Return result with warning; suggest re-upload with better lighting |
| LLM timeout/failure | Return cached/partial results; queue for retry; show "processing" state |
| Translation failure | Fall back to English with "Translation unavailable" notice |
| STT failure | Show "Could not understand" prompt; suggest text input fallback |
| Vital out-of-range | Prompt user to re-enter; do not store without confirmation |
| Rate limit exceeded | Return 429 with upgrade prompt for free-tier users |
| Emergency detected | Bypass all queues; return emergency response immediately |

### Monitoring & Observability

| Tool | Purpose |
|------|---------|
| Prometheus + Grafana | System metrics (latency, throughput, error rates per service) |
| Sentry | Application error tracking and alerting |
| Elasticsearch + Kibana | Centralized logging, request tracing |
| OpenTelemetry | Distributed tracing across microservices |
| Custom dashboards | Business metrics: daily active users, vitals logged/day, report generation rate, AI queries/day |

---

## Security Design

### Data Encryption

| Data State | Method |
|-----------|--------|
| At rest (S3) | AES-256 server-side encryption (SSE-S3) |
| At rest (PostgreSQL) | Column-level encryption for health PII (pgcrypto) |
| In transit | TLS 1.3 for all API communication |
| Client storage | Android Keystore / iOS Keychain for tokens; encrypted SQLite for offline data |

### Data Privacy

- **Health data as sensitive PII:** All health records classified as sensitive personal data under DPDPA 2023. Requires explicit, granular consent.
- **Minimal data collection:** Only collect what's needed for each feature.
- **No data selling:** Health data is NEVER shared with insurers, pharma companies, or advertisers.
- **Right to erasure:** User can delete all health data at any time. Deletion is permanent and includes backups within 30 days.
- **Audit logging:** All data access logged with user_id, action, timestamp. Logs retained for 1 year.
- **Anonymized analytics:** Business metrics use aggregated, de-identified data only.

---

## Deployment Architecture

```
+---------------------------------------------------+
|              AWS Cloud (ap-south-1, Mumbai)         |
|                                                     |
|  +--------------+     +------------------------+   |
|  | Route 53      |---->| CloudFront CDN          |   |
|  | (DNS)         |     | (Static assets + API)   |   |
|  +--------------+     +------------+-----------+   |
|                                     |               |
|  +--------------------------------------------------+
|  |  EKS (Kubernetes)                                 |
|  |  +----------+ +----------+ +----------+          |
|  |  | Auth     | | Onboard  | | Health   |          |
|  |  | Service  | | Service  | | Tracking |          |
|  |  | (2 pods) | | (2 pods) | | (4 pods) |          |
|  |  +----------+ +----------+ +----------+          |
|  |  +----------+ +----------+ +----------+          |
|  |  | AI Asst. | | Report   | | Notif.   |          |
|  |  | Service  | | Service  | | Service  |          |
|  |  | (4 pods) | | (2 pods) | | (2 pods) |          |
|  |  +----------+ +----------+ +----------+          |
|  |  +----------+                                     |
|  |  | Doctor   |                                     |
|  |  | Rec.Svc  |                                     |
|  |  | (2 pods) |                                     |
|  |  +----------+                                     |
|  +--------------------------------------------------+
|                                                     |
|  +------------+ +------------+ +------------------+ |
|  | RDS         | | ElastiCa   | | S3               | |
|  | PostgreSQL  | | che Redis  | | (Prescriptions   | |
|  | (Multi-AZ)  | | (Cluster)  | |  & Reports)      | |
|  +------------+ +------------+ +------------------+ |
+---------------------------------------------------+
```

- **Region:** ap-south-1 (Mumbai) for low latency to Indian users.
- **Multi-AZ:** PostgreSQL and Redis deployed across 2 availability zones.
- **Auto-scaling:** HPA on CPU/memory. AI Assistant and Health Tracking scale up during peak hours.
- **Blue-green deployments:** Zero-downtime deploys via Kubernetes rolling updates.

---

## MVP Scope (Phase 1 -- 12 Weeks)

| Week | Deliverable |
|------|-------------|
| 1-2 | Project setup: CI/CD, DB schema, auth service, S3 bucket, base API framework |
| 3-4 | Onboarding: Health profile collection, BMI calculation, basic UI |
| 5-6 | Health Tracking: Vital logging, medication management, prescription OCR, trend charts |
| 7-8 | AI Assistant: Bhashini STT/TTS integration, medical RAG pipeline, multilingual chat |
| 9-10 | Reports + Notifications: Weekly report generation, health tips engine, push notifications |
| 11 | Doctor Recommendation: Symptom parser, specialty mapping, emergency detection |
| 12 | Integration testing, performance tuning, security audit, beta launch |

---

**Document Version:** 1.0
**Last Updated:** February 15, 2026
