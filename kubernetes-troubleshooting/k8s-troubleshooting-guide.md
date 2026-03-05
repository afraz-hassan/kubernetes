# Kubernetes Troubleshooting Guide — Enterprise Level

> **Author:** Afraz Hassan <br>
> **Last Updated:** March 5, 2026

---
Disclaimer: This document provides general Kubernetes troubleshooting guidance. All resource names, examples, and commands are illustrative and use placeholders. No production infrastructure details are disclosed.

## Table of Contents

1. [Environment Overview](#1-environment-overview)
2. [Quick Reference — Cluster Access](#2-quick-reference--cluster-access)
3. [Node Problems](#3-node-problems)
   - 3.1 [Node NotReady](#31-node-notready)
   - 3.2 [Kured Reboots](#32-kured-reboots)
   - 3.3 [Azure-Initiated Node Restarts](#33-azure-initiated-node-restarts)
   - 3.4 [CPU Pressure / Memory Pressure](#34-cpu-pressure--memory-pressure)
   - 3.5 [Disk Pressure](#35-disk-pressure)
   - 3.6 [PID Pressure](#36-pid-pressure)
   - 3.7 [Node Scaling Issues (VMSS)](#37-node-scaling-issues-vmss)
4. [Pod / Container Restarts](#4-pod--container-restarts)
   - 4.1 [CrashLoopBackOff](#41-crashloopbackoff)
   - 4.2 [OOMKilled](#42-oomkilled)
   - 4.3 [Pod Stuck in Pending](#43-pod-stuck-in-pending)
   - 4.4 [Pod Stuck in Terminating](#44-pod-stuck-in-terminating)
   - 4.5 [ImagePullBackOff](#45-imagepullbackoff)
   - 4.6 [Pod Evicted](#46-pod-evicted)
   - 4.7 [Init Container Failures](#47-init-container-failures)
5. [Networking Issues](#5-networking-issues)
   - 5.1 [Service Unreachable](#51-service-unreachable)
   - 5.2 [Ingress / NGINX Issues](#52-ingress--nginx-issues)
   - 5.3 [Istio Service Mesh Issues](#53-istio-service-mesh-issues)
   - 5.4 [DNS Resolution Failures](#54-dns-resolution-failures)
   - 5.5 [NAT Gateway / External Connectivity](#55-nat-gateway--external-connectivity)
6. [Resource Quota & Limits](#6-resource-quota--limits)
7. [Helm / Deployment Failures](#7-helm--deployment-failures)
8. [Storage Issues](#8-storage-issues)
9. [Certificate & Secret Issues](#9-certificate--secret-issues)
10. [Monitoring & Observability](#10-monitoring--observability)
    - 10.1 [Metrics Server Issues](#102-metrics-server-issues)
11. [Security — Twistlock / Pod Security](#11-security--twistlock--pod-security)
12. [HPA / KEDA Autoscaling Issues](#12-hpa--keda-autoscaling-issues)
13. [Redis Issues](#13-redis-issues)
14. [Azure-Specific Troubleshooting](#14-azure-specific-troubleshooting)
15. [AWS EKS-Specific Troubleshooting](#15-aws-eks-specific-troubleshooting)
16. [Emergency Procedures](#16-emergency-procedures)
17. [Useful One-Liners & Scripts](#17-useful-one-liners--scripts)

---
## 1. Environment Overview

###  Pre-Requisites
Before starting, make sure you have info of Cluster, Region, Resource Group, and Environment.


### Kured Reboot Window
It is recommended to get KURED Reboot schedule which will help you following this document. For Example:
- **Days:** Sunday, Wednesday, Friday
- **Start:** 08:00 AM ET
- **End:** 010:00 AM ET
- **Period:** 30 minutes

--- 

## 2. Quick Reference — Cluster Access

### AKS — Get Credentials

```bash
# Pattern: az aks get-credentials --resource-group <RG> --name <CLUSTER> --subscription <SUB>
```

### Verify Current Context

```bash
kubectl config current-context
kubectl config get-contexts
kubectl cluster-info
```

### Switch Context

```bash
kubectl config use-context <CONTEXT_NAME>
```
<CONTEXT_NAME> is the name is the cluster.

---

## 3. Node Problems

### 3.1 Node NotReady

A node in `NotReady` state means the kubelet is not communicating with the API server.

#### Diagnostic Checklist

- [ ] Is the node visible in `kubectl get nodes`?
- [ ] Was there a kured reboot scheduled?
- [ ] Did Azure perform a maintenance event?
- [ ] Is the node under resource pressure (CPU, memory, disk)?
- [ ] Is the kubelet running on the node?
- [ ] Is the VMSS instance healthy?

#### Step-by-Step Commands

**Step 1: Check node status**
```bash
kubectl get nodes -o wide
```

**Step 2: Get detailed node info**
```bash
kubectl describe node <NODE_NAME>
```
Look for:
- `Conditions` section — check `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`
- `Events` section — look for `NodeNotReady`, `NodeHasSufficientMemory`, etc.
- `Taints` — look for `node.kubernetes.io/not-ready` or `node.kubernetes.io/unreachable`

**Step 3: Check node events (last 1 hour)**
```bash
kubectl get events --field-selector involvedObject.kind=Node,involvedObject.name=<NODE_NAME> --sort-by='.lastTimestamp'
```

**Step 4: Check all cluster events for node issues**
```bash
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | grep -i "node\|NotReady\|reboot\|drain"
```

**Step 5: Check if kured initiated a reboot**
```bash
# Check kured logs
kubectl logs -n kube-system -l app.kubernetes.io/name=kured --tail=200

# Check for kured reboot annotation on nodes
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.weave\.works/kured-reboot-in-progress}{"\n"}{end}'
```

**Step 6: Check Azure Activity Log for the node's VMSS**
```bash
# List VMSS instances
az vmss list-instances --resource-group <RG> --name <VMSS_NAME> -o table

# Check Azure Activity Log for restarts
az monitor activity-log list --resource-group <RG> \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query "[?contains(operationName.value, 'restart') || contains(operationName.value, 'deallocate') || contains(operationName.value, 'start')].{Time:eventTimestamp, Op:operationName.value, Status:status.value}" \
  -o table
```

**Step 7: Check VMSS instance health**
```bash
az vmss get-instance-view --resource-group <RG> --name <VMSS_NAME> --instance-id <INSTANCE_ID>
```

**Step 8: If node stays NotReady — cordon and drain (Not Recommeded For Enterprise (Production)**
```bash
# Cordon the node (prevent new pods)
kubectl cordon <NODE_NAME>

# Drain the node (evict existing pods)
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data --force --grace-period=60

# After node recovers, uncordon
kubectl uncordon <NODE_NAME>
```

---

### 3.2 Kured Reboots

Kured (KUbernetes REboot Daemon) automatically reboots nodes when `/var/run/reboot-required` is present (e.g., after kernel updates).

**Kured has different schedules defined.**

#### Diagnostic Checklist

- [ ] Is it within the kured reboot window?
- [ ] Does the node have the reboot annotation?
- [ ] Is kured running on all nodes?
- [ ] Was the reboot successful?

#### Step-by-Step Commands

**Step 1: Check kured daemonset status**
```bash
kubectl get ds kured -n kube-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=kured -o wide
```

**Step 2: Check kured logs for reboot activity**
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=kured --tail=500 | grep -i "reboot\|drain\|uncordon\|lock"
```

**Step 3: Check if any node has a reboot-in-progress annotation**
```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.weave\.works/kured-reboot-in-progress}{"\n"}{end}'
```

**Step 4: Check if a node is stuck with reboot annotation**
If a node has the annotation but hasn't rebooted (e.g., draining took too long):
```bash
# Remove the stuck annotation to unlock kured
kubectl annotate node <NODE_NAME> weave.works/kured-reboot-in-progress-

# Verify annotation is removed
kubectl get node <NODE_NAME> -o jsonpath='{.metadata.annotations}'
```

**Step 5: Check if nodes need reboot (from kured perspective)**
```bash
# SSH into the node or exec into a privileged pod and check:
cat /var/run/reboot-required
```

**Step 6: Manually block kured if needed (emergency)**
```bash
# Lock kured by adding an annotation to ALL nodes (to block reboots during incident)
kubectl annotate nodes --all kured/reboot-blocked="true"

# Unlock when ready
kubectl annotate nodes --all kured/reboot-blocked-
```

---

### 3.3 Azure-Initiated Node Restarts

Azure may restart nodes for:
- Planned maintenance (host OS updates)
- Unplanned hardware failures
- Live migration events

#### Diagnostic Checklist

- [ ] Check Azure Service Health for planned maintenance
- [ ] Check VMSS instance events
- [ ] Check Azure Activity Log for the resource group
- [ ] Check Scheduled Events API from the node

#### Step-by-Step Commands

**Step 1: Check Azure Service Health**
```bash
# Via Azure CLI
az monitor activity-log list --resource-group <RG> \
  --start-time $(date -u -d '48 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query "[].{Time:eventTimestamp, Operation:operationName.value, Status:status.value, Caller:caller}" \
  -o table
```

**Step 2: Check VMSS events**
```bash
az vmss list-instances --resource-group <RG> --name <VMSS_NAME> \
  --query "[].{Name:name, State:instanceView.statuses[0].displayStatus, ProvisioningState:provisioningState}" -o table
```

**Step 3: Check for Azure Scheduled Events (from inside a node)**
```bash
# If you have node access:
curl -H Metadata:true http://<IP-adress>/metadata/scheduledevents?api-version=2020-07-01
```

**Step 4: Check AKS maintenance configuration**
```bash
az aks maintenanceconfiguration list --resource-group <RG> --cluster-name <CLUSTER> -o table
```

**Step 5: Check node uptime to identify recent restarts**
```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\t"}{.status.conditions[?(@.type=="Ready")].lastTransitionTime}{"\n"}{end}'
```

---

### 3.4 CPU Pressure / Memory Pressure

#### Diagnostic Checklist

- [ ] Which node is under pressure?
- [ ] What pods are consuming the most resources?
- [ ] Are resource limits properly set?
- [ ] Is the node overcommitted?
- [ ] Are there pods without resource limits?

#### Step-by-Step Commands

**Step 1: Check node conditions**
```bash
kubectl get nodes -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,CPU_PRESSURE:.status.conditions[?(@.type=="MemoryPressure")].status,MEM_PRESSURE:.status.conditions[?(@.type=="MemoryPressure")].status,DISK_PRESSURE:.status.conditions[?(@.type=="DiskPressure")].status,PID_PRESSURE:.status.conditions[?(@.type=="PIDPressure")].status'
```

**Step 2: Check node resource usage with metrics-server**
```bash
kubectl top nodes
```

**Step 3: Check node resource allocation (requests vs allocatable)**
```bash
kubectl describe node <NODE_NAME> | grep -A 20 "Allocated resources"
```

**Step 4: Find top resource-consuming pods on a specific node**
```bash
kubectl top pods --all-namespaces --sort-by=cpu | head -30
kubectl top pods --all-namespaces --sort-by=memory | head -30
```

**Step 5: Find pods on a specific node**
```bash
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<NODE_NAME>
```

**Step 6: Find pods WITHOUT resource limits (they can cause pressure)**
```bash
kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.spec.containers[].resources.limits == null) | "\(.metadata.namespace)/\(.metadata.name)"'
```

**Step 7: Check if node is overcommitted**
```bash
# Total requests vs capacity
kubectl describe node <NODE_NAME> | grep -E "cpu|memory" | head -10
```

**Step 8: Quick view — all nodes resource usage percentage**
```bash
kubectl top nodes --no-headers | awk '{printf "%-50s CPU: %s/%s (%s)  MEM: %s/%s (%s)\n", $1, $2, $3, $4, $5, $6, $7}'
```

---

### 3.5 Disk Pressure

#### Diagnostic Checklist

- [ ] Is the node under DiskPressure condition?
- [ ] Are old container images filling the disk?
- [ ] Are emptyDir volumes consuming space?
- [ ] Are container logs too large?

#### Step-by-Step Commands

**Step 1: Check for DiskPressure condition**
```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="DiskPressure")].status}{"\n"}{end}'
```

**Step 2: Check kubelet disk thresholds (default: 85% hard eviction)**
```bash
kubectl get node <NODE_NAME> -o jsonpath='{.status.allocatable}' | jq .
```

**Step 3: If you have node access — check disk usage**
```bash
# From a privileged pod or SSH:
df -h
du -sh /var/lib/docker/* 2>/dev/null | sort -rh | head -10
du -sh /var/log/containers/* 2>/dev/null | sort -rh | head -10
crictl images | sort -k3 -rh | head -20
```

**Step 4: Trigger image garbage collection**
```bash
# On AKS, you can use the docker-disk-cleanup approach:
# crictl rmi --prune     (removes unused images)
```

**Step 5: Find pods using most ephemeral storage**
```bash
kubectl get pods --all-namespaces -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name) \(.spec.containers[].resources.limits.["ephemeral-storage"] // "no-limit")"'
```

---

### 3.6 PID Pressure

#### Step-by-Step Commands

**Step 1: Check for PIDPressure**
```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t PIDPressure: "}{.status.conditions[?(@.type=="PIDPressure")].status}{"\n"}{end}'
```

**Step 2: Find pods with excessive processes**
```bash
# Get pods on the affected node
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<NODE_NAME>

# Check process count from node (if accessible)
ps aux | wc -l
```

---

### 3.7 Node Scaling Issues (VMSS)

#### Step-by-Step Commands

**Step 1: Check AKS cluster autoscaler status**
```bash
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
```

**Step 2: Check autoscaler logs**
```bash
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=200
```

**Step 3: Check VMSS scaling events**
```bash
az vmss list --resource-group <RG> -o table
az vmss show --resource-group <RG> --name <VMSS_NAME> --query "sku" -o table
```

**Step 4: Manually scale VMSS (if autoscaler is not working)**
```bash
az vmss scale --resource-group <RG> --name <VMSS_NAME> --new-capacity <COUNT>
```

---

## 4. Pod / Container Restarts

### 4.1 CrashLoopBackOff

The container keeps crashing and Kubernetes keeps restarting it with an exponential backoff.

#### Diagnostic Checklist

- [ ] What is the exit code?
- [ ] Are liveness/readiness probes failing?
- [ ] Is the application running out of memory?
- [ ] Are required config maps/secrets missing?
- [ ] Are required dependencies (DB, Redis, external APIs) reachable?
- [ ] Is the container image correct and pullable?

#### Step-by-Step Commands

**Step 1: Identify pods in CrashLoopBackOff**
```bash
kubectl get pods -n <NAMESPACE> | grep -i "crash\|error\|backoff"
```

**Step 2: Get pod details**
```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```
Look for:
- `Last State` → `Terminated` → `Exit Code`
- `Restart Count`
- `Events` → container start/failure messages
- `Liveness` / `Readiness` probe results

**Step 3: Check current and previous container logs**
```bash
# Current logs
kubectl logs <POD_NAME> -n <NAMESPACE> -c <CONTAINER_NAME>

# Previous crashed container logs
kubectl logs <POD_NAME> -n <NAMESPACE> -c <CONTAINER_NAME> --previous
```

**Step 4: Common exit codes**
| Exit Code | Meaning | Common Cause |
|-----------|---------|-------------|
| 0 | Success | Application exited cleanly (but K8s restarts it) |
| 1 | Application Error | Code exception, missing config |
| 126 | Permission denied | Cannot execute the entrypoint |
| 127 | Command not found | Wrong entrypoint/command in container spec |
| 137 | SIGKILL (OOMKilled) | Container exceeded memory limit |
| 139 | SIGSEGV | Segmentation fault |
| 143 | SIGTERM | Graceful termination (normal) |

**Step 5: Check if probes are the problem**
```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.containers[*].livenessProbe}' | jq .
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.containers[*].readinessProbe}' | jq .
```

**Step 6: Temporarily disable probes for debugging (do NOT do in prod without approval)**
Modify the deployment to increase probe timeouts or comment out probes to isolate the issue.

**Step 7: Exec into the pod (if it stays up briefly)**
```bash
kubectl exec -it <POD_NAME> -n <NAMESPACE> -c <CONTAINER_NAME> -- /bin/sh
```

---

### 4.2 OOMKilled

Container killed because it exceeded its memory limit.

#### Diagnostic Checklist

- [ ] What is the current memory limit?
- [ ] Is the application leaking memory?
- [ ] Is the JVM heap configured correctly (for Java apps)?
- [ ] Did the workload change (more data, more requests)?

#### Step-by-Step Commands

**Step 1: Confirm OOMKilled**
```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE> | grep -A 5 "Last State"
# Look for: Reason: OOMKilled
```

**Step 2: Check current memory limit**
```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.containers[*].resources}' | jq .
```

**Step 3: Check actual memory usage**
```bash
kubectl top pod <POD_NAME> -n <NAMESPACE> --containers
```

**Step 4: Check memory usage over time (Dynatrace or any monitoring you use)**
Go to Dynatrace → Infrastructure → Kubernetes Workloads → Select the workload → Check memory usage graph.

**Step 5: For Java/Scala applications — check JVM settings**
```bash
kubectl exec -it <POD_NAME> -n <NAMESPACE> -- env | grep -i "java\|jvm\|heap\|xmx\|xms"
```

**Step 6: Increase memory limit if needed**
Your Helm values files define resource limits. For example:
```yaml
resources:
  requests:
    cpu: 100m
  limits:
    memory: 6G    # Increase this
    cpu: 1
```

Update the appropriate values file in your infrastructure repository repo and deploy through the pipeline.

**Step 7: Check namespace quota to ensure room**
```bash
kubectl describe resourcequota -n <NAMESPACE>
kubectl get resourcequota -n <NAMESPACE> -o yaml
```

---

### 4.3 Pod Stuck in Pending

#### Diagnostic Checklist

- [ ] Is there enough node capacity (CPU/memory)?
- [ ] Is the namespace resource quota exceeded?
- [ ] Are there node affinity/anti-affinity constraints?
- [ ] Are there taints preventing scheduling?
- [ ] Is there a missing PersistentVolume?

#### Step-by-Step Commands

**Step 1: Find pending pods**
```bash
kubectl get pods --all-namespaces --field-selector status.phase=Pending
```

**Step 2: Describe the pending pod**
```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
# Look at Events section for scheduling errors:
#   - "Insufficient cpu"
#   - "Insufficient memory"
#   - "0/N nodes are available: N node(s) had taint..."
#   - "exceeded quota"
```

**Step 3: Check namespace quota utilization**
```bash
kubectl describe resourcequota -n <NAMESPACE>
```

**Step 4: Check available resources on nodes**
```bash
kubectl top nodes
kubectl describe nodes | grep -A 10 "Allocated resources"
```

**Step 5: Check for taints on nodes**
```bash
kubectl get nodes -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints'
```

**Step 6: Check node affinity rules on the pod**
```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.affinity}' | jq .
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.nodeSelector}' | jq .
```

---

### 4.4 Pod Stuck in Terminating

#### Diagnostic Checklist

- [ ] Is the pod stuck due to a finalizer?
- [ ] Is the node unreachable?
- [ ] Is there a PDB (Pod Disruption Budget) preventing termination?
- [ ] Is Istio sidecar preventing graceful shutdown?

#### Step-by-Step Commands

**Step 1: Check stuck terminating pods**
```bash
kubectl get pods --all-namespaces | grep Terminating
```

**Step 2: Check for finalizers**
```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.metadata.finalizers}'
```

**Step 3: Force delete the pod (last resort)**
```bash
kubectl delete pod <POD_NAME> -n <NAMESPACE> --grace-period=0 --force
```

**Step 4: If the node is unreachable and pods are stuck**
```bash
# The pods will remain Terminating until the node comes back
# If the node is permanently gone, delete it from Kubernetes
kubectl delete node <NODE_NAME>
# The VMSS autoscaler or AKS will provision a replacement
```

---

### 4.5 ImagePullBackOff

#### Diagnostic Checklist

- [ ] Is the image tag correct?
- [ ] Is the container registry accessible?
- [ ] Are the image pull secrets valid?
- [ ] Is there a network issue to the registry?
- [ ] Is the registry rate-limited?

#### Step-by-Step Commands

**Step 1: Find ImagePullBackOff pods**
```bash
kubectl get pods -n <NAMESPACE> | grep -i "imagepull\|errimagepull"
```

**Step 2: Check the exact error**
```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE> | grep -A 10 "Events"
```

**Step 3: Check which image is failing**
```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.containers[*].image}'
```

**Step 4: Verify image pull secrets**
```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.imagePullSecrets}'
kubectl get secret <SECRET_NAME> -n <NAMESPACE> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

**Step 5: Test registry access manually**
```bash
# For ACR:
az acr login --name <ACR_NAME>
az acr repository show-tags --name <ACR_NAME> --repository <REPO> -o table

# Check if the specific tag exists
az acr repository show --name <ACR_NAME> --image <REPO>:<TAG>
```

---

### 4.6 Pod Evicted

#### Diagnostic Checklist

- [ ] Was the node under memory/disk pressure?
- [ ] Is the pod using more ephemeral storage than allowed?
- [ ] Are the pod's QoS class and priority appropriate?

#### Step-by-Step Commands

**Step 1: Find evicted pods**
```bash
kubectl get pods --all-namespaces --field-selector status.phase=Failed | grep Evicted
```

**Step 2: Check eviction reason**
```bash
kubectl describe pod <EVICTED_POD> -n <NAMESPACE> | grep -i "evict\|reason\|message"
```

**Step 3: Clean up evicted pods**
```bash
kubectl get pods --all-namespaces --field-selector status.phase=Failed -o json | \
  jq -r '.items[] | select(.status.reason=="Evicted") | "\(.metadata.namespace) \(.metadata.name)"' | \
  xargs -L1 bash -c 'kubectl delete pod $1 -n $0'
```

**Step 4: Check pod QoS class**
```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.status.qosClass}'
# Guaranteed > Burstable > BestEffort (eviction priority order)
```

---

### 4.7 Init Container Failures

#### Step-by-Step Commands

**Step 1: Check init container status**
```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE> | grep -A 20 "Init Containers"
```

**Step 2: Check init container logs**
```bash
kubectl logs <POD_NAME> -n <NAMESPACE> -c <INIT_CONTAINER_NAME>
```

**Step 3: Common init container issues for Istio (istio-init)**
```bash
# Check if istio-init is failing
kubectl logs <POD_NAME> -n <NAMESPACE> -c istio-init
# Usually related to iptables rules — check pod security policies
```

---

## 5. Networking Issues

### 5.1 Service Unreachable

#### Diagnostic Checklist

- [ ] Is the service endpoint healthy?
- [ ] Is the pod behind the service running?
- [ ] Is the service selector matching pod labels?
- [ ] Is there a NetworkPolicy blocking traffic?

#### Step-by-Step Commands

**Step 1: Check service and its endpoints**
```bash
kubectl get svc <SERVICE_NAME> -n <NAMESPACE>
kubectl get endpoints <SERVICE_NAME> -n <NAMESPACE>
```

**Step 2: Verify endpoints have IP addresses**
```bash
# If endpoints list is empty, the service selector doesn't match any running pods
kubectl describe endpoints <SERVICE_NAME> -n <NAMESPACE>
```

**Step 3: Check labels match**
```bash
# Get service selector
kubectl get svc <SERVICE_NAME> -n <NAMESPACE> -o jsonpath='{.spec.selector}'

# Get pod labels
kubectl get pods -n <NAMESPACE> --show-labels | grep <APP_NAME>
```

**Step 4: Test connectivity from another pod**
```bash
kubectl run test-curl --image=curlimages/curl --restart=Never -n <NAMESPACE> -- \
  curl -v http://<SERVICE_NAME>.<NAMESPACE>.svc.cluster.local:<PORT>/health
kubectl logs test-curl -n <NAMESPACE>
kubectl delete pod test-curl -n <NAMESPACE>
```

**Step 5: Check NetworkPolicies**
```bash
kubectl get networkpolicies -n <NAMESPACE>
kubectl describe networkpolicy <POLICY_NAME> -n <NAMESPACE>
```

---

### 5.2 Ingress / NGINX Issues

#### Diagnostic Checklist

- [ ] Is the ingress resource configured correctly?
- [ ] Is the NGINX ingress controller running?
- [ ] Is the TLS certificate valid?
- [ ] Is the backend service healthy?

#### Step-by-Step Commands

**Step 1: Check ingress resources**
```bash
kubectl get ingress -n <NAMESPACE>
kubectl describe ingress <INGRESS_NAME> -n <NAMESPACE>
```

**Step 2: Check NGINX ingress controller pods**
```bash
kubectl get pods -n <ingress-nginx-namespace>
kubectl logs -n <ingress-nginx-namespace> -l app.kubernetes.io/component=controller --tail=200
```

**Step 3: Check NGINX ingress controller for errors**
```bash
kubectl logs -n <ingress-nginx-namespace> -l app.kubernetes.io/component=controller --tail=500 | grep -i "error\|warn\|503\|502\|504"
```

**Step 4: Check if external ingress controller is also healthy**
```bash
kubectl get pods -n <ingress-nginx-ext-namespace>
kubectl logs -n <ingress-nginx-ext-namespace> -l app.kubernetes.io/component=controller --tail=200
```

**Step 5: Verify ingress annotations are correct**
Your services typically use:
```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: 1024m
```

**Step 6: Test ingress endpoint directly**
```bash
curl -vk https://<INGRESS_HOST>/<PATH>
```

**Step 7: Check NGINX configuration for specific backend**
```bash
kubectl exec -it -n <ingress-nginx-namespace> <NGINX_POD> -- cat /etc/nginx/nginx.conf | grep -A 10 "<SERVICE_NAME>"
```

---

### 5.3 Istio Service Mesh Issues

#### Diagnostic Checklist

- [ ] Is istio-proxy sidecar injected in the pod?
- [ ] Is istiod (control plane) running?
- [ ] Are AuthorizationPolicies blocking traffic?
- [ ] Is mTLS configured correctly?

#### Step-by-Step Commands

**Step 1: Check istiod status**
```bash
kubectl get pods -n <istio-system-namespace>
kubectl logs -n <istio-system-namespace> -l app=istiod --tail=200
```

**Step 2: Check if sidecar is injected**
```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.containers[*].name}'
# Should see: <app-container> istio-proxy
```

**Step 3: Check istio-proxy logs**
```bash
kubectl logs <POD_NAME> -n <NAMESPACE> -c istio-proxy --tail=200
```

**Step 4: Check proxy sync status**
```bash
# Using istioctl if available:
istioctl proxy-status
istioctl analyze -n <NAMESPACE>
```

**Step 5: Check AuthorizationPolicies**
```bash
kubectl get authorizationpolicies -n <NAMESPACE>
kubectl describe authorizationpolicy <POLICY_NAME> -n <NAMESPACE>
```

**Step 6: Check if mTLS is enforced**
```bash
kubectl get peerauthentication --all-namespaces
```

**Step 7: Debug Envoy proxy configuration**
```bash
kubectl exec -it <POD_NAME> -n <NAMESPACE> -c istio-proxy -- pilot-agent request GET clusters
kubectl exec -it <POD_NAME> -n <NAMESPACE> -c istio-proxy -- pilot-agent request GET config_dump
```

---

### 5.4 DNS Resolution Failures

#### Step-by-Step Commands

**Step 1: Check CoreDNS is running**
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
```

**Step 2: Test DNS from a pod**
```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never -- nslookup kubernetes.default
kubectl logs dns-test
kubectl delete pod dns-test
```

**Step 3: Test external DNS resolution**
```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never -- nslookup google.com
kubectl logs dns-test
kubectl delete pod dns-test
```

**Step 4: Check CoreDNS configmap**
```bash
kubectl get configmap coredns -n kube-system -o yaml
```

**Step 5: Check if nodelocaldns is running (for EKS clusters)**
```bash
kubectl get ds nodelocaldns -n kube-system
kubectl get pods -n kube-system -l k8s-app=node-local-dns
```

---

### 5.5 NAT Gateway / External Connectivity

For AKS clusters, outbound traffic flows through NAT Gateway.

#### Step-by-Step Commands

**Step 1: Verify NAT Gateway**
```bash
az network nat gateway show --resource-group <RG> --name <NAT_GW_NAME>
```

**Step 2: Check NAT Gateway public IP**
```bash
az network nat gateway show --resource-group <RG> --name <NAT_GW_NAME> \
  --query "publicIpAddresses[].id" -o table
```

**Step 3: Test outbound connectivity from a pod**
```bash
kubectl run test-curl --image=curlimages/curl --restart=Never -n <NAMESPACE> -- \
  curl -s ifconfig.me
kubectl logs test-curl -n <NAMESPACE>
kubectl delete pod test-curl -n <NAMESPACE>
```

---

## 6. Resource Quota & Limits

#### Step-by-Step Commands

**Step 1: Check all resource quotas in a namespace**
```bash
kubectl describe resourcequota -n <NAMESPACE>
```

**Step 2: Check quota usage across all namespaces**
```bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  echo "=== $ns ==="
  kubectl describe resourcequota -n $ns 2>/dev/null
done
```

**Step 3: Check LimitRange in a namespace**
```bash
kubectl get limitrange -n <NAMESPACE>
kubectl describe limitrange -n <NAMESPACE>
```

**Step 4: Calculate total memory limits for all deployments in a namespace**
```bash
NAMESPACE="<NAMESPACE>"
kubectl get pods -n $NAMESPACE -o json | jq '[.items[].spec.containers[].resources.limits.memory // "0" | rtrimstr("Gi") | rtrimstr("Mi") | tonumber] | add'
```

**Step 5: Check if deployments count quota is exceeded**
```bash
kubectl get resourcequota -n <NAMESPACE> -o jsonpath='{.items[*].status}'  | jq .
```

---

## 7. Helm / Deployment Failures

#### Diagnostic Checklist

- [ ] Is the Helm release in a failed state?
- [ ] Did the Helm chart values change?
- [ ] Is there a version mismatch?
- [ ] Did the deployment timeout?

#### Step-by-Step Commands

**Step 1: Check Helm release status**
```bash
helm list -n <NAMESPACE>
helm list -n <NAMESPACE> --failed
```

**Step 2: Check Helm release history**
```bash
helm history <RELEASE_NAME> -n <NAMESPACE>
```

**Step 3: Check what changed in the last release**
```bash
helm get values <RELEASE_NAME> -n <NAMESPACE>
helm get manifest <RELEASE_NAME> -n <NAMESPACE>
```

**Step 4: Debug a failed Helm install/upgrade**
```bash
helm upgrade --install <RELEASE_NAME> <CHART> -n <NAMESPACE> --debug --dry-run -f values.yaml
```

**Step 5: Rollback a failed release**
```bash
helm rollback <RELEASE_NAME> <REVISION> -n <NAMESPACE>
```

**Step 6: Force delete a stuck release**
```bash
helm uninstall <RELEASE_NAME> -n <NAMESPACE>
# If that fails:
kubectl delete secret -n <NAMESPACE> -l owner=helm,name=<RELEASE_NAME>
```

**Step 7: Check deployment rollout status**
```bash
kubectl rollout status deployment/<DEPLOYMENT_NAME> -n <NAMESPACE>
kubectl rollout history deployment/<DEPLOYMENT_NAME> -n <NAMESPACE>
```

**Step 8: Rollback a deployment**
```bash
kubectl rollout undo deployment/<DEPLOYMENT_NAME> -n <NAMESPACE>
# Or to a specific revision:
kubectl rollout undo deployment/<DEPLOYMENT_NAME> -n <NAMESPACE> --to-revision=<REVISION>
```

---

## 8. Storage Issues

#### Diagnostic Checklist

- [ ] Is the PVC bound?
- [ ] Is the StorageClass available?
- [ ] Is the PV provisioner healthy?
- [ ] Are Azure Disks/Files quotas exhausted?

#### Step-by-Step Commands

**Step 1: Check PVC status**
```bash
kubectl get pvc -n <NAMESPACE>
kubectl describe pvc <PVC_NAME> -n <NAMESPACE>
```

**Step 2: Check PV status**
```bash
kubectl get pv
kubectl describe pv <PV_NAME>
```

**Step 3: Check StorageClasses**
```bash
kubectl get storageclass
```

**Step 4: Check CSI driver pods (Azure)**
```bash
kubectl get pods -n kube-system -l app=csi-azuredisk-node
kubectl get pods -n kube-system -l app=csi-azurefile-node
```

**Step 5: Check CSI secrets store (for KeyVault)**
```bash
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get secretproviderclass -n <NAMESPACE>
```

---

## 9. Certificate & Secret Issues

#### Diagnostic Checklist

- [ ] Is the TLS certificate expired?
- [ ] Is the SPN (Service Principal) credential expired?
- [ ] Are KeyVault secrets synced?
- [ ] Is the CSI SecretProviderClass configured correctly?

#### Step-by-Step Commands

**Step 1: Check TLS secret**
```bash
kubectl get secret <TLS_SECRET> -n <NAMESPACE>
kubectl get secret <TLS_SECRET> -n <NAMESPACE> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates -subject
```

**Step 2: Check SPN expiry dates**

```bash
# Check SPN in Azure AD
az ad sp credential list --id <SPN_APP_ID> -o table
```

**Step 3: Check KeyVault CSI sync status**
```bash
kubectl get secretproviderclasspodstatus -n <NAMESPACE>
kubectl describe secretproviderclass <SPC_NAME> -n <NAMESPACE>
```

**Step 4: Check if KeyVault secrets are accessible**
```bash
az keyvault secret list --vault-name <VAULT_NAME> -o table
# e.g.: az keyvault secret list --vault-name <KEYVAULT_NAME> -o table
```

---

## 10. Monitoring & Observability

### 10.1 Metrics Server Issues

#### Step-by-Step Commands

**Step 1: Check metrics server**
```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes  # If this fails, metrics server is down
kubectl top pods -n <NAMESPACE>
```

**Step 2: Check metrics server logs**
```bash
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=100
```

---

## 11. Security — Twistlock / Pod Security

Check the twistlock security version.

#### Step-by-Step Commands

**Step 1: Check Twistlock Defender**
```bash
kubectl get ds -n twistlock
kubectl get pods -n twistlock -o wide
```

**Step 2: Check Twistlock Defender logs**
```bash
kubectl logs -n twistlock <DEFENDER_POD> --tail=200
```

**Step 3: Check Pod Security Standards (PSS) violations**
```bash
# Check namespace labels for PSS enforcement
kubectl get ns <NAMESPACE> -o jsonpath='{.metadata.labels}' | jq . | grep "pod-security"
```

**Step 4: Check for PSS-blocked pods**
```bash
kubectl get events --all-namespaces | grep -i "forbidden\|violat\|security"
```

---

## 12. HPA / KEDA Autoscaling Issues

#### Diagnostic Checklist

- [ ] Is the HPA configured?
- [ ] Is KEDA installed and running?
- [ ] Are metrics available (CPU/custom)?
- [ ] Is the target deployment correct?

#### Step-by-Step Commands

**Step 1: Check HPA status**
```bash
kubectl get hpa -n <NAMESPACE>
kubectl describe hpa <HPA_NAME> -n <NAMESPACE>
```

**Step 2: Check current vs desired replicas**
```bash
kubectl get hpa -n <NAMESPACE> -o custom-columns='NAME:.metadata.name,MIN:.spec.minReplicas,MAX:.spec.maxReplicas,CURRENT:.status.currentReplicas,DESIRED:.status.desiredReplicas,CPU_UTIL:.status.currentMetrics[0].resource.current.averageUtilization'
```

**Step 3: Check KEDA operator**
```bash
kubectl get pods -n keda
kubectl logs -n keda -l app=keda-operator --tail=200
```

**Step 4: Check KEDA ScaledObjects**
```bash
kubectl get scaledobjects -n <NAMESPACE>
kubectl describe scaledobject <SO_NAME> -n <NAMESPACE>
```

**Step 5: Check if metrics are being reported (common HPA issue)**
```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/<NAMESPACE>/pods" | jq .
```

---

## 13. Redis Issues

Many of your services use embedded Redis (via Helm charts with `redis.enabled: true`).

#### Diagnostic Checklist

- [ ] Is the Redis pod running?
- [ ] Is Redis responding to PING?
- [ ] Is Redis using too much memory?
- [ ] Is Redis in read-only mode (maxmemory reached)?

#### Step-by-Step Commands

**Step 1: Find Redis pods**
```bash
kubectl get pods -n <NAMESPACE> | grep redis
```

**Step 2: Check Redis pod logs**
```bash
kubectl logs <REDIS_POD> -n <NAMESPACE> --tail=200
```

**Step 3: Test Redis connectivity**
```bash
kubectl exec -it <REDIS_POD> -n <NAMESPACE> -- redis-cli ping
# Response should be: PONG
```

**Step 4: Check Redis memory usage**
```bash
kubectl exec -it <REDIS_POD> -n <NAMESPACE> -- redis-cli info memory
```

**Step 5: Check Redis configuration**
```bash
kubectl exec -it <REDIS_POD> -n <NAMESPACE> -- redis-cli config get maxmemory
kubectl exec -it <REDIS_POD> -n <NAMESPACE> -- redis-cli config get maxmemory-policy
```
Your LRU-configured Redis instances use:
```
maxmemory 400mb
maxmemory-policy allkeys-lru
```

**Step 6: Flush Redis cache (if needed, non-prod only)**
```bash
kubectl exec -it <REDIS_POD> -n <NAMESPACE> -- redis-cli FLUSHALL
```

---

## 14. Azure-Specific Troubleshooting

### AKS Cluster Health

```bash
# Check AKS cluster provisioning state
az aks show --resource-group <RG> --name <CLUSTER> --query "provisioningState" -o tsv

# Check AKS cluster power state
az aks show --resource-group <RG> --name <CLUSTER> --query "powerState" -o tsv

# Check node pool status
az aks nodepool list --resource-group <RG> --cluster-name <CLUSTER> -o table

# Check for Azure Advisor recommendations
az advisor recommendation list --resource-group <RG> -o table
```

### Azure Resource Quotas

```bash
# Check VM quota usage in the region
az vm list-usage --location <AZURE_REGION> -o table | grep -i "Standard E"
az vm list-usage --location <AZURE_REGION> -o table | grep -i "Standard E"
```

### AKS Upgrade Status

```bash
# Check available upgrades
az aks get-upgrades --resource-group <RG> --name <CLUSTER> -o table

# Check current version
az aks show --resource-group <RG> --name <CLUSTER> --query "kubernetesVersion" -o tsv
```

### Azure Network Issues

```bash
# Check NSG rules
az network nsg list --resource-group <RG> -o table
az network nsg rule list --resource-group <RG> --nsg-name <NSG_NAME> -o table

# Check route table
az network route-table list --resource-group <RG> -o table
```

---

## 15. AWS EKS-Specific Troubleshooting

### EKS Cluster Health

```bash
# Check cluster status
aws eks describe-cluster --name <CLUSTER> --query "cluster.status" --output text

# Check node groups
aws eks list-nodegroups --cluster-name <CLUSTER>
aws eks describe-nodegroup --cluster-name <CLUSTER> --nodegroup-name <NG_NAME>
```

### EKS Auth Issues

```bash
# Check aws-auth configmap
kubectl get configmap aws-auth -n kube-system -o yaml
```

### VPC CNI Issues

```bash
# Check VPC CNI pods
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100

# Check available IPs
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.pods}{"\n"}{end}'
```

### EFS/EBS Issues

```bash
# Check EFS CSI driver
kubectl get pods -n kube-system -l app=efs-csi-node

# Check EBS CSI driver
kubectl get pods -n kube-system -l app=ebs-csi-node
```

---

## 16. Emergency Procedures

### 16.1 Emergency Pod Restart (Single Deployment)

```bash
kubectl rollout restart deployment/<DEPLOYMENT_NAME> -n <NAMESPACE>
```

### 16.2 Emergency — Restart All Pods in a Namespace

```bash
# WARNING: This will restart ALL deployments in the namespace
kubectl get deploy -n <NAMESPACE> -o name | xargs -I{} kubectl rollout restart {} -n <NAMESPACE>
```

### 16.3 Emergency — Cordon a Problematic Node

```bash
# Prevent new pods from being scheduled
kubectl cordon <NODE_NAME>

# Move existing pods off the node
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data --force --grace-period=120

# After the issue is resolved
kubectl uncordon <NODE_NAME>
```

### 16.4 Emergency — Scale Down / Scale Up

```bash
# Scale down (e.g., to stop a misbehaving deployment)
kubectl scale deployment/<DEPLOYMENT_NAME> -n <NAMESPACE> --replicas=0

# Scale back up
kubectl scale deployment/<DEPLOYMENT_NAME> -n <NAMESPACE> --replicas=<DESIRED>
```

### 16.5 Emergency — Block Kured Reboots

```bash
# Block all kured reboots cluster-wide
kubectl annotate nodes --all kured/reboot-blocked="emergency-$(date +%Y%m%d)"

# Unblock when ready
kubectl annotate nodes --all kured/reboot-blocked-
```

### 16.6 Emergency — Delete All Evicted Pods

```bash
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.status.phase=="Failed" and .status.reason=="Evicted") | "\(.metadata.namespace) \(.metadata.name)"' | \
  while read ns pod; do kubectl delete pod $pod -n $ns; done
```

### 16.7 Emergency — Force Delete Stuck Namespace

```bash
# If a namespace is stuck in Terminating:
kubectl get namespace <NAMESPACE> -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw "/api/v1/namespaces/<NAMESPACE>/finalize" -f -
```

---

## 17. Useful One-Liners & Scripts

### Get All Pods with High Restart Counts

```bash
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.status.containerStatuses[]?.restartCount > 5) | "\(.metadata.namespace)\t\(.metadata.name)\t\(.status.containerStatuses[].restartCount)"' | \
  sort -t$'\t' -k3 -rn | column -t
```

### Get All Pods Not in Running State

```bash
kubectl get pods --all-namespaces --field-selector 'status.phase!=Running,status.phase!=Succeeded' -o wide
```

### Get All Failed Events in Last Hour

```bash
kubectl get events --all-namespaces --sort-by='.lastTimestamp' --field-selector type=Warning | tail -50
```

### Node Resource Summary

```bash
echo "NODE                                CPU_REQ  CPU_LIM  MEM_REQ    MEM_LIM"
kubectl get nodes --no-headers | while read node rest; do
  echo -n "$node  "
  kubectl describe node $node | grep -A 4 "Allocated resources" | tail -3 | head -2 | awk '{printf "%s %s  ", $2, $3}'
  echo
done
```

### Get Total Memory Limits per Namespace

```bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  total=$(kubectl get pods -n $ns -o json 2>/dev/null | jq '[.items[].spec.containers[].resources.limits.memory // "0" | gsub("Gi";"") | gsub("Mi";"") | gsub("G";"") | tonumber] | add // 0')
  if [ "$total" != "0" ] && [ "$total" != "null" ]; then
    echo "$ns: ${total}"
  fi
done
```

### Find Pods Consuming Most CPU

```bash
kubectl top pods --all-namespaces --sort-by=cpu --no-headers | head -20
```

### Find Pods Consuming Most Memory

```bash
kubectl top pods --all-namespaces --sort-by=memory --no-headers | head -20
```

### Watch Pod Restarts in Real-Time

```bash
watch -n 5 'kubectl get pods -n <NAMESPACE> --sort-by=".status.containerStatuses[0].restartCount" | tail -20'
```

### Quick Cluster Health Check Script

```bash
#!/bin/bash
echo "========== CLUSTER HEALTH CHECK =========="
echo ""
echo "--- Cluster Info ---"
kubectl cluster-info
echo ""

echo "--- Node Status ---"
kubectl get nodes -o wide
echo ""

echo "--- Node Resource Usage ---"
kubectl top nodes
echo ""

echo "--- Nodes with Conditions ---"
kubectl get nodes -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,MEM_PRESSURE:.status.conditions[?(@.type=="MemoryPressure")].status,DISK_PRESSURE:.status.conditions[?(@.type=="DiskPressure")].status,PID_PRESSURE:.status.conditions[?(@.type=="PIDPressure")].status'
echo ""

echo "--- Pods Not Running ---"
kubectl get pods --all-namespaces --field-selector 'status.phase!=Running,status.phase!=Succeeded' --no-headers 2>/dev/null | head -20
echo ""

echo "--- Pods with High Restart Count (>5) ---"
kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.status.containerStatuses[]?.restartCount > 5) | "\(.metadata.namespace)\t\(.metadata.name)\t\(.status.containerStatuses[].restartCount)"' | sort -t$'\t' -k3 -rn | head -20
echo ""

echo "--- Recent Warning Events ---"
kubectl get events --all-namespaces --sort-by='.lastTimestamp' --field-selector type=Warning --no-headers 2>/dev/null | tail -20
echo ""

echo "--- Kured Reboot Status ---"
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}: kured-reboot={.metadata.annotations.weave\.works/kured-reboot-in-progress}{"\n"}{end}'
echo ""

echo "--- System Pods Health ---"
echo "kube-system:"
kubectl get pods -n kube-system --no-headers | grep -v Running | grep -v Completed
echo "ingress-nginx:"
kubectl get pods -n ingress-nginx --no-headers 2>/dev/null | grep -v Running
echo "istio-system:"
kubectl get pods -n istio-system --no-headers 2>/dev/null | grep -v Running
echo "dynatrace:"
kubectl get pods -n dynatrace --no-headers 2>/dev/null | grep -v Running
echo "twistlock:"
kubectl get pods -n twistlock --no-headers 2>/dev/null | grep -v Running
echo ""

echo "========== HEALTH CHECK COMPLETE =========="
```

### Compare Namespace Quota Usage

```bash
kubectl get resourcequota --all-namespaces -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,CPU_USED:.status.used.limits\.cpu,CPU_HARD:.status.hard.limits\.cpu,MEM_USED:.status.used.limits\.memory,MEM_HARD:.status.hard.limits\.memory'
```

### Watch Deployments in a Namespace

```bash
watch -n 10 'kubectl get deploy -n <NAMESPACE> -o custom-columns="NAME:.metadata.name,READY:.status.readyReplicas,DESIRED:.spec.replicas,AVAILABLE:.status.availableReplicas,UP-TO-DATE:.status.updatedReplicas"'
```

---

## Quick Decision Tree

```
Problem Detected
│
├── Node Issue?
│   ├── NotReady → §3.1
│   ├── Reboot occurred? → Check kured §3.2, then Azure §3.3
│   ├── High CPU/Mem? → §3.4
│   ├── Disk full? → §3.5
│   └── Scaling needed? → §3.7
│
├── Pod Issue?
│   ├── CrashLoopBackOff → §4.1
│   ├── OOMKilled → §4.2
│   ├── Pending → §4.3
│   ├── Terminating (stuck) → §4.4
│   ├── ImagePullBackOff → §4.5
│   ├── Evicted → §4.6
│   └── Init container fail → §4.7
│
├── Network Issue?
│   ├── Service unreachable → §5.1
│   ├── Ingress 502/503/504 → §5.2
│   ├── Istio blocking → §5.3
│   ├── DNS failure → §5.4
│   └── External connectivity → §5.5
│
├── Deployment Issue?
│   ├── Helm failed → §7
│   └── Quota exceeded → §6
│
├── Storage Issue? → §8
├── Certificate/Secret expired? → §9
├── Monitoring broken? → §10
├── Security blocking? → §11
├── Autoscaling not working? → §12
└── Redis issue? → §13
```





