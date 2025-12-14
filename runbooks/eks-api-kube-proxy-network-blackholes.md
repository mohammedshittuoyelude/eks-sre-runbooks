# Runbook: EKS API Server, kube-proxy, and Networking Blackholes

## Purpose
Diagnose and resolve issues where Kubernetes workloads experience:
- Intermittent or total loss of connectivity
- API server timeouts or permission errors
- Services failing despite healthy pods
- Traffic “blackholes” where packets are silently dropped

These issues are often caused by failures in:
- EKS API server connectivity
- kube-proxy rules (iptables/IPVS)
- Node-level networking, security groups, or routing

---

## Common Symptoms
- `kubectl` commands intermittently time out
- Pods cannot reach Kubernetes API (`https://kubernetes.default.svc`)
- Service-to-service traffic fails, but pod-to-pod IP works
- Requests hang (no errors, no responses)
- Issues correlate with node scale-up/down events

---

### 1. Validate Kubernetes API availability
From a workstation:
```bash
kubectl get nodes
kubectl get pods -A
````

If slow or failing:

* API server may be unreachable
* Network path to control plane may be broken

---

### 2. Test API connectivity from a pod

```bash
kubectl run -it --rm api-debug \
  --image=public.ecr.aws/docker/library/busybox:1.36 \
  --restart=Never -- sh
```

Inside the pod:

```sh
wget -qO- https://kubernetes.default.svc/api --no-check-certificate | head
```

Failure here indicates:

* kube-proxy or service routing issue
* Security group or NACL blocking traffic
* Control-plane endpoint unreachable

---

### 3. Check kube-proxy health

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=200
```

Look for:

* iptables or IPVS sync errors
* Node-specific failures
* CrashLoopBackOff

---

## Deep Diagnosis

### 4. Verify service routing vs pod routing

Test pod-to-pod:

```bash
ping <pod-ip>
```

Test service IP:

```bash
curl http://<service-cluster-ip>:<port>
```

Interpretation:

* Pod IP works, service IP fails → kube-proxy issue
* Both fail → node or network-level issue

---

### 5. Inspect kube-proxy mode

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml
```

Check:

* iptables vs IPVS mode
* Feature mismatches after cluster upgrades

Misaligned kube-proxy config can cause silent traffic drops.

---

### 6. Check node-level networking

```bash
kubectl describe node <node-name>
```

Look for:

* `NetworkUnavailable`
* Node conditions flapping
* Frequent reboots or replacements

If node access is available:

```bash
sudo iptables -L -n | wc -l
sudo ipvsadm -Ln
sudo dmesg | tail -n 50
```

Red flags:

* Extremely large iptables rule sets
* Kernel networking errors
* Packet drop warnings

---

### 7. Validate security groups and NACLs

Confirm:

* Worker node SG allows outbound traffic to API server
* Control-plane SG allows inbound from nodes on port 443
* NACLs are not blocking ephemeral ports

A single incorrect rule can create a “blackhole” with no logs.

---

### 8. Identify node-specific blackholes

```bash
kubectl get pods -o wide -A
```

If failures correlate to specific nodes:

* Cordon the node

```bash
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets
```

If issues disappear → node networking corruption confirmed.

---

## Common Root Causes and Resolutions

### A. kube-proxy desynchronization

**Fix**

* Restart kube-proxy pods

```bash
kubectl rollout restart daemonset kube-proxy -n kube-system
```

---

### B. Node networking corruption

**Fix**

* Drain and terminate affected nodes
* Allow autoscaler/Karpenter to replace them

---

### C. Security group or NACL regression

**Fix**

* Revert recent network rule changes
* Validate control-plane ↔ node connectivity explicitly

---

### D. API endpoint access misconfiguration

**Fix**

* Validate public/private API endpoint settings
* Ensure worker nodes have network access to endpoint

---

## Post-Fix Verification

### 1. Validate API access

```bash
kubectl get nodes
kubectl get pods -A
```

### 2. Validate service routing

```bash
curl http://<service-cluster-ip>:<port>
```

### 3. Monitor for recurrence

* Watch kube-proxy logs
* Track node churn vs incidents
* Monitor API server latency metrics

---

## Real Incident Example

### Incident Summary

After a node scale-down event, multiple services experienced intermittent request hangs with no corresponding application errors.

### Impact

* Requests stalled without responses
* Elevated latency across several services
* On-call engineers paged for degraded performance

### Root Cause

A subset of nodes had corrupted kube-proxy iptables rules following repeated scale events, causing silent packet drops for service traffic.

### Resolution

* Identified affected nodes by correlating failures
* Drained and replaced impacted nodes
* Restarted kube-proxy to resync service rules

### Prevention / Follow-up

* Added monitoring for kube-proxy health
* Reduced iptables rule growth by limiting service churn
* Included node replacement as a first-class remediation step

