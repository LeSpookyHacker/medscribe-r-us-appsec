# Attack Surface Document — MedScribe-R-Us

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P1

---

## Overview

This document enumerates the complete attack surface of the MedScribe-R-Us platform —
every exposed endpoint, authentication mechanism, external integration, and data
store that an attacker could target. It is the reference used to scope penetration
testing engagements and DAST configuration.

---

## 1. Internet-Facing Attack Surface

### 1.1 API Gateway (Primary Ingress)

| Endpoint Group | Auth Mechanism | Methods | Notes |
|---|---|---|---|
| `/api/v1/auth/*` | None (auth initiation) | POST | OAuth2 / OIDC login flows; PKCE required |
| `/api/v1/sessions/*` | JWT Bearer | GET, POST, DELETE | Recording session management |
| `/api/v1/audio/stream` | JWT Bearer + WebSocket | WS | Real-time audio stream; PHI in transit |
| `/api/v1/notes/*` | JWT Bearer | GET, PUT, PATCH | SOAP note retrieval, editing, approval |
| `/api/v1/patients/*` | JWT Bearer + ABAC | GET | Patient record access; care team scoped |
| `/api/v1/admin/*` | JWT Bearer + Admin role | GET, POST, PUT, DELETE | Clinic admin functions; tenant-scoped |
| `/api/v1/webhooks/*` | HMAC signature | POST | Inbound webhooks from EMR systems |
| `/api/v1/fhir/*` | SMART on FHIR token | GET, POST | FHIR R4 endpoints for EMR integration |
| `/health` | None | GET | Health check; must return no internal detail |
| `/metrics` | IP allowlist | GET | Prometheus metrics; restricted to monitoring systems |

**Attack surface notes:**
- The WebSocket endpoint (`/audio/stream`) is the highest-bandwidth PHI flow and
  the most complex to secure — authentication must persist across the WebSocket
  lifecycle, not just at handshake time
- `/api/v1/webhooks/*` accepts inbound data from EMR systems; HMAC signature
  validation must occur before any webhook payload is processed
- `/health` must never expose version numbers, dependency details, or stack traces

### 1.2 Clinician Portal (Next.js)

| Surface | Auth | Notes |
|---|---|---|
| Login page | OAuth2 PKCE flow | SSO via health system IdP preferred |
| Note editor | JWT session | Rich text editor; XSS surface for stored content |
| Evidence viewer | JWT session | Renders transcript segments; injection surface |
| File upload (future) | JWT session | Not currently implemented; high risk when added |

**Attack surface notes:**
- The note editor renders AI-generated content — stored XSS risk if output is not
  sanitized before rendering
- Content Security Policy (CSP) must prevent inline script execution

### 1.3 Patient Portal (Next.js)

| Surface | Auth | Notes |
|---|---|---|
| Login | Email magic link or SSO | Lower security stakes than clinician portal |
| Visit summary view | JWT session | Read-only PHI; must be scoped to own records only |
| Consent management | JWT session | Consent records must be immutable after signing |

### 1.4 Admin Portal (Next.js)

| Surface | Auth | Notes |
|---|---|---|
| Clinic Admin login | JWT + MFA required | Tenant-scoped; highest-risk portal after clinician |
| Platform Admin login | JWT + MFA + IP allowlist | Cross-tenant access; internal only |
| Clinician account management | JWT + tenant scope | CRUD on clinician accounts |
| Integration settings | JWT + tenant scope | EMR connection config; stores OAuth client secrets |

---

## 2. Internal Attack Surface (GCP VPC)

These endpoints are not internet-facing but are accessible to any compromised
internal Cloud Run service.

| Service | Internal Port | Auth | Notes |
|---|---|---|---|
| Audio Ingestion Service | 8080 | Service-to-service mTLS | Receives audio from API Gateway |
| Transcription Service | 8080 | Service-to-service mTLS | Calls Google STT |
| PHI Scrubbing Layer | 8080 | Service-to-service mTLS | Critical — output goes to LLM |
| AI Summarization Engine | 8080 | Service-to-service mTLS | Calls Vertex AI |
| Output Validation Service | 8080 | Service-to-service mTLS | Gate before MongoDB write |
| EMR Integration Service | 8080 | Service-to-service mTLS | FHIR write-back |
| Notification Service | 8080 | Service-to-service mTLS | Outbound webhooks |

