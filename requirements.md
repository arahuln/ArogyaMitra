# REQUIREMENTS: ArogyaMitra

> **Theme:** AI for Bharat | **Hackathon:** AWS AI for Bharat
> **Version:** 2.0 | **Last Updated:** February 15, 2026

---

## Problem Statement

India has 100M+ chronic disease patients who take 3-8 medications daily — many on complex schedules (alternate days, tapering doses, cyclical courses) that no standard reminder app can handle. Health information is in English with medical jargon; most patients never read or understand it. Medical records are scattered across paper prescriptions, hospital files, and WhatsApp photos. When a patient visits a new doctor, there is no single place to see their complete history. Doctors make decisions with incomplete information.

**ArogyaMitra** is a voice-first AI health assistant in 12+ Indian languages that solves this by combining smart medication scheduling, chronic disease tracking, a centralized health passport, and symptom-to-specialty guidance — all accessible through voice with zero English literacy required.

---

## Goals

1. **Remove the language barrier** — let users manage their health entirely via voice in their native Indian language
2. **Handle complex medication schedules** that standard reminders can't (alternate days, every-N-days, tapering, cyclical, split doses)
3. **Centralize scattered health records** into a single doctor-ready health passport shareable via QR code
4. **Guide users to the right specialist** based on symptoms, with safety guardrails and emergency escalation
5. **Reduce missed medications** and improve chronic disease management adherence

## Non-Goals

- We do NOT diagnose diseases or prescribe medications
- We do NOT replace doctor consultations or provide telemedicine
- We do NOT integrate with hospital EMR/EHR systems (post-MVP)
- We do NOT sell insurance, medicines, or doctor appointments
- We do NOT store Aadhaar or government ID data

---

## Personas

### P1: Ramesh (Chronic Disease Patient)
- **Age:** 55 | **Location:** Tier-2 city | **Language:** Hindi
- **Conditions:** Type 2 diabetes + hypertension + thyroid
- **Takes:** 6 medications — Metformin (twice daily after food), Thyronorm (empty stomach), Amlodipine (once daily), Methotrexate (every Sunday), Prednisolone (tapering: 30mg this week, 25mg next), Insulin (10u morning, 6u night)
- **Pain:** Can't remember which pill to take when. Misses Methotrexate some weeks. Doesn't know today's Prednisolone dose. Wife has to remind him.
- **Goal:** One app that tells him exactly what to take and when, in Hindi, via voice

### P2: Lakshmi (Elderly Low-Literacy User)
- **Age:** 72 | **Location:** Village, Tamil Nadu | **Language:** Tamil only
- **Conditions:** Hypertension, knee pain
- **Pain:** Can't read English. Can't type. Paper prescriptions lost. Forgot what the doctor said.
- **Goal:** Talk to the phone in Tamil, it tells her what medicine to take and when

### P3: Priya (Caregiver Managing Parents)
- **Age:** 30 | **Location:** Bangalore | **Language:** English + Kannada
- **Role:** Managing medications for both parents remotely (father: diabetic, mother: cardiac)
- **Pain:** Can't be there in person. Doesn't know if parents took their medicines. No way to share their health history with new specialists.
- **Goal:** Get alerts when parents miss doses. Share health passport with their doctors.

### P4: Dr. Sharma (Doctor)
- **Age:** 45 | **Location:** Delhi clinic
- **Pain:** Patients come without records. They can't recall past medications, dosages, or test results. First 10 minutes of every consultation is spent piecing together history.
- **Goal:** Scan a QR code and see the patient's complete health summary in 30 seconds

---

## User Journeys

### Journey 1: Ramesh sets up his medications (Day 1)
1. Opens app → selects Hindi → signs in with phone OTP
2. Onboarding: answers health questions via voice ("mujhe diabetes hai, BP hai, thyroid hai")
3. Photographs his 3 prescriptions → OCR extracts 6 medications with schedules
4. Reviews extracted medications → app auto-detects: Metformin = daily, Methotrexate = every 7 days, Prednisolone = tapering
5. Confirms schedules → sees unified "Today's Medicines" card on home screen

