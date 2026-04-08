# MedScribe-R-Us — Application Security Program

> ⚠️ **This is a fictional company created for portfolio and educational purposes.**
> MedScribe-R-Us does not exist. This repository demonstrates how a first AppSec hire
> would build a security program from scratch at an AI-powered healthcare startup.

---

## About MedScribe-R-Us (Fictional)

MedScribe-R-Us is a fictional healthcare AI company that transforms patient-clinician
conversations into structured clinical notes using large language models. Clinicians
record appointments, and the platform produces AI-generated SOAP notes, diagnoses,
and treatment plan summaries — all linked back to timestamped evidence in the original
transcript. These summaries are written directly into the clinic's Electronic Medical
Record (EMR) system.

**The core value proposition:** Reduce clinician charting time from ~2 hours/day to
under 20 minutes, letting doctors spend more time with patients.

### Fictional Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (React) — clinician web portal |
| Backend API | Python FastAPI — REST + WebSocket |
| AI/ML | Google Vertex AI (Gemini) — summarization & NLP |
| Transcription | Google Speech-to-Text API |
| Database | MongoDB Atlas — clinical documents & audit logs |
| Storage | Google Cloud Storage (GCS) — encrypted audio files |
| Infrastructure | Google Cloud Platform (GCP) — Cloud Run, VPC, Secret Manager |
| Auth | OAuth 2.0 + OIDC, SMART on FHIR for EHR launch |
| EMR Integration | HL7 FHIR R4 APIs — Epic, Cerner |
| Monitoring | Google Cloud Logging + Datadog |

### The Conversation Pipeline (Simplified)

```
[Clinician records appointment]
        |
        v
[Audio captured → GCS (encrypted)]
        |
        v
[Google Speech-to-Text → raw transcript]
        |
        v
[PHI scrubbing layer → de-identified text for LLM]
        |
        v
[Vertex AI (Gemini) → SOAP note + evidence links]
        |
        v
[Output validation → check for hallucinated clinical facts]
        |
        v
[Clinician review → approve / edit]
        |
        v
[FHIR write-back → Epic / Cerner EMR]
```

### User Roles

| Role | Description |
|---|---|
| **Clinician** | Records appointments, reviews/approves AI-generated notes |
| **Patient** | Consents to recording; may access their own visit summary |
| **Clinic Admin** | Manages clinician accounts, billing, integration settings |
| **Integration Partner** | EMR vendors (Epic, Cerner) accessing FHIR endpoints |
| **MedScribe Platform Admin** | Internal team managing the SaaS platform |

---

## Regulatory Scope

MedScribe-R-Us handles Protected Health Information (PHI) as defined under HIPAA and
is subject to the following compliance frameworks:

| Framework | Applicability |
|---|---|
| **HIPAA / HITECH** | Primary — PHI handling, BAA with customers, breach notification |
| **SOC 2 Type II** | SaaS trust requirement for enterprise health system customers |
| **HITRUST CSF** | Target for Year 2 — common healthcare enterprise requirement |
| **OWASP LLM Top 10** | AI-specific security risks in the summarization pipeline |
| **NIST CSF 2.0** | Security program framework alignment |

---

## This Repository: AppSec Program Build-Out

This repo documents the work of MedScribe-R-Us's **first AppSec hire** — building a
security program from zero. It is organized in phases:

| Phase | Focus | Status |
|---|---|---|
| **P0** | Company definition, architecture, regulatory context, data classification | ✅ Complete |
| **P1** | DFDs, STRIDE threat modeling, AI pipeline threat model | ✅ Complete |
| **P2** | CI/CD security pipeline — SAST, SCA, secrets, container, DAST | ✅ Complete |
| **P3** | Vulnerability management program, secure architecture reference | 🔲 Planned |
| **P4** | LLM/AI security deep-dive, bug bounty, program maturity | 🔲 Planned |

### Repository Structure

```
medscribe-r-us-appsec/
├── README.md                        ← You are here
├── docs/
│   ├── architecture/
│   │   ├── system-overview.md       ← Full pipeline narrative
│   │   ├── system-diagram.png       ← High-level architecture diagram
│   │   ├── iam-design.md            ← RBAC/ABAC model (P3)
│   │   ├── phi-data-flow.md         ← PHI lifecycle (P3)
│   │   └── network-security.md      ← GCP VPC design (P3)
│   ├── policies/
│   │   ├── data-classification.md   ← PHI tiers, handling rules, retention
│   │   └── regulatory-context.md    ← HIPAA, SOC 2, HITRUST framing
│   ├── threat-model/                ← P1 deliverables
│   ├── sdlc/                        ← P2 deliverables
│   ├── vuln-mgmt/                   ← P3 deliverables
│   ├── ai-security/                 ← P4 deliverables
│   └── program/                     ← P4 deliverables
├── .github/
│   └── workflows/                   ← P2: SAST, SCA, secrets, container, DAST
└── semgrep-rules/                   ← P2: Custom PHI + auth rules
```

---

## Author

**Manuel Del Rio** — Senior AppSec Engineer  
This repository is a portfolio project demonstrating the ability to build a zero-to-one
application security program in a healthcare AI startup context.

- Synack Red Team Member
- Published CVEs in medical device security research
- ~10 years AppSec experience across MITRE, Accenture, and independent research

---

*All company names, patient data, clinical notes, and system details in this repository
are entirely fictional and created for educational purposes only.*
