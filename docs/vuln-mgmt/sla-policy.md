# Vulnerability Management — SLA Policy

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P3

---

## 1. Purpose

This policy defines the Service Level Agreements (SLAs) for identifying, triaging,
and remediating security vulnerabilities across MedScribe-R-Us's products and
infrastructure. It establishes accountability for timely remediation and provides
the escalation framework when SLAs are at risk of being breached.

---

## 2. Severity Definitions

Severity is assigned using CVSS 3.1 base scores, modified by HIPAA contextual
factors defined in `docs/vuln-mgmt/risk-scoring.md`.

| Severity | CVSS Range | MedScribe Definition |
|---|---|---|
| 🔴 **Critical** | 9.0–10.0 | Direct PHI breach risk, authentication bypass, RCE, or any finding that constitutes a probable HIPAA breach without remediation |
| 🟠 **High** | 7.0–8.9 | Significant exploitable risk with clear attack path; PHI exposure under specific conditions |
| 🟡 **Medium** | 4.0–6.9 | Meaningful risk requiring specific conditions or attacker prerequisites to exploit |
| 🟢 **Low** | 0.1–3.9 | Limited risk; defence-in-depth improvement; no direct PHI exposure path |

### HIPAA Severity Escalation

A finding is automatically escalated one severity tier if any of the following apply:

- The vulnerability provides a path to access PHI without authorization
- The vulnerability would bypass the clinician approval gate (T-009)
- The vulnerability affects a HIPAA-required control (access control, audit logging,
  transmission security, integrity)
- A working exploit exists in the wild targeting this specific vulnerability

---

## 3. SLA Tiers

| Severity | Time to Acknowledge | Time to Remediate | Time to Verify |
|---|---|---|---|
| 🔴 Critical | 1 hour | **24 hours** | 4 hours post-fix |
| 🟠 High | 4 hours | **7 days** | 24 hours post-fix |
| 🟡 Medium | 24 hours | **30 days** | 72 hours post-fix |
| 🟢 Low | 72 hours | **90 days** | 1 week post-fix |

### Why 24 Hours for Critical?

The 24-hour Critical SLA is derived directly from HIPAA breach notification
requirements. Under 45 CFR §164.404, a Covered Entity (and by extension, its
Business Associates) must notify affected individuals of a breach "without
unreasonable delay and in no case later than 60 days." The internal discovery-to-
notification clock starts the moment a breach is suspected.

If a Critical vulnerability constitutes a potential breach, MedScribe has 60 days
for external notification — but the internal response must begin immediately.
Getting from discovery to remediation decision within 24 hours ensures the security
team has time to assess breach status, preserve forensic evidence, and begin
notification preparation if required, all before any external deadline pressure.

### SLA Clock Rules

- **Start time:** When the finding is first confirmed (not when it is assigned
  or triaged — confirmed)
- **Pause conditions:** SLA is paused only if the vendor has no available fix and
  MedScribe has implemented a documented compensating control
- **Restart conditions:** SLA restarts from zero if a new exploit is published for
  a previously paused finding
- **Extension process:** SLA extensions require written approval from the Security
  Lead and Engineering Lead, with a documented risk acceptance statement

---

## 4. Vulnerability Sources and Intake

| Source | Intake Process | Initial Severity |
|---|---|---|
| Semgrep (SAST) | Auto-creates GitHub Issue via workflow | Per rule severity |
| pip-audit / npm audit (SCA) | Auto-creates GitHub Issue via workflow | Based on CVSS |
| Trivy (container) | Auto-creates GitHub Issue via workflow | Based on CVSS |
| OWASP ZAP (DAST) | Auto-creates GitHub Issue via workflow | Based on ZAP risk |
| Internal penetration test | Manual GitHub Issue creation by AppSec | AppSec-assigned |
| External bug report | Triaged via SECURITY.md process | AppSec-assigned |
| Threat model review | Manual GitHub Issue creation by AppSec | AppSec-assigned |

All findings, regardless of source, enter the same tracking workflow in GitHub Issues.

---

## 5. Tracking and Labels

Every vulnerability tracking issue must carry the following labels:

**Severity:** `severity:critical`, `severity:high`, `severity:medium`, `severity:low`

**Source:** `source:sast`, `source:sca`, `source:dast`, `source:pentest`, `source:threat-model`

**Status:** `status:open`, `status:in-remediation`, `status:pending-verification`,
`status:accepted-risk`, `status:false-positive`, `status:resolved`

**Special flags:**
- `phi-exposure:yes` — finding has a PHI access path; triggers mandatory AppSec review
- `hipaa-control` — finding affects a HIPAA-required technical safeguard
- `sla:at-risk` — SLA breach is within 48 hours; triggers escalation notification
- `sla:breached` — SLA has been missed; auto-applied by nightly GitHub Actions workflow

### Nightly SLA Audit Workflow

A GitHub Actions workflow runs nightly at 00:00 UTC and:
1. Finds all open vulnerability issues past their SLA due date
2. Applies the `sla:breached` label
3. Posts a comment on the issue tagging the assigned engineer and security team lead
4. Includes the issue in the weekly security metrics report

---

## 6. Accepted Risk Process

If a finding cannot be remediated within SLA due to technical constraints or vendor
dependency, it may be accepted as a risk with the following requirements:

1. **Documented rationale** — written explanation of why remediation is not feasible
   within SLA
2. **Compensating controls** — description of any mitigating controls in place
3. **Re-review date** — maximum 90 days for High/Critical; 180 days for Medium/Low
4. **Approval** — Security Lead + Engineering Lead sign-off required for High/Critical

Accepted risk findings carry the `status:accepted-risk` label and a mandatory
re-review date comment. They are included in the monthly security risk register
presented to leadership.

---

## 7. Escalation Path

| Trigger | Escalation Action |
|---|---|
| Critical finding discovered | Immediate: PagerDuty → Security Lead |
| Critical SLA at 12 hours with no fix | Security Lead + VP Engineering notified |
| Critical SLA breached | Incident declared; HIPAA breach assessment initiated |
| High finding SLA at 5 days with no fix | Security Lead + Engineering Manager notified |
| High SLA breached | Security Lead review; potential risk acceptance required |
| 3+ Medium findings SLA-breached simultaneously | Security Lead + Engineering Lead notified |
| PHI-exposure finding at any severity | Mandatory AppSec review before closure |

---

## 8. Metrics

The following metrics are reported weekly to engineering leadership and monthly
to the executive team:

| Metric | Definition | Target |
|---|---|---|
| **MTTR — Critical** | Mean time to remediate Critical findings | < 20 hours |
| **MTTR — High** | Mean time to remediate High findings | < 5 days |
| **MTTR — Medium** | Mean time to remediate Medium findings | < 25 days |
| **SLA Compliance Rate** | % of findings resolved within SLA | > 95% |
| **Open Critical Count** | Number of open Critical findings at report time | 0 |
| **PHI Exposure Findings** | Count of open findings with `phi-exposure:yes` | 0 |
| **False Positive Rate** | % of findings marked false-positive by source | < 15% per tool |
| **Vulnerability Backlog Age** | Average age of open Medium/Low findings | < 45 days |

---

## 9. Policy Review

Reviewed annually or when any of the following occur:
- Material change to the platform architecture
- Change in applicable regulation
- SLA compliance rate drops below 90% for two consecutive months
- A HIPAA breach notification is issued

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | Phase 3 | AppSec (M. Del Rio) | Initial version |
