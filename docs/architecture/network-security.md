# Network Security — GCP VPC Design

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P3
>
> **Addresses:** T-002 (API Gateway Bypass), T-012 (DoS via Audio), T-013 (Secret Manager Compromise)

---

## 1. Core Principle

**The API Gateway is the only internet ingress point.**

Every other service in MedScribe-R-Us runs on a private GCP VPC with no public
IP address and no internet ingress route. There is no way to reach an internal
service from the internet except through the API Gateway. This is enforced at
the GCP networking layer, not the application layer.

---

## 2. Network Topology

```
Internet
    │
    ▼
Cloud Armor WAF
(DDoS, IP reputation, WAF rules)
    │
    ▼
Global HTTPS Load Balancer
    │
    ▼
API Gateway (Cloud Run — Public ingress)
    │  Ingress: all
    │  Egress: VPC connector only
    │
    ▼ (via VPC Connector — 10.0.0.0/28)
┌─────────────────────────────────────────────────┐
│  GCP Private VPC — us-central1                  │
│  CIDR: 10.0.0.0/16                              │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Cloud Run Services (Private VPC)         │   │
│  │ Ingress: internal-and-cloud-load-bal.    │   │
│  │ Egress: VPC connector                   │   │
│  │                                          │   │
│  │  Audio Ingestion    PHI Scrubbing        │   │
│  │  Transcription      AI Summarization     │   │
│  │  Output Validation  EMR Integration      │   │
│  │  Clinician Portal   Admin Portal         │   │
│  │  Patient Portal     Notification Svc     │   │
│  └──────────────────────────────────────────┘   │
│                │                                 │
│                ▼                                 │
│  ┌─────────────────────────┐                    │
│  │ Private Service Connect │                    │
│  │ (VPC peering endpoints) │                    │
│  └────────────┬────────────┘                    │
│               │                                 │
└───────────────┼─────────────────────────────────┘
                │
    ┌───────────┴───────────┐
    │                       │
    ▼                       ▼
MongoDB Atlas            GCP Services
(Private endpoint)       (VPC Service Controls)
                         - Cloud KMS
                         - Secret Manager
                         - Vertex AI
                         - Speech-to-Text
                         - GCS
                         - Artifact Registry
```

---

## 3. Cloud Run Ingress Configuration

**Addressing T-002 — API Gateway Auth Bypass**

All Cloud Run services except the API Gateway are configured with:

```yaml
# Cloud Run service configuration (all internal services)
ingress: INGRESS_TRAFFIC_INTERNAL_LOAD_BALANCER
```

This is the GCP Cloud Run setting `internal-and-cloud-load-balancing` which means:
- Traffic from the internet: **rejected at GCP networking layer** (not app layer)
- Traffic from the VPC: **allowed**
- Traffic from Cloud Load Balancer: **allowed** (only the API Gateway sits behind the LB)

This is enforced by GCP's infrastructure. Even if a service has no auth middleware,
it cannot receive internet traffic because GCP drops it before it reaches the container.

**Automated audit:**
A GitHub Actions workflow runs on every merge to `main` and verifies all Cloud Run
services have the correct ingress setting via the GCP API:

```bash
gcloud run services describe SERVICE_NAME \
  --region=us-central1 \
  --format='value(spec.template.metadata.annotations[run.googleapis.com/ingress])'
# Expected: internal-and-cloud-load-balancing
# Any other value fails the deployment gate
```

---

## 4. VPC Service Controls Perimeter

**Addressing T-013 — Secret Manager Compromise**

VPC Service Controls create a security perimeter around sensitive GCP services.
Even if an internal service account credential is compromised, the attacker
cannot access resources outside the VPC perimeter.

**Services inside the perimeter:**
- Cloud KMS (encryption keys)
- Secret Manager (all credentials)
- Vertex AI API (LLM calls)
- GCS (audio storage)
- Artifact Registry (container images)

**Access policy:**
- Only requests originating from within the MedScribe VPC are permitted
- Requests from outside the VPC (even with valid service account credentials)
  return a `PERMISSION_DENIED` error
- All access attempts (allow and deny) logged to GCP Audit Logs

This means: if a Secret Manager credential is leaked to an external attacker,
they cannot use it from outside the VPC perimeter to access the secret value.
The credential is useless without VPC access.

---

## 5. Cloud Armor WAF Rules

