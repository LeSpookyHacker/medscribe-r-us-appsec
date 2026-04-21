# Program Maturity Assessment — OWASP SAMM

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P4
>
> **Framework:** OWASP Software Assurance Maturity Model (SAMM) v2.0

---

## Overview

This assessment scores MedScribe-R-Us's AppSec program against OWASP SAMM v2.0
at two points in time: **program inception** (before any AppSec work) and
**end of Phase 4** (current state). Target scores for Year 2 are also defined.

SAMM defines five Business Functions, each with three Security Practices,
each scored 0–3 (0 = not performed, 1 = ad hoc, 2 = defined/repeatable,
3 = optimizing/measuring).

##### Note
This SAMM assessment is my first time running through the maturity model, so there might be some discrepancies or errors. Some liberties were taken as well as this is a fictional product in a fictional company. 

---

## Scoring Summary

| Business Function | Security Practice | Inception | End P4 | Year 2 Target |
|---|---|---|---|---|
| **Governance** | Strategy & Metrics | 0 | 2 | 3 |
| | Policy & Compliance | 0 | 2 | 3 |
| | Education & Guidance | 0 | 1 | 2 |
| **Design** | Threat Assessment | 0 | 3 | 3 |
| | Security Requirements | 0 | 2 | 3 |
| | Security Architecture | 0 | 2 | 3 |
| **Implementation** | Secure Build | 0 | 2 | 3 |
| | Secure Deployment | 0 | 2 | 3 |
| | Defect Management | 0 | 2 | 3 |
| **Verification** | Architecture Assessment | 0 | 2 | 3 |
| | Requirements-driven Testing | 0 | 2 | 3 |
| | Security Testing | 0 | 2 | 3 |
| **Operations** | Incident Management | 0 | 1 | 2 |
| | Environment Management | 0 | 2 | 3 |
| | Operational Management | 0 | 1 | 2 |
| **Overall Average** | | **0.0** | **1.93** | **2.80** |

---

## Detailed Scoring by Practice

### Governance

#### Strategy & Metrics — Score: 2

**What exists:**
- AppSec program charter with defined scope and objectives
- Phase-based roadmap (P0–P4) with measurable deliverables
- Security metrics tracked: MTTR, SLA compliance, finding counts by source
- Quarterly metrics review with engineering leadership

**What's missing for Level 3:**
- Board-level security reporting
- Formal risk appetite statement from executive leadership
- Benchmarking against industry peers (BSIMM data)

---

#### Policy & Compliance — Score: 2

**What exists:**
- Data classification policy (Tier 1–5, `docs/policies/data-classification.md`)
- Regulatory context documented: HIPAA, SOC 2, HITRUST, OWASP LLM Top 10
- Vulnerability management SLA policy with defined enforcement
- Security gates policy for CI/CD pipeline
- Developer security guide with clear expectations

**What's missing for Level 3:**
- Formal annual policy review process with stakeholder sign-off
- Compliance tracking against SOC 2 controls (controls mapped but not formally tracked)
- HITRUST CSF gap analysis

---

#### Education & Guidance — Score: 1

**What exists:**
- Developer security guide (`docs/sdlc/developer-guide.md`) covering tool outputs,
  suppression policy, and escalation paths
- "Golden rules" embedded in developer onboarding documentation
- Security-focused code review comments on flagged PRs

**What's missing for Level 2:**
- Formal security training curriculum for engineers (planned for Year 2)
- Role-specific training (clinician portal engineers vs. AI pipeline engineers)
- Security champions program
- Training completion tracking

---

### Design

#### Threat Assessment — Score: 3

**What exists:**
- L0, L1, L2 DFD threat models documenting all data flows and trust boundaries
- STRIDE analysis across all 19 L1 data flows (14 findings, fully documented)
- AI-specific threat model: OWASP LLM Top 10 + MITRE ATLAS mapping
- Attack surface document enumerating all endpoints, auth mechanisms, integrations
- Threat register updated through P4 with remediation status per finding
- Threat model review process: security review required for architectural changes

**Why Level 3:** Threat modeling is integrated into the development lifecycle
(not just point-in-time), covers AI-specific threats that most AppSec programs
miss, and is documented at sufficient depth to drive specific technical controls.

---

#### Security Requirements — Score: 2

**What exists:**
- PHI handling requirements derived from data classification policy
- Authentication requirements: MFA, token lifetime, session binding
- ABAC requirements for clinical context enforcement
- LLM output handling requirements from secure AI design guide
- Security review request process for architectural changes

**What's missing for Level 3:**
- Security requirements formally integrated into product requirements system (Jira)
- Requirements traceability matrix linking security requirements to test cases
- Customer-facing security requirements (what MedScribe guarantees to health systems)

---

#### Security Architecture — Score: 2

**What exists:**
- IAM design (RBAC + ABAC, `docs/architecture/iam-design.md`)
- PHI data flow lifecycle (`docs/architecture/phi-data-flow.md`)
- Network security design (`docs/architecture/network-security.md`)
- Secure AI design guide with prompt construction, validation pipeline, supply chain
- Distroless container images, VPC Service Controls, Cloud Armor WAF

**What's missing for Level 3:**
- Reusable security architecture patterns library for new services
- Architecture review board process (currently ad hoc)
- Formal threat model update cadence for architectural changes

---

### Implementation

#### Secure Build — Score: 2

