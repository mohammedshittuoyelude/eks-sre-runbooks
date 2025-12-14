# Runbook: Karpenter Node Provisioning Failures (EKS)

## Purpose
Diagnose and resolve failures where Karpenter does not provision nodes, provisions incorrect nodes, or provisions nodes that fail to join the cluster.

Common symptoms include:
- Pods stuck in `Pending`
- Karpenter logs showing `Failed to provision node`
- Nodes created in EC2 but never join EKS
- Nodes immediately terminating after creation
- Capacity imbalance across AZs
- IP exhaustion or instance launch failures

---

## Quick Triage (5–10 minutes)

### 1. Identify pending pods
```bash
kubectl get pods -A | grep Pending
````

Describe one pending pod:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

* `Insufficient cpu`
* `Insufficient memory`
* `node(s) didn't match Pod's node affinity`
* `no nodes available to schedule pods`

---

### 2. Check Karpenter controller health

```bash
kubectl get pods -n karpenter
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=200
```

Common error patterns:

* `no instance types satisfy requirements`
* `launch template not found`
* `insufficient capacity`
* `access denied`
* `failed to create instance`

---

### 3. Confirm EC2 instances are being created

```bash
aws ec2 describe-instances \
  --filters "Name=tag:karpenter.sh/nodepool,Values=*" \
  --query 'Reservations[].Instances[].InstanceId'
```

Outcomes:

* No instances → provisioning blocked
* Instances exist but not joining → bootstrap / IAM / networking issue
* Instances terminate quickly → permissions or misconfiguration

---

## Deep Diagnosis

### 4. Validate NodePool / Provisioner requirements

```bash
kubectl get nodepool
kubectl describe nodepool <nodepool-name>
```

Check for:

* Instance type constraints too narrow
* AZ restrictions (`topology.kubernetes.io/zone`)
* Capacity type mismatches (spot vs on-demand)
* Overly strict labels / taints

Common mistake:

* Requesting instance types not available in the AZ
* Requesting GPU instances without GPU AMI

---

### 5. Validate EC2NodeClass / Launch Template

```bash
kubectl get ec2nodeclass
kubectl describe ec2nodeclass <nodeclass-name>
```

Verify:

* Subnets are correct and have free IPs
* Security groups allow EKS control-plane access
* AMI family matches workload (AL2 / Bottlerocket / AL2023)
* UserData/bootstrap is present

Red flags:

* Wrong AMI family
* Missing bootstrap configuration
* Subnets with IP exhaustion

---

### 6. Check IAM permissions (VERY COMMON FAILURE)

Karpenter node role must allow:

* `ec2:RunInstances`
* `ec2:CreateTags`
* `ec2:Describe*`
* `iam:PassRole`

Check CloudTrail for errors:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances
```

Look for:

* `AccessDenied`
* `UnauthorizedOperation`

---

### 7. Nodes created but not joining cluster

If instances exist but do not appear in Kubernetes:

```bash
kubectl get nodes
```

Check instance user data:

* Correct cluster name
* Correct API endpoint access
* Correct CA bundle

Common causes:

* Wrong cluster name in bootstrap
* Private API endpoint unreachable
* Security groups blocking port 443 to API server

---

### 8. Capacity or AZ imbalance issues

```bash
kubectl get nodes -L topology.kubernetes.io/zone
```

Symptoms:

* Nodes heavily skewed to one AZ
* Frequent `Insufficient capacity` errors

Fix:

* Broaden instance type selection
* Allow more AZs
* Use mixed capacity types

---

### 9. IP exhaustion blocking node creation

```bash
aws ec2 describe-subnets --subnet-ids <subnet-id>
```

If available IPs are low:

* Karpenter cannot attach ENIs
* Node launch fails silently

Fix:

* Expand subnet CIDR
* Add additional subnets
* Reduce pod density

---

## Common Root Causes and Resolutions

### A. Overly restrictive instance requirements

**Fix**

* Expand allowed instance families
* Allow multiple sizes
* Avoid single-AZ constraints

---

### B. IAM or PassRole failure

**Fix**

* Validate node IAM role
* Ensure `iam:PassRole` permission exists
* Match role ARN exactly

---

### C. Incorrect AMI or bootstrap config

**Fix**

* Use EKS-optimized AMI
* Validate bootstrap script
* Match AMI family to Karpenter config

---

### D. Spot capacity interruptions

**Fix**

* Add fallback on-demand NodePool
* Increase instance diversity
* Monitor Spot interruption events

---

### E. Subnet or IP exhaustion

**Fix**

* Use larger CIDR ranges
* Enable prefix delegation
* Spread nodes across subnets

---

## Post-Fix Verification

### 1. Trigger new scheduling

```bash
kubectl delete pod <pending-pod> -n <namespace>
```

### 2. Confirm node provisioning

```bash
kubectl get nodes -w
```

### 3. Verify workload placement

```bash
kubectl get pods -n <namespace> -o wide
```

Success criteria:

* New nodes appear within minutes
* Pods move to `Running`
* Karpenter logs show successful provisioning

---

## Long-Term SRE Improvements

* Use multiple instance families and sizes
* Monitor Karpenter provisioning latency
* Alert on repeated provisioning failures
* Track AZ capacity balance
* Test scale-up during load tests and deployments