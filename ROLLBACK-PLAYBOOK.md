# Rollback Playbook

## Overview

This playbook provides step-by-step recovery procedures for deployment failures affecting the Checkout Service. It is designed to eliminate the ad-hoc, undocumented rollback responses that have historically caused extended downtime (see Incident 2: 45-minute rollback delay in `docs/incident-log.md`).

Every engineer involved in deployment must read and understand this playbook **before** participating in any release.

---

## Scenario 1: Immediate Rollback Procedure

### What Happened

A deployment has been executed and health verification checks are failing. The new release is causing active customer impact — elevated error rates, failed transactions, or service unavailability. The situation requires an immediate return to the previous known-good version.

**Real-world example:** Incident 1 (2026-05-18) — A feature branch deployed directly to production caused the checkout service to return incorrect totals for 30 minutes.

### Trigger Conditions

An immediate rollback **must** be initiated when any of the following conditions are met:

| Condition | Threshold | Detection Method |
|-----------|-----------|-----------------|
| Error rate spike | > 1% for ≥ 2 minutes | APM alerting (Datadog/New Relic) |
| Latency spike | p95 > 500ms for ≥ 2 minutes | APM alerting |
| Health endpoint failure | ≥ 50% of pods returning non-200 | Kubernetes health probes |
| Pod crash loops | Any pod in CrashLoopBackOff | `kubectl get pods` monitoring |
| Key transaction failure | Checkout smoke test fails | Automated post-deployment test |
| Business metric drop | Checkout completion rate drops > 5% | Business metrics dashboard |
| Customer reports | Multiple customer complaints received | Support channel escalation |

### Decision Owner

The **Release Owner** (named in the release request) is the sole decision-maker for triggering an immediate rollback. If the Release Owner is unreachable, the **on-call Senior Engineer** assumes decision authority.

### First Actions (Execute in Order)

**Time target: Complete rollback within 5 minutes of trigger detection.**

#### Step 1: Announce the Rollback (T+0 minutes)

```
Post to #incidents Slack channel:
"🚨 ROLLBACK IN PROGRESS — Checkout Service release [VERSION] is being rolled back due to [BRIEF REASON]. Release Owner: [NAME]. Updates to follow."
```

#### Step 2: Execute the Rollback (T+1 minute)

```bash
# Roll back to the previous deployment revision
kubectl rollout undo deployment/checkout-service -n production

# Monitor rollback progress
kubectl rollout status deployment/checkout-service -n production --timeout=300s
```

#### Step 3: Verify Rollback Success (T+3 minutes)

```bash
# Confirm the previous version is running
kubectl get deployment checkout-service -n production \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check all pods are running and healthy
kubectl get pods -n production -l app=checkout-service

# Verify health endpoint
curl -s -o /dev/null -w "%{http_code}" https://checkout.example.com/health
```

#### Step 4: Confirm Service Restoration (T+5 minutes)

- Verify error rate has returned to baseline
- Verify latency has returned to baseline
- Run key transaction smoke test
- Confirm customer-facing functionality is operational

#### Step 5: Communicate Resolution (T+5 minutes)

```
Post to #incidents Slack channel:
"✅ ROLLBACK COMPLETE — Checkout Service has been rolled back to [PREVIOUS VERSION]. Service is restored. Incident report to follow within 24 hours."
```

### How Service Is Restored

Service is restored by reverting the Kubernetes deployment to the previous known-good revision. Kubernetes handles the pod replacement automatically through its rolling update strategy. The previous container image is pulled from the registry and deployed to all pods.

### Post-Rollback Actions

1. File an incident report within 24 hours
2. Identify root cause of the failure
3. Do NOT re-deploy the failed version without fixes and re-validation
4. Schedule a post-incident review within 1 week

---

## Scenario 2: Partial Failure Procedure

### What Happened

A deployment has been executed but is exhibiting partial degradation. The service is not fully down — some requests succeed while others fail, or specific features are broken while core functionality remains operational. The failure is isolated to a subset of traffic, pods, or functionality.

**Real-world example:** Incident 3 (2026-04-29) — A release bypassed functional validation and shipped a broken checkout flow. Several orders failed while others completed normally.

