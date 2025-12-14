# Runbook: EKS API Server, kube-proxy, and Networking Blackholes

## Purpose
Diagnose and resolve issues where Kubernetes workloads experience:
- Intermittent or total loss of connectivity
- API server timeouts or permission errors
- Services failing despite healthy pods
- Traffic “blackholes” where packets are silently dropped

These issues are often caused by failures in:
- EKS API Server connectivity
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

## Advanced Deep Dive: kube-proxy, iptables, and IPVS Blackholes

This section provides deeper technical context for diagnosing silent networking failures caused by kube-proxy rule desynchronization, iptables corruption, or IPVS state issues.

---

### 1. kube-proxy Modes and Traffic Flow

kube-proxy programs service routing rules on each node using one of two modes:

#### iptables mode (default in many clusters)
- Uses iptables NAT rules to translate Service IPs to Pod IPs
- Scales poorly with very large numbers of services/endpoints
- Large rule sets increase sync time and rule churn

#### IPVS mode
- Uses Linux IPVS for kernel-level load balancing
- Better performance at scale
- Still relies on iptables for packet marking and filtering

If kube-proxy fails to correctly program rules, traffic may be:
- Dropped silently
- Routed to non-existent endpoints
- Stuck in stale NAT mappings

---

### 2. How Blackholes Commonly Form

Blackholes typically appear after:
- Rapid node scale-up / scale-down
- Large service churn (many Services or Endpoints changing)
- kube-proxy restarts during heavy load
- Node reboots with stale iptables state

Common failure mechanisms:
- Partial rule updates applied
- Old endpoints not removed
- Rule chains exceed kernel processing limits
- iptables sync interrupted mid-update

Result:
- Traffic appears to “hang” with no errors
- Retries do not help
- Only specific nodes are affected

---

### 3. iptables Rule Explosion Detection

Check total rule count:
```bash
sudo iptables -L -n | wc -l
````

Red flags:

* Tens of thousands of rules
* Dramatic growth over time
* Long delays when listing rules

Inspect Kubernetes-specific chains:

```bash
sudo iptables -t nat -L KUBE-SERVICES -n
sudo iptables -t nat -L KUBE-NODEPORTS -n
```

Symptoms of corruption:

* Missing rules for known Services
* Duplicate or conflicting rules
* Chains referencing non-existent endpoints

---

### 4. IPVS-Specific Diagnostics (if enabled)

Check IPVS tables:

```bash
sudo ipvsadm -Ln
```

Look for:

* Services with zero backends
* Backends pointing to deleted Pod IPs
* Extremely large IPVS tables

If IPVS services exist but traffic still fails:

* Check iptables `mangle` and `nat` tables
* Verify kube-proxy health and sync loops

---

### 5. Confirm kube-proxy Rule Synchronization

Check kube-proxy logs for sync behavior:

```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=200
```

Indicators of trouble:

* Repeated full resyncs
* Long sync durations
* Errors applying rules

A kube-proxy that cannot complete a sync reliably can leave nodes in a partially programmed state.

---

### 6. Node-Level Packet Drops and Kernel Signals

Check kernel logs:

```bash
sudo dmesg | grep -i drop | tail -n 20
```

Look for:

* Conntrack table exhaustion
* Packet drop warnings
* Netfilter errors

Check conntrack usage:

```bash
sudo sysctl net.netfilter.nf_conntrack_count
sudo sysctl net.netfilter.nf_conntrack_max
```

Conntrack exhaustion can amplify blackhole symptoms even when iptables rules are correct.

---

### 7. When Restarting kube-proxy Is NOT Enough

If restarting kube-proxy does not resolve the issue:

* iptables state may already be corrupted
* Kernel networking state may be inconsistent

In these cases:

* Drain the node
* Terminate and replace it
* Allow fresh rule programming on a clean node

This is often faster and safer than attempting in-place repair.

---

### 8. SRE Decision Rule (Practical Guidance)

**If:**

* Issues are node-specific
* Traffic hangs silently
* kube-proxy logs show sync instability
* iptables/IPVS state is questionable

**Then:**
Drain and replace the node

Node replacement is a valid, production-safe SRE remediation for networking blackholes.

---

### 9. Preventive Controls

* Limit excessive Service and Endpoint churn
* Prefer IPVS mode for large clusters
* Monitor iptables rule count growth
* Alert on kube-proxy restart loops
* Periodically rotate nodes to clear accumulated state
