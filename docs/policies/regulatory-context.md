# Regulatory Context — MedScribe-R-Us

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Effective: Phase 0

---

## 1. Purpose

This document outlines the regulatory and compliance frameworks that govern
MedScribe-R-Us as a healthcare AI company handling Protected Health Information (PHI).
It establishes the compliance baseline that the AppSec program is designed to support.

---

## 2. HIPAA / HITECH

### Applicability
MedScribe-R-Us is a **HIPAA Business Associate (BA)** for every health system customer
it serves. As a BA, MedScribe-R-Us:
- Processes PHI on behalf of Covered Entities (health systems, clinics)
- Is required to execute a **Business Associate Agreement (BAA)** with each customer
- Is directly liable under HIPAA for safeguarding PHI per the Security Rule and
  Privacy Rule
- Must notify Covered Entities within **60 days** of discovering a PHI breach
  (Breach Notification Rule, 45 CFR Part 164 Subpart D)

### Key HIPAA Rules for MedScribe-R-Us

**Security Rule (45 CFR Part 164, Subpart C):**

| Safeguard Category | Key Requirements | MedScribe Application |
|---|---|---|
| Administrative | Risk analysis, workforce training, incident response | AppSec program roadmap (this repo) |
| Physical | Physical access controls to systems containing PHI | GCP data center (Google's responsibility under shared model) |
| Technical | Access controls, audit controls, integrity, transmission security | Encryption, RBAC, audit logs, TLS |

**Privacy Rule (45 CFR Part 164, Subpart E):**
- Minimum Necessary: Access to PHI limited to what is required for the specific task
- Patient Rights: Patients have rights to access, amend, and receive an accounting
  of disclosures — MedScribe must support these workflows for its customers

**Breach Notification Rule (45 CFR Part 164, Subpart D):**
- Unsecured PHI breach → notify the Covered Entity without unreasonable delay and
  within 60 days
- Covered Entity notifies patients; MedScribe supports this process
- Cryptographic encryption of PHI at rest and in transit constitutes a "safe harbor"
  — breached data that was properly encrypted may not require notification if the
  key was not compromised

### HITECH Act
HITECH (Health Information Technology for Economic and Clinical Health Act) strengthened
HIPAA enforcement and extended Business Associate liability. Key implications:
- Civil monetary penalties now apply directly to BAs
- Tier structure: $100–$50,000 per violation; up to $1.9M per violation category per year
- Criminal penalties possible for willful neglect

### AppSec Program Impact
Every control in this AppSec program is traceable back to a HIPAA Security Rule
safeguard. The threat model (P1) and vulnerability management program (P3) will map
findings to specific HIPAA requirements.

---

## 3. SOC 2 Type II

### Applicability
Enterprise health system customers will require MedScribe-R-Us to demonstrate SOC 2
Type II compliance before signing contracts. This is a standard commercial requirement
for SaaS vendors handling sensitive health data.

### Trust Services Criteria
MedScribe-R-Us targets the following SOC 2 Trust Services Criteria (TSC):

| Criteria | Relevance to MedScribe |
|---|---|
| **CC — Security (Common Criteria)** | Required. Covers access controls, encryption, monitoring, incident response, change management |
| **A — Availability** | Required. Clinical workflows depend on platform uptime; SLA commitments to health systems |
| **C — Confidentiality** | Required. PHI confidentiality is core to the product |
| **PI — Processing Integrity** | Applicable. AI-generated notes must be accurate and complete |
| **P — Privacy** | Applicable if MedScribe collects patient-facing data directly |

### SOC 2 Type II vs. Type I
- **Type I:** Point-in-time assessment of control design — achievable in Year 1
- **Type II:** 6–12 month observation period of control effectiveness — target for Year 2

### AppSec Program Impact
The CI/CD security pipeline (P2) and vulnerability management program (P3) directly
support SOC 2 CC6 (Logical and Physical Access Controls) and CC7 (System Operations)
requirements. The threat model (P1) supports CC3 (Risk Assessment).

---

## 4. HITRUST CSF

### Applicability
HITRUST CSF (Common Security Framework) is increasingly required by large health
systems (enterprise contracts) as a comprehensive security certification. It incorporates
HIPAA, NIST, ISO 27001, PCI DSS, and other frameworks into a unified control set.

**Target:** Year 2 certification (e18 Validated Assessment)

### HITRUST e1, i1, r2 Assessment Levels
| Level | Control Count | Target |
|---|---|---|
| e1 (Essential) | ~44 controls | Year 1 self-assessment |
| i1 (Implemented) | ~182 controls | Year 2 |
| r2 (Risk-based) | ~2,000+ controls | Year 3+ |

### AppSec Program Impact
HITRUST domain mapping will be incorporated into the P3 vulnerability management
program and the P4 maturity assessment.

---

## 5. OWASP LLM Top 10

### Applicability
The OWASP LLM Top 10 is not a regulatory framework but a foundational security
reference for AI applications. Given MedScribe-R-Us's core AI functionality, it
is treated as a required technical standard.

| OWASP LLM Risk | MedScribe Exposure | Phase Addressed |
|---|---|---|
| LLM01 — Prompt Injection | Clinician notes as indirect injection vectors | P1 (threat model), P4 (AI security) |
| LLM02 — Insecure Output Handling | AI note output written to EMR | P1, P4 |
| LLM03 — Training Data Poisoning | Future model fine-tuning risk | P4 |
| LLM04 — Model Denial of Service | Large transcript inputs | P1 |
| LLM05 — Supply Chain Vulnerabilities | Vertex AI API dependency | P2 (SCA), P4 |
| LLM06 — Sensitive Info Disclosure | PHI leakage through LLM | P1, P4 |
| LLM07 — Insecure Plugin Design | FHIR write-back as "plugin" | P1 |
| LLM08 — Excessive Agency | Auto-write to EMR without approval | Product control (clinician approval gate) |
| LLM09 — Overreliance | Clinicians over-trusting AI notes | Product control (evidence links) |
| LLM10 — Model Theft | Model exfiltration via API | P4 |

---

## 6. NIST Cybersecurity Framework 2.0

### Applicability
NIST CSF 2.0 provides the structural language for the overall AppSec program. The
six functions map directly to this repo's phases:

| NIST CSF Function | AppSec Program Coverage |
|---|---|
| **Govern** | Data classification (P0), policies, maturity assessment (P4) |
| **Identify** | System architecture (P0), threat model, attack surface (P1) |
| **Protect** | CI/CD security pipeline (P2), secure architecture (P3) |
| **Detect** | SAST/DAST (P2), vulnerability management (P3) |
| **Respond** | Incident response (P3), output validation (P1) |
| **Recover** | DR architecture, RTO/RPO targets (future) |

---

## 7. Compliance Roadmap Summary

| Framework | Year 1 Target | Year 2 Target |
|---|---|---|
| HIPAA Security Rule | Full control implementation | Annual risk assessment |
| SOC 2 Type I | Complete | — |
| SOC 2 Type II | Observation period begins | Certification |
| HITRUST e1 | Self-assessment | — |
| HITRUST i1 | — | Validated assessment |
| OWASP LLM Top 10 | Full mapping + mitigations | Continuous |
| NIST CSF 2.0 | Program alignment | Maturity score improvement |
