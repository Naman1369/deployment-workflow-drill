# Deployment Workflow Diagram

## Overview

This document presents the complete deployment workflow as a visual diagram. It shows the release movement from code commit through production success, including all approval points, validation checkpoints, and recovery paths.

---

## Primary Deployment Flow

```
┌──────────────────┐
│   Code Commit    │
│  (PR to main)    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐      ┌─────────────────────────┐
│ Build Validation │─No──▶│  Validation Failure     │
│                  │      │  ─ Notify developer      │
│ • Build compile  │      │  ─ Block PR merge        │
│ • Unit tests     │      │  ─ Fix and resubmit      │
│ • Integration    │      └─────────────────────────┘
│ • Security scan  │
│ • Coverage check │
└────────┬─────────┘
         │ Pass
         ▼
┌──────────────────┐      ┌─────────────────────────┐
│ Approval Gate    │─No──▶│  Approval Rejected      │
│                  │      │  ─ Address feedback       │
│ • Tech Lead      │      │  ─ Update release notes   │
│ • Release Mgr    │      │  ─ Re-request approval    │
│ • 2 approvals    │      └─────────────────────────┘
│ • Rollback plan  │
└────────┬─────────┘
         │ Approved
         ▼
┌──────────────────┐      ┌─────────────────────────┐
│ Production       │─No──▶│  Readiness Check Failed │
│ Readiness Gate   │      │  ─ Resolve blockers      │
│                  │      │  ─ Re-verify readiness    │
│ • Checklist done │      └─────────────────────────┘
│ • Monitoring up  │
│ • On-call ready  │
│ • No incidents   │
└────────┬─────────┘
         │ Ready
         ▼
┌──────────────────┐      ┌─────────────────────────┐
│ Deployment       │─Fail▶│  Deployment Failure     │
│                  │      │  ─ Halt rollout           │
│ • Canary (5%)    │      │  ─ Trigger rollback       │
│ • Rolling (25%)  │      │  ─ See Recovery Path      │
│ • Rolling (50%)  │      └───────────┬─────────────┘
│ • Full (100%)    │                  │
└────────┬─────────┘                  │
         │ Success                    │
         ▼                            │
┌──────────────────┐      ┌───────────▼─────────────┐
│ Health           │─Fail▶│  Health Check Failure   │
│ Verification     │      │  ─ Assess severity       │
│                  │      │  ─ Trigger rollback       │
│ • /health 200    │      │  ─ See Recovery Path      │
│ • /ready 200     │      └───────────┬─────────────┘
│ • Error rate OK  │                  │
│ • Latency OK     │                  │
│ • No restarts    │                  ▼
│ • Smoke test OK  │      ┌─────────────────────────┐
└────────┬─────────┘      │   ROLLBACK TRIGGER      │
         │ Pass           │                         │
         ▼                │ kubectl rollout undo    │
┌──────────────────┐      │ deployment/checkout-    │
│  ✅ Production   │      │ service -n production   │
│     Success      │      │                         │
│                  │      │  ─ Verify restoration    │
│ • Release logged │      │  ─ File incident report  │
│ • Team notified  │      │  ─ Post-incident review  │
│ • Monitoring on  │      └─────────────────────────┘
└──────────────────┘
```

---

## Failure Paths Detail

### Failure Path 1: Validation Failure

```
Build Validation Stage
        │
        ▼
   ┌─────────┐
   │  FAIL   │
   └────┬────┘
        │
        ▼
┌──────────────────────────────────────────────────┐
│ Validation Failure Response                      │
├──────────────────────────────────────────────────┤
│ 1. CI pipeline reports failure with details      │
│ 2. PR is blocked from merging                    │
│ 3. Developer is notified via GitHub notification │
│ 4. Developer fixes the issue                     │
│ 5. New commit triggers re-validation             │
│ 6. Cycle repeats until all checks pass           │
└──────────────────────────────────────────────────┘
        │
        ▼
  Return to Build Validation
```

**What triggers it:** Build compilation error, test failure, security vulnerability, coverage drop below 80%.

**Who responds:** Developer (fixes code), CI Pipeline (re-runs checks).

**Impact:** Release is delayed until code meets quality standards. No production impact since the code never leaves the build stage.

---

### Failure Path 2: Deployment Failure

```
Deployment Execution Stage
        │
        ▼
   ┌─────────┐
   │  FAIL   │
   └────┬────┘
        │
        ▼
┌──────────────────────────────────────────────────┐
│ Deployment Failure Response                      │
├──────────────────────────────────────────────────┤
│ 1. Release Owner detects failure                 │
│    (pod crash, image pull error, timeout)         │
│ 2. Rollout is halted immediately                 │
│ 3. Announcement posted to #incidents             │
│ 4. Rollback command executed:                    │
│    kubectl rollout undo deployment/              │
│    checkout-service -n production                │
│ 5. Rollback verified (pods healthy, version OK)  │
│ 6. Service restoration confirmed                 │
│ 7. Incident report filed within 24 hours         │
└──────────────────────────────────────────────────┘
        │
        ▼
  Incident Report → Post-Incident Review
```

**What triggers it:** Container image pull failure, pod CrashLoopBackOff, Kubernetes resource limits exceeded, configuration errors, timeout during rollout.

**Who responds:** Release Owner (makes rollback decision), Deployment Engineer (executes rollback), Incident Commander (if escalation needed).

**Impact:** Partial production impact during the failed rollout window. Impact is minimized by canary deployment strategy (only 5% traffic affected initially).

---

### Failure Path 3: Health Check Failure