**Attack surface notes:**
- Lateral movement between internal services is possible if mTLS is not enforced
- Each service's IAM service account must be scoped to only the GCP resources it needs
- Internal services should validate that requests arrive from expected upstream services

---

## 3. Data Store Attack Surface

| Store | Technology | Access Control | PHI? | Encryption |
|---|---|---|---|---|
| Clinical data | MongoDB Atlas | Service account + IP allowlist + mTLS | ✅ Yes | AES-256, CMEK |
| Audio files | GCS | Service account + bucket IAM | ✅ Yes | AES-256, CMEK |
| Secrets | GCP Secret Manager | Per-service IAM least privilege | 🔵 Keys | GCP-managed |
| Audit logs | MongoDB Atlas (separate collection) | Append-only service account | ⚠️ PII | AES-256 |
| Session tokens | In-memory (Redis future) | Not currently persisted | ⚠️ PII | N/A |

---

## 4. External Integration Attack Surface

| Integration | Direction | Protocol | Auth | PHI Shared |
|---|---|---|---|---|
| Epic EMR | Bidirectional | FHIR R4 / HTTPS | SMART on FHIR OAuth 2.0 | ✅ Yes |
| Cerner EMR | Bidirectional | FHIR R4 / HTTPS | SMART on FHIR OAuth 2.0 | ✅ Yes |
| Google Speech-to-Text | Outbound | gRPC (GCP internal) | GCP Service Account | ✅ Yes (audio) |
| Vertex AI (Gemini) | Outbound | REST (GCP VPC perimeter) | GCP Service Account | ❌ De-identified only |
| Datadog | Outbound | HTTPS agent | API Key (Secret Manager) | ❌ No PHI |
| SendGrid (email notifications) | Outbound | HTTPS | API Key (Secret Manager) | ⚠️ PII (clinician email) |

---

## 5. Authentication Mechanisms

| Mechanism | Used By | Token Lifetime | MFA Required |
|---|---|---|---|
| OAuth2 + OIDC (PKCE) | Clinician, Patient, Clinic Admin portal login | 15-min access token, 8-hour refresh | Required for Clinic Admin; recommended for clinicians |
| SMART on FHIR | EMR integration (both directions) | Per-encounter scope; 1-hour lifetime | N/A (machine-to-machine) |
| GCP Service Account (Workload Identity) | Internal service-to-service | Automatic rotation | N/A |
| HMAC-SHA256 | Inbound webhooks from EMR | Per-request signature | N/A |
| JWT (internal) | API Gateway → internal services | 15 minutes | Inherits from login |

---

## 6. Penetration Testing Scope

Based on this attack surface enumeration, the recommended scope for internal
penetration testing engagements is:

**In scope — every engagement:**
- API Gateway endpoints (all groups listed in section 1.1)
- Clinician Portal (authenticated, with test clinician account)
- Admin Portal (authenticated, with test Clinic Admin and Platform Admin accounts)
- Authentication flows (OAuth2 PKCE, JWT session management)
- Multi-tenancy isolation (cross-tenant access attempts)

**In scope — AI-focused engagement:**
- AI Summarization Engine prompt injection surface
- Output Validation Service bypass attempts
- PHI scrubbing validation (using synthetic PHI test data)
- Approval gate bypass

**Out of scope:**
- Google Cloud infrastructure (GCP's responsibility)
- Epic/Cerner FHIR endpoints (customer/vendor responsibility)
- Production patient data (use synthetic test data only)
- Denial of service testing (requires pre-approval and off-hours window)

---

## 7. Attack Surface Change Management

The attack surface must be re-reviewed when any of the following occur:

- A new API endpoint is added or an existing endpoint's auth mechanism changes
- A new external integration is added
- A new data store is provisioned
- A new user role or permission scope is defined
- A major dependency version is updated (e.g., FastAPI, Next.js major versions)

The security team must be notified via the security review request process
(`docs/sdlc/developer-guide.md`) before any of the above changes are shipped to production.
