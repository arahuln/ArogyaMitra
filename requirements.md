# Requirements: BimaSahayak (Insurance Helper)

## Requirement 1: Trap Detector — Policy Risk Auditing

### User Story
As a **policyholder**, I want to **upload my insurance policy PDF and get highlighted red flags and hidden risks**, so that **I can make informed decisions and avoid claim-time surprises**.

### Acceptance Criteria
- [ ] User can upload a policy document in PDF format (up to 50MB, scanned or digital)
- [ ] System extracts full text from the PDF using OCR for scanned documents and direct parsing for digital PDFs
- [ ] AI identifies and highlights critical risk clauses including: waiting periods (disease-specific and initial), room rent sub-limits, co-payment clauses, pre-existing condition exclusions, specific treatment exclusions, and network hospital restrictions
- [ ] Each red flag is presented with a severity indicator: ⚠️ High Risk, ⚡ Medium Risk, ℹ️ Info
- [ ] Each red flag includes a plain-language explanation at 8th-grade reading level (e.g., "Room Rent Capping is ₹2,000. If you take a private room, you will pay 50% of the bill from your pocket.")
- [ ] Results are displayed within 30 seconds of upload for a standard 20-page policy
- [ ] System identifies at least 90% of standard risk clauses when benchmarked against manual expert review
- [ ] Red flags are grouped by category (Waiting Periods, Sub-Limits, Exclusions, Co-Payments, Network Restrictions)
- [ ] User can tap any red flag to see the original clause text from the policy document
- [ ] Results are available in the user's selected vernacular language

---

## Requirement 2: Pre-Claim Document Triage

### User Story
As a **patient about to leave the hospital**, I want to **photograph my hospital documents and get them validated for insurance claim readiness**, so that **my claim is not rejected due to missing or inconsistent paperwork**.

### Acceptance Criteria
- [ ] User can capture photos of hospital documents using the mobile camera or upload existing images/scans
- [ ] System performs OCR on both printed and handwritten documents with ≥90% text extraction accuracy
- [ ] System checks for the presence of all mandatory claim documents: Discharge Summary, Original Bills & Receipts, Prescriptions, Diagnostic Reports, Claim Form, and Photo ID Proof
- [ ] Missing documents trigger a 🛑 Stop alert with actionable guidance (e.g., "You are missing the 'Discharge Summary'. Go back to the nurse station at counter 3.")
- [ ] System cross-verifies data consistency: patient name matches across all documents, policy number matches, date ranges are consistent, hospital name matches empanelled network
- [ ] Consistent data fields receive a ✅ Pass confirmation (e.g., "Name on bill matches Policy ID. You are safe to submit.")
- [ ] Complete document validation is finished within 60 seconds of upload
- [ ] System generates a downloadable checklist PDF summarizing pass/fail status for all documents
- [ ] User can re-upload corrected documents and re-validate
- [ ] System achieves ≥95% accuracy in identifying missing documents

---

## Requirement 3: Voice-First Vernacular Interface

### User Story
As a **non-English speaking user**, I want to **ask insurance questions in my native language using voice and receive spoken responses**, so that **I can understand my coverage without needing to read English text**.

### Acceptance Criteria
- [ ] System supports voice input (Speech-to-Text) in at least 10 Indian languages: Hindi, Tamil, Telugu, Marathi, Bengali, Gujarati, Kannada, Malayalam, Punjabi, and Odia
- [ ] System provides voice output (Text-to-Speech) in all supported languages
- [ ] Voice recognition accuracy is ≥85% for supported languages in typical ambient noise conditions
- [ ] Response time from voice query to audio response is <5 seconds for standard queries
- [ ] Responses cite specific clauses from the user's uploaded policy or government scheme (e.g., "As per Clause 4.2 of your policy...")
- [ ] Visual cues (icons, color coding, simple graphics) accompany all audio responses for reinforcement
- [ ] System supports multi-turn conversational flow with context retention (follow-up questions)
- [ ] Full text transcription of voice interactions is displayed on screen and stored in history
- [ ] System handles common insurance intents: coverage verification, claim status, hospital network lookup, and premium payment queries
- [ ] Interface works on low-bandwidth (3G) connections with graceful audio quality degradation

---

## Requirement 4: Ayushman Bharat Eligibility Bridge

### User Story
As an **uninsured citizen**, I want to **check if I qualify for the Ayushman Bharat (PM-JAY) government health scheme**, so that **I can access the ₹5 Lakh free healthcare cover I may be entitled to**.

### Acceptance Criteria
- [ ] User provides basic information: household income, family size, location (state/district), occupation, and housing type
- [ ] System applies current SECC (Socio-Economic Caste Census) deprivation criteria and PM-JAY eligibility rules
- [ ] Eligibility determination is provided within 30 seconds with a clear Yes/No result and explanation
- [ ] If eligible, system displays nearest Common Service Centers (CSCs) on a map with distance, address, and operating hours
- [ ] System provides a step-by-step enrollment guide with required documents checklist (Aadhaar, ration card, income certificate, etc.)
- [ ] If not eligible for Ayushman Bharat, system suggests alternative state-level health schemes the user may qualify for
- [ ] CSC location data is updated at least monthly from government sources
- [ ] Eligibility check and results are available in all supported vernacular languages
- [ ] User can set a reminder to complete enrollment at the CSC
- [ ] System does not store sensitive income/caste data beyond the active session unless user explicitly opts in

---

## Requirement 5: Policy Document Management

### User Story
As a **user with multiple insurance policies**, I want to **store, organize, and get reminders for all my family's policies in one place**, so that **I can quickly access any policy when needed**.

