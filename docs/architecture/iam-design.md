# IAM Design — MedScribe-R-Us

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P3
>
> **Addresses:** T-001 (Credential Spoofing), T-011 (Privilege Escalation)

---

## 1. Overview

MedScribe-R-Us uses a layered identity and access model combining Role-Based
Access Control (RBAC) for coarse-grained permissions with Attribute-Based Access
Control (ABAC) for clinical context enforcement. HIPAA's "minimum necessary"
standard requires that access to PHI be limited not just by role, but by clinical
relationship — a cardiologist has no business accessing a patient's psychiatry notes.

ABAC is the mechanism that enforces this distinction.

---

## 2. Identity Providers

| User Type | Identity Provider | Auth Protocol | MFA |
|---|---|---|---|
| Clinicians | Health system IdP (Epic, Cerner SSO) | OIDC / SAML 2.0 | Enforced by health system IdP |
| Patients | MedScribe Auth (Auth0) | OAuth 2.0 PKCE | Optional (magic link default) |
| Clinic Admins | MedScribe Auth (Auth0) | OAuth 2.0 PKCE | **Required** |
| Platform Admins | MedScribe Auth (Auth0) + HW key | OAuth 2.0 PKCE | **Required — hardware key only** |
| Internal services | GCP Workload Identity | Service account federation | N/A (mTLS) |

### MFA Policy Detail

**Clinic Admins** must enroll in MFA before their account is activated. Accepted
factors: TOTP authenticator app, WebAuthn hardware key. SMS is not accepted due to
SIM-swapping risk in a HIPAA context.

**Platform Admins** require hardware security keys (YubiKey or equivalent FIDO2
device). Password + TOTP is not sufficient for cross-tenant administrative access.
Platform Admin sessions expire after 4 hours with no extension; re-authentication
is required.

**Clinicians** authenticate via their health system's IdP. MedScribe relies on the
health system to enforce MFA per their own HIPAA policies. Clinician JWT sessions
issued by MedScribe expire after 15 minutes; refresh tokens are valid for 8 hours
(the duration of a typical clinic day).

---

## 3. Role Definitions (RBAC)

| Role | Scope | Capabilities |
|---|---|---|
| `clinician` | Own appointments + care team patients | Record, view, edit, approve notes for patients on their care team |
| `patient` | Own records only | Read-only view of approved visit summaries for their own encounters |
| `clinic_admin` | Their `tenant_id` only | Manage clinician accounts, integration settings, billing contacts within their health system |
| `platform_admin` | All tenants | Full administrative access across all customers; strictly audited |
| `integration_partner` | FHIR scope only | Read/write FHIR R4 resources scoped to the integration agreement |
| `service_account` | Per-service GCP IAM | Machine-to-machine; no human login |

### JWT Claim Structure

All MedScribe JWTs carry the following claims:

```json
{
  "sub": "user_uuid",
  "role": "clinician",
  "tenant_id": "health_system_uuid",
  "npi": "1234567890",
  "iss": "https://auth.medscribe-r-us.fake",
  "aud": "medscribe-api",
  "iat": 1700000000,
  "exp": 1700000900,
  "jti": "unique_token_id"
}
```

`tenant_id` is always set server-side from the verified identity provider claim.
It is never accepted from the request payload, path parameter, or query string.
This is the primary enforcement mechanism for T-011 (privilege escalation).

---

## 4. ABAC — Clinical Context Enforcement

RBAC alone does not satisfy HIPAA's minimum necessary standard. A clinician role
permits accessing clinical data, but not *all* clinical data — only data for patients
where a care team relationship exists.

### Care Team Membership

The ABAC layer enforces a single core rule:

> **A clinician may only access a patient's data if an active care team
> relationship exists between that clinician and that patient for the
> encounter being accessed.**

Care team membership is determined at request time by querying the ABAC
authorization service, which holds a synchronized cache of care team data
from the health system's EMR (synced via FHIR CareTeam resources).

### ABAC Check Flow

