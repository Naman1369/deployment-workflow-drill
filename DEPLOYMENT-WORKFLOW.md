# Controlled Deployment Workflow

## Overview

This document defines the controlled deployment workflow for the Checkout Service. It establishes a predictable, repeatable, auditable, and safe release movement process that replaces the current uncontrolled direct-to-production deployment.

Every release must progress through five sequential stages. No stage may be skipped or bypassed.

```
Code Commit → Build Validation → Release Approval → Deployment Execution → Health Verification → Production Success
                                                                                                    ↓ (if failure)
                                                                                                 Recovery Stage
```

---

## Stage 1: Build Validation

### Purpose

Ensure that every release candidate meets minimum quality, security, and functional correctness standards before it is eligible for approval. This stage blocks defective code from advancing through the pipeline.

### What Gets Validated

| Check | Description | Tool/Method |
|-------|-------------|-------------|
| **Build Compilation** | Source code compiles without errors | CI build step (e.g., `npm run build`, `docker build`) |
| **Unit Tests** | All unit tests pass with ≥80% code coverage | Test runner with coverage reporting |
| **Integration Tests** | Service integrations function correctly | Integration test suite against test environment |
| **Linting & Static Analysis** | Code meets style and quality standards | ESLint, SonarQube, or equivalent |
| **Security Scan** | No critical or high-severity vulnerabilities | Snyk, Trivy, or equivalent dependency/container scanner |
| **Container Build** | Docker image builds successfully and passes health probe | Docker build + container health check |
| **Staging Deployment** | Application deploys and operates correctly in staging | Automated staging deployment + smoke tests |

### Who Owns Validation

- **Primary Owner:** CI/CD Pipeline (automated execution)
- **Escalation Owner:** Build Engineer or DevOps Lead
- The CI pipeline executes all validation checks automatically on every pull request targeting `main`.
- The Build Engineer investigates and resolves validation failures.

### What Blocks Release Progression

The release **cannot advance** to the Approval Stage if any of the following occur:

- Build compilation fails
- Any unit test fails
- Code coverage drops below 80%
- Any critical or high-severity security vulnerability is detected
- Integration tests fail
- Staging smoke tests fail
- Container health probe fails

All blocking conditions produce a clear failure message with actionable remediation guidance.

---

## Stage 2: Release Approval

### Purpose

Ensure that every production release is explicitly authorized by designated approvers who have reviewed the release scope, risk, and readiness. This stage enforces separation of duties between code authors and release authorizers.

### Who Approves

| Role | Responsibility | Required |
|------|---------------|----------|
| **Tech Lead** | Reviews code changes, architecture impact, and test results | Yes (mandatory) |
| **Release Manager** | Reviews release scope, deployment plan, and rollback readiness | Yes (mandatory) |
| **QA Lead** | Confirms testing completeness and staging verification | Conditional (for major releases) |

- Minimum **two approvals** are required for every production release.
- The code author **cannot** approve their own release.

### Approval Criteria

Approvers must verify the following before granting approval:

1. All Build Validation checks have passed (green CI status)
2. Release notes are complete and accurate
3. Rollback plan is documented and attached to the release request
4. Deployment window is confirmed (no release freezes, no active incidents)
5. Release Owner is named and available for the deployment window
6. Stakeholder notification has been sent (relevant teams informed)

### Escalation Process

| Condition | Action |
|-----------|--------|
| Approver unavailable for >2 hours | Escalate to backup approver (Engineering Manager) |
| Approval disagreement between reviewers | Escalate to VP of Engineering for final decision |
| Emergency hotfix requiring expedited approval | Follow Emergency Release Procedure (minimum 1 Senior Engineer + 1 Manager approval) |
| Release blocked by open questions | Approver documents questions and blocks release until resolved |

### Approval Documentation

Every approval must be recorded with:
- Approver name and role
- Timestamp of approval
- Approval conditions or caveats (if any)
- Link to the release request and associated PR

---

## Stage 3: Deployment Execution

### Purpose

Execute the production deployment in a controlled, sequenced manner with clear ownership and defined procedures. This stage ensures that deployment follows a repeatable process with no ad-hoc manual steps.

