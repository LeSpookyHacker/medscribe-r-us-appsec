# Security Gates Policy — MedScribe-R-Us CI/CD Pipeline

> **Fictional company — portfolio/educational purposes only.**
>
> **Document Status:** Draft v1.0 | Owner: AppSec | Phase: P2

---

## Overview

This document defines the security gate thresholds for MedScribe-R-Us's CI/CD
pipeline. Gates are the enforcement mechanism that translates threat model
findings and vulnerability policy into build-time controls.

A "gate" blocks a PR merge or a deployment until its condition is resolved.
Gates are distinct from "signals" — signals generate notifications or issues
but do not block.

---

## Gate Summary

| Tool | Trigger | Gate Condition | On Failure |
|---|---|---|---|
| Semgrep (custom rules) | Every PR | Any finding | Block PR merge |
| Semgrep (community) | Every PR | ERROR severity on `main` | Block merge to `main` |
| pip-audit / npm audit | Every PR | CVSS ≥ 9.0 with fix available | Block PR merge |
| OWASP Dependency-Check | Every PR | Critical with known exploit | Block PR merge |
| Gitleaks | Every push | Any secret detected | Block PR merge + alert |
| Trivy (image CVEs) | Push to `main`/`staging` | Critical or High with fix | Block deployment |
| Trivy (misconfigurations) | Push to `main`/`staging` | Critical or High | Block deployment |
| OWASP ZAP (baseline) | Post-deploy to `staging` | High severity alert | Block next promotion |
| OWASP ZAP (authenticated) | Post-deploy to `staging` | High severity alert | Block next promotion |

---

## Tool-by-Tool Gate Logic

### 1. Semgrep (SAST)

**Custom rules** (`semgrep-rules/` directory):

Custom rules are MedScribe-specific and cover PHI-in-logs patterns, missing
FastAPI auth dependencies, tenant scope bypass, and insecure LLM output
handling. These rules are tuned to this codebase — false positive tolerance
is zero.

- **Any finding at any severity → block PR merge**
- Suppression requires `# nosemgrep: rule-id` with a comment explaining why
  the finding is a false positive or accepted risk
- Suppressions are reviewed quarterly by the security team

**Community rules** (`p/python`, `p/javascript`, `p/owasp-top-ten`, `p/jwt`):

- `ERROR` severity → block merge to `main`; create GitHub issue for feature branches
- `WARNING` severity → create GitHub issue with 30-day SLA; does not block
- `INFO` severity → surfaced in SARIF dashboard only; no action required

---

### 2. SCA — pip-audit / npm audit / OWASP Dependency-Check

**Gate threshold:** CVSS base score ≥ 9.0 with an available fix

The ≥ 9.0 threshold targets Critical CVEs only. This keeps the gate signal-to-noise
ratio high — the gate fires on findings that are both severe and actionable.

**Important:** A CVSS 8.x CVE in a package that handles PHI (e.g., the MongoDB
driver, the FHIR client, the JWT library) is escalated to the same urgency as a
CVSS 9.0+ finding. PHI-touching libraries are subject to a tighter threshold
enforced by the security team during triage, not by automated gate logic.

**Unfixed CVEs:** If no fix is available, the gate does not block but a GitHub
issue is created with the `sla:accepted-risk` label and a 90-day re-review date.
The security team must document why the unfixed CVE is acceptable for the
MedScribe threat model.

**Scheduled scans:** SCA also runs daily at 06:00 UTC against pinned dependency
manifests. This catches newly published CVEs against existing dependencies between
PRs. Daily scan findings that exceed gate threshold generate a P1 GitHub issue
for immediate triage.

---

### 3. Secrets Detection — Gitleaks

**Gate threshold:** Any detected secret, regardless of severity

There is no warn-only mode for secrets. A detected secret means:

1. The credential must be assumed compromised immediately
2. Rotation is required before any merge is permitted
3. The security team must be notified
4. Git history must be cleaned

The gate fires on the full commit history, not just the PR diff. A secret
committed three months ago and deleted in a later commit is still in the
repository and still requires rotation.

**False positive handling:** The `.gitleaks.toml` allowlist handles known
false positive patterns (GitHub Actions `${{ secrets.* }}` syntax, placeholder
values in docs). Engineers should not disable the tool to resolve a false
positive — update the allowlist with a PR that the security team reviews.

---

### 4. Container Scanning — Trivy

**Gate threshold:** Critical or High CVE with an available fix

Trivy scans two surfaces per image:

- **OS packages** — vulnerabilities in the base image's OS-level packages
- **Library vulnerabilities** — CVEs in Python/Node packages installed in the image
- **Misconfigurations** — Dockerfile security issues (running as root, secrets in
  layers, exposed ports, missing USER directive)

**Distroless recommendation:** Both the backend (Python FastAPI) and frontend
(Next.js) services use distroless base images. Distroless removes the shell,
package manager, and most OS-level packages — significantly reducing the CVE
surface that Trivy scans and making the images harder to use as an attacker
foothold post-exploitation.

**Delta report:** Every PR to `main` receives a comment showing only the CVEs
*introduced* by that PR, compared to the currently deployed image. This keeps
review focused on new risk rather than pre-existing backlog.

---

### 5. DAST — OWASP ZAP

**Gate threshold:** High severity ZAP alert blocks the next environment promotion

DAST runs post-deploy to staging, not on PR. It requires a running application
to test against. Three scan contexts:

- **Unauthenticated baseline** — security headers, error disclosure, common injections
- **Authenticated (clinician role)** — PHI access controls, note approval flow
- **Authenticated (admin role)** — tenant isolation (T-011), privilege escalation

**Medium severity findings** do not block but auto-create GitHub Issues with the
`severity:medium`, `source:dast`, `status:open` labels and a 30-day SLA
(aligned to the vulnerability management policy).

**ZAP rules configuration** (`.zap/rules.tsv`): Some ZAP alerts are explicitly
set to IGNORE for MedScribe-R-Us. For example, "Cookie No HttpOnly Flag" is
set to IGNORE for the WebSocket audio stream session cookie because the
HttpOnly flag cannot be applied to cookies used by JavaScript WebSocket clients.
All IGNORE decisions are documented in the rules file with a justification comment.

---

## Suppression Policy

**Semgrep suppressions:**

```python
# nosemgrep: phi-variable-in-logger
# Justification: `session_id` variable here contains only a UUID —
# no PHI value. Confirmed by code review 2024-01-15 (M. Del Rio).
logger.info("Session completed", session_id=session_id)
```

**Trivy suppressions** (`.trivyignore`):

```
# CVE-2024-XXXXX
# Justification: Affects only the `urllib3` ProxyManager class.
# MedScribe does not use HTTP proxies in any Cloud Run service.
# Re-review date: 2024-04-01
CVE-2024-XXXXX
```

All suppressions are reviewed quarterly. A pattern of suppressions in the same
area of code is a signal for a deeper security review of that component.

---

## Escalation Path

| Event | Escalation |
|---|---|
| Secrets gate fires | Immediate: security team lead notified via PagerDuty |
| Critical CVE gate fires | Same business day: security team + engineering lead |
| High CVE gate fires | Next business day: engineering team |
| DAST High finding | Same business day: security team |
| SLA breach (any finding) | Weekly security metrics review + team lead notification |
