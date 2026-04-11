# PHI Data Flow — Lifecycle Documentation

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P3
>
> **Addresses:** T-004 (Transcript Tampering), T-005 (Audit Log Repudiation),
> T-009 (Approval Gate Bypass)

---

## 1. Purpose

This document traces every PHI category handled by MedScribe-R-Us from its
point of entry to its point of deletion. It is the primary reference for HIPAA
audit preparation, incident response investigations, and security architecture
reviews.

For each PHI category, this document records: where it enters, where it is
stored, who can access it, when and how it is de-identified, and how it is
eventually deleted.

---

## 2. PHI Categories Processed

| Category | HIPAA Identifier Type | Entry Point |
|---|---|---|
| Patient voice (audio) | Biometric identifier | Audio Ingestion Service |
| Clinician voice (audio) | Name (indirect) | Audio Ingestion Service |
| Patient name | Name | Raw transcript |
| Date of birth | Date | Raw transcript |
| Dates of service | Date | Raw transcript, SOAP note |
| Geographic data | Geographic | Raw transcript |
| Phone numbers | Phone | Raw transcript |
| Email addresses | Email | Raw transcript |
| MRN / account numbers | Account number | Raw transcript, EMR data |
| Diagnosis codes (ICD-10) | Medical record | SOAP note |
| Medications / dosages | Medical record | SOAP note |
| Treatment plans | Medical record | SOAP note |
| Social Security Number | SSN | Raw transcript (rare) |

---

## 3. PHI Lifecycle by Stage

### Stage 1: Audio Capture

**PHI present:** Patient voice, clinician voice (biometric), all spoken PHI

**Storage:**
- Written to GCS bucket: `gs://medscribe-phi-audio-{tenant_id}/`
- Encryption: AES-256, CMEK via GCP Cloud KMS per-tenant key
- Path: `/{tenant_id}/{appointment_id}/{recording_timestamp}.opus`

**Access controls:**
- Write: `audio-ingest` service account only
- Read: `transcription` service account only
- No human user has direct GCS bucket access
- Bucket-level access logging enabled

**Integrity control (T-003):**
- SHA-256 hash of each audio segment computed at ingestion
- Hash stored in MongoDB `recordings` collection alongside `appointment_id`
- Hash verified by Transcription Service before processing begins
- Mismatch triggers alert and halts processing

**Retention:** Per customer BAA; configurable 1–10 years; default 7 years

**Deletion:** CMEK key deletion (cryptographic erasure) + GCS object deletion
confirmed by the Data Lifecycle Service

---

### Stage 2: Transcription

**PHI present:** All spoken PHI — names, dates, diagnoses, medications

**Storage:**
- MongoDB Atlas `transcripts` collection
- Encryption: AES-256 at rest (MongoDB Atlas native encryption)
- Field-level encryption on `content` and `speaker_segments` fields
- Document structure: `{ appointment_id, tenant_id, content, speaker_segments, created_at, integrity_hash }`

**Access controls:**
- Write: `transcription` service account only
- Read: `phi-scrub` service account (for scrubbing), `transcription` service account
- No clinician-facing API exposes raw transcripts (only de-identified versions)
- MongoDB RBAC: `transcription` and `phi-scrub` roles have collection-level access only

**Integrity control (T-004):**
- Transcript stored with HMAC-SHA256 of content, keyed with a transcript-specific
  secret from Secret Manager
- HMAC verified before any downstream processing
- Append-only: transcripts cannot be updated after creation (MongoDB validator rule)
- Any modification attempt is rejected at the database layer and logged

---

### Stage 3: De-identification (PHI Scrubbing)

**PHI present (input):** Full transcript with all identifiers

**PHI present (output):** Zero — de-identified under HIPAA Safe Harbor (45 CFR §164.514(b))

**Token map storage:**
- MongoDB Atlas `token_maps` collection — separate collection from transcripts
- Structure: `{ session_id, patient_id, tenant_id, token: "PATIENT_NAME_1", value: "Jane Smith", created_at }`
- Field-level encryption on `value` field
- Access: `phi-scrub` service account (write), `output-val` service account (read for re-injection)
- Token maps are NEVER replicated to any external system or sent to Vertex AI
- Retention: same as the associated transcript; deleted simultaneously

**Session binding (T-008):**
- `session_id` in the token map is a cryptographic binding of `appointment_id +
  patient_id + tenant_id`, HMAC-signed