### Deployment Sequence

```
1. Pre-deployment checklist verification
       ↓
2. Production deployment announcement (Slack/#releases channel)
       ↓
3. Enable maintenance mode / traffic drain (if applicable)
       ↓
4. Execute deployment to production (canary → rolling → full)
       ↓
5. Verify deployment command completion
       ↓
6. Advance to Health Verification Stage
```

#### Deployment Strategy: Canary → Rolling → Full

1. **Canary Phase (5% traffic):** Deploy to a single canary pod. Monitor for 5 minutes. Check error rates, latency, and key business metrics.
2. **Rolling Phase (25% → 50% → 100%):** If canary passes, incrementally roll out to remaining pods in stages with 3-minute observation windows between each stage.
3. **Full Phase:** All pods running the new version. Advance to Health Verification.

At any point during deployment, the Release Owner can halt the rollout and trigger a rollback.

### Release Ownership

| Role | Person | Responsibility |
|------|--------|---------------|
| **Release Owner** | Named engineer from the release request | Leads deployment execution, monitors rollout, makes go/no-go decisions |
| **Deployment Engineer** | On-call DevOps engineer | Executes deployment commands, monitors infrastructure health |
| **Incident Commander** (standby) | On-call Senior Engineer | Available for immediate escalation if deployment fails |

The Release Owner is the single decision-maker during deployment execution. They have the authority to:
- Proceed with the rollout
- Pause the rollout
- Trigger an immediate rollback

### Production Release Procedure

#### Pre-Deployment Checklist

Before executing deployment, the Release Owner must confirm:

- [ ] Build Validation stage passed (all green)
- [ ] Release Approval granted (two approvals documented)
- [ ] Rollback plan reviewed and ready
- [ ] Previous known-good version identified: `v____`
- [ ] Monitoring dashboards open and accessible
- [ ] On-call engineer notified and available
- [ ] Deployment announcement sent to `#releases` channel
- [ ] No active incidents or release freezes in effect

#### Deployment Commands

```bash
# 1. Verify release candidate
kubectl get deployment checkout-service -n production -o wide

# 2. Execute canary deployment
kubectl set image deployment/checkout-service \
  checkout=registry.example.com/checkout:${RELEASE_VERSION} \
  -n production --record

# 3. Monitor canary
kubectl rollout status deployment/checkout-service -n production --timeout=300s

# 4. Verify pod health
kubectl get pods -n production -l app=checkout-service
```

---

## Stage 4: Health Verification

### Purpose

Confirm that the deployed release is operating correctly in production. This stage validates that the application is healthy, performing within acceptable thresholds, and not causing customer-facing errors.

### Verification Checks

| Check | Method | Frequency | Timeout |
|-------|--------|-----------|---------|
| **Endpoint Health Probe** | HTTP GET `/health` returns 200 | Every 30 seconds | 10 seconds |
| **Readiness Check** | HTTP GET `/ready` returns 200 | Every 30 seconds | 10 seconds |
| **Key Transaction Test** | Automated smoke test executing checkout flow | Once after deployment | 60 seconds |
| **Error Rate Check** | Monitor application error rate via APM | Continuous for 10 minutes | N/A |
| **Latency Check** | Monitor p95 latency via APM | Continuous for 10 minutes | N/A |
| **Pod Stability Check** | Verify no pod restarts or CrashLoopBackOff | Continuous for 10 minutes | N/A |
| **Log Analysis** | Scan application logs for error patterns | Once at T+5 minutes | N/A |

### Success Criteria

The deployment is considered **successful** when ALL of the following conditions are met during the 10-minute observation window:

- [ ] `/health` endpoint returns HTTP 200 on all pods
- [ ] `/ready` endpoint returns HTTP 200 on all pods
- [ ] Key transaction smoke test completes successfully
- [ ] Error rate is ≤ 0.1% (or within baseline ± 0.05%)
- [ ] p95 latency is ≤ 200ms (or within baseline ± 20%)
- [ ] Zero pod restarts during observation window
- [ ] No critical error patterns in application logs

