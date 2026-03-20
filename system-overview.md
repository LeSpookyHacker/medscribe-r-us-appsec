# System Overview — MedScribe-R-Us Platform

> **Fictional company — portfolio/educational purposes only.**

## 1. Purpose

This document provides a narrative description of the MedScribe-R-Us platform
architecture — how data flows from a patient-clinician appointment through AI
processing to a completed clinical note in the EMR. It is the foundational reference
for all threat modeling, secure design, and security testing work in this program.

---

## 2. High-Level Architecture

MedScribe-R-Us is a multi-tenant SaaS platform deployed on Google Cloud Platform (GCP).
Health systems (customers) connect their EMR systems via FHIR R4 APIs and provision
clinician access through the MedScribe web portal. The platform operates as a HIPAA
Business Associate for each customer health system.

### Core Services

| Service | Runtime | Responsibility |
|---|---|---|
| **API Gateway** | GCP Cloud Run | Auth enforcement, rate limiting, request routing |
| **Audio Ingestion Service** | Cloud Run (Python FastAPI) | Receives audio, validates format, writes to GCS |
| **Transcription Service** | Cloud Run (Python FastAPI) | Calls Google Speech-to-Text, stores raw transcript |
| **PHI Scrubbing Layer** | Cloud Run (Python) | De-identifies transcript before LLM submission |
| **AI Summarization Engine** | Cloud Run (Python FastAPI) | Constructs prompts, calls Vertex AI (Gemini), parses output |
| **Output Validation Service** | Cloud Run (Python) | Checks for hallucinated clinical facts, policy violations |
| **EMR Integration Service** | Cloud Run (Python FastAPI) | FHIR R4 write-back to Epic/Cerner |
| **Clinician Portal** | Cloud Run (Next.js 15) | Web UI for review, approval, editing of AI notes |
| **Admin Portal** | Cloud Run (Next.js 15) | Clinic admin and MedScribe internal admin functions |
| **Notification Service** | Cloud Run (Python) | Async webhooks, email alerts, in-app notifications |

All services communicate over private GCP VPC. No service is directly internet-exposed
except through the API Gateway.

---

## 3. The Conversation Pipeline — Step by Step

### Step 1: Appointment Recording

The clinician initiates a recording session from the MedScribe portal (browser-based)
or mobile app. The patient is notified and must have provided prior consent (managed
by the health system per their HIPAA policies).

- Audio is streamed in real-time to the **Audio Ingestion Service** over WebSocket
  (TLS 1.3)
- Each audio segment is encrypted (AES-256) and written to **GCS** under a
  customer-scoped bucket with CMEK (Customer-Managed Encryption Keys)
- A `recording_id` and `appointment_id` are created and logged to MongoDB Atlas

**Data at this stage:** Raw audio containing PHI (voices of clinician and patient)

### Step 2: Transcription

After the session ends, the **Transcription Service** pulls the audio from GCS and
submits it to the **Google Speech-to-Text API** (a HIPAA-eligible GCP service under
BAA).

- The raw transcript (full verbatim text) is stored in MongoDB Atlas, encrypted at rest
- Speaker diarization attempts to label utterances as "Clinician" or "Patient"
- The transcript remains internally to MedScribe and is never sent to external systems
  in raw form

**Data at this stage:** Verbatim transcript containing PHI (names, diagnoses, symptoms,
medications, dates of birth, etc.)

### Step 3: PHI Scrubbing

Before the transcript is sent to the LLM, it passes through the **PHI Scrubbing Layer**
— a rule-based + ML NER (Named Entity Recognition) pipeline that identifies and masks
the following PHI categories per HIPAA Safe Harbor (45 CFR §164.514(b)):

- Patient name → `[PATIENT_NAME]`
- Geographic data (street, city, zip) → `[LOCATION]`
- Dates (except year) → `[DATE]`
- Phone numbers → `[PHONE]`
- Email addresses → `[EMAIL]`
- SSN, MRN, account numbers → `[ID_NUMBER]`
- Device identifiers → `[DEVICE_ID]`

**Important:** The scrubbed transcript retains clinical meaning (symptoms, diagnoses,
medications, treatment plans) while removing direct identifiers. The mapping between
masked tokens and real values is stored separately in MongoDB and never sent to Vertex AI.

**Data at this stage:** De-identified transcript safe for LLM submission

### Step 4: AI Summarization

The **AI Summarization Engine** constructs a structured prompt from the de-identified
transcript and submits it to **Vertex AI (Gemini)** via the Vertex AI API.

