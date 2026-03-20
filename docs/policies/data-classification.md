# Data Classification Policy — MedScribe-R-Us

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Effective: Phase 0

---

## 1. Purpose

This policy defines how MedScribe-R-Us classifies the data it creates, processes,
stores, and transmits — and the baseline security controls required for each
classification tier. It is the foundational data governance document for the AppSec
program and informs threat modeling, secure design, and compliance work.

---

## 2. Classification Tiers

### Tier 1 — Protected Health Information (PHI) 🔴

**Definition:** Any information that relates to a patient's past, present, or future
physical or mental health condition; the provision of healthcare to an individual; or
payment for healthcare — when that information can identify the individual. Governed
by HIPAA 45 CFR Part 164.

**Examples in MedScribe-R-Us:**
- Raw audio recordings of patient-clinician appointments
- Verbatim transcripts of appointments
- AI-generated SOAP notes (pre-approval and post-approval)
- ICD-10 diagnosis codes linked to a patient
- Medication lists, dosages, treatment plans
- Patient names, dates of birth, MRNs, addresses, phone numbers
- Any data element that could be used to identify a patient

**Required controls:**
- Encryption at rest: AES-256, CMEK via GCP Cloud KMS
- Encryption in transit: TLS 1.3 minimum; no unencrypted channels
- Access control: Role-based; minimum necessary; logged and audited
- De-identification required before LLM submission (see system-overview.md Step 3)
- Retention: Per customer BAA; default 7 years (HIPAA minimum)
- Deletion: Cryptographic erasure (delete CMEK key) + data destruction confirmation
- Logging: All access to PHI must produce an audit log entry in MongoDB Atlas
- Breach notification: HIPAA 60-day notification window; internal incident response
  team notified within 1 hour of suspected breach

**Prohibited:**
- PHI must never appear in application logs (Datadog, Cloud Logging)
- PHI must never be stored in client-side browser storage (localStorage, cookies)
- PHI must never be sent to any LLM API without prior de-identification
- PHI must never be transmitted over non-TLS connections
- PHI must never be included in error messages or stack traces

---

### Tier 2 — Personally Identifiable Information (PII) 🟠

**Definition:** Information that can be used to identify a specific individual, not
directly tied to a health condition. Often overlaps with PHI but may exist independently
(e.g., clinician PII that is not patient health data).

**Examples in MedScribe-R-Us:**
- Clinician name, email, NPI number, credential information
- Clinic admin names and contact information
- Patient portal login credentials
- IP addresses and device identifiers associated with user sessions
- Billing contact information (non-PHI portions)

**Required controls:**
- Encryption at rest: AES-256
- Encryption in transit: TLS 1.3 minimum
- Access control: Role-based; need-to-know; audited
- No logging of PII values in plain text (hash or omit)
- Retention: Retained for duration of account + 2 years; deleted on account closure
- GDPR consideration: If any EU patients or clinicians exist, full GDPR obligations apply

**Prohibited:**
- PII must not appear in application error logs
- PII must not be transmitted to third parties without explicit DPA/contractual basis

---

### Tier 3 — De-Identified Clinical Data 🟡

**Definition:** Clinical information from which all HIPAA Safe Harbor identifiers have
been removed (45 CFR §164.514(b)). This data retains medical meaning (diagnoses,
treatment patterns, medication classes) but cannot be linked back to an individual
without re-identification.

**Examples in MedScribe-R-Us:**
- De-identified transcripts submitted to Vertex AI
- Aggregate clinical statistics (e.g., "most common diagnoses in Q3")
- Anonymized model training datasets (future use case)

**Required controls:**
- De-identification must be validated before any data is classified in this tier
- PHI scrubbing pipeline output must be audited periodically for de-identification gaps
- Re-identification risk assessment required before using for model training
- Access control: Restricted; data science and ML teams only
- No customer-identifiable information may be inferred from aggregate data

**Note:** If re-identification risk analysis finds that combining de-identified data
with external datasets could re-identify individuals, the data must be reclassified to
Tier 1 (PHI).

---

### Tier 4 — Internal / Confidential 🔵

**Definition:** Non-public business information not directly tied to patient data.
Disclosure could harm MedScribe-R-Us's competitive position or customer relationships.

**Examples in MedScribe-R-Us:**
- Source code (all repositories)
- System architecture documentation
- Security assessments, vulnerability reports, penetration test findings
- Customer contracts and BAAs
- Internal security policies and runbooks
- API keys, service account credentials, secrets (should be in Secret Manager)
- Employee performance data, compensation, HR records

**Required controls:**
- Encryption at rest and in transit
- Access control: Employee need-to-know; contractor NDA required
- Source code: Private GitHub repos; branch protection; required PR reviews
- Secrets: GCP Secret Manager only; no plaintext in code, configs, or env vars
- Security findings: Restricted to security team + affected engineering teams

---

### Tier 5 — Public 🟢

**Definition:** Information approved for public release with no restrictions.

**Examples in MedScribe-R-Us:**
- Marketing website content
- Published API documentation (public FHIR endpoint schemas)
- Press releases
- This portfolio repository (fictional, educational)

**Required controls:**
- Review and approval before publication
- No accidental inclusion of Tier 1–4 data in public content
- Public-facing APIs must still enforce authentication for any non-public endpoints

---

## 3. Data Flow Summary

| Data Type | Classification | Where It Lives | LLM Exposure |
|---|---|---|---|
| Raw audio | Tier 1 (PHI) | GCS (CMEK encrypted) | ❌ Never |
| Verbatim transcript | Tier 1 (PHI) | MongoDB Atlas | ❌ Never |
| De-identified transcript | Tier 3 | In-memory (scrubbing layer) | ✅ Yes — Vertex AI only |
| AI-generated SOAP note (draft) | Tier 1 (PHI) | MongoDB Atlas | ❌ Never |
| Approved SOAP note | Tier 1 (PHI) | MongoDB Atlas + Epic/Cerner | ❌ Never |
| Audit logs | Tier 2 (PII) | MongoDB Atlas + Cloud Logging | ❌ Never |
| Clinician account data | Tier 2 (PII) | MongoDB Atlas | ❌ Never |
| System logs (no PHI/PII) | Tier 4 (Internal) | Datadog, Cloud Logging | ❌ Never |

---

## 4. Handling Violations

Any employee, contractor, or system process that handles data in violation of this
policy must:

1. Immediately notify the Security team (security@medscribe-r-us.fake)
2. Preserve evidence; do not attempt to remediate without Security guidance
3. Treat the incident as a potential HIPAA breach until confirmed otherwise

Violations are tracked in the incident response system and reviewed in the quarterly
security review.

---

## 5. Policy Review

This policy is reviewed annually or whenever a material change occurs in:
- Platform architecture
- Applicable regulation
- Data types processed
- Third-party integrations

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | Phase 0 | AppSec (M. Del Rio) | Initial version |