Cloud Armor sits at the internet perimeter in front of the Global HTTPS Load Balancer.

**Active rule sets:**

| Rule | Action | Priority |
|---|---|---|
| OWASP CRS — SQLi | Deny 403 | 1000 |
| OWASP CRS — XSS | Deny 403 | 1001 |
| OWASP CRS — LFI/RFI | Deny 403 | 1002 |
| Log4Shell signatures | Deny 403 | 1003 |
| GCP Threat Intelligence — known malicious IPs | Deny 403 | 1004 |
| Rate limit — per IP (audio endpoint) | Throttle | 2000 |
| Rate limit — per IP (auth endpoint) | Throttle | 2001 |
| Rate limit — per tenant (global) | Throttle | 2002 |

**Addressing T-012 — DoS via Large Audio Submission:**

Two-layer defense:

1. **Cloud Armor rate limiting** — limits audio stream requests per IP to 10
   concurrent sessions and 100 new sessions per hour. Clinic-level rate limiting
   caps total concurrent sessions per tenant.

2. **Application-layer limits** (API Gateway):
   - Maximum audio session duration: 4 hours
   - Maximum audio file size on upload: 500MB
   - WebSocket sessions time out after 10 minutes of inactivity
   - Per-tenant token budget enforced for Vertex AI calls

---

## 6. Private Connectivity

### MongoDB Atlas (Private Endpoint)

MongoDB Atlas is connected via GCP Private Service Connect — no public endpoint
exists. The connection string uses the private endpoint DNS name:

```
mongodb+srv://cluster.private.mongodb.net
```

Network-level access is restricted to the MedScribe VPC CIDR (`10.0.0.0/16`)
via MongoDB Atlas IP Access List. Traffic never traverses the public internet.

### Google APIs (Private Google Access)

Cloud Run services access Google APIs (Cloud KMS, Secret Manager, Vertex AI,
Speech-to-Text) via Private Google Access — traffic routes through Google's
internal network, not the public internet, even when originating from the VPC.

---

## 7. TLS Configuration

**Minimum TLS version:** 1.3 on all external endpoints

**Certificate management:** GCP-managed certificates via Google Certificate Manager
with automatic renewal. No manual certificate management.

**Internal service communication:** All Cloud Run service-to-service calls use
mTLS via GCP Traffic Director. Service identity is established via Workload
Identity — no shared secrets or passwords between services.

**Cipher suites (TLS 1.3):** GCP manages cipher suite selection; MedScribe
relies on GCP's security policy which enforces TLS 1.3-compatible suites only:
- `TLS_AES_128_GCM_SHA256`
- `TLS_AES_256_GCM_SHA384`
- `TLS_CHACHA20_POLY1305_SHA256`

---

## 8. Firewall Rules

| Rule Name | Direction | Source | Destination | Port | Action |
|---|---|---|---|---|---|
| `allow-internal-vpc` | Ingress | `10.0.0.0/16` | All Cloud Run | 8080 | Allow |
| `allow-health-checks` | Ingress | `35.191.0.0/16, 130.211.0.0/22` | API Gateway | 443 | Allow |
| `deny-all-ingress` | Ingress | `0.0.0.0/0` | All | All | Deny (default) |
| `allow-egress-google` | Egress | All Cloud Run | Google APIs | 443 | Allow |
| `allow-egress-mongo` | Egress | All Cloud Run | MongoDB PSC endpoint | 27017 | Allow |
| `deny-all-egress` | Egress | All | `0.0.0.0/0` | All | Deny (default) |

The deny-all default rules ensure no unintentional egress path exists. A Cloud
Run service cannot make outbound calls to arbitrary internet endpoints — only
to Google APIs and the MongoDB private endpoint. This prevents SSRF exploitation
from pivoting to external infrastructure.

---

## 9. Network Security Monitoring

| Signal | Detection | Response |
|---|---|---|
| Cloud Armor rule trigger | Cloud Armor logs → Datadog alert | Auto-block IP after 5 rule hits in 1 min |
| Unusual egress traffic | VPC Flow Logs anomaly detection | Alert to security team |
| Failed Cloud Run ingress (non-VPC) | GCP Audit Logs | Alert — potential ingress misconfiguration |
| VPC Service Controls deny | GCP Audit Logs → alert | Potential credential misuse outside perimeter |
| mTLS failure between services | Cloud Run logs | Alert — potential lateral movement attempt |