The prompt instructs the model to produce:
- **SOAP Note:** Subjective, Objective, Assessment, Plan
- **Evidence Links:** Each clinical claim mapped to the transcript segment that supports it
- **ICD-10 Suggestions:** Potential diagnosis codes for clinician review
- **Medication Reconciliation:** Any medications mentioned, with dosage and frequency

**Security considerations at this stage:**
- Prompt is constructed server-side; no user-controlled content is injected without sanitization
- Vertex AI is called over a private VPC Service Controls perimeter (no public egress)
- LLM output is treated as untrusted until validated

**Data at this stage:** LLM output (untrusted — may contain hallucinated clinical facts)

### Step 5: Output Validation

The **Output Validation Service** performs structured checks on the LLM response before
it is surfaced to the clinician:

- **Schema validation:** Output conforms to expected SOAP structure
- **Hallucination detection:** Cross-references ICD codes and medication names against
  controlled vocabularies (SNOMED CT, RxNorm)
- **PHI re-injection audit:** Ensures the de-identified transcript tokens are re-mapped
  correctly when evidence links are reconstructed
- **Policy checks:** Flags any LLM output that references protected categories
  (e.g., mental health diagnoses) for mandatory clinician review before EMR write

**Data at this stage:** Validated, structured clinical note draft (still not in EMR)

### Step 6: Clinician Review & Approval

The validated draft is surfaced to the clinician in the **Clinician Portal**. The
clinician can:
- Read and edit any section of the note
- Click any claim to see the source evidence (timestamped transcript segment)
- Approve, reject, or flag for secondary review
- Add free-text addenda

**No AI-generated note is written to the EMR without explicit clinician approval.**
This is a hard product requirement and a critical security/compliance control.

### Step 7: EMR Write-Back

Upon clinician approval, the **EMR Integration Service** constructs a FHIR R4
DocumentReference resource and submits it to the health system's Epic or Cerner
FHIR endpoint using OAuth 2.0 client credentials (scoped to the specific patient
encounter via SMART on FHIR).

- The write operation is logged to MongoDB (audit trail)
- A webhook notification is sent to the health system confirming the write
- The original audio and transcript are retained per the customer's data retention
  policy (configurable, default 7 years per HIPAA minimum)

---

## 4. External Integrations & Trust Boundaries

| Integration | Protocol | Auth Mechanism | Data Shared |
|---|---|---|---|
| **Epic EMR** | FHIR R4 over HTTPS | SMART on FHIR OAuth 2.0 | Approved clinical notes (PHI) |
| **Cerner EMR** | FHIR R4 over HTTPS | SMART on FHIR OAuth 2.0 | Approved clinical notes (PHI) |
| **Google Speech-to-Text** | gRPC (GCP internal) | GCP Service Account | Raw audio (PHI) |
| **Vertex AI (Gemini)** | REST (GCP internal) | GCP Service Account | De-identified transcript only |
| **MongoDB Atlas** | TLS, Private endpoint | mTLS + IP allowlist | All platform data (PHI) |
| **Datadog** | HTTPS agent | API Key (Secret Manager) | Logs, metrics (no PHI) |

---

## 5. Deployment Architecture

- **GCP Project:** One project per environment (dev, staging, prod)
- **Region:** `us-central1` (primary), `us-east1` (DR)
- **Compute:** All services run as Cloud Run jobs (serverless containers)
- **Networking:** All inter-service traffic on private VPC; API Gateway is the only
  internet ingress point
- **Secrets:** All credentials stored in GCP Secret Manager; no secrets in code or
  environment variables in plain text
- **Container Registry:** Artifact Registry with vulnerability scanning enabled
- **CI/CD:** GitHub Actions → Cloud Build → Cloud Run

---

## 6. Key Security Properties (Current State — Pre-AppSec Program)

This table represents the assumed security posture at program inception — before
formal threat modeling, controls implementation, or CI/CD security gates.

| Property | Current State | Risk Level |
|---|---|---|
| Threat modeling | None performed | High |
| SAST/DAST | Not integrated into CI/CD | High |
| Dependency scanning | Not automated | High |
| Secrets detection | Not enforced | High |
| Container scanning | Not integrated | High |
| PHI scrubbing validation | Ad hoc testing only | Critical |
| LLM prompt injection testing | None | Critical |
| Incident response playbook | None | High |
| Security training | None for engineers | Medium |
| Vulnerability SLAs | Undefined | High |

These gaps drive the AppSec program roadmap documented in this repository.
