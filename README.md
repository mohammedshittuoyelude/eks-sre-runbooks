# EKS SRE Runbooks

This repository contains **production-grade Site Reliability Engineering (SRE) runbooks** focused on **Amazon EKS** and cloud-native workloads.

The runbooks are written from real-world incident patterns and are designed to:
- Reduce mean time to resolution (MTTR)
- Provide structured triage and deep-dive diagnosis
- Serve as operational references for on-call engineers
- Demonstrate SRE thinking around scalability, reliability, and observability

---

##  Runbooks Included

### 1️ CoreDNS DNS Latency & Timeouts
**Focus:** Cluster DNS failures, latency, and intermittent resolution issues  
**Covers:**
- `EAI_AGAIN`, `SERVFAIL`, DNS timeouts
- CoreDNS scaling and saturation
- Kubernetes API connectivity
- Node-level networking and conntrack pressure

📄 `runbooks/coredns-dns-latency-timeouts.md`

---

### 2️ VPC CNI IP Exhaustion & Prefix Delegation
**Focus:** Pod scheduling failures caused by IPAM and ENI exhaustion  
**Covers:**
- Pods stuck in `Pending` / `ContainerCreating`
- Subnet IP exhaustion
- Prefix delegation tuning
- ENI limits and instance sizing

📄 `runbooks/vpc-cni-ip-exhaustion-prefix-delegation.md`

---

### 3️ Pod OOMKills & CPU Throttling
**Focus:** Application instability caused by memory and CPU constraints  
**Covers:**
- OOMKilled diagnosis
- CPU throttling impact on latency
- Node pressure and evictions
- Runtime-specific behavior (.NET, JVM, Python)

 `runbooks/pod-oomkill-cpu-throttling.md`

---

### 4️ Karpenter Node Provisioning Failures
**Focus:** Cluster autoscaling and node provisioning issues  
**Covers:**
- Pending pods due to capacity
- NodePool / EC2NodeClass misconfiguration
- IAM and bootstrap failures
- AZ imbalance and IP exhaustion
- Spot vs on-demand behavior

 `runbooks/karpenter-node-provisioning-failures.md`

---

##  Who This Is For
- SREs and Platform Engineers
- Kubernetes and EKS operators
- On-call engineers supporting production workloads
- Engineers preparing for SRE / Platform interviews

---

## Design Philosophy
- Incident-driven, not theory-driven
- Clear separation of **triage**, **diagnosis**, and **resolution**
- Emphasis on **why** failures occur, not just how to fix them
- Optimized for fast decision-making during incidents

---

## Future Additions
- EKS API server & kube-proxy networking failures
- Node-level packet drops and blackholes
- Service mesh & sidecar impact on performance
- Real incident timelines and postmortems

---

## 📌 Disclaimer
These runbooks are based on real-world operational patterns but are generalized and sanitized.  
They are intended for learning, reference, and interview discussion purposes.