- Re-injection step verifies this binding before substituting any value
- Cross-patient token substitution is architecturally impossible without breaking
  the HMAC binding

---

### Stage 4: AI Summarization

**PHI present:** None — de-identified transcript only crosses the Vertex AI boundary

**What crosses the GCP VPC Service Controls perimeter:**
- De-identified transcript (Tier 3)
- System prompt (no PHI)
- De-identified prior note context (Tier 3)

**What never crosses:**
- Raw transcript
- Token map
- Patient identifiers of any kind

**Logging at this stage:**
- Each Vertex AI API call logged: `{ session_id, prompt_token_count, response_token_count, timestamp }`
- Content of the prompt and response is NOT logged — only metadata
- GCP Cloud Logging audit log captures Vertex AI API call metadata automatically

---

### Stage 5: SOAP Note (Pre-approval)

**PHI present:** PHI re-injected from token map — back to Tier 1

**Storage:**
- MongoDB Atlas `notes` collection
- Fields: `{ appointment_id, patient_id, tenant_id, content, status: "draft", approved_by: null, created_at }`
- `approved_by` field: write-protected — can only be set by the Note Approval endpoint
  authenticated as the assigned clinician; cannot be set by the EMR Integration Service
  or any other service

**Approval gate enforcement (T-009):**
- EMR Integration Service reads `notes` collection directly
- Before initiating FHIR write, validates: `status == "approved" AND approved_by != null`
- Validation is performed by reading from MongoDB — never from the API request payload
- If validation fails, write is rejected and the event is logged as an anomalous access attempt

---

### Stage 6: SOAP Note (Post-approval / EMR Write-back)

**PHI present:** Full SOAP note with all clinical content

**FHIR patient binding enforcement (T-010):**
- SMART on FHIR token requested with `patient/{patient_id} encounter/{encounter_id}` scope
- Token's `patient` claim validated against the appointment's `patient_id` binding
  before FHIR write is initiated
- FHIR write response validated: response `subject` resource matches expected patient
- Binding mismatch triggers immediate alert and halts the write

**EMR storage:** PHI leaves MedScribe's system boundary and is governed by
the health system's own HIPAA controls from this point forward.

**MedScribe copy:** The approved note is retained in MongoDB per the customer
BAA retention period. The MedScribe copy is the audit record.

---

### Stage 7: Audit Logging

**Addresses T-005 — Audit Log Repudiation**

Every PHI access event generates an audit log entry containing:

| Field | Value |
|---|---|
| `event_id` | UUID |
| `timestamp` | ISO 8601, UTC |
| `actor_id` | User UUID or service account |
| `actor_role` | JWT role claim |
| `tenant_id` | Customer tenant UUID |
| `action` | READ / WRITE / DELETE / APPROVE / DENY |
| `resource_type` | transcript / note / audio / token_map |
| `resource_id` | appointment_id |
| `patient_id` | Hashed (SHA-256) — stored in audit log, not plain text |
| `outcome` | SUCCESS / FAILURE |
| `hmac` | HMAC-SHA256 of the log entry, keyed with audit log signing key |

**Tamper-evidence controls:**
- Audit logs written to a separate MongoDB collection: `audit_events`
- The `phi-audit-writer` service account has INSERT-only access — no UPDATE or DELETE
- Logs replicated in real-time to GCP Cloud Logging (external to MongoDB)
- HMAC on each entry using a dedicated signing key in Secret Manager
- Any attempt to modify an existing audit log entry breaks the HMAC chain
- GCP Cloud Logging audit logs themselves are immutable (GCP platform guarantee)

**Retention:** 6 years minimum (HIPAA requirement for BA documentation)

---

## 4. PHI Deletion Process

PHI deletion is triggered by:
- Account closure (customer offboarding)
- Patient deletion request (where applicable under state law)
- Retention period expiry (enforced by the Data Lifecycle Service)

**Deletion sequence:**
1. CMEK key deletion in Cloud KMS (cryptographic erasure of GCS audio)
2. GCS object deletion confirmed
3. MongoDB document deletion (`transcripts`, `notes`, `token_maps` collections)
4. Deletion event written to audit log (the audit log entry itself is retained per HIPAA)
5. Customer notified of deletion completion with confirmation timestamp

**Deletion verification:**
- Automated test: after deletion, attempt to read the deleted document — expect 404
- GCS bucket inventory confirms object removal
- Deletion confirmation stored in a separate `deletion_records` collection (no PHI)
