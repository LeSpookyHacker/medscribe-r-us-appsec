# Data Flow Diagram — Level 1 (Component Diagram)

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P1

---

## Overview

The L1 diagram decomposes the MedScribe-R-Us platform into its component services,
data stores, and internal data flows. Trust boundaries are drawn explicitly.

This diagram is the primary reference for the STRIDE threat analysis. Each numbered
data flow in the diagram corresponds to a row in the STRIDE threat register.

---

## Diagram

```mermaid
flowchart TD
    %% ── External Entities ──────────────────────────────────────────
    CLI["👤 Clinician"]
    PAT["👤 Patient"]
    EMR["🏥 EMR\n(Epic/Cerner)"]
    STT["☁️ Google\nSpeech-to-Text"]
    VAI["☁️ Vertex AI\n(Gemini)"]

    %% ── Internet Trust Boundary ─────────────────────────────────────
    subgraph INTERNET ["🌐 Internet Trust Boundary"]
        APIGW["API Gateway\n(Cloud Armor + Rate Limit)"]
    end

    %% ── Internal VPC ────────────────────────────────────────────────
    subgraph VPC ["🔒 GCP Private VPC"]

        subgraph INGESTION ["Audio Pipeline"]
            AUD["Audio Ingestion\nService"]
            TRANS["Transcription\nService"]
            SCRUB["PHI Scrubbing\nLayer"]
        end

        subgraph AIPIPE ["AI Pipeline"]
            AISUM["AI Summarization\nEngine"]
            OUTVAL["Output Validation\nService"]
        end

        subgraph PORTAL ["Portals"]
            CLINP["Clinician\nPortal (Next.js)"]
            ADMINP["Admin\nPortal (Next.js)"]
            PATP["Patient\nPortal (Next.js)"]
        end

        EMRINT["EMR Integration\nService"]
        NOTIFY["Notification\nService"]

        %% ── Data Stores ─────────────────────────────────────────────
        GCS[("🗄️ GCS\nAudio Store\n(CMEK)")]
        MONGO[("🗄️ MongoDB Atlas\nClinical Data\n(Encrypted)")]
        SECMGR[("🔑 Secret Manager\nCredentials")]
    end

    %% ── Flow 1: Clinician → API Gateway ────────────────────────────
    CLI -->|"①  Audio stream (PHI) / API calls [TLS 1.3]"| APIGW

    %% ── Flow 2: Patient → API Gateway ──────────────────────────────
    PAT -->|"②  Portal login / consent [TLS 1.3]"| APIGW

    %% ── Flow 3: API Gateway → Services ─────────────────────────────
    APIGW -->|"③  Authenticated audio stream"| AUD
    APIGW -->|"③  Authenticated API requests"| CLINP
    APIGW -->|"③  Authenticated API requests"| ADMINP
    APIGW -->|"③  Authenticated API requests"| PATP

    %% ── Flow 4: Audio → GCS ─────────────────────────────────────────
    AUD -->|"④  Encrypted audio segments (PHI)"| GCS

    %% ── Flow 5: GCS → Transcription ─────────────────────────────────
    GCS -->|"⑤  Audio file (PHI)"| TRANS

    %% ── Flow 6: Transcription → Speech-to-Text ──────────────────────
    TRANS -->|"⑥  Audio (PHI) [GCP VPC / BAA]"| STT
    STT -->|"⑥  Raw transcript (PHI)"| TRANS

    %% ── Flow 7: Transcript → MongoDB ────────────────────────────────
    TRANS -->|"⑦  Raw transcript (PHI)"| MONGO

    %% ── Flow 8: Transcript → PHI Scrubber ───────────────────────────
    TRANS -->|"⑧  Raw transcript (PHI)"| SCRUB

    %% ── Flow 9: Scrubber → AI Summarization ─────────────────────────
    SCRUB -->|"⑨  De-identified transcript [Tier 3]"| AISUM

    %% ── Flow 10: AI Summarization → Vertex AI ───────────────────────
    AISUM -->|"⑩  De-identified prompt [GCP VPC perimeter]"| VAI
    VAI -->|"⑩  LLM response (untrusted)"| AISUM

    %% ── Flow 11: AI Summary → Output Validation ─────────────────────
    AISUM -->|"⑪  Raw LLM output (untrusted)"| OUTVAL

    %% ── Flow 12: Validated Note → MongoDB ───────────────────────────
    OUTVAL -->|"⑫  Validated SOAP note draft (PHI)"| MONGO

    %% ── Flow 13: Draft Note → Clinician Portal ───────────────────────
    MONGO -->|"⑬  SOAP note draft (PHI)"| CLINP
    CLINP -->|"⑬  Clinician edits + approval"| MONGO

    %% ── Flow 14: Approved Note → EMR Integration ─────────────────────
    MONGO -->|"⑭  Approved note (PHI) + approval flag"| EMRINT

    %% ── Flow 15: EMR Integration → Epic/Cerner ───────────────────────
    EMRINT -->|"⑮  FHIR DocumentReference (PHI) [OAuth 2.0]"| EMR
    EMR -->|"⑮  Prior visit notes (PHI) [OAuth 2.0]"| EMRINT

    %% ── Flow 16: EMR Integration → MongoDB (prior notes) ─────────────
    EMRINT -->|"⑯  Prior notes stored as LLM context (PHI)"| MONGO

    %% ── Flow 17: Patient Portal ──────────────────────────────────────
    MONGO -->|"⑰  Visit summary (PHI, post-approval only)"| PATP
    PATP -->|"⑰  Summary delivered [TLS 1.3]"| PAT

    %% ── Flow 18: Secrets ─────────────────────────────────────────────
    SECMGR -->|"⑱  API keys / service account creds"| AISUM
    SECMGR -->|"⑱  API keys / service account creds"| EMRINT
    SECMGR -->|"⑱  API keys / service account creds"| TRANS

    %% ── Flow 19: Notifications ───────────────────────────────────────
    OUTVAL -->|"⑲  Validation alerts"| NOTIFY
    EMRINT -->|"⑲  Write-back confirmation"| NOTIFY
    NOTIFY -->|"⑲  Webhook / email [TLS 1.3]"| EMR

    %% ── Styling ──────────────────────────────────────────────────────
    classDef external fill:#e8f4fd,stroke:#2196F3,stroke-width:1.5px,color:#0d47a1
    classDef service fill:#f3e5f5,stroke:#9C27B0,stroke-width:1px,color:#4a148c
    classDef datastore fill:#fff9c4,stroke:#F9A825,stroke-width:1.5px,color:#f57f17
    classDef gateway fill:#fce4ec,stroke:#E91E63,stroke-width:2px,color:#880e4f
    classDef google fill:#fff3e0,stroke:#FF9800,stroke-width:1.5px,color:#e65100

    class CLI,PAT,EMR external
    class STT,VAI google
    class APIGW gateway
    class AUD,TRANS,SCRUB,AISUM,OUTVAL,CLINP,ADMINP,PATP,EMRINT,NOTIFY service
    class GCS,MONGO,SECMGR datastore
```

