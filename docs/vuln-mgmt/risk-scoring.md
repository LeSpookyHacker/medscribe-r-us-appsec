# Vulnerability Management — Risk Scoring Framework

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P3

---

## 1. Purpose

This document defines how MedScribe-R-Us scores and prioritizes vulnerabilities
beyond CVSS base scores. CVSS alone is insufficient for HIPAA-regulated environments
because it does not account for the PHI exposure path, regulatory impact, or the
specific architecture of the system being scored.

---

## 2. Base Score — CVSS 3.1

All findings begin with their CVSS 3.1 base score as published by the NVD or the
security researcher who identified the finding. For internally discovered findings
(threat model, code review, pen test), AppSec assigns a CVSS score based on the
standard CVSS 3.1 vector.

If a CVSS score is not available (e.g., a novel finding with no CVE), AppSec
scores the vulnerability using the CVSS 3.1 calculator with the following
vector components:

| Component | Description |
|---|---|
| **AV** | Attack Vector (Network, Adjacent, Local, Physical) |
| **AC** | Attack Complexity (Low, High) |
| **PR** | Privileges Required (None, Low, High) |
| **UI** | User Interaction (None, Required) |
| **S** | Scope (Unchanged, Changed) |
| **C/I/A** | Confidentiality / Integrity / Availability Impact (None, Low, High) |

---

## 3. Contextual Modifier System

After establishing the CVSS base score, AppSec applies contextual modifiers
specific to MedScribe-R-Us's architecture and regulatory obligations. The
effective score determines the final severity tier and SLA.

### Upward Modifiers — Increase Effective Score

| Modifier | Score Increase | Condition |
|---|---|---|
| **PHI on path** | +2.0 | The vulnerability provides a path to read, write, or delete PHI |
| **Auth bypass** | +1.5 | The vulnerability bypasses authentication or session validation |
| **Approval gate** | +2.0 | The vulnerability bypasses the clinician approval gate (T-009) |
| **Internet-facing** | +1.0 | The vulnerable component is directly internet-accessible |
| **Active exploit** | +1.5 | A working exploit or exploit kit exists in the wild |
| **Cross-tenant** | +2.0 | The vulnerability allows access to another customer's data |
| **HIPAA control** | +1.0 | The finding directly affects a HIPAA-required technical safeguard |
| **Multi-tenant blast radius** | +1.5 | Successful exploitation affects all tenants simultaneously |

### Downward Modifiers — Decrease Effective Score

| Modifier | Score Decrease | Condition |
|---|---|---|
| **Auth required** | -1.0 | Exploitation requires valid authentication as any role |
| **High-priv required** | -1.5 | Exploitation requires Admin-level authentication |
| **No sensitive data** | -1.0 | No PHI, PII, or credential data accessible via exploitation |
| **Compensating control** | -1.0 | A documented, independent control mitigates the primary risk |
| **Internal only** | -0.5 | Vulnerable component is on private VPC with no internet path |

### Modifier Cap Rules

- **Minimum effective score:** 0.0 (cannot be negative)
- **Maximum effective score:** 10.0 (cannot exceed CVSS ceiling)
- **Mandatory Critical floor:** Any finding with both `PHI on path` and `Internet-facing`
  modifiers has a minimum effective score of 9.0 (Critical), regardless of base score

---

## 4. Scoring Examples

### Example A — PHI Scrubbing Gap (T-007)

A gap in the NER-based PHI scrubber allows a patient name to pass unmasked
into a Vertex AI API prompt.

| Component | Value |
|---|---|
| CVSS Base | 6.5 (AV:N/AC:H/PR:N/UI:N/S:U/C:H/I:N/A:N) |
| PHI on path | +2.0 |
| Internet-facing | +0.0 (scrubber is internal VPC) |
| Active exploit | +0.0 |
| Auth required | -1.0 |
| **Effective Score** | **7.5 — High** |
| **SLA** | 7 days |

Without contextual modifiers, this finding would be Medium (6.5). The PHI
exposure path elevates it to High, reflecting the HIPAA breach risk.

---

