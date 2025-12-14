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

# Runtime-Specific Memory & CPU Tuning (EKS)

## Purpose

Provide language/runtime-specific guidance for diagnosing and preventing:

* OOMKills
* CPU throttling
* Memory fragmentation
* GC-related latency
* Heap vs container memory mismatch

Applies to workloads running on **Amazon EKS** using:

* .NET (.NET 6 / .NET 8)
* JVM (Java 11/17/21)
* Python
* Node.js / Go (briefly)

---

## 1️ .NET (.NET 6 / .NET 8) on EKS

### Common Failure Patterns

**Symptoms**

* Pods OOMKilled even though heap usage looks “normal”
* Memory usage grows slowly over time
* Latency spikes under load
* Restarts “fix” the issue temporarily

**Root Causes**

* Server GC using too much memory inside container
* GC heap not respecting container limits (older versions)
* LOH (Large Object Heap) fragmentation
* Thread pool growth under load
* Native memory (non-heap) growth

---

### Key Checks

#### 1. Confirm GC mode

```bash
kubectl exec -it <pod> -- printenv | grep -i gc
```

Check runtime info:

```bash
kubectl exec -it <pod> -- dotnet --info
```

**Red flags**

* `Server GC: true` + small container memory
* No explicit memory limits or GC configuration

---

### Recommended .NET Container Settings (EKS)

#### Environment variables (VERY IMPORTANT)

```yaml
env:
  - name: DOTNET_GC_SERVER
    value: "1"
  - name: DOTNET_GC_HEAP_HARD_LIMIT_PERCENT
    value: "75"
  - name: DOTNET_GC_CONCURRENT
    value: "1"
```

**Why**

* Prevents GC from consuming 100% of container memory
* Leaves headroom for native allocations and threads

---

### Resource Sizing Guidance

* **Memory limit**: at least **1.3–1.5× expected steady-state**
* **Memory request**: close to real usage (avoid overpacking nodes)
* Avoid extremely tight memory limits with Server GC

---

### Advanced Diagnostics

```bash
kubectl exec -it <pod> -- dotnet-counters monitor
kubectl exec -it <pod> -- dotnet-gcdump collect
```

Look for:

* Heap growth over time
* LOH fragmentation
* High allocation rate

---

## 2️ JVM (Java 11 / 17 / 21) on EKS

### Common Failure Patterns

**Symptoms**

* Pod OOMKilled but heap never reached `-Xmx`
* High GC pause times
* CPU throttling during GC
* Sudden memory spikes

**Root Causes**

* JVM heap not container-aware
* Heap too large relative to pod memory
* Metaspace / native memory exhaustion
* GC threads throttled by CPU limits

---

### MUST-HAVE JVM Flags (Containers)

```bash
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=70
-XX:InitialRAMPercentage=50
-XX:+ExitOnOutOfMemoryError
```

**Why**

* Prevents JVM from assuming host memory
* Leaves room for:

  * Metaspace
  * Thread stacks
  * Direct buffers
  * JIT

---

### GC Recommendations

* **G1GC** (default in modern JVMs) → good for most services
* For latency-sensitive services:

  ```bash
  -XX:MaxGCPauseMillis=200
  ```

---

### What to Monitor

* Heap usage vs MaxRAMPercentage
* GC pause time
* Allocation rate
* Native memory tracking (if enabled)

```bash
jcmd <pid> GC.heap_info
jcmd <pid> VM.native_memory summary
```

---

### CPU Throttling Impact on JVM

**Very important**

* GC threads are CPU-bound
* CPU throttling → longer GC → latency spikes

**Recommendation**

* Set **CPU requests realistically**
* Avoid extremely low CPU limits
* Prefer HPA scaling over tight CPU caps

---

## 3️ Python on EKS

### Common Failure Patterns

**Symptoms**

* Gradual memory growth
* OOMKill after hours/days
* CPU spikes under load
* Works fine locally, fails in Kubernetes

**Root Causes**

* Python memory fragmentation
* Objects not released to OS
* Large in-memory data structures
* Too many workers (gunicorn/uvicorn)

---

### Key Characteristics (important for SREs)

* Python does **not** return memory to OS reliably
* RSS may only grow, never shrink
* Garbage collection is not heap-compacting

---

### Best Practices

#### Gunicorn / Uvicorn

```bash
--max-requests 1000
--max-requests-jitter 100
```

**Why**

* Forces worker recycle
* Prevents unbounded memory growth

---

### Resource Sizing Guidance

* Set memory limit with **ample headroom**
* Avoid ultra-tight memory limits
* Monitor RSS, not just heap-like metrics

---

### Debugging Tools

```bash
tracemalloc
objgraph
gc.get_stats()
```

Watch for:

* Steady RSS increase
* Worker count vs memory per worker

---

## 4️ Node.js (brief but important)

### Common Issues

* V8 heap limit too large
* Event loop blocking
* Memory leak in closures

### Must-have flag

```bash
--max-old-space-size=70% of container memory
```

---

## 5️ Go (brief)

### Characteristics

* GC is concurrent and container-aware
* Usually fewer memory surprises
* OOMKills often due to:

  * Large in-memory caches
  * Unbounded goroutines

### Key checks

* Goroutine count
* Heap growth trend
* Memory request sizing

---

## 6️ Cross-Runtime Rules (VERY IMPORTANT)

### 1. Never size heap = container memory

Always leave room for:

* Native memory
* Threads
* Buffers
* Sidecars (Envoy, logging agents)

---

### 2. CPU limits can cause latency

* CPU throttling affects:

  * GC
  * Thread scheduling
  * Event loops
* Prefer:

  * Reasonable CPU requests
  * Autoscaling
  * Fewer hard CPU limits for latency-critical services

---

### 3. Watch node pressure

Even “healthy” pods can be evicted when:

* Node memory pressure rises
* Requests are too low
* Node is overpacked

---

## 7️ Alerts to Add

* Pod OOMKilled count
* Memory usage > 80% of limit
* CPU throttling ratio
* GC pause time
* Restart count increase
* Node memory pressure
