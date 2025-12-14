# Runbook: VPC CNI IP Exhaustion / Pod IP Allocation Failures (EKS)

## Purpose
Diagnose and resolve pod scheduling failures caused by Amazon VPC CNI IP exhaustion, including issues related to:
- Insufficient available IPs on nodes or subnets
- Prefix delegation misconfiguration
- ENI/IPAM limits being reached

Common symptoms include:
- Pods stuck in `Pending` or `ContainerCreating`
- Errors such as `failed to assign an IP address to container`
- `Insufficient pods` or `no available IP addresses`
- VPC CNI (`aws-node`) errors in logs

---

## Quick Triage (5–10 minutes)

### 1. Identify affected pods
```bash
kubectl get pods -A | grep -E 'Pending|ContainerCreating'
Describe one affected pod:

bash
Copy code
kubectl describe pod <pod-name> -n <namespace>
Look for events like:

FailedCreatePodSandBox

failed to allocate IP address

no free IPs available in the subnet

2. Check VPC CNI (aws-node) pod health
bash
Copy code
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl logs -n kube-system -l k8s-app=aws-node --tail=200
Common log messages:

failed to assign an IP address to container

IPAM exhausted

no available IP addresses

failed to attach ENI

3. Check node pod capacity
bash
Copy code
kubectl get nodes -o wide
kubectl describe node <node-name>
Pay attention to:

Allocatable: pods

Current pod count vs max pods

Whether the node has reached its pod limit

Deep Diagnosis
4. Verify subnet IP availability
Identify the subnet(s) used by the node:

bash
Copy code
kubectl describe node <node-name> | grep -i subnet
Then check subnet free IPs (AWS Console or CLI):

bash
Copy code
aws ec2 describe-subnets --subnet-ids <subnet-id>
Red flags:

Subnet with very low available IP addresses

Small CIDR ranges (e.g., /27, /28) used for worker nodes

5. Check ENI and IP limits for instance type
Confirm instance type:

bash
Copy code
kubectl get node <node-name> -o jsonpath='{.metadata.labels.node\.kubernetes\.io/instance-type}'
echo
Compare against AWS limits:

Max ENIs

IPs per ENI

Effective max pods per node

If the node hits ENI/IP limits, no more pods can be scheduled.

6. Verify prefix delegation configuration
Check CNI config:

bash
Copy code
kubectl get configmap aws-node -n kube-system -o yaml
Look for values like:

ENABLE_PREFIX_DELEGATION: "true"

WARM_PREFIX_TARGET: "1"

If prefix delegation is disabled:

Nodes consume individual IPs rapidly

IP exhaustion occurs sooner

7. Check warm IP / prefix targets
Review these environment variables in the aws-node DaemonSet:

WARM_IP_TARGET

MINIMUM_IP_TARGET

WARM_PREFIX_TARGET

bash
Copy code
kubectl describe daemonset aws-node -n kube-system | grep -A5 -E 'WARM_|MINIMUM_'
Misconfigured values may cause:

Slow IP allocation

Bursty pod creation failures during scaling events

Common Root Causes and Resolutions
A. Subnet IP exhaustion
Symptoms

Multiple nodes affected

AWS console shows low available IPs in subnet

Resolution

Expand subnet CIDR

Add larger or additional subnets

Move node groups to larger subnets

Prevention

Use /23 or larger subnets for EKS nodes

Monitor available IPs with CloudWatch

B. Prefix delegation disabled
Symptoms

Nodes hit pod limit early

ENI IP exhaustion occurs quickly

Resolution
Enable prefix delegation and restart aws-node:

bash
Copy code
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
kubectl rollout restart daemonset aws-node -n kube-system
Prevention

Enable prefix delegation by default for EKS clusters

Validate after cluster upgrades

C. Node reached max pod capacity
Symptoms

Node shows Allocatable pods fully consumed

New pods remain pending despite free CPU/memory

Resolution

Scale node group

Use larger instance types

Reduce pod density per node

Prevention

Right-size instance types

Validate max-pods configuration

D. Warm IP / prefix targets too low
Symptoms

Pods fail during rapid scale-up

IPs eventually allocate but with delays

Resolution
Example tuning (adjust to your environment):

bash
Copy code
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
kubectl rollout restart daemonset aws-node -n kube-system
Prevention

Tune warm targets based on scaling patterns

Monitor IP allocation latency

Post-Fix Verification
1. Restart a test pod
bash
Copy code
kubectl run ip-test --image=nginx --restart=Never
kubectl describe pod ip-test
2. Verify aws-node stability
bash
Copy code
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100
Success criteria:

Pods move to Running

No new IPAM or ENI errors

Node pod count increases successfully

Long-Term SRE Improvements
Enable prefix delegation cluster-wide

Use larger CIDR subnets for worker nodes

Monitor subnet available IPs

Alert on IPAM errors from aws-node

Test scaling behavior during load tests