### Journey 2: Ramesh's daily medication routine
1. 7:30 AM push notification: "Thyronorm 50mcg — khali pet leni hai" (empty stomach)
2. Ramesh taps "Le li" (taken) ✅
3. 8:30 AM after breakfast: "Metformin 500mg, Amlodipine 5mg — khana khane ke baad"
4. Ramesh doesn't respond → 30 min follow-up: "Dawai li kya?"
5. Still no response → Priya (caregiver) gets alert: "Papa ne subah ki dawai nahi li"
6. Sunday morning: "Aaj Methotrexate ka din hai — 15mg leni hai"
7. Evening: voice logs sugar: "Mera sugar aaj 145 hai" → app records, shows trend

### Journey 3: Lakshmi visits a new doctor
1. Opens app → taps QR code on home screen
2. Doctor scans QR → sees one-page summary: conditions, medications, recent BP readings, allergies
3. Doctor says "I can see her BP has been elevated for 3 weeks, and she's on Amlodipine 5mg — I'll increase to 10mg"
4. Lakshmi photographs new prescription → app updates medications

### Journey 4: Priya checks on her father
1. Gets push notification: "Papa ne subah ki dawai nahi li (Metformin + Amlodipine)"
2. Calls father, reminds him
3. Opens app → switches to father's profile → sees weekly adherence: 85%
4. Father has a doctor appointment → Priya generates health passport PDF, WhatsApps it to the doctor

### Journey 5: Ramesh feels unwell
1. Opens app → speaks: "Mujhe 2 din se chakkar aa rahe hain aur haath mein jhunjhuni hai"
2. Bot asks follow-up: "Kya yeh ek taraf hai ya dono taraf?"
3. Bot asks: "Kya bolne mein dikkat ho rahi hai ya chehra ek taraf jhuk raha hai?"
4. If no red-flag signs → "Aapke symptoms ke liye Neurologist se milna chahiye. Yeh urgent nahi lagta, 2-3 din mein appointment le lein."
5. If stroke signs detected → RED ALERT: "Yeh emergency ho sakti hai. Abhi 108 call karein." + one-tap call button

---

## Functional Requirements

### FR-1: Voice-First Multilingual AI Bot

The AI bot is the **primary interface** — not an add-on feature. Users can do everything via voice.

| ID | Requirement | Acceptance Criteria |
|----|-------------|-------------------|
| FR-1.1 | Voice input/output in Indian languages | STT + TTS via Bhashini in 12+ languages. MVP demo: Hindi + Telugu + Tamil |
| FR-1.2 | Voice as universal controller | Every app feature accessible via voice: log vitals, set reminders, ask questions, check reports, upload documents |
| FR-1.3 | Code-mix support | Handles Hindi-English ("mera sugar check karo"), regional-English mix naturally |
| FR-1.4 | Simple, jargon-free responses | Medical terms explained in everyday language. 8th-grade reading level. "BP high hai" not "hypertensive readings detected" |
| FR-1.5 | Multi-turn conversation | Maintains context across follow-up questions within a session (sliding window of 8 turns) |
| FR-1.6 | RAG-grounded responses | Answers sourced from curated medical KB (WHO, ICMR guidelines). NOT unverified internet content |
| FR-1.7 | Safety disclaimers | Every health response includes: "Yeh medical salah nahi hai. Doctor se zaroor milein." (spoken + displayed) |
| FR-1.8 | Emergency escalation | Detects emergency symptoms (chest pain, stroke signs, breathing difficulty) → immediately shows 108/112 with call button |
| FR-1.9 | Text fallback | Users can switch to text input anytime. UI is minimal and accessible |
| FR-1.10 | Low-bandwidth mode | Audio compressed to OGG Opus 24kbps for 3G. Graceful degradation on slow networks |

### FR-2: Smart Medication Scheduler

The core differentiator — handles every medication schedule pattern that chronic disease patients actually have.