### Detection Method

Partial failures are typically more difficult to detect than full failures. Use the following methods:

| Detection Method | What to Look For |
|-----------------|------------------|
| **APM Error Breakdown** | Elevated error rate on specific endpoints (e.g., `/checkout/payment` failing while `/checkout/cart` works) |
| **Pod-Level Monitoring** | Some pods healthy, others restarting or returning errors |
| **Canary Metrics** | Canary pod showing degraded metrics while existing pods are healthy |
| **Customer Reports** | Intermittent failure reports (e.g., "sometimes checkout works, sometimes it doesn't") |
| **Log Analysis** | Error patterns appearing on specific pods or for specific request types |

### Containment Strategy

**Time target: Contain partial failure within 10 minutes of detection.**

#### Step 1: Isolate the Affected Component (T+0 minutes)

```bash
# Identify which pods are affected
kubectl get pods -n production -l app=checkout-service -o wide

# Check logs on affected pods
kubectl logs -n production [AFFECTED_POD_NAME] --tail=100

# If canary deployment, halt the rollout
kubectl rollout pause deployment/checkout-service -n production
```

#### Step 2: Assess Scope and Impact (T+2 minutes)

Determine:
- What percentage of traffic is affected?
- Which specific features or endpoints are failing?
- Is the failure consistent or intermittent?
- Are affected pods in a specific availability zone or node?

#### Step 3: Decide on Response Path (T+5 minutes)

| Condition | Response |
|-----------|----------|
| < 5% traffic affected, issue is isolated and understood | **Monitor and fix-forward** — deploy a patch |
| 5-20% traffic affected, root cause unclear | **Pause rollout and investigate** — 15-minute investigation window |
| > 20% traffic affected OR customer impact confirmed | **Full rollback** — execute Immediate Rollback Procedure |
| Investigation window exceeded (15 min) without resolution | **Full rollback** — do not extend investigation |

#### Step 4: Execute Containment

**If fixing forward:**
```bash
# Resume rollout after applying fix
kubectl rollout resume deployment/checkout-service -n production
```

**If rolling back:**
```bash
# Execute full rollback
kubectl rollout undo deployment/checkout-service -n production
kubectl rollout status deployment/checkout-service -n production --timeout=300s
```

### Recovery Path

1. Contain the failure using the steps above
2. Restore full service (either through fix-forward or rollback)
3. Verify all health checks pass
4. Document the partial failure and its root cause
5. Add detection mechanisms for this failure mode to prevent recurrence

### How Service Is Restored

For partial failures, service is restored either by rolling back the entire deployment (if the scope is large or growing) or by deploying a targeted fix (if the issue is isolated and well-understood). The decision must be made within 15 minutes — if the root cause is not identified in that time, the default action is a full rollback.

---

## Scenario 3: Full Deployment Failure Procedure

### What Happened

A deployment has resulted in complete service failure. The Checkout Service is entirely unavailable — all pods are down, all health endpoints are failing, and no customer requests are being served. This is a critical production incident.

**Real-world example:** A combination of Incidents 1 and 4 — wrong version deployed to production combined with no validation gates, resulting in a completely non-functional checkout service.

### Escalation Process

**Time target: Service restored within 15 minutes. Escalation within 5 minutes.**

| Time | Action | Owner |
|------|--------|-------|
| T+0 min | Failure detected, Release Owner notified | Monitoring system / On-call engineer |
| T+1 min | Immediate rollback initiated | Release Owner |
| T+2 min | Incident Commander activated | On-call Senior Engineer |
| T+5 min | If rollback has not succeeded: escalate to Engineering Manager | Incident Commander |
| T+10 min | If service not restored: escalate to VP of Engineering | Engineering Manager |
| T+15 min | If service not restored: activate Major Incident Protocol | VP of Engineering |

### Communication Steps

#### Internal Communication

| Channel | Message | Timing |
|---------|---------|--------|
| `#incidents` (Slack) | 🚨 P1 Incident: Checkout Service is DOWN. Rollback in progress. Incident Commander: [NAME] | T+0 min |
| `#incidents` (Slack) | Status update every 5 minutes until resolved | Every 5 min |
| Engineering All-Hands (email) | Incident summary if not resolved within 30 minutes | T+30 min |
| `#incidents` (Slack) | ✅ Service restored. Root cause investigation underway. | Upon resolution |