If any criterion fails, the Release Owner must immediately trigger the Recovery Stage.

### Monitoring Requirements

The following monitoring must be active and visible during and after deployment:

| Dashboard | Tool | Key Metrics |
|-----------|------|-------------|
| **Application Health** | Grafana/Datadog | Request rate, error rate, latency (p50, p95, p99) |
| **Infrastructure** | Kubernetes Dashboard | Pod status, CPU/memory usage, restart counts |
| **Business Metrics** | Custom dashboard | Checkout completion rate, order success rate |
| **Alerting** | PagerDuty/OpsGenie | Error rate spike, latency spike, pod failure alerts |

### Post-Verification Actions

Upon successful health verification:

1. Release Owner marks the deployment as **Production Success** in the release log
2. Release record is updated with health check results and timestamps
3. Deployment announcement is updated in `#releases` with success confirmation
4. Monitoring transitions from active observation to standard alerting

---

## Stage 5: Recovery Stage

### Purpose

Define the structured response process when a deployment fails at any stage. This stage ensures that failures are handled consistently, service is restored quickly, and incidents are properly documented.

### Failure Handling Process

```
Failure Detected
       ↓
Release Owner assesses severity
       ↓
  ┌─────────────┬──────────────────┐
  │             │                  │
Minor Issue   Major Issue     Critical Issue
  │             │                  │
Monitor &     Pause Rollout    Immediate
Investigate   & Investigate    Rollback
  │             │                  │
  └─────────────┴──────────────────┘
                 ↓
        Incident Report Filed
                 ↓
        Post-Incident Review Scheduled
```

#### Severity Classification

| Severity | Criteria | Response Time |
|----------|----------|---------------|
| **Critical** | Customer-facing errors, data loss, complete service failure | Immediate rollback (< 5 minutes) |
| **Major** | Degraded performance, partial feature failure, elevated error rates | Investigate within 10 minutes, rollback if unresolved in 15 minutes |
| **Minor** | Non-critical log errors, cosmetic issues, no customer impact | Monitor for 30 minutes, fix-forward if isolated |

### Rollback Trigger Conditions

A rollback **must** be triggered immediately when any of the following occur:

1. Error rate exceeds 1% for more than 2 minutes
2. p95 latency exceeds 500ms for more than 2 minutes
3. Any pod enters CrashLoopBackOff state
4. Health endpoint returns non-200 status on ≥ 50% of pods
5. Key transaction smoke test fails
6. Checkout completion rate drops by > 5% from baseline
7. Release Owner determines customer impact is occurring

### Rollback Execution

```bash
# Immediate rollback to previous version
kubectl rollout undo deployment/checkout-service -n production

# Verify rollback completion
kubectl rollout status deployment/checkout-service -n production --timeout=300s

# Confirm previous version is running
kubectl get deployment checkout-service -n production -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### Incident Ownership

| Role | Responsibility |
|------|---------------|
| **Release Owner** | Makes the rollback decision, coordinates response, updates stakeholders |
| **Deployment Engineer** | Executes rollback commands, verifies infrastructure recovery |
| **Incident Commander** | Escalated to if rollback fails or incident extends beyond 30 minutes |
| **Communication Lead** | Sends status updates to `#incidents` channel and stakeholders |

### Post-Incident Requirements

After every deployment failure:

1. Incident report filed within 24 hours
2. Root cause analysis completed within 48 hours
3. Post-incident review meeting scheduled within 1 week
4. Corrective actions documented and tracked to completion
5. Release process improvements identified and implemented

---

## Workflow Summary

| Stage | Owner | Gate | Failure Action |
|-------|-------|------|----------------|
| Build Validation | CI Pipeline / Build Engineer | All checks pass | Fix and re-submit |
| Release Approval | Tech Lead + Release Manager | Two approvals granted | Address feedback and re-request |
| Deployment Execution | Release Owner | Pre-deployment checklist complete | Halt deployment |
| Health Verification | Release Owner | All success criteria met for 10 min | Trigger Recovery |
| Recovery | Release Owner / Incident Commander | Service restored to known-good state | Escalate to VP Engineering |