| ID | Requirement | Acceptance Criteria |
|----|-------------|-------------------|
| FR-2.1 | Standard daily reminders | Daily at fixed times with before/after food instructions |
| FR-2.2 | Alternate-day dosing | Every other day (e.g., steroids Mon/Wed/Fri) |
| FR-2.3 | Every-N-days scheduling | Every 2, 3, 5, 7 days (e.g., Methotrexate weekly) |
| FR-2.4 | Specific-days-of-week | Custom day selection (e.g., Mon + Thu only) |
| FR-2.5 | Tapering schedules | Auto-decreasing dose: start 40mg, reduce 5mg/week, min 5mg. Reminder shows today's calculated dose |
| FR-2.6 | Cyclical patterns | X days on, Y days off (e.g., 21 on / 7 off for chemo drugs) |
| FR-2.7 | Split dosing | Different doses at different times (e.g., insulin 10u morning, 6u night) |
| FR-2.8 | "For X days" courses | Short courses: "Take for 5 days" → auto-stops after 5 days |
| FR-2.9 | Food/timing instructions | Before food, after food, empty stomach, with food, bedtime — displayed on every reminder |
| FR-2.10 | Unified daily schedule card | Single "Today's Medicines" view grouped by time slot. Not 6 separate notifications |
| FR-2.11 | Voice dose confirmation | User says "le li" (taken) or "nahi li" (skipped) by voice or one-tap button |
| FR-2.12 | Missed dose follow-up | If no response in 30 min → follow-up reminder. If still no response → log as missed |
| FR-2.13 | Caregiver alerts | Optional: if patient misses dose, send push/SMS to linked caregiver (Priya gets "Papa ne dawai nahi li") |
| FR-2.14 | Prescription OCR setup | Photograph prescription → OCR extracts drug names, doses, frequency (parses "1-0-1", "OD", "BD", "AC/PC") → auto-creates schedule |
| FR-2.15 | Voice setup | Set up medications via voice: "Metformin 500mg din mein do baar khane ke baad" |
| FR-2.16 | Adherence tracking | Weekly/monthly adherence percentage per medication and overall |

### FR-3: Chronic Disease Vital Tracking

| ID | Requirement | Acceptance Criteria |
|----|-------------|-------------------|
| FR-3.1 | Voice vital logging | "Mera sugar aaj 145 hai" → parsed and stored with timestamp |
| FR-3.2 | BP logging | Systolic + diastolic. Context: resting/post-exercise |
| FR-3.3 | Blood sugar logging | Value + context: fasting / post-meal / random / HbA1c |
| FR-3.4 | Additional vitals | Weight, temperature, heart rate, SpO2 (manual entry) |
| FR-3.5 | Trend visualization | Simple line charts: BP and sugar over weeks/months. Normal range bands shown |
| FR-3.6 | Weekly trend summary | Voice-readable: "Is hafte aapka sugar average 155 tha, pichle hafte se 10 kam" |
| FR-3.7 | Critical alerts | BP > 180/120, sugar > 400, SpO2 < 90 → immediate alert + "Doctor se milein" |
| FR-3.8 | Input validation | Physiological range checks (BP 60-250, sugar 30-600). Out-of-range prompts re-entry |

### FR-4: Health Passport (Centralized Records)

| ID | Requirement | Acceptance Criteria |
|----|-------------|-------------------|
| FR-4.1 | Document upload | Camera photo, gallery image, PDF, WhatsApp forward. Accepts: prescriptions, lab reports, discharge summaries, X-rays, vaccination records |
| FR-4.2 | Auto-categorization | OCR + LLM classifies document type (prescription / lab report / discharge / scan / vaccination). >90% accuracy |
| FR-4.3 | Data extraction | Extracts: medication names, test values + normal ranges, doctor name, hospital, date, diagnosis |
| FR-4.4 | Indian lab report parsing | Understands: CBC, LFT, KFT, HbA1c, Lipid Profile, Thyroid Panel, Urine Routine. Flags abnormals |
| FR-4.5 | Searchable timeline | All documents organized chronologically. Filter by type, condition, date, doctor |
| FR-4.6 | Doctor Visit Summary | One-page PDF: active medications + dosages, recent vitals + trends, conditions, allergies, key lab results |
| FR-4.7 | QR code sharing | Home screen QR code → doctor scans → read-only web page with health summary. 24-hour expiry link. No app install needed for doctor |
| FR-4.8 | WhatsApp/link sharing | Share summary or specific documents via WhatsApp, email, or direct link. Consent-based |
| FR-4.9 | Full export | Download complete health record as single PDF |
| FR-4.10 | Condition tagging | Tag documents by condition (e.g., "Diabetes", "Knee Surgery 2024") |
| FR-4.11 | Offline access | Previously synced documents viewable without internet |

