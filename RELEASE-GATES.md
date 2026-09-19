# Release Gate Standards

## Overview

Release gates are mandatory checkpoints that a release candidate must pass before advancing to the next stage of the deployment workflow. Each gate enforces specific quality, security, or governance criteria. **No gate may be bypassed** without explicit executive-level approval and documented risk acceptance.

These gates exist because the current deployment process has no validation checkpoints, which has directly caused production incidents (see `DEPLOYMENT-AUDIT.md`).

---

## Gate 1: Build Success Gate

### Purpose

Verify that the application source code compiles successfully and produces a deployable artifact (container image) without errors. This is the most fundamental quality check — if the code does not build, it cannot be deployed.

### Validation Criteria

| Criterion | Threshold | Tool |
|-----------|-----------|------|
| Source code compiles without errors | Zero compilation errors | Build system (e.g., `npm run build`, `go build`) |
| Docker container image builds successfully | Exit code 0 | `docker build` |
| Build artifact is tagged with commit SHA and version | Artifact exists in registry | Container registry (e.g., ECR, GCR) |
| Build completes within timeout | ≤ 10 minutes | CI pipeline timeout |

### Failure Consequence

- **Pipeline Action:** Release pipeline halts immediately. The release candidate is rejected.
- **Developer Action:** The developer must fix the build failure and submit a new commit.
- **Impact of Bypass:** Deploying a non-building or partially-built artifact can cause container crashes, CrashLoopBackOff states, and complete service outages. The current workflow has no build verification (`deploy.yml` contains only `echo` commands), which means broken builds could theoretically reach production.

### Responsible Owner

- **Primary:** CI/CD Pipeline (automated)
- **Escalation:** Build Engineer / DevOps Lead

### Why Bypassing Creates Risk

A failed build indicates that the code is fundamentally broken. Deploying a broken build to production guarantees a service outage. Without this gate, the team discovered broken checkout flows in production (Incident 3, 2026-04-29) because validation was skipped entirely.

---

## Gate 2: Test Coverage Gate

### Purpose

Ensure that the release candidate has been tested to an acceptable level of coverage and that all tests pass. This gate prevents untested or regression-bearing code from advancing.

### Validation Criteria

| Criterion | Threshold | Tool |
|-----------|-----------|------|
| All unit tests pass | 100% pass rate | Test runner (Jest, Mocha, pytest, etc.) |
| Code coverage meets minimum threshold | ≥ 80% line coverage | Coverage reporter (Istanbul, Coveralls) |
| No test regressions from previous release | Zero new test failures | CI diff comparison |
| Integration tests pass against staging | 100% pass rate | Integration test suite |

### Failure Consequence

- **Pipeline Action:** Release pipeline halts. The release candidate cannot proceed to the Approval Gate.
- **Developer Action:** Fix failing tests, increase coverage, or justify exceptions with Tech Lead approval.
- **Impact of Bypass:** Untested code paths can contain bugs, regressions, or unintended behavior changes that only manifest in production under real user traffic.

### Responsible Owner

- **Primary:** CI/CD Pipeline (automated test execution)
- **Escalation:** QA Lead / Tech Lead

### Why Bypassing Creates Risk

The current process has zero test requirements (see `deploy.yml`). Incident 3 documented a release that "bypassed functional validation and shipped a broken checkout flow," resulting in failed orders and manual remediation. Test coverage gates prevent this class of incident entirely.

---

## Gate 3: Security Scan Gate

### Purpose

Identify and block releases containing known security vulnerabilities in application dependencies, container base images, or application code. This gate prevents the deployment of vulnerable software to production.

### Validation Criteria

| Criterion | Threshold | Tool |
|-----------|-----------|------|
| No critical-severity vulnerabilities | Zero critical CVEs | Snyk, Trivy, or Grype |
| No high-severity vulnerabilities | Zero high CVEs (or documented exception) | Snyk, Trivy, or Grype |
| Container base image is up-to-date | Base image ≤ 30 days old | Container scanner |
| No hardcoded secrets or credentials | Zero findings | GitLeaks, TruffleHog |
| SAST scan completes without critical findings | Zero critical findings | SonarQube, Semgrep |

### Failure Consequence

- **Pipeline Action:** Release pipeline halts. The security report is attached to the release request.
- **Developer Action:** Remediate vulnerabilities (update dependencies, patch base image, remove secrets). If a vulnerability cannot be immediately remediated, a documented risk exception must be approved by the Security Lead.
- **Impact of Bypass:** Deploying vulnerable code exposes the organization to security breaches, data exfiltration, and compliance violations.

### Responsible Owner

- **Primary:** CI/CD Pipeline (automated scanning)
- **Escalation:** Security Lead / CISO

### Why Bypassing Creates Risk

The current workflow has no security scanning of any kind. A single unpatched dependency with a known exploit could lead to a data breach affecting customer payment information in the checkout service. Security gates are non-negotiable for any service handling financial transactions.

---

## Gate 4: Approval Gate

### Purpose

Ensure that every production release receives explicit human authorization from designated approvers who have reviewed the release scope, risk, readiness, and rollback plan. This gate enforces the separation of duties between code authors and release authorizers.

### Validation Criteria