### Acceptance Criteria
- [ ] User can upload and securely store multiple policy documents in cloud storage with end-to-end encryption
- [ ] System auto-extracts key metadata from uploaded policies: insurer name, policy number, coverage amount, premium, start date, and expiry date
- [ ] User receives push/SMS notifications 30 days and 7 days before policy renewal dates
- [ ] User can link family members and associate policies to each family member
- [ ] Search and filter is available by insurer, policy type (health, life, motor), family member, and expiry date
- [ ] Policies are accessible offline after initial sync on the device

---

## Requirement 6: Claim Filing Assistance (Premium)

### User Story
As a **policyholder filing a claim**, I want to **get step-by-step guidance through the claim process with document organization and tracking**, so that **I maximize my chances of claim approval**.

### Acceptance Criteria
- [ ] System provides a guided, step-by-step claim form filling workflow with field-level help text
- [ ] User can upload, tag, and organize all claim-related documents within the app
- [ ] System tracks claim submission status and sends notifications on status changes
- [ ] System provides editable communication templates for insurer correspondence (follow-up, escalation, appeal)
- [ ] For rejected claims, system analyzes the rejection reason and provides specific appeal guidance with supporting clause references
- [ ] Estimated claim amount calculator shows expected payout based on policy terms and submitted bills

---

## Non-Functional Requirements

### NFR1: Performance
- [ ] 90% of API responses complete within 3 seconds
- [ ] System supports 10,000 concurrent users
- [ ] PDF processing completes within 60 seconds for documents up to 20 pages
- [ ] 99.5% uptime during business hours (9 AM–9 PM IST)

### NFR2: Security & Privacy
- [ ] All document storage uses end-to-end encryption (AES-256)
- [ ] Authentication via OTP-based login with optional biometric
- [ ] Compliant with India's Digital Personal Data Protection Act (DPDPA) 2023
- [ ] No user data shared with third parties without explicit consent
- [ ] Regular third-party security audits (annual minimum)
- [ ] Data retention follows IRDAI norms (7 years for claim records)

### NFR3: Accessibility & Device Support
- [ ] WCAG 2.1 Level AA compliant
- [ ] Functional on low-end Android devices (2GB RAM, Android 8.0+)
- [ ] App install size <50MB
- [ ] Functional on 3G networks (degraded media quality acceptable)
- [ ] Offline mode for stored policies, checklists, and cached results
- [ ] High-contrast mode and adjustable font sizes

### NFR4: Localization
- [ ] Full UI localization for 10+ Indian languages
- [ ] All AI-generated content (red flags, guidance, responses) available in the user's selected language
- [ ] Indian currency (₹) and date formats (DD/MM/YYYY) used throughout

### NFR5: Compliance
- [ ] IRDAI regulations compliance (cannot act as broker without license)
- [ ] Consumer Protection Act, 2019 compliance
- [ ] System includes disclaimers that it does not provide legal or medical advice
- [ ] Maintains insurer neutrality — no favoritism in recommendations

---

## Business Requirements

### Revenue Tiers

| Tier | Price | Features |
|------|-------|----------|
| **Free** | ₹0 | Trap Detector (2 policies), Ayushman eligibility check, basic voice queries, single policy storage |
| **Premium** | ₹99/mo or ₹999/yr | Unlimited policy analysis, Pre-Claim Triage, Claim Filing Assistance, family account (5 members), priority support |
| **B2B Hospital** | ₹50K–₹2L/yr | White-label solution, HMS integration, analytics dashboard, bulk patient document triage |
| **B2G Government** | Contract-based | Ayushman enrollment facilitation, state scheme integration, usage analytics for government |

### Success Metrics (Year 1)
- [ ] 100,000 registered users
- [ ] 30% reduction in claim rejection rate for active users
- [ ] 50,000 Ayushman Bharat enrollments facilitated
- [ ] User satisfaction score ≥4.2/5.0
- [ ] 5% free-to-premium conversion rate

---

## Constraints

| Category | Constraint |
|----------|------------|
| Legal | Cannot provide legal advice or act as insurance broker without IRDAI license |
| Medical | Cannot make medical diagnoses or treatment recommendations |
| Technical | Must work on 3G networks; app size <50MB; limited to publicly available policy data |
| Data | Cannot guarantee claim approval — only improves documentation quality |
| Regulatory | Must comply with DPDPA 2023, IRDAI norms, IT Act 2000 |

---

## Out of Scope (Post-MVP)

- Hospital EMR/EHR system integration
- Predictive claim approval probability scoring
- Community forum and user experience sharing
- Insurance marketplace and policy comparison shopping
- Cashless claim facilitation and real-time TPA integration
- Health/fitness tracker integration
- Blockchain-based document verification

---

## Glossary

| Term | Definition |
|------|------------|
| **PM-JAY** | Pradhan Mantri Jan Arogya Yojana (Ayushman Bharat) — government health insurance scheme providing ₹5L cover |
| **SECC** | Socio-Economic Caste Census — data used to determine PM-JAY eligibility |
| **CSC** | Common Service Center — government-authorized service points for scheme enrollment |
| **IRDAI** | Insurance Regulatory and Development Authority of India |
| **RAG** | Retrieval-Augmented Generation — AI technique grounding responses in source documents |
| **OCR** | Optical Character Recognition — converting images of text into machine-readable text |
| **Sub-limit** | Maximum payable amount for a specific item within overall policy coverage |
| **Co-payment** | Percentage of claim amount the policyholder must pay out-of-pocket |
| **TPA** | Third Party Administrator — intermediary managing cashless claims between insurer and hospital |
| **DPDPA** | Digital Personal Data Protection Act, 2023 — India's data privacy legislation |

---

**Document Version:** 1.0
**Last Updated:** February 15, 2026
