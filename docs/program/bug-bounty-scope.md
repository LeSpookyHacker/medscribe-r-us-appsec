# Bug Bounty Program — Scope & Policy

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P4

---

## Program Overview

MedScribe-R-Us operates a private bug bounty program for security researchers.
We are committed to working with the security community to identify and remediate
vulnerabilities before they affect our customers and their patients.

This is a **private program** — invitations are extended to vetted researchers only.
Unsolicited reports that follow this policy will still be reviewed and rewarded
if they meet the criteria below.

---

## Safe Harbor

MedScribe-R-Us will not pursue legal action against security researchers who:

- Act in good faith to identify and report vulnerabilities
- Do not access, modify, or exfiltrate real patient data
- Use only test accounts and synthetic data provided by MedScribe-R-Us
- Do not perform denial of service, social engineering, or physical attacks
- Report findings to us before public disclosure and allow 90 days for remediation
- Do not share findings with third parties before coordinated disclosure

We consider good-faith security research conducted within this policy to be
authorized access under the Computer Fraud and Abuse Act and equivalent laws.

---

## In Scope

### Web Applications

| Target | URL | Auth Required |
|---|---|---|
| Clinician Portal | `https://app.medscribe-r-us.fake` | Yes — test account provided |
| Patient Portal | `https://patient.medscribe-r-us.fake` | Yes — test account provided |
| Admin Portal | `https://admin.medscribe-r-us.fake` | Yes — test Clinic Admin account |
| REST API | `https://api.medscribe-r-us.fake/api/v1/` | Yes — test JWT provided |
| WebSocket Audio Endpoint | `wss://api.medscribe-r-us.fake/api/v1/audio/stream` | Yes |

### Specifically High Interest

The following vulnerability classes are of particular interest and eligible
for maximum reward tiers:

- **Cross-tenant data access** — any mechanism by which a Clinic Admin or
  clinician accesses another health system's patient data
- **PHI exposure without authentication** — any endpoint that returns patient
  data without a valid, properly scoped JWT
- **Approval gate bypass** — any mechanism by which an AI-generated note
  reaches the EMR without explicit clinician approval
- **FHIR write-back patient spoofing** — writing a SOAP note to the wrong
  patient's EMR record
- **Prompt injection** — direct or indirect injection that produces output
  materially different from the clinical conversation content
- **Authentication bypass** — any mechanism to obtain a valid session without
  valid credentials
- **Privilege escalation** — any mechanism to access resources beyond the
  scope of the authenticated role

---

## Out of Scope

The following are explicitly out of scope. Testing against these may result
in removal from the program:

- Google Cloud Platform infrastructure (AWS, GCP, Azure shared responsibility)
- Epic and Cerner EMR systems (customer infrastructure)
- MedScribe-R-Us corporate network, employee endpoints, or internal tooling
- Social engineering of MedScribe-R-Us employees
- Physical attacks against MedScribe-R-Us facilities
- Denial of service or volumetric attacks
- Automated scanning without prior written approval from the security team
- Testing against production systems or real patient data (test environment only)
- Vulnerabilities requiring physical access to a clinician's device
- Self-XSS or issues requiring a user to execute malicious code in their own browser
- Rate limiting issues without a demonstrated security impact
- Missing security headers without a demonstrated exploitable vector
- TLS configuration issues on non-critical endpoints
- Software version disclosure without exploitability demonstrated
- Clickjacking on pages with no sensitive actions

---

## Reward Tiers

Rewards are based on the effective severity score (CVSS + contextual modifiers,
per `docs/vuln-mgmt/risk-scoring.md`) and the quality of the report.

| Effective Severity | Reward Range |
|---|---|
| 🔴 Critical (9.0–10.0) | $5,000 – $15,000 |
| 🟠 High (7.0–8.9) | $1,500 – $5,000 |
| 🟡 Medium (4.0–6.9) | $250 – $1,500 |
| 🟢 Low (0.1–3.9) | $50 – $250 |
| Informational | Recognition only (Hall of Fame) |

### Reward Modifiers

| Condition | Modifier |
|---|---|
| First report of this vulnerability class | +25% |
| Proof-of-concept exploit provided | +15% |
| Fix verified by researcher | +10% |
| Incomplete report requiring significant investigation | -25% |
| Duplicate finding | $0 (recognition only) |

### Specifically High Interest Bonus

Findings in the "Specifically High Interest" category above receive an
additional $2,000 bonus on top of the standard reward tier, if confirmed.

---

## Submission Requirements

A complete report must include:

1. **Vulnerability description** — what the vulnerability is and why it is a
   security risk
2. **Steps to reproduce** — precise, numbered steps a MedScribe engineer can
   follow to reproduce the finding in our test environment
3. **Impact statement** — what an attacker could achieve by exploiting this
   vulnerability; what data could be accessed or modified
4. **Affected endpoints or components** — specific URLs, API routes, or
   system components
5. **Proof of concept** — screenshots, HTTP request/response logs, or a
   working exploit (using test data only)
6. **Suggested remediation** — optional but appreciated

Reports missing steps to reproduce or proof of concept will be asked for
clarification before triage begins.

---

## Response Commitments

| Milestone | Target Timeframe |
|---|---|
| Acknowledgment of report | 1 business day |
| Initial triage and severity assessment | 3 business days |
| Remediation timeline communicated | 7 business days |
| Patch for Critical/High | 24 hours / 7 days (per SLA policy) |
| Researcher notification of patch | Within 2 business days of fix |
| Coordinated public disclosure | 90 days after initial report |
| Reward payment | Within 14 days of validation |

---

## Test Account Provisioning

Test accounts for each role are provisioned on request. To request test
credentials:

1. Email `security@medscribe-r-us.fake` with subject: `Bug Bounty Test Account Request`
2. Include your researcher name/handle and the role(s) you need
3. Test accounts are provisioned within 2 business days
4. Test accounts operate against a synthetic patient dataset — no real PHI

**Important:** Test credentials are for your use only. Do not share test
accounts with other researchers. Account sharing is grounds for removal from
the program.

---

## Disclosure Policy

MedScribe-R-Us follows a coordinated disclosure model:

- **90-day disclosure deadline:** We commit to patching Critical and High findings
  within SLA. If we cannot patch within 90 days, we will communicate with the
  researcher about an extension or accept a public disclosure with a remediation
  timeline.
- **Premature disclosure:** Public disclosure before 90 days without MedScribe-R-Us
  consent voids the safe harbor and may affect reward eligibility.
- **Hall of Fame:** Researchers who submit valid findings are credited in our
  public Hall of Fame unless they request anonymity.

---

## Hall of Fame

*The following researchers have responsibly disclosed security vulnerabilities
to MedScribe-R-Us. We are grateful for their contributions to patient data
security.*

*(Hall of Fame entries will appear here as the program receives validated submissions.)*