**What exists:**
- SAST: Semgrep with custom PHI-aware and auth rules + community packs
- SCA: pip-audit + npm audit + OWASP Dependency-Check + daily scheduled scans
- Secrets: Gitleaks with custom MedScribe patterns, pre-commit hook + CI gate
- Container: Trivy CVE + misconfiguration scanning with delta reports
- Custom Semgrep rules: `phi-in-logs.yml`, `auth-missing.yml`, `llm-output-handling.yml`
- Gate policy: defined thresholds with zero-tolerance for custom rule findings

**What's missing for Level 3:**
- IAST (instrumented runtime analysis) in staging environment
- Automatic PR assignment to security team for findings above threshold
- Formal suppression review workflow (currently quarterly manual review)

---

#### Secure Deployment — Score: 2

**What exists:**
- Distroless base images for all Cloud Run services
- GCP Workload Identity (no service account key files)
- Secret Manager for all credentials; no plaintext secrets in code or config
- VPC Service Controls perimeter around sensitive GCP APIs
- Automated ingress configuration audit on every deployment
- DAST (ZAP) runs post-deploy to staging with three auth contexts

**What's missing for Level 3:**
- Infrastructure-as-code security scanning (Terraform/Deployment Manager)
- Automated rollback on DAST High finding post-deployment
- Canary deployments with security monitoring before full rollout

---

#### Defect Management — Score: 2

**What exists:**
- GitHub Issues as unified vulnerability tracker
- Mandatory label taxonomy: severity, source, status, PHI exposure
- SLA tiers with automated breach detection (nightly workflow)
- Accepted risk process with documented rationale and re-review dates
- Metrics dashboard tracking MTTR, SLA compliance, false positive rates
- Escalation paths for Critical findings (PagerDuty)

**What's missing for Level 3:**
- Root cause analysis on all Critical findings (trend identification)
- Vulnerability KPIs in engineering OKRs (currently security-team-only)
- Formal defect aging heatmap with team-level accountability

---

### Verification

#### Architecture Assessment — Score: 2

**What exists:**
- DFDs reviewed against implementation (P1 → P3 cross-check)
- Attack surface document with all endpoints, auth mechanisms, integrations
- Internal penetration testing scope defined in attack surface document
- Security review required for changes to attack surface

**What's missing for Level 3:**
- Formal annual third-party architecture review
- Purple team exercises (collaborative red/blue team assessments)
- Automated architecture drift detection

---

#### Requirements-driven Testing — Score: 2

**What exists:**
- Prompt injection test suite (TC-001–TC-010) mapped to specific threat findings
- DAST admin-authenticated scan specifically targeting T-011 (privilege escalation)
- PHI scrubber validation against labeled test corpus
- Token budget enforcement tests
- Re-injection audit tests for T-008

**What's missing for Level 3:**
- Full requirements traceability: every security requirement has at least one test
- Automated security regression run on every deployment (currently CI-triggered)
- Formal test coverage reporting for security requirements

---

#### Security Testing — Score: 2

**What exists:**
- Five-tool automated security testing pipeline (SAST, SCA, Secrets, Container, DAST)
- AI-specific test suite: 10 test cases covering direct injection, indirect
  injection, and output manipulation
- Custom Semgrep rules with 0% false positive rate
- Authenticated DAST in three role contexts
- Bug bounty program (private, invitation-only)

**What's missing for Level 3:**
- Annual third-party penetration test with formal report
- Continuous bug bounty (currently private/invitation-only)
- Red team exercises (adversary simulation)

---

### Operations

#### Incident Management — Score: 1

**What exists:**
- Escalation paths defined in SLA policy
- PagerDuty integration for Critical findings
- Breach assessment process initiated on Critical security events
- Security team on-call defined

**What's missing for Level 2:**
- Documented incident response playbook with specific scenarios
  (PHI breach, credential compromise, prompt injection confirmed exploitation)
- Tabletop exercises (planned for Year 2)
- Post-incident review process
- HIPAA breach notification workflow automation

---

#### Environment Management — Score: 2

**What exists:**
- Separate GCP projects for dev, staging, production
- VPC Service Controls applied in production
- Distroless images, no shell access in production containers
- Secret Manager; no hardcoded credentials
- Production access via GCP IAM with audit logging
- Automated security configuration audit on deployment

**What's missing for Level 3:**
- Formal change management process for production environment changes
- Infrastructure-as-code with automated security policy enforcement
- Production access just-in-time (JIT) provisioning

---

#### Operational Management — Score: 1

**What exists:**
- GCP Cloud Logging + Datadog for operational monitoring
- Anomaly detection in DAST authenticated scan
- PHI access audit logging with HMAC integrity
- VPC Flow Logs for network anomaly detection

**What's missing for Level 2:**
- Security Information and Event Management (SIEM) integration
- Threat hunting playbooks for the AI pipeline attack surface
- Formal operational security review cadence

---

## Year 2 Priorities

Based on the gap analysis above, the following initiatives are highest priority
for reaching the Year 2 target scores:

1. **Incident response playbook** — document specific scenarios for PHI breach,
   credential compromise, and prompt injection exploitation (Operations: 1 → 2)

2. **Security training curriculum** — role-specific security training for engineers
   with completion tracking (Education: 1 → 2)

3. **Annual third-party penetration test** — external validation of the controls
   built across P1–P4 (Security Testing: 2 → 3)

4. **SIEM integration** — centralized security event management connecting
   audit logs, DAST findings, and anomaly detection (Operational: 1 → 2)

5. **SOC 2 Type II observation period** — begin formal compliance evidence
   collection (Policy & Compliance: 2 → 3)