---

## Data Flow Index

| Flow | From | To | Data | PHI? |
|---|---|---|---|---|
| ① | Clinician | API Gateway | Audio stream, API calls | ✅ Yes |
| ② | Patient | API Gateway | Login, consent | ⚠️ PII |
| ③ | API Gateway | Internal services | Authenticated requests | Varies |
| ④ | Audio Ingestion | GCS | Encrypted audio segments | ✅ Yes |
| ⑤ | GCS | Transcription Service | Audio file | ✅ Yes |
| ⑥ | Transcription ↔ Google STT | Raw audio / raw transcript | ✅ Yes |
| ⑦ | Transcription | MongoDB | Raw transcript | ✅ Yes |
| ⑧ | Transcription | PHI Scrubbing Layer | Raw transcript | ✅ Yes |
| ⑨ | PHI Scrubbing | AI Summarization | De-identified transcript | ❌ Tier 3 |
| ⑩ | AI Summarization ↔ Vertex AI | De-identified prompt / LLM response | ❌ Tier 3 |
| ⑪ | AI Summarization | Output Validation | Raw LLM output (untrusted) | ⚠️ Potential |
| ⑫ | Output Validation | MongoDB | Validated SOAP note draft | ✅ Yes |
| ⑬ | MongoDB ↔ Clinician Portal | SOAP note draft / clinician edits | ✅ Yes |
| ⑭ | MongoDB | EMR Integration | Approved note + approval flag | ✅ Yes |
| ⑮ | EMR Integration ↔ Epic/Cerner | FHIR DocumentReference / prior notes | ✅ Yes |
| ⑯ | EMR Integration | MongoDB | Prior notes as LLM context | ✅ Yes |
| ⑰ | MongoDB → Patient Portal → Patient | Visit summary (post-approval) | ✅ Yes |
| ⑱ | Secret Manager | Internal services | API keys, service account credentials | 🔵 Internal |
| ⑲ | Services | Notification Service → EMR | Webhooks, write-back confirmations | ⚠️ PII |

---

## Trust Boundaries at L1

| Boundary | Location | What Crosses It |
|---|---|---|
| **Internet perimeter** | Between external entities and API Gateway | Flows ①, ②, ⑰ |
| **GCP API boundary** | Between VPC and Google APIs | Flows ⑥ (PHI), ⑩ (de-identified) |
| **EMR integration boundary** | Between EMR Integration Service and Epic/Cerner | Flow ⑮ |
| **PHI scrubbing boundary** | Between PHI data and LLM submission | Flow ⑨ — critical single point of control |
| **Approval gate boundary** | Between MongoDB and EMR Integration | Flow ⑭ — enforced server-side |

---

## Key Observations at L1

1. **Flow ⑨ is the most critical control point** — the PHI scrubbing boundary between raw transcript data and Vertex AI. A failure here sends PHI to a third-party LLM API. This generates the highest-severity threat findings in the STRIDE analysis.

2. **Flow ⑯ creates an indirect injection surface** — prior EMR notes ingested as LLM context come from Epic/Cerner, a system MedScribe does not control. Adversarially crafted prior notes become an injection vector into the AI pipeline.

3. **Flow ⑭ must enforce the approval gate server-side** — the approval flag on the approved note must be validated by the EMR Integration Service, not trusted from the client. Frontend-only enforcement is bypassable via direct API calls.

4. **Secret Manager (flow ⑱) is a high-value target** — compromise of Secret Manager credentials gives an attacker access to the Vertex AI API key, the MongoDB connection string, and the EMR OAuth client secret simultaneously.