### FR-5: Symptom-to-Specialty Guidance

| ID | Requirement | Acceptance Criteria |
|----|-------------|-------------------|
| FR-5.1 | Voice symptom intake | User describes symptoms in local language via voice or text |
| FR-5.2 | Follow-up questions | Bot asks 2-3 clarifying questions to narrow down: location, duration, severity, one-side-or-both, associated symptoms |
| FR-5.3 | Specialty mapping | Maps symptoms to: GP, Cardiologist, Dermatologist, Orthopedist, ENT, Ophthalmologist, Gynecologist, Pediatrician, Neurologist, Gastroenterologist, Pulmonologist, Endocrinologist, Psychiatrist, Urologist |
| FR-5.4 | Primary + alternatives | Recommends 1 primary specialty + up to 2 alternatives with clear reasons |
| FR-5.5 | Urgency classification | Routine (within a week) / Soon (2-3 days) / Urgent (24 hours) / Emergency (go now) |
| FR-5.6 | Basic home care tips | For non-urgent: suggest rest, hydration, ice/heat, safe OTC options. Condition-aware (diabetic with fever → "sugar zyada check karein") |
| FR-5.7 | Red-flag detection | Chest pain, stroke signs (face droop, arm weakness, speech slur), breathing difficulty, severe bleeding → IMMEDIATE emergency screen with 108/112 call button. Bypasses all other processing |
| FR-5.8 | No diagnosis | Never says "you might have X disease." Only: "these symptoms are typically addressed by a [Specialty] doctor" |
| FR-5.9 | Symptom log saved | Logged in health timeline. User can show doctor: "I reported these symptoms on Feb 10" |

---

## Non-Functional Requirements

| ID | Area | Requirement | Target |
|----|------|-------------|--------|
| NFR-1 | Latency | Voice query → audio response | < 5 seconds (P90) |
| NFR-2 | Latency | API response time | < 3 seconds (P90) |
| NFR-3 | Latency | Medication reminder delivery | Within 60 seconds of scheduled time |
| NFR-4 | Throughput | Concurrent users | 10,000 |
| NFR-5 | Availability | Uptime | 99.5% (health queries can be urgent) |
| NFR-6 | Storage | Prescription/document storage | 500MB per user, S3 with AES-256 |
| NFR-7 | Offline | Cached data available offline | Medication schedule, recent vitals, documents |
| NFR-8 | Device support | Minimum Android | Android 8.0+, 2GB RAM |
| NFR-9 | App size | Install size | < 40MB |
| NFR-10 | Network | Minimum bandwidth | Functional on 3G (degraded audio quality OK) |
| NFR-11 | OCR | Prescription text extraction accuracy | >= 90% for printed text |
| NFR-12 | STT | Voice recognition accuracy | >= 85% in typical ambient noise |
| NFR-13 | Report | Weekly report generation | < 30 seconds per user |
| NFR-14 | Scale | QR link page load | < 2 seconds |

---

## Data Requirements

| Data | Storage | Retention | Encryption |
|------|---------|-----------|------------|
| User profile (name, phone, health profile) | PostgreSQL | While account active | Column-level (pgcrypto) |
| Vital logs (BP, sugar, weight, etc.) | PostgreSQL | While account active | Column-level |
| Medication records + schedules | PostgreSQL | While account active | Column-level |
| Prescription/document images | AWS S3 | While account active | AES-256 SSE |
| Chat sessions | Redis (30 min) + PostgreSQL (history) | 30 min live / 1 year history | In-transit TLS 1.3 |
| Weekly reports (PDFs) | AWS S3 | 52 weeks rolling | AES-256 SSE |
| Medication adherence logs | PostgreSQL | While account active | Standard |
| Caregiver links | PostgreSQL | While account active | Standard |
| Audit logs | Elasticsearch | 1 year | Standard |

---

## Privacy & Safety Requirements