```
Clinician requests GET /api/v1/notes/{appointment_id}
        |
        v
API Gateway validates JWT (role = clinician, tenant_id = xyz)
        |
        v
Authorization Middleware:
  1. Extract patient_id from appointment record
  2. Query ABAC service: is(clinician_npi, patient_id, tenant_id) on_care_team?
  3. If NO → 403 Forbidden (logged as access attempt)
  4. If YES → request proceeds
        |
        v
Note data returned to clinician
```

### ABAC Service Architecture

The ABAC service is a lightweight internal Cloud Run service that:
- Maintains a 5-minute TTL cache of care team memberships per tenant
- Refreshes from the health system FHIR `CareTeam` resource on cache miss
- Logs every access decision (allow and deny) to the audit log
- Returns deny-by-default if the FHIR CareTeam resource is unavailable
  (fail-closed, not fail-open)

**Fail-closed is critical.** If the ABAC service is unavailable, access is
denied and the clinician sees an error. This is operationally inconvenient but
prevents unauthorized PHI access during an outage.

---

## 5. Session Management

| Parameter | Value | Rationale |
|---|---|---|
| Access token lifetime | 15 minutes | Limits window of stolen token abuse |
| Refresh token lifetime | 8 hours (clinician), 4 hours (admin) | Matches clinical workflow duration |
| Refresh token rotation | On every use | Detects refresh token theft |
| Session binding | IP + User-Agent fingerprint | Detects session hijacking |
| Concurrent session limit | 3 per user | Detects credential sharing |
| Anomalous login detection | Geography + device | Flags credential compromise |

### Anomalous Login Handling

If a clinician login originates from an unusual geography (>500 miles from their
last login) or an unrecognized device fingerprint, the session is flagged for
step-up authentication. The clinician must complete an additional TOTP challenge
before the session is granted. The event is logged and the security team is notified
if the geographic anomaly exceeds 1,000 miles.

---

## 6. Service-to-Service Identity (GCP Workload Identity)

All Cloud Run services use GCP Workload Identity Federation — no service account
key files exist on disk. Each service has a dedicated service account with only
the IAM permissions it requires:

| Service | GCP Service Account | Permissions Granted |
|---|---|---|
| Audio Ingestion | `audio-ingest@project.iam` | GCS write (audio bucket only) |
| Transcription | `transcription@project.iam` | GCS read (audio bucket), STT API, Secret Manager (STT key only) |
| PHI Scrubbing | `phi-scrub@project.iam` | MongoDB read (transcript collection) |
| AI Summarization | `ai-summary@project.iam` | Vertex AI API, Secret Manager (Vertex key only) |
| Output Validation | `output-val@project.iam` | MongoDB write (notes collection) |
| EMR Integration | `emr-integration@project.iam` | MongoDB read (approved notes), Secret Manager (FHIR OAuth only) |
| Secret Manager | — | Accessed per-service; no cross-service secret access |

No service account has `roles/secretmanager.admin` or any wildcard IAM role.
Secret Manager access is restricted to the specific secret versions each
service needs.

---

## 7. Addressing T-001 and T-011

### T-001 — Clinician Credential Spoofing

Controls implemented:
- MFA enforced at health system IdP level for all clinicians
- 15-minute access token lifetime limits stolen-token window
- Refresh token rotation detects theft
- Anomalous login detection triggers step-up auth
- Concurrent session limit (3) detects credential sharing
- All session events logged to tamper-evident audit trail

### T-011 — Admin Portal Horizontal Privilege Escalation

Controls implemented:
- `tenant_id` claim set server-side from verified IdP; never from request
- Authorization middleware validates `current_user.tenant_id` against requested
  resource's `tenant_id` on every admin endpoint
- Platform Admin tokens are issued separately from Clinic Admin tokens with
  a distinct `role` claim; Platform Admin sessions require hardware key MFA
- Automated DAST test (P2) continuously validates tenant isolation in staging
- Quarterly penetration test scope includes explicit cross-tenant access testing