```
Health Verification Stage
        │
        ▼
   ┌─────────┐
   │  FAIL   │
   └────┬────┘
        │
        ▼
┌──────────────────────────────────────────────────┐
│ Health Check Failure Response                    │
├──────────────────────────────────────────────────┤
│ 1. Health verification reports failure           │
│    (error rate, latency, endpoint down)           │
│ 2. Release Owner assesses severity:              │
│    ┌─────────────────────────────────────┐       │
│    │ Critical → Immediate rollback       │       │
│    │ Major    → 15-min investigation     │       │
│    │ Minor    → Monitor and fix-forward  │       │
│    └─────────────────────────────────────┘       │
│ 3. If rollback: execute recovery procedure       │
│ 4. If fix-forward: deploy patch through          │
│    full pipeline (no gate bypass)                │
│ 5. Verify service health after response          │
│ 6. Document outcome and lessons learned          │
└──────────────────────────────────────────────────┘
        │
        ▼
  Incident Report → Post-Incident Review
```

**What triggers it:** Elevated error rates (>1%), latency spike (p95 >500ms), pod restarts, failed smoke tests, customer-reported issues.

**Who responds:** Release Owner (severity assessment, rollback decision), Deployment Engineer (executes rollback), Communication Lead (stakeholder updates).

**Impact:** Production is impacted — this is a post-deployment failure. Customer transactions may be failing. Speed of response directly determines incident severity and duration.

---

### Failure Path 4: Rollback Trigger

```
   Any Stage Post-Deployment
        │
        ▼
┌───────────────────┐
│ ROLLBACK TRIGGER  │
│                   │
│ Error rate > 1%   │
│ Latency > 500ms   │
│ Pod crash loops    │
│ Health probe fail  │
│ Smoke test fail    │
│ Business metrics ↓ │
└────────┬──────────┘
         │
         ▼
┌──────────────────────────────────────────────────┐
│ Rollback Execution                               │
├──────────────────────────────────────────────────┤
│                                                  │
│ Tier 1: Standard Rollback                        │
│   kubectl rollout undo deployment/               │
│   checkout-service -n production                 │
│        │                                         │
│        ▼ (if Tier 1 fails)                       │
│ Tier 2: Force Version Deploy                     │
│   kubectl set image deployment/checkout-service  │
│   checkout=[KNOWN_GOOD_IMAGE] -n production      │
│        │                                         │
│        ▼ (if Tier 2 fails)                       │
│ Tier 3: Scale Down and Redeploy                  │
│   kubectl scale deployment/checkout-service      │
│   -n production --replicas=0                     │
│   kubectl scale deployment/checkout-service      │
│   -n production --replicas=[COUNT]               │
│                                                  │
└────────┬─────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────┐
│ Post-Rollback Verification                       │
├──────────────────────────────────────────────────┤
│ 1. Confirm previous version is running           │
│ 2. All health endpoints return 200               │
│ 3. Error rate returns to baseline                │
│ 4. Latency returns to baseline                   │
│ 5. Key transactions complete successfully        │
│ 6. Announce resolution to #incidents             │
└──────────────────────────────────────────────────┘
         │
         ▼
   Incident Report → Root Cause Analysis → Process Improvement
```

---

## Complete Flow Summary

```
                        ┌──────────────┐
                        │  Code Commit │
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐     ┌──────────────┐
                        │    Build     │────▶│  Fix & Retry  │
                        │  Validation  │Fail └──────────────┘
                        └──────┬───────┘
                               │ Pass
                        ┌──────▼───────┐     ┌──────────────┐
                        │   Approval   │────▶│   Address    │
                        │    Gate      │Deny │  Feedback    │
                        └──────┬───────┘     └──────────────┘
                               │ Approved
                        ┌──────▼───────┐     ┌──────────────┐
                        │  Production  │────▶│   Resolve    │
                        │  Readiness   │Fail │  Blockers    │
                        └──────┬───────┘     └──────────────┘
                               │ Ready
                        ┌──────▼───────┐     ┌──────────────┐
                        │  Deployment  │────▶│   ROLLBACK   │
                        │  Execution   │Fail │   TRIGGER    │
                        └──────┬───────┘     └──────┬───────┘
                               │ OK                 │
                        ┌──────▼───────┐            │
                        │   Health     │────────────┘
                        │ Verification │Fail
                        └──────┬───────┘
                               │ Pass
                        ┌──────▼───────┐
                        │  ✅ SUCCESS  │
                        │  Production  │
                        │   Release    │
                        └──────────────┘
```

---

## Key

| Symbol | Meaning |
|--------|---------|
| `─Pass─▶` | Gate passed, release advances |
| `─Fail─▶` | Gate failed, failure path activated |
| `─Deny─▶` | Approval denied, feedback required |
| `✅` | Production success — release complete |
| `🚨` | Rollback triggered — recovery in progress |

---

## Approval Points

1. **Build Validation** — Automated approval (all CI checks must pass)
2. **Approval Gate** — Human approval (Tech Lead + Release Manager)
3. **Production Readiness** — Release Owner verification (checklist completion)

## Validation Checkpoints

1. **Build Compilation** — Code compiles without errors
2. **Test Execution** — Unit and integration tests pass with ≥80% coverage
3. **Security Scan** — No critical/high vulnerabilities
4. **Staging Smoke Test** — Application functions correctly in staging
5. **Health Verification** — Production health checks pass for 10 minutes

## Recovery Paths

1. **Validation Failure** → Fix code → Re-submit PR → Re-run validation
2. **Approval Rejection** → Address feedback → Update release → Re-request approval
3. **Deployment Failure** → Halt rollout → Rollback → Incident report
4. **Health Check Failure** → Assess severity → Rollback or fix-forward → Incident report
5. **Rollback Trigger** → Tier 1/2/3 rollback → Verify restoration → Post-incident review