### Example B — Admin Portal Privilege Escalation (T-011)

A Clinic Admin can access another tenant's clinician accounts via a missing
tenant_id scope check.

| Component | Value |
|---|---|
| CVSS Base | 7.1 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N) |
| PHI on path | +2.0 |
| Cross-tenant | +2.0 |
| Internet-facing | +1.0 |
| Auth required | -1.0 |
| **Effective Score** | **10.0 (capped) — Critical** |
| **SLA** | 24 hours |

The cross-tenant modifier combined with PHI and internet-facing access pushes this
to the maximum score. This reflects the reality: a Clinic Admin exploiting this
can access the PHI of every patient across every tenant simultaneously.

---

### Example C — Stale Dependency (Low-Severity CVE)

A transitive Python dependency has a CVSS 3.2 CVE affecting a code path
not used by MedScribe.

| Component | Value |
|---|---|
| CVSS Base | 3.2 (AV:L/AC:H/PR:L/UI:N/S:U/C:L/I:N/A:N) |
| No sensitive data | -1.0 |
| Internal only | -0.5 |
| Compensating control | -1.0 |
| **Effective Score** | **0.7 — Low** |
| **SLA** | 90 days |

---

## 5. STRIDE Finding Scoring Reference

Scores for all 14 STRIDE findings from P1, post-modifier:

| ID | Finding | Base CVSS | Modifiers Applied | Effective Score | Severity |
|---|---|---|---|---|---|
| T-001 | Clinician Credential Spoofing | 7.4 | PHI (+2), Internet (+1), Auth req (-1) | 9.4 | 🔴 Critical |
| T-002 | API Gateway Auth Bypass | 9.8 | PHI (+2), Internet (+1) [capped] | 10.0 | 🔴 Critical |
| T-003 | Audio Stream Tampering | 5.9 | PHI (+2), Compensating (-1), Internal (-0.5) | 6.4 | 🟡 Medium |
| T-004 | Transcript Tampering in MongoDB | 6.8 | PHI (+2), Auth req (-1), Internal (-0.5) | 7.3 | 🟠 High |
| T-005 | Audit Log Repudiation | 6.5 | HIPAA control (+1), Auth req (-1) | 6.5 | 🟡 Medium → escalated to 🟠 High |
| T-006 | PHI in Application Logs | 7.5 | PHI (+2), Internet (+0), Auth req (-1) | 8.5 | 🔴 Critical |
| T-007 | PHI Scrubbing Gap | 6.5 | PHI (+2), Auth req (-1) | 7.5 | 🟠 High |
| T-008 | Cross-Patient Token Map Leak | 8.1 | PHI (+2), Cross-tenant (+2) [capped] | 10.0 | 🔴 Critical |
| T-009 | Approval Gate Bypass | 8.3 | Approval gate (+2), PHI (+2) [capped] | 10.0 | 🔴 Critical |
| T-010 | FHIR Patient Spoofing | 8.1 | PHI (+2), Internet (+1) [capped] | 10.0 | 🔴 Critical |
| T-011 | Admin Privilege Escalation | 7.1 | PHI (+2), Cross-tenant (+2), Internet (+1), Auth (-1) [capped] | 10.0 | 🔴 Critical |
| T-012 | DoS via Large Audio | 5.3 | HIPAA control (+1), Auth req (-1) | 5.3 | 🟡 Medium |
| T-013 | Secret Manager Compromise | 9.0 | PHI (+2), Multi-tenant (+1.5) [capped] | 10.0 | 🔴 Critical |
| T-014 | Indirect Prompt Injection | 7.5 | PHI (+2), Auth req (-1) | 8.5 | 🔴 Critical |

---

## 6. Score Review and Appeals

If an engineering team believes a score is inaccurate:

1. Open a comment on the GitHub Issue with the alternative scoring rationale
2. Tag the AppSec team for review
3. AppSec reviews within 2 business days and either confirms, adjusts, or escalates

Score disputes do not pause the SLA clock. If a finding is later downgraded,
the SLA is retroactively updated. If upgraded, the new SLA applies from the
upgrade date.