| ID | Requirement |
|----|-------------|
| PS-1 | All health data classified as **sensitive personal data** under DPDPA 2023. Explicit, granular consent required before collection |
| PS-2 | Health data is **NEVER** shared with insurers, pharma companies, or advertisers |
| PS-3 | QR code sharing is **consent-based** and **time-limited** (24-hour expiry). User explicitly generates it each time |
| PS-4 | Caregiver access requires **explicit patient consent**. Patient can revoke at any time |
| PS-5 | User can **delete all data** at any time (right to erasure). Deletion includes backups within 30 days |
| PS-6 | Every health response includes a **spoken + displayed disclaimer**: "This is not a medical diagnosis" |
| PS-7 | System **never diagnoses diseases** or prescribes medications. Only maps symptoms to specialties |
| PS-8 | **Emergency detection** bypasses all queues — immediate escalation with helpline numbers |
| PS-9 | Income/caste/religion data is **never collected** |
| PS-10 | All API communication over **TLS 1.3**. Data at rest encrypted with **AES-256** |
| PS-11 | Compliant with: DPDPA 2023, IT Act 2000, Telemedicine Practice Guidelines 2020 |

---

## Accessibility Requirements

| ID | Requirement |
|----|-------------|
| A-1 | **Voice-first by design** — every feature usable without reading or typing |
| A-2 | **Large touch targets** (48dp minimum) for elderly users |
| A-3 | **High-contrast mode** and adjustable font sizes (up to 2x) |
| A-4 | **Screen reader compatible** (TalkBack on Android) |
| A-5 | **Simple, icon-heavy UI** — minimal text, visual cues for all states |
| A-6 | App works on **low-end devices** (2GB RAM, Android 8.0+) |
| A-7 | Full **UI localization** for 12+ Indian languages (MVP: 2-3) |
| A-8 | **WCAG 2.1 Level AA** compliant |

---

## Assumptions

1. Bhashini (AI4Bharat) APIs are available and free for STT/TTS/NMT in supported languages
2. Users have a basic smartphone (Android 8+) with a phone number for OTP login
3. Internet connectivity (at least 3G) is available for voice features; offline mode covers cached data
4. Users are willing to speak to their phone for health management (validated in user interviews)
5. Caregivers and patients are in a trust relationship — caregiver linking is consensual
6. Indian prescription shorthand ("1-0-1", "OD", "BD", "AC/PC") is parseable with LLM + rules
7. Curated medical knowledge base (WHO, ICMR) is sufficient for health Q&A without needing real-time medical data

---

## Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Bhashini STT accuracy < 85% for some languages | Core feature degraded | Medium | Fallback to text input; focus MVP on Hindi + Telugu + Tamil where accuracy is highest |
| Users give medical info bot too much trust | Unsafe health decisions | Medium | Disclaimers on every response; never diagnose; emergency escalation; "consult doctor" prompts |
| OCR fails on handwritten prescriptions | Medication setup fails | High | Google Vision for printed; flag low-confidence results for manual entry; support voice setup as alternative |
| Complex medication schedules confuse users | Low adoption | Low | Voice-guided setup; visual confirmation of schedule; "Tomorrow you'll take X at Y" preview |
| Caregiver alert fatigue | Alerts ignored | Medium | Smart alerting: only after 2 missed follow-ups; configurable quiet hours; daily summary option |
| Low-end phones can't handle voice streaming | Exclusion of target users | Medium | Chunked HTTP upload fallback instead of WebSocket streaming; compressed audio (24kbps OGG) |
| Doctor QR scanning adoption | Health Passport underused | Medium | QR page is a simple web page — no app needed for doctor; promote via onboarding |

---

## Success Metrics

### Hackathon Demo
- [ ] End-to-end voice flow in Hindi: onboard → set medication → voice confirm dose → log vital → get trend → check symptoms → see doctor recommendation
- [ ] Health Passport: upload prescription photo → auto-extract → generate QR → scan and view summary
- [ ] Caregiver alert demo: patient misses dose → caregiver gets notification

### Post-Launch (Year 1)
| Metric | Target |
|--------|--------|
| Registered users | 200,000 |
| Weekly active users | 40% retention |
| Medication adherence improvement | 25% increase vs. baseline |
| Users with 3+ vitals logged per week | 50,000 |
| Health Passport QR scans by doctors | 10,000 |
| User satisfaction | >= 4.3 / 5.0 |
| Free-to-premium conversion | 7% |

---

## Out of Scope (Post-MVP)

