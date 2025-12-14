# Runbook: Pod OOMKilled & CPU Throttling (Kubernetes / EKS)

## Purpose
Diagnose and resolve performance degradation and pod restarts caused by:
- Container memory exhaustion (OOMKilled)
- CPU throttling leading to latency and timeouts
- Mis-sized resource requests/limits
- Memory leaks or traffic spikes

Common symptoms:
- Pods restarting unexpectedly
- `OOMKilled` in pod status or events
- Latency spikes, timeouts, intermittent 5xx
- High CPU usage with throttling (even when limits appear “high enough”)

---

## Quick Triage (5–10 minutes)

### 1) Identify affected pods and restart patterns
```bash
kubectl get pods -A --sort-by=.status.containerStatuses[0].restartCount
kubectl get pods -A | grep -E 'CrashLoopBackOff|Error|OOMKilled'
````

Pick one pod and inspect:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

* `Last State: Terminated` → `Reason: OOMKilled`
* Events showing eviction or OOM
* `CrashLoopBackOff` with frequent restarts

---

### 2) Check current resource settings (requests/limits)

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].name}{"\n"}{.spec.containers[*].resources}{"\n"}'
echo
```

Quick interpretation:

* If **memory limit is too low**, OOMKills are likely.
* If **CPU limit is set**, throttling is possible under load.
* If **requests are far below reality**, nodes overpack pods → noisy neighbor issues.

---

### 3) Check live usage (if Metrics Server exists)

```bash
kubectl top pod <pod-name> -n <namespace>
kubectl top pod -n <namespace> --sort-by=memory
kubectl top pod -n <namespace> --sort-by=cpu
```

If `kubectl top` is unavailable, rely on:

* Application metrics (Prometheus/CloudWatch/Datadog)
* Container logs and restart reasons

---

## Deep Diagnosis

### 4) Confirm OOMKilled vs Application Crash

OOMKill confirmation:

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}'
echo
```

Also check events:

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp | tail -n 30
```

If **not** OOMKilled:

* Check app logs for crash reasons
* Check readiness/liveness probes
* Check dependencies (DB, DNS, downstreams)

---

### 5) Check node pressure and eviction signals

```bash
kubectl describe node <node-name> | sed -n '1,220p'
```

Look for:

* `MemoryPressure=True` / `DiskPressure=True`
* Recent evictions
* High pod density on the node

Check node usage:

```bash
kubectl top node <node-name>
```

If the node is near memory limits, even “normal” pods can be evicted.

---

### 6) CPU throttling diagnosis (Kubernetes view)

CPU throttling happens when a container hits its CPU **limit**.

First, see if limits exist:

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].name}{"\n"}{.spec.containers[*].resources.limits.cpu}{"\n"}'
echo
```

If using Prometheus, check (examples):

* `container_cpu_cfs_throttled_periods_total`
* `container_cpu_cfs_periods_total`
* throttling ratio = throttled_periods / total_periods

If you don’t have Prometheus, symptoms still help:

* High request latency during CPU spikes
* CPU usage pinned near limit
* Improved behavior after raising CPU limit or removing it (for Burstable workloads)

---

### 7) Check for memory leak patterns

Signals of a leak:

* Memory usage steadily increases over time
* Restart “fixes” the problem temporarily
* OOMKill occurs after predictable runtime duration

What to check:

* Application heap metrics (JVM/.NET/Python)
* GC logs if applicable
* Traffic changes (new endpoints, payload size changes)

---

## Common Root Causes → Fixes

### A) Memory limit too low (OOMKilled)

**Symptoms**

* Pod shows `OOMKilled`
* Memory usage near limit shortly before restart

**Fix**

* Increase memory limit
* Increase memory request (to reduce overpacking on nodes)
* Add HPA based on workload/latency (not just CPU)

**Example patch (Deployment)**

```bash
kubectl -n <namespace> edit deploy <deployment-name>
```

Update container resources:

* requests.memory: increase to realistic baseline
* limits.memory: increase with headroom

---

### B) CPU throttling due to strict CPU limits

**Symptoms**

* Latency spikes under load
* CPU usage hits limit
* Throttling metrics increase (if available)

**Fix options**

1. Increase CPU limit
2. Consider removing CPU limits (common SRE practice for latency-sensitive services) while keeping CPU requests
3. Scale out (HPA) earlier to avoid hitting limits

---

### C) Node overcommit / noisy neighbor

**Symptoms**

* Node has many pods and high memory usage
* Multiple pods exhibit instability on the same node

**Fix**

* Increase node size or add nodes
* Increase requests to reflect real usage
* Use pod anti-affinity / topology spread constraints for critical services

---

### D) Bad liveness/readiness probes causing restarts

**Symptoms**

* Restarts occur but not OOMKilled
* Logs show probe failures
* App needs more startup time under load

**Fix**

* Increase `initialDelaySeconds`, `timeoutSeconds`, `failureThreshold`
* Prefer readiness probe to gate traffic instead of killing the container

---

## Post-Fix Verification

### 1) Confirm pods stabilize

```bash
kubectl get pods -n <namespace> -w
```

### 2) Confirm restarts stop increasing

```bash
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace> | sed -n '1,160p'
```

### 3) Validate latency and error rate

* p95 latency returns to baseline
* 5xx reduces
* CPU throttling decreases (if tracked)
* memory usage below new limit with headroom

---

## Long-Term SRE Improvements

* Establish standard resource sizing (requests/limits) per service tier
* Alert on:

  * OOMKilled events
  * throttling ratio
  * memory approaching limit
* Add load tests to validate scaling behavior
* Track memory leak indicators and automate heap/diagnostic capture where possible