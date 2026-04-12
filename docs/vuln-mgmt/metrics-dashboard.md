# Security Metrics Dashboard — MedScribe-R-Us

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P3
>
> **Reporting Period:** Program inception through end of P3
> **Audience:** Engineering leadership, Executive team

---
### Note: This file is populated with both real metrics (such as the threats from P1 being addressed) but also plausible fake metrics, the idea is that this metric dashboard will be viewed by internall app sec engineers once the AppSec program has been rolled out in stages

## Program Health Summary

| Metric | Current | Target | Status |
|---|---|---|---|
| Open Critical findings | 0 | 0 | ✅ |
| Open PHI-exposure findings | 0 | 0 | ✅ |
| SLA compliance rate (all severity) | 91% | ≥ 95% | ⚠️ |
| MTTR — Critical | 18.5 hrs | < 20 hrs | ✅ |
| MTTR — High | 4.8 days | < 5 days | ✅ |
| MTTR — Medium | 22 days | < 25 days | ✅ |
| False positive rate (SAST) | 12% | < 15% | ✅ |
| False positive rate (DAST) | 8% | < 15% | ✅ |

> **SLA compliance note:** The 91% rate reflects the program's first 90 days.
> Three Medium findings from the initial threat model review were not resolved
> within SLA due to engineering resource constraints during a product launch.
> All three have since been remediated. Target of ≥ 95% is the steady-state goal.

---

## Finding Counts by Phase

### P1 — Threat Model (14 findings identified)

| Severity | Identified | Resolved | In Remediation | Accepted Risk |
|---|---|---|---|---|
| 🔴 Critical | 9 | 9 | 0 | 0 |
| 🟠 High | 3 | 3 | 0 | 0 |
| 🟡 Medium | 2 | 2 | 0 | 0 |
| 🟢 Low | 0 | — | — | — |
| **Total** | **14** | **14** | **0** | **0** |

### P2 — CI/CD Pipeline (findings surfaced by tooling in first 30 days)

| Source | Findings | Resolved | FP | Accepted |
|---|---|---|---|---|
| Semgrep — custom rules | 7 | 7 | 0 | 0 |
| Semgrep — community | 23 | 19 | 4 | 0 |
| pip-audit (Python) | 4 | 4 | 0 | 0 |
| npm audit (Node) | 2 | 2 | 0 | 0 |
| Gitleaks | 1 | 1 | 0 | 0 |
| Trivy — CVEs | 11 | 10 | 0 | 1 |
| Trivy — misconfigs | 3 | 3 | 0 | 0 |
| ZAP — unauthenticated | 6 | 5 | 1 | 0 |
| ZAP — clinician auth | 4 | 4 | 0 | 0 |
| ZAP — admin auth | 3 | 3 | 0 | 0 |
| **Total** | **64** | **58** | **5** | **1** |

**Notable findings:**
- The Gitleaks finding was a GCP service account key fragment in a comment block
  dating from an early development sprint. The key had already been rotated but
  was not cleaned from history. Git history was cleaned; no breach assessment required.
- The accepted-risk Trivy finding is a CVSS 4.1 CVE in `libexpat` affecting an
  XML parsing code path not exercised by any MedScribe service. Re-review: 90 days.

---

## MTTR Trend — Trailing 90 Days

| Week | Critical MTTR | High MTTR | Medium MTTR | SLA Compliance |
|---|---|---|---|---|
| Week 1–2 (P1 findings) | 21.2 hrs | 6.1 days | 28 days | 86% |
| Week 3–4 | 19.8 hrs | 5.4 days | 24 days | 89% |
| Week 5–6 (P2 pipeline active) | 17.1 hrs | 4.9 days | 21 days | 91% |
| Week 7–8 | 16.3 hrs | 4.7 days | 22 days | 93% |
| Week 9–10 (P3 controls live) | 15.9 hrs | 4.2 days | 20 days | 95% |
| **Current (Week 11–12)** | **18.5 hrs** | **4.8 days** | **22 days** | **91%** |

> Week 11–12 SLA compliance dip reflects three Medium findings from the ZAP
> admin-authenticated scan that entered the queue simultaneously. Engineering
> capacity was allocated to a feature release. Findings are now resolved.

---

## Vulnerability Aging — Open Findings

As of end of P3, zero open Critical or High findings. The open backlog:

| Finding | Source | Severity | Age | SLA Due | Status |
|---|---|---|---|---|---|
| Missing rate limiting on `/api/v1/audio/stream` per-IP | ZAP | Medium | 18 days | Day 30 | In remediation |
| Server version disclosure in error responses | ZAP | Low | 44 days | Day 90 | In remediation |
| `libexpat` CVE in frontend image | Trivy | Low | 31 days | Day 90 | Accepted risk |

---

## Tool Performance — Signal vs. Noise

| Tool | Total Findings | True Positives | False Positives | FP Rate |
|---|---|---|---|---|
| Semgrep — custom rules | 7 | 7 | 0 | 0% |
| Semgrep — community | 23 | 19 | 4 | 17% |
| pip-audit | 4 | 4 | 0 | 0% |
| npm audit | 2 | 2 | 0 | 0% |
| Gitleaks | 1 | 1 | 0 | 0% |
| Trivy — CVEs | 11 | 10 | 0 | 0% (1 accepted risk) |
| Trivy — misconfigs | 3 | 3 | 0 | 0% |
| ZAP — all contexts | 13 | 12 | 1 | 8% |

**Observations:**
- Custom Semgrep rules have 0% false positive rate — confirms the rules are
  tightly scoped to actual MedScribe code patterns
- Community Semgrep rules have a 17% FP rate, slightly above the 15% target.
  Four false positives are in the JWT community ruleset flagging a non-standard
  but intentional JWT implementation pattern. A suppression allowlist has been
  added; FP rate will normalize to ~9% in next reporting period
- ZAP's single false positive was "Cookie Same-Site attribute not set" on a
  read-only session cookie with no sensitive data — added to `.zap/rules.tsv`

---

## HIPAA Audit Readiness Indicators

| HIPAA Requirement | Control | Evidence Location | Status |
|---|---|---|---|
| §164.308(a)(1) — Risk Analysis | Threat model + risk scoring | `docs/threat-model/` | ✅ Complete |
| §164.308(a)(6) — Incident Procedures | Vulnerability escalation path | `docs/vuln-mgmt/sla-policy.md` | ✅ Complete |
| §164.312(a)(1) — Access Control | IAM design + ABAC | `docs/architecture/iam-design.md` | ✅ Complete |
| §164.312(b) — Audit Controls | Audit logging design | `docs/architecture/phi-data-flow.md` | ✅ Complete |
| §164.312(c)(1) — Integrity | Audio hash + MongoDB controls | `docs/architecture/system-overview.md` | ✅ Complete |
| §164.312(e)(1) — Transmission Security | TLS 1.3 + VPC controls | `docs/architecture/network-security.md` | ✅ Complete |

---

## Next Reporting Period Targets

| Metric | Current | Next Period Target |
|---|---|---|
| SLA compliance rate | 91% | ≥ 95% |
| Open Medium findings | 2 | 0 |
| Semgrep community FP rate | 17% | ≤ 12% |
| MTTR — Critical | 18.5 hrs | ≤ 16 hrs |
| HIPAA audit readiness | 6/6 controls | Maintain + add P4 AI controls |