- Telemedicine video consultation
- Wearable device integration (smartwatch, fitness bands)
- Lab report automated interpretation (beyond extraction)
- Pharmacy ordering / medicine delivery
- Hospital/clinic appointment booking
- Insurance integration
- Mental health screening
- Pregnancy and childcare tracking
- Community forums
- EMR/EHR system integration

---

## Future Roadmap

| Phase | Timeline | Features |
|-------|----------|----------|
| **MVP (Hackathon)** | Week 0-2 | Voice bot (2-3 languages), medication scheduler (all patterns), vital logging (BP + sugar), Health Passport (upload + QR), symptom-to-specialty |
| **Phase 1** | Month 1-3 | 12 language expansion, caregiver dashboard, weekly reports, notification engine |
| **Phase 2** | Month 3-6 | Wearable integration, lab report interpretation, family profiles (5 members), offline voice |
| **Phase 3** | Month 6-12 | B2B clinic dashboard, EMR integration, telemedicine referral, regional pharmacy tie-ups |

---

## Demo Plan (2-Minute Flow)

### Setup (15 sec)
> "Meet Ramesh, a 55-year-old diabetic in Lucknow with 6 medications. He speaks only Hindi."

### Act 1: Voice Onboarding (20 sec)
> Ramesh opens ArogyaMitra → selects Hindi → says: "Mujhe diabetes hai, BP hai, thyroid hai"
> Photographs his prescription → app extracts 6 medications including Methotrexate (weekly) and Prednisolone (tapering)

### Act 2: Smart Reminders (25 sec)
> Next morning: notification says "Thyronorm 50mcg — khali pet" → Ramesh says "le li" ✅
> Sunday: "Aaj Methotrexate ka din hai — 15mg" (app knows it's his weekly dose day)
> Prednisolone shows "Aaj ki dose: 25mg" (auto-calculated from taper schedule, was 30mg last week)
> Ramesh misses afternoon dose → his daughter Priya gets alert: "Papa ne dawai nahi li"

### Act 3: Vital Tracking (15 sec)
> Ramesh says: "Mera sugar aaj 145 hai" → app records, shows trend chart
> Voice summary: "Is hafte average 150, pichle hafte se 8% kam — achi baat hai!"

### Act 4: Health Passport (20 sec)
> Ramesh visits a new cardiologist → opens QR on home screen
> Doctor scans → sees one-page summary: conditions, 6 medications with exact dosages, BP trend (elevated), recent HbA1c: 7.2
> Doctor says: "I can see the full picture in 30 seconds"

### Act 5: Symptom Guidance (25 sec)
> Ramesh says: "Mujhe 2 din se chakkar aa rahe hain aur haath mein jhunjhuni hai"
> Bot asks: "Kya yeh ek taraf hai ya dono taraf?" → "Dono taraf"
> Bot asks: "Kya bolne mein dikkat ho rahi hai?" → "Nahi"
> Bot: "Aapko Neurologist se milna chahiye. Yeh urgent nahi lagta, 2-3 din mein appointment lein. Agar achanak chehra ek taraf jhuke ya bolne mein dikkat ho, turant 108 call karein."

### Closing (10 sec)
> "ArogyaMitra — your voice-first health companion in every Indian language."

---

## Glossary

| Term | Definition |
|------|------------|
| **ICMR** | Indian Council of Medical Research — India's apex body for biomedical research |
| **WHO** | World Health Organization — international health guidelines |
| **DPDPA** | Digital Personal Data Protection Act, 2023 — India's data privacy law |
| **Bhashini** | Government of India's AI language platform for Indian languages |
| **STT/TTS** | Speech-to-Text / Text-to-Speech — voice recognition and synthesis |
| **RAG** | Retrieval-Augmented Generation — AI technique grounding responses in source documents |
| **CDSCO** | Central Drugs Standard Control Organisation — India's drug regulatory body |
| **OD/BD/TDS** | Once daily / Twice daily / Three times daily — prescription shorthand |
| **AC/PC** | Ante Cibum (before food) / Post Cibum (after food) — prescription shorthand |
| **1-0-1** | Indian prescription notation: Morning-Afternoon-Night (1 = take, 0 = skip) |
| **HbA1c** | Glycated hemoglobin — 3-month average blood sugar indicator |
| **SpO2** | Blood oxygen saturation level |

---

**Document Version:** 2.0
**Last Updated:** February 15, 2026