#### External Communication (if customer-facing)

| Audience | Message | Timing | Owner |
|----------|---------|--------|-------|
| Status Page | "We are experiencing issues with our checkout service. Our team is investigating." | T+5 min | Communication Lead |
| Status Page | "The issue has been identified and a fix is being deployed." | When rollback starts | Communication Lead |
| Status Page | "The issue has been resolved. All services are operating normally." | Upon resolution | Communication Lead |
| Customer Support | Internal talking points and ETA for resolution | T+5 min | Support Lead |

### Recovery Workflow

#### Phase 1: Emergency Rollback (T+0 to T+5 minutes)

```bash
# 1. Attempt standard rollback
kubectl rollout undo deployment/checkout-service -n production
kubectl rollout status deployment/checkout-service -n production --timeout=300s

# 2. If standard rollback fails, force deployment of known-good version
kubectl set image deployment/checkout-service \
  checkout=registry.example.com/checkout:[LAST_KNOWN_GOOD_VERSION] \
  -n production --record

# 3. If deployment is completely broken, scale down and redeploy
kubectl scale deployment/checkout-service -n production --replicas=0
kubectl scale deployment/checkout-service -n production --replicas=[ORIGINAL_REPLICA_COUNT]
```

#### Phase 2: Service Verification (T+5 to T+10 minutes)

```bash
# Verify pods are running
kubectl get pods -n production -l app=checkout-service

# Check health endpoints
curl -s -o /dev/null -w "%{http_code}" https://checkout.example.com/health

# Run smoke test
./scripts/smoke-test.sh production

# Check error rates in monitoring
# (Manual verification via Grafana/Datadog dashboard)
```

#### Phase 3: Stabilization (T+10 to T+30 minutes)

- Confirm all health checks pass for at least 10 minutes
- Verify error rate and latency have returned to baseline
- Confirm customer-facing transactions are completing successfully
- Stand down escalation (notify all stakeholders of resolution)

#### Phase 4: Post-Incident (T+30 minutes onward)

1. **Incident Report:** Filed within 24 hours with timeline, root cause, and impact assessment
2. **Root Cause Analysis:** Completed within 48 hours
3. **Post-Incident Review:** Scheduled within 1 week with all involved engineers
4. **Corrective Actions:** Documented, assigned owners, and tracked to completion
5. **Process Improvements:** Any gaps in the deployment workflow identified and addressed

### How Service Is Restored

For full deployment failures, service is restored through a three-tier rollback strategy:

1. **Standard rollback** (`kubectl rollout undo`) — reverts to the previous deployment revision
2. **Forced version deployment** — explicitly sets the container image to the last known-good version
3. **Scale-down and redeploy** — last resort if the deployment is in a corrupted state

Each tier is attempted in sequence, escalating only if the previous tier fails.

---

## Quick Reference Card

Print this card and keep it accessible during every deployment.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ROLLBACK QUICK REFERENCE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ANNOUNCE:  Post to #incidents "ROLLBACK IN PROGRESS"        │
│                                                                 │
│  2. ROLLBACK:  kubectl rollout undo deployment/checkout-service │
│                -n production                                    │
│                                                                 │
│  3. VERIFY:    kubectl rollout status deployment/checkout-      │
│                service -n production --timeout=300s             │
│                                                                 │
│  4. CONFIRM:   curl https://checkout.example.com/health         │
│                                                                 │
│  5. COMMUNICATE: Post to #incidents "ROLLBACK COMPLETE"         │
│                                                                 │
│  DECISION OWNER: Release Owner → On-Call Sr. Engineer           │
│  ESCALATION: 5 min → Eng Manager, 10 min → VP Engineering      │
│  TARGET: Service restored within 5 minutes                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Playbook Maintenance

- This playbook must be reviewed and updated quarterly
- After every deployment incident, verify that the playbook covers the failure scenario
- All engineers must complete a rollback drill exercise during onboarding
- Rollback procedures must be tested in staging at least once per month
