# Runbook: CoreDNS DNS Latency / Timeouts (EKS)

## Purpose
Diagnose and resolve DNS resolution latency and timeout issues in Kubernetes (Amazon EKS), including symptoms such as:
- `EAI_AGAIN`
- `Temporary failure in name resolution`
- `i/o timeout` during service-to-service or external DNS lookups
- Intermittent application 5xx errors caused by failed name resolution

This runbook helps identify whether the root cause is related to:
1. CoreDNS capacity or configuration
2. Node or pod networking
3. Upstream DNS resolver (VPC resolver / NAT)
4. Cluster resource bottlenecks (CPU, conntrack, ENI/IP pressure)
5. Excessive DNS traffic from a specific workload or namespace

---

### 1. Confirm scope and impact
Determine:
- Is the issue cluster-wide or namespace-specific?
- Are failures affecting internal names (`*.svc.cluster.local`), external domains, or both?
- Is the issue intermittent or persistent?
- Did the issue begin after a deployment, node scale-down, add-on upgrade, or CNI change?

Check CoreDNS status:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get deploy coredns -n kube-system
kubectl get svc kube-dns -n kube-system
kubectl get endpoints kube-dns -n kube-system

---

Expected:

* CoreDNS pods are Running and Ready
* `kube-dns` service has populated endpoints

---

### 2. Validate DNS resolution from within the cluster

Launch a temporary debug pod:

```bash
kubectl run -it --rm dns-debug \
  --image=public.ecr.aws/docker/library/busybox:1.36 \
  --restart=Never -- sh
```

Inside the pod:

```sh
cat /etc/resolv.conf
nslookup kubernetes.default.svc.cluster.local
nslookup google.com
```

Interpretation:

* Internal lookup fails → CoreDNS, kube-dns service, or Kubernetes API connectivity
* External lookup fails → upstream DNS / egress / NAT issue
* Both fail intermittently → capacity, throttling, or packet drops

Exit:

```sh
exit
```

---

## Deep Diagnosis

### 3. Inspect CoreDNS pods and logs

#### 3.1 Pod health and restarts

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
kubectl describe pod -n kube-system -l k8s-app=kube-dns
```

Look for:

* Frequent restarts
* Liveness/readiness probe failures
* OOMKilled events
* Pod evictions or scheduling delays

---

#### 3.2 Review CoreDNS logs

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=200
```

Common indicators:

* `plugin/errors` spikes
* `SERVFAIL` or excessive `NXDOMAIN`
* `i/o timeout` contacting upstream resolvers
* Kubernetes API watch/list failures

---

### 4. Check CoreDNS resource usage and saturation

```bash
kubectl top pods -n kube-system -l k8s-app=kube-dns
kubectl get deploy coredns -n kube-system -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

Signals:

* High CPU usage or throttling
* Memory pressure leading to restarts

---

### 5. Review CoreDNS configuration (Corefile)

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Verify:

* `kubernetes cluster.local` plugin block exists
* `forward . /etc/resolv.conf` or valid upstream resolvers
* `cache` plugin enabled
* No misconfigured rewrite or loop rules

Red flags:

* Forwarding to unreachable DNS servers
* Cache disabled in high-QPS clusters
* Incorrect or missing Kubernetes plugin

---

### 6. Validate CoreDNS connectivity to Kubernetes API

CoreDNS relies on the Kubernetes API for service discovery.

```bash
kubectl exec -n kube-system deploy/coredns -- \
  wget -qO- https://kubernetes.default.svc/api --no-check-certificate | head
```

If this fails:

* Verify node-to-control-plane connectivity
* Check security groups, NACLs, and routing
* Validate EKS endpoint access (private/public)

---

### 7. Rule out node-level networking issues

DNS relies heavily on UDP and is sensitive to packet drops.

Check node health:

```bash
kubectl top nodes
kubectl describe node <node-name>
```

If you have node access:

```bash
sudo sysctl net.netfilter.nf_conntrack_count
sudo sysctl net.netfilter.nf_conntrack_max
sudo dmesg | tail -n 50
```

Indicators:

* Conntrack table exhaustion
* Node CPU saturation
* Kernel-level packet drops

---

## Common Root Causes and Resolutions

### A. CoreDNS under-provisioned

Symptoms:

* DNS failures during traffic spikes
* High CPU usage on CoreDNS pods

Resolution:

```bash
kubectl scale deploy coredns -n kube-system --replicas=6
```

Increase resource requests/limits:

* CPU request: 200m
* CPU limit: 1000m
* Memory: 256–512Mi

Prevention:

* Enable HPA for CoreDNS
* Add PodDisruptionBudget
* Spread pods across AZs using anti-affinity

---

### B. Upstream DNS / egress issues

Symptoms:

* Internal DNS works
* External lookups fail or timeout

Resolution:

* Verify VPC DNS resolution and hostnames enabled
* Check NAT Gateway health and routes
* Validate outbound UDP/TCP 53 is allowed

Prevention:

* Enable NodeLocal DNSCache
* Increase caching efficiency

---

### C. CoreDNS disruption during node scale events

Symptoms:

* DNS issues following node scale-down or upgrades

Resolution:

* Add PodDisruptionBudget
* Increase replica count
* Ensure anti-affinity rules are in place

---

### D. Kubernetes API connectivity issues

Symptoms:

* `*.cluster.local` lookups fail
* CoreDNS logs show API watch/list errors

Resolution:

* Fix node-to-API routing
* Validate kube-proxy and service networking
* Review EKS endpoint access configuration

---

## Post-Fix Verification

Run validation tests:

```bash
kubectl run -it --rm dns-verify \
  --image=public.ecr.aws/docker/library/busybox:1.36 \
  --restart=Never -- sh
```

Inside:

```sh
time nslookup kubernetes.default.svc.cluster.local
time nslookup google.com
```

Success criteria:

* Low latency responses
* No intermittent failures
* Application error rate returns to baseline

---

## Long-Term SRE Improvements

* Deploy NodeLocal DNSCache
* Add CoreDNS autoscaling
* Monitor DNS latency and error rates (p50/p95)
* Track SERVFAIL and timeout rates via metrics

```