| Criterion | Threshold | Tool |
|-----------|-----------|------|
| Minimum two approvals granted | ≥ 2 approvals from authorized reviewers | GitHub Environment protection rules |
| Code author has not self-approved | Author ≠ Approver | GitHub branch protection |
| All prior gates (Build, Test, Security) have passed | Green CI status | CI pipeline status checks |
| Rollback plan is attached to the release request | Document exists | Release request template |
| Release notes are complete | Changelog attached | Release request template |
| Deployment window confirmed (no freezes/incidents) | Verified by Release Manager | Release calendar |

### Failure Consequence

- **Pipeline Action:** Deployment job remains in "Waiting for approval" state indefinitely until approvals are granted.
- **Developer Action:** Address reviewer feedback, update release documentation, and re-request approval.
- **Impact of Bypass:** An unapproved release has no peer review of its risk, scope, or readiness. The team discovered through Incident 1 that unrestricted deployments cause customer-facing errors.

### Responsible Owner

- **Primary:** Tech Lead + Release Manager (human reviewers)
- **Escalation:** Engineering Manager / VP of Engineering (for emergency releases)

### Why Bypassing Creates Risk

The current process states _"No human approval step is required"_ (`current-release-process.md`). Every incident in the incident log occurred because no human reviewed or approved the release before it reached production. The Approval Gate is the single most important governance control in the entire workflow.

---

## Gate 5: Production Readiness Gate

### Purpose

Perform a final pre-deployment verification to confirm that all operational prerequisites are in place before deployment execution begins. This gate catches last-minute issues that could affect deployment safety.

### Validation Criteria

| Criterion | Threshold | Tool |
|-----------|-----------|------|
| Pre-deployment checklist completed by Release Owner | All items checked | Checklist in release request |
| Monitoring dashboards are accessible and functioning | Verified by Release Owner | Grafana/Datadog |
| On-call engineer is notified and available | Acknowledged confirmation | PagerDuty/Slack |
| Rollback commands are tested in staging | Verified in staging | Manual verification |
| Previous known-good version is identified and recorded | Version documented | Release request |
| No active production incidents | Zero active P1/P2 incidents | Incident management system |
| Deployment announcement sent | Posted to `#releases` | Slack |

### Failure Consequence

- **Pipeline Action:** Deployment execution is blocked until the Release Owner confirms all readiness criteria are met.
- **Developer Action:** Resolve outstanding readiness items and re-confirm.
- **Impact of Bypass:** Deploying without confirming operational readiness means the team may not have monitoring visibility, rollback capability, or incident response coverage during deployment.

### Responsible Owner

- **Primary:** Release Owner (named in the release request)
- **Escalation:** Release Manager / DevOps Lead

### Why Bypassing Creates Risk

Incident 2 (2026-05-12) showed that the team spent 45 minutes determining the rollback procedure because readiness was never verified before deployment. The Production Readiness Gate ensures that the team is prepared for both success and failure before any code reaches production.

---

## Gate 6: Health Verification Gate

### Purpose

Confirm that the deployed release is operating correctly in production after deployment execution. This gate validates that the application is healthy, performant, and not causing customer-facing issues before the release is marked as complete.

### Validation Criteria

| Criterion | Threshold | Tool |
|-----------|-----------|------|
| Health endpoint returns HTTP 200 on all pods | 100% of pods healthy | Kubernetes health probes |
| Readiness endpoint returns HTTP 200 on all pods | 100% of pods ready | Kubernetes readiness probes |
| Error rate within acceptable range | ≤ 0.1% (or baseline ± 0.05%) | APM (Datadog, New Relic) |
| p95 latency within acceptable range | ≤ 200ms (or baseline ± 20%) | APM |
| Zero pod restarts during observation window | Zero restarts in 10 minutes | `kubectl get pods` |
| Key transaction smoke test passes | Checkout flow completes successfully | Automated smoke test |
| No critical error patterns in logs | Zero critical log entries | Log aggregator (ELK, Loki) |

### Failure Consequence

- **Pipeline Action:** The release is marked as **failed**. The Recovery Stage is activated immediately.
- **Release Owner Action:** Assess severity and trigger rollback if customer impact is detected.
- **Impact of Bypass:** Declaring a release successful without health verification means the team will not detect silent failures, degraded performance, or partial outages until customers report them.

### Responsible Owner

- **Primary:** Release Owner (monitors health verification)
- **Escalation:** Incident Commander (if health checks fail)

### Why Bypassing Creates Risk

The deployment history shows that most "successful" deployments have no health verification records (`deployment-history.md`). Release v1.4.1 had _"no post-release verification captured"_ and v1.3.7 had _"no health verification documented."_ Without this gate, the team has no way to distinguish between a genuinely successful deployment and a deployment that is silently failing.

---

## Gate Summary

| Gate | Stage | Type | Bypass Allowed |
|------|-------|------|---------------|
| Build Success | Build Validation | Automated | **No** |
| Test Coverage | Build Validation | Automated | **No** |
| Security Scan | Build Validation | Automated | Exception only (Security Lead approval) |
| Approval | Release Approval | Human | Emergency procedure only (2 senior approvals) |
| Production Readiness | Pre-Deployment | Human + Automated | **No** |
| Health Verification | Post-Deployment | Automated + Human | **No** |

---

## Gate Bypass Policy

In extraordinary circumstances (e.g., critical security patch, active production incident requiring hotfix), a gate may be bypassed **only** with:

1. Written approval from VP of Engineering or higher
2. Documented risk acceptance
3. Compensating controls identified and implemented
4. Post-deployment review scheduled within 24 hours
5. Bypass event logged in the audit trail

All bypass events are reviewed in the monthly operational review meeting.
