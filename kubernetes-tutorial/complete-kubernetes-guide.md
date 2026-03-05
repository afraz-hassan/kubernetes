# Kubernetes: Complete Guide (Beginner → Advanced)

> **Author:** Afraz Hassan <br>
> **Last Updated:** March 2, 2026
---

## Table of contents

1. [What Kubernetes is (and isn’t)](#what-kubernetes-is-and-isnt)
2. [Core ideas you must understand first](#core-ideas-you-must-understand-first)
3. [Kubernetes architecture](#kubernetes-architecture)
4. [kubectl, YAML, and the API](#kubectl-yaml-and-the-api)
5. [Workloads: Pods, Deployments, Jobs, StatefulSets, DaemonSets](#workloads-pods-deployments-jobs-statefulsets-daemonsets)
6. [Configuration: Labels, Selectors, Annotations](#configuration-labels-selectors-annotations)
7. [ConfigMaps, Secrets, and environment injection](#configmaps-secrets-and-environment-injection)
8. [Services, Endpoints, Ingress, and Gateway API](#services-endpoints-ingress-and-gateway-api)
9. [Networking fundamentals (CNI, DNS, kube-proxy)](#networking-fundamentals-cni-dns-kube-proxy)
10. [Storage: Volumes, PV/PVC, StorageClasses, CSI](#storage-volumes-pvpvc-storageclasses-csi)
11. [Scheduling: nodes, taints/tolerations, affinity, topology](#scheduling-nodes-taintstolerations-affinity-topology)
12. [Resource management: requests/limits, QoS, quotas](#resource-management-requestslimits-qos-quotas)
13. [Autoscaling: HPA, VPA, Cluster Autoscaler, KEDA](#autoscaling-hpa-vpa-cluster-autoscaler-keda)
14. [Health, rollouts, and disruption](#health-rollouts-and-disruption)
15. [Security: RBAC, service accounts, Pod Security, network policies](#security-rbac-service-accounts-pod-security-network-policies)
16. [Observability: logs, metrics, traces, events](#observability-logs-metrics-traces-events)
17. [Troubleshooting playbook](#troubleshooting-playbook)
18. [Operating clusters: upgrades, backups, DR, lifecycle](#operating-clusters-upgrades-backups-dr-lifecycle)
19. [Multi-tenancy and enterprise patterns](#multi-tenancy-and-enterprise-patterns)
20. [GitOps and continuous delivery](#gitops-and-continuous-delivery)
21. [Service meshes and advanced traffic management](#service-meshes-and-advanced-traffic-management)
22. [Cost, performance, and capacity planning](#cost-performance-and-capacity-planning)
23. [A practical “day-2” checklist](#a-practical-day-2-checklist)
24. [Glossary (quick definitions)](#glossary-quick-definitions)
25. [Deep dive: Pod and container lifecycle](#deep-dive-pod-and-container-lifecycle)
26. [Deep dive: The Kubernetes API request path](#deep-dive-the-kubernetes-api-request-path)
27. [Deep dive: Controllers, reconciliation, and watches](#deep-dive-controllers-reconciliation-and-watches)
28. [Deep dive: CRDs and Operators (extending Kubernetes)](#deep-dive-crds-and-operators-extending-kubernetes)
29. [Deep dive: Services and load balancing internals](#deep-dive-services-and-load-balancing-internals)
30. [Deep dive: NetworkPolicy patterns](#deep-dive-networkpolicy-patterns)
31. [Deep dive: Storage details you’ll need in production](#deep-dive-storage-details-youll-need-in-production)
32. [Deep dive: Runtime and node security](#deep-dive-runtime-and-node-security)
33. [Enterprise runbooks (what operators actually do)](#enterprise-runbooks-what-operators-actually-do)
34. [Deep dive: Kubernetes object model (metadata, deletion, finalizers)](#deep-dive-kubernetes-object-model-metadata-deletion-finalizers)
35. [Deep dive: RBAC in practice (patterns that scale)](#deep-dive-rbac-in-practice-patterns-that-scale)
36. [Deep dive: TLS, certificates, and cert-manager](#deep-dive-tls-certificates-and-cert-manager)
37. [Deep dive: Advanced scheduling (priority, preemption, descheduler)](#deep-dive-advanced-scheduling-priority-preemption-descheduler)
38. [Deep dive: Progressive delivery (canary, blue-green)](#deep-dive-progressive-delivery-canary-blue-green)
39. [Deep dive: Multi-cluster and fleet basics](#deep-dive-multi-cluster-and-fleet-basics)
40. [Appendix: kubectl cheat sheet](#appendix-kubectl-cheat-sheet)

---

## What Kubernetes is (and isn’t)

Kubernetes (often written as **K8s**) is a system that runs containerized applications across many machines and keeps them healthy over time.

### What Kubernetes does well

- **Schedules** containers onto machines (nodes).
- **Keeps things running**: if a container crashes, Kubernetes restarts it.
- **Scales**: more replicas when load increases.
- **Rolls out updates** safely (and can roll back).
- **Provides networking** and service discovery.
- **Manages configuration** and secrets.
- **Lets you declare** the desired state (“I want 6 replicas”) and continuously tries to make reality match.

### What Kubernetes is NOT

- Not a CI/CD tool (it runs what you ship, but it doesn’t build your code).
- Not a logging/monitoring tool by itself.
- Not “just Docker.” Kubernetes can run container runtimes other than Docker.
- Not a silver bullet: you still need good app design, security practices, and operational discipline.

### Kubernetes mental model (the simplest one)

Think of Kubernetes like a **very strict, automated operations team**:

- You tell it what you want (desired state).
- It checks the cluster continuously.
- When something drifts (a pod dies, a node disappears), it fixes it.

---

## Core ideas you must understand first

### 1) Desired state

Kubernetes works from *declarations*.

- You don’t normally say: “start container X now.”
- You say: “I want a Deployment with 3 replicas.”

Kubernetes will keep trying to ensure 3 healthy pods exist.

### 2) Controllers

A **controller** is a loop that watches the cluster and takes actions.

Examples:

- Deployment controller ensures the right number of replicas.
- Job controller ensures a job completes.
- Node controller reacts to node failures.

### 3) The API is the center of everything

Everything in Kubernetes is an **API object** stored in a database (etcd) and managed via the Kubernetes API server.

- `kubectl` talks to the API.
- Controllers talk to the API.
- Operators talk to the API.
- Even “kubectl apply” is basically “send JSON/YAML to the API server.”

### 4) Kubernetes is built for failure

Nodes die, networks split, containers crash. Kubernetes assumes that happens and is designed to recover.

---

## Kubernetes architecture

### Cluster components (big picture)

A Kubernetes cluster has:

- **Control plane**: brains of the cluster.
- **Worker nodes**: run your application pods.

### Control plane components

1) **kube-apiserver**

- The front door.
- Validates requests (authn/authz/admission).
- Reads/writes state in etcd.

2) **etcd**

- The cluster’s database.
- Stores all Kubernetes object state.
- If etcd is unhealthy, the cluster is in serious trouble.

3) **kube-scheduler**

- Decides which node should run a newly-created pod.
- Uses resource info, affinity rules, taints/tolerations, and more.

4) **kube-controller-manager**

- Runs core controllers (deployment, node, endpoints, etc.).

5) **cloud-controller-manager** (in cloud environments)

- Integrates with cloud APIs (load balancers, routes, node lifecycle).

### Worker node components

1) **kubelet**

- Agent on each node.
- Ensures containers for pods are running.
- Reports node/pod status back to the API server.

2) **container runtime**

- Actually runs containers (containerd, CRI-O, etc.).

3) **kube-proxy**

- Implements Service networking (usually via iptables/ipvs rules).

4) **CNI plugin**

- Provides pod networking (Calico, Cilium, Flannel, etc.).

### Namespaces (a basic but important boundary)

A **namespace** is a logical grouping of resources.

- It helps organize things.
- It’s also used for RBAC separation and quotas.

It is **not** a strong security boundary by itself. Treat it as “organize + policy boundary,” not “hard isolation.”

---

## kubectl, YAML, and the API

### The three `kubectl` commands you use constantly

- `kubectl get ...` → list things
- `kubectl describe ...` → detailed view + events
- `kubectl logs ...` → application logs

### Use contexts correctly (avoid disasters)

Kubeconfig can contain multiple clusters/contexts.

- Always check: `kubectl config current-context`
- Consider setting a strong prompt to show context and namespace.

### YAML basics

Kubernetes resources are described in YAML.

Every object commonly has:

- `apiVersion`
- `kind`
- `metadata` (name, namespace, labels)
- `spec` (desired state)
- `status` (actual state; set by Kubernetes)

Example (a tiny Pod):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
  labels:
    app: hello
spec:
  containers:
  - name: hello
    image: nginx:1.27
    ports:
    - containerPort: 80
```

### Imperative vs declarative

Imperative (quick):

```bash
kubectl create deployment web --image=nginx
kubectl scale deployment web --replicas=5
```

Declarative (preferred for real systems):

```bash
kubectl apply -f deployment.yaml
kubectl diff -f deployment.yaml
```

### `apply` and “last applied configuration”

`kubectl apply` stores a copy of what you applied so it can calculate a patch later.

In GitOps environments, you typically:

- Keep YAML in Git
- Reconcile continuously (Argo CD / Flux)

---

## Workloads: Pods, Deployments, Jobs, StatefulSets, DaemonSets

### Pod

A **Pod** is the smallest runnable unit.

- A pod can have **one or more containers**.
- Containers in the same pod share:
  - network namespace (same IP/port space)
  - volumes

#### When multi-container pods make sense

- Sidecar pattern (log shipper, proxy, agent)
- Init containers (one-time setup)

### Deployment (stateless apps)

A **Deployment** manages ReplicaSets which manage Pods.

You use Deployments for:

- web services
- APIs
- background workers that can be replicated

Key behaviors:

- rolling updates
- rollbacks
- scaling

Example Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myrepo/api:1.0.0
        ports:
        - containerPort: 8080
```

### Job and CronJob (run-to-completion)

- **Job**: run a task until it completes.
- **CronJob**: run Jobs on a schedule.

Use for:

- database migrations (carefully)
- nightly batch tasks
- report generation

### StatefulSet (stateful apps)

StatefulSet is for workloads where identity matters:

- stable pod names (e.g., `db-0`, `db-1`)
- stable storage (PVC per replica)
- ordered start/stop (optional)

Use for:

- databases (when you truly must)
- Kafka/ZooKeeper style systems

Real-life tip: running stateful systems in Kubernetes is doable, but operationally heavier. Managed DB services often reduce your risk.

### DaemonSet (one pod per node)

DaemonSet ensures a copy of a pod runs on each node.

Use for:

- log collectors
- node monitoring agents
- CNI components

---

## Configuration: Labels, Selectors, Annotations

### Labels

Labels are key/value pairs for grouping and selecting.

Examples:

- `app: payments`
- `env: prod`
- `tier: backend`

They drive:

- Services selecting pods
- Deployments selecting pods
- NetworkPolicies selecting pods
- Prometheus scraping rules

### Selectors

Selectors are how one object targets others.

Common patterns:

- Service selects pods by label `app=api`.
- Deployment uses a selector that must match the pod template labels.

### Annotations

Annotations are for “extra metadata” that isn’t used for selection.

Examples:

- build info
- owner/team
- load balancer config hints

---

## ConfigMaps, Secrets, and environment injection

### ConfigMap

A ConfigMap holds non-sensitive configuration.

You can mount it:

- as env vars
- as files

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  LOG_LEVEL: "info"
  FEATURE_X: "enabled"
```

### Secret

A Secret holds sensitive data.

Important reality check:

- By default, Kubernetes Secrets are base64-encoded, not encrypted.
- In many clusters you should enable encryption-at-rest for etcd and use external secret management.

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=
  password: c2VjdXJl
```

### Better secret handling (enterprise)

Common approaches:

- Sealed Secrets
- External Secrets Operator
- CSI Secret Store (cloud key vault integration)

---

## Services, Endpoints, Ingress, and Gateway API

### Why Services exist

Pods come and go. IPs change.

A **Service** gives a stable virtual IP (ClusterIP) and DNS name.

### Service types

1) **ClusterIP** (default)

- Internal-only
- Most common

2) **NodePort**

- Exposes a port on each node
- Usually not used directly in enterprise setups (often behind LB)

3) **LoadBalancer**

- In cloud: provisions an external load balancer

4) **ExternalName**

- DNS alias to an external name

Example Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: ClusterIP
```

### Ingress (HTTP/HTTPS routing)

Ingress is a Kubernetes object that describes HTTP routing rules. It needs an **Ingress Controller** (like NGINX Ingress, HAProxy Ingress, etc.) to actually work.

Example Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80
```

### Gateway API (modern replacement direction)

Gateway API is newer and more expressive than Ingress.

- Better separation between platform team (Gateways) and app teams (Routes)
- Supports more traffic patterns cleanly

---

## Networking fundamentals (CNI, DNS, kube-proxy)

Networking is one of the most important parts of Kubernetes.

### Pod-to-Pod model

Kubernetes assumes:

- Every pod gets its own IP
- Pods can reach each other without NAT

How that happens depends on the **CNI plugin**.

### CNI (Container Network Interface)

The CNI plugin is responsible for:

- giving pods IP addresses
- setting up routes / overlay / eBPF
- enforcing network policy (sometimes)

Popular CNIs:

- Calico (common, strong policy)
- Cilium (eBPF based, powerful)
- Flannel (simple)

### kube-dns / CoreDNS

DNS inside the cluster is usually provided by CoreDNS.

Typical service DNS names:

- `api.default.svc.cluster.local`

### kube-proxy

kube-proxy implements Services:

- using iptables/ipvs, or
- in some setups, replaced/augmented by eBPF solutions.

---

## Storage: Volumes, PV/PVC, StorageClasses, CSI

### The problem storage solves

Containers are ephemeral. If a pod restarts, its container filesystem resets.

So we attach **Volumes** for data that must survive.

### PV and PVC (the contract)

- **PV (PersistentVolume)**: a piece of storage in the cluster.
- **PVC (PersistentVolumeClaim)**: a request for storage.

This separation allows:

- app teams to request storage
- platform teams to control how storage is provisioned

### StorageClass

StorageClass defines *how* dynamic provisioning works.

- e.g., fast SSD vs cheap HDD
- encryption options
- replication

### CSI (Container Storage Interface)

CSI drivers let Kubernetes talk to storage systems.

Cloud examples:

- Azure Disk / Azure Files CSI
- AWS EBS / EFS CSI
- GCE PD CSI

---

## Scheduling: nodes, taints/tolerations, affinity, topology

Scheduling is “where pods land.”

### Node labels

Nodes often have labels like:

- `nodepool=general`
- `zone=us-east-1a`

### Taints and tolerations

**Taints** keep pods away from nodes unless the pod tolerates it.

Typical use cases:

- dedicated nodes for databases
- GPU nodes
- “spot” nodes

### Affinity/anti-affinity

Affinity rules guide scheduling:

- “Put this pod near that one” (affinity)
- “Keep these pods apart” (anti-affinity)

Anti-affinity is key for high availability: don’t schedule all replicas on one node.

### Topology spread constraints

A clean modern way to spread pods across zones/nodes.

---

## Resource management: requests/limits, QoS, quotas

### Requests and limits

For each container you can set:

- **requests**: what you need (used for scheduling)
- **limits**: hard cap (what you cannot exceed)

CPU is compressible (throttled), memory is not (OOMKill risk).

### QoS classes

Based on requests/limits, Kubernetes assigns QoS:

- Guaranteed
- Burstable
- BestEffort

In resource pressure, BestEffort goes first.

### ResourceQuota and LimitRange

At the namespace level you can enforce:

- maximum total CPU/memory
- max pods
- default requests/limits

Enterprise tip: quotas prevent “noisy neighbor” problems and make cost attribution easier.

---

## Autoscaling: HPA, VPA, Cluster Autoscaler, KEDA

### HPA (Horizontal Pod Autoscaler)

Scales replica count based on metrics.

Common signals:

- CPU %
- memory (less ideal)
- custom metrics (requests per second)

Requires metrics pipeline (metrics-server + optionally Prometheus adapter).

### VPA (Vertical Pod Autoscaler)

Adjusts requests/limits over time.

Good for:

- batch jobs
- workloads with stable patterns

Be careful with stateful/latency-sensitive services; resizing can cause restarts.

### Cluster Autoscaler

Adds/removes nodes based on pending pods.

Managed Kubernetes often supports it.

### KEDA

Event-driven autoscaling (e.g., scale based on queue length).

---

## Health, rollouts, and disruption

### Probes (readiness, liveness, startup)

- **Readiness**: can the pod receive traffic?
- **Liveness**: is the pod stuck and should be restarted?
- **Startup**: give slow apps time before liveness starts killing them.

### Deployment rollout mechanics

A Deployment uses a rolling update strategy:

- create new ReplicaSet
- gradually shift pods
- keep minimum available

Useful commands:

```bash
kubectl rollout status deployment/api
kubectl rollout history deployment/api
kubectl rollout undo deployment/api
```

### PodDisruptionBudget (PDB)

PDB protects availability during voluntary disruptions:

- node drains
- upgrades

It doesn’t prevent failures, but it prevents you from *causing* too much downtime at once.

---

## Security: RBAC, service accounts, Pod Security, network policies

Security in Kubernetes is layered. You don’t rely on one control.

### Authentication vs authorization

- **Authentication**: who are you?
- **Authorization**: what are you allowed to do?

### RBAC (Roles and Bindings)

RBAC defines permissions.

- `Role` / `RoleBinding` are namespace-scoped.
- `ClusterRole` / `ClusterRoleBinding` can be cluster-wide.

Basic advice:

- start with least privilege
- separate human access from workload access
- avoid giving `cluster-admin` broadly

### ServiceAccounts

Pods use ServiceAccounts to call the API.

Best practice:

- create a dedicated ServiceAccount per app
- bind only needed permissions

### Admission control

Before objects are stored, admission controllers can enforce policy.

Common enterprise solutions:

- OPA Gatekeeper
- Kyverno

### Pod Security Standards

Pod Security replaced PodSecurityPolicy.

Profiles:

- privileged
- baseline
- restricted

Goal: prevent dangerous pod settings (privileged containers, hostPath mounts, etc.).

### NetworkPolicies

Without NetworkPolicies, many CNIs allow all pod-to-pod traffic by default.

NetworkPolicies allow you to say:

- “Only these pods can talk to that database.”

Enterprise approach:

- default-deny in sensitive namespaces
- explicitly allow required flows

### Image security

Key points:

- scan images in CI
- pin images by digest for high security
- use minimal base images
- sign images (Sigstore/cosign)

---

## Observability: logs, metrics, traces, events

### Events

Start with events when something is wrong:

```bash
kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp
```

### Logs

```bash
kubectl logs -n <ns> pod/<pod> -c <container>
kubectl logs -n <ns> deployment/<name> --tail=200
```

In production you usually centralize logs:

- Fluent Bit / Fluentd / Vector
- send to Elasticsearch / Splunk / Loki / cloud logging

### Metrics

You typically have:

- metrics-server (basic)
- Prometheus (rich)
- Grafana dashboards

### Tracing

Distributed tracing helps find latency bottlenecks:

- OpenTelemetry instrumentation
- Jaeger / Tempo / vendor APM

---

## Troubleshooting playbook

This section is intentionally practical.

### Step 1: Is it Kubernetes or the app?

- If the pod is running and ready but app is returning errors → likely app-level.
- If pods are crashlooping, pending, not ready → likely Kubernetes/platform.

### Step 2: Check object status and events

```bash
kubectl get pods -n <ns> -o wide
kubectl describe pod -n <ns> <pod>
kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp | tail
```

### Common pod states

- `Pending` → scheduling issues (resources, taints, PVC not bound)
- `CrashLoopBackOff` → app exits or health checks failing
- `ImagePullBackOff` → registry auth, image name, network
- `ErrImagePull` → image doesn’t exist, permissions

### Diagnose Pending

Check:

- node capacity (`kubectl describe node <node>`)
- taints (`kubectl describe node`)
- PVC binding (`kubectl get pvc -n <ns>`)
- affinity/anti-affinity rules

### Diagnose CrashLoopBackOff

- check logs: `kubectl logs ... --previous`
- confirm env vars/config mounted
- check probes are correct

### DNS troubleshooting

Symptoms:

- service name doesn’t resolve
- random timeouts

Quick checks:

- `kubectl get svc -n kube-system` for CoreDNS
- run a debug pod and `nslookup` service names

### Networking troubleshooting

- Check NetworkPolicies (are you blocking traffic?)
- Check Service selectors match pod labels
- Check endpoints: `kubectl get endpoints -n <ns> <svc>`

### Node issues

- node NotReady
- disk pressure
- memory pressure

Check:

```bash
kubectl describe node <node>
```

Then look at:

- kubelet logs (on the node)
- runtime logs
- CNI health

---

## Operating clusters: upgrades, backups, DR, lifecycle

This is where you move from “developer” to “platform operator.”

### Upgrades (control plane and nodes)

Principles:

- upgrade regularly (avoid big jumps)
- read release notes
- stage in non-prod first

Typical safe flow:

1. Upgrade control plane (managed service often does this).
2. Upgrade node pools one by one.
3. Drain nodes safely (respect PDBs).
4. Watch SLOs and key workloads.

### Backups

There are two kinds:

1) **Cluster state** (etcd / API objects)
2) **Persistent data** (PV snapshots)

Popular tools:

- Velero (cluster resources + PV snapshots)

### Disaster recovery

Questions you must answer:

- RPO: how much data loss is acceptable?
- RTO: how long can recovery take?

Approaches:

- multi-zone cluster + resilient apps
- multi-region active/standby
- restore-from-backup to a new cluster

### Add-ons lifecycle

Treat cluster add-ons like products:

- version them
- document ownership
- upgrade them regularly

---

## Multi-tenancy and enterprise patterns

### Teams, namespaces, and guardrails

Enterprise clusters often host many teams.

Core controls:

- namespaces per team/app
- RBAC: restrict who can do what
- quotas: prevent resource overuse
- policies: enforce security defaults
- network policies: control east-west traffic

### Separate environments

Common models:

- separate clusters per environment (dev/qa/prod)
- separate node pools + namespaces within one cluster

Safer model for enterprise: separate clusters for prod vs non-prod.

### Platform responsibilities

A good platform team provides:

- standard ingress + TLS
- standard observability
- standard policy enforcement
- curated base images and runtime standards

---

## GitOps and continuous delivery

GitOps is “Git is the source of truth for desired state.”

### Why GitOps is popular

- audit trail
- easy rollback
- consistent environments
- less “kubectl drift”

### Typical GitOps loop

1. Developer opens PR to change YAML/Helm/Kustomize.
2. CI validates manifests.
3. GitOps controller (Argo CD/Flux) syncs changes into cluster.
4. Drift detection alerts when cluster differs from Git.

### Helm vs Kustomize (simple view)

- Helm: templating + packaging. Great for complex parameterized apps.
- Kustomize: patching overlays. Great for environment differences.

Many enterprises use both.

---

## Service meshes and advanced traffic management

A service mesh adds a proxy layer (sidecars or node-level) to manage service-to-service traffic.

### Why use a mesh

- mTLS between services
- traffic splitting (canary)
- retries/timeouts/circuit breaking
- better observability

### Costs

- extra complexity
- more resource usage
- more moving parts

Only adopt a mesh when you need the capabilities.

---

## Cost, performance, and capacity planning

### The basics of capacity

To size clusters, you need:

- CPU and memory requests
- expected peak traffic
- headroom for failures

### Over-requesting vs under-requesting

- Over-requesting wastes money but increases stability.
- Under-requesting saves money but increases risk of OOMs and eviction.

Good practice:

- start conservative
- observe real usage
- tune gradually

### Node pools strategy

- general pool for most workloads
- dedicated pools for special needs (GPU, high-memory)
- spot/preemptible pools for safe-to-interrupt workloads

---

## A practical “day-2” checklist

Use this as a quick audit list.

### Cluster basics

- Control plane HA (where applicable)
- Node pool strategy documented
- Cluster upgrades scheduled and tested

### Security

- RBAC least privilege
- Pod Security enforcement baseline/restricted
- NetworkPolicies for sensitive namespaces
- Secrets management approach documented

### Reliability

- PDBs for critical workloads
- readiness/liveness probes set
- proper requests/limits
- backups tested (restore drill)

### Observability

- centralized logs
- metrics + dashboards
- alerting on SLOs, not only CPU

### Delivery

- GitOps in place
- CI validation for manifests
- rollbacks practiced

---

## Glossary (quick definitions)

- **Cluster**: a set of machines (nodes) managed by Kubernetes.
- **Node**: a machine (VM or physical) that runs pods.
- **Control plane**: API server + scheduler + controllers (cluster brain).
- **Pod**: smallest deployable unit; one or more containers.
- **Deployment**: controller for stateless apps; manages ReplicaSets.
- **Service**: stable virtual IP/DNS name to reach pods.
- **Ingress**: HTTP routing rules (needs an ingress controller).
- **CNI**: networking plugin that gives pods IPs.
- **PV/PVC**: persistent storage abstraction.
- **RBAC**: permission model for Kubernetes API.
- **Namespace**: logical partition for resources and policy.

---

## Where to go next (recommended practice path)

If you want to truly master Kubernetes, reading isn’t enough—you need repetition and real problems.

A solid practice sequence:

1. Deploy a simple web app (Deployment + Service).
2. Add ConfigMap + Secret.
3. Add Ingress or Gateway and TLS.
4. Add HPA.
5. Add NetworkPolicy.
6. Run a controlled failure drill (kill pods, drain nodes).
7. Practice a version upgrade in a non-prod cluster.
8. Build a GitOps pipeline for a few namespaces.

---

### Notes on scope

Kubernetes is huge. This guide is “complete enough to operate real clusters,” but the ecosystem is bigger than any single document.

If you tell me your target environment (AKS/EKS/GKE/on-prem), your CNI, and what kind of workloads you run, I can tailor a second document with concrete, environment-specific runbooks (upgrades, ingress patterns, secret management, and standard policies).

---

## Deep dive: Pod and container lifecycle

If you understand this section well, you will avoid many of the “mystery” outages people hit in Kubernetes.

### Pod phases (high level)

- **Pending**: Kubernetes accepted the pod, but it isn’t running yet.
  - Common reasons: no nodes fit, image pull waiting, PVC not bound.
- **Running**: at least one container is running.
- **Succeeded**: all containers exited successfully (usually Jobs).
- **Failed**: containers exited with a non-zero status.
- **Unknown**: Kubernetes can’t get state (network issues to node).

Important: “Running” does not mean “serving traffic.” That’s what **readiness** is for.

### Container states

Each container can be:

- **Waiting** (image pull, crash backoff, creating)
- **Running**
- **Terminated** (with exit code and reason)

### Readiness vs liveness (simple rule)

- **Readiness** answers: “Should this pod receive traffic?”
- **Liveness** answers: “Should Kubernetes restart this container?”

If you only remember one thing: **be conservative with liveness**. Bad liveness probes cause self-inflicted outages.

### Startup probes (when your app needs time)

If the app takes a while to start, use **startupProbe** so liveness does not kill it during warm-up.

### Init containers

Init containers run before app containers. They are great for:

- waiting for dependencies (careful—don’t hide broken design)
- downloading configs
- preparing permissions or files

Example:

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ["sh", "-c", "until nc -z db 5432; do echo waiting; sleep 2; done"]
  containers:
  - name: api
    image: myrepo/api:1.0.0
```

### Sidecars (the common “helper container” pattern)

Sidecars share the pod network and volumes.

Common sidecars:

- service mesh proxy
- log shipper
- config reloader

Sidecar gotcha: sidecars can keep a pod running even when the main container exited. For Jobs, this matters.

### Graceful shutdown (termination flow)

When Kubernetes terminates a pod:

1. It marks the pod as **not ready** (it should stop receiving traffic).
2. It sends **SIGTERM** to containers.
3. It waits up to `terminationGracePeriodSeconds`.
4. If still running, it sends **SIGKILL**.

To shut down cleanly:

- your app must handle SIGTERM
- readiness should fail quickly during shutdown

### Draining nodes (why pods get deleted)

When you do:

```bash
kubectl drain <node> --ignore-daemonsets
```

Kubernetes evicts pods (respecting PDBs) and reschedules them elsewhere. This is the normal path during upgrades.

### Debugging running pods without “ssh into container” habits

Kubernetes gives you safer tools than baking SSH into images.

- Exec into a container:

```bash
kubectl exec -n <ns> -it pod/<pod> -- sh
```

- Use an ephemeral debug container (when enabled):

```bash
kubectl debug -n <ns> -it pod/<pod> --image=busybox:1.36
```

---

## Deep dive: The Kubernetes API request path

Almost every “why is this forbidden / denied / stuck” question becomes easier once you know the request flow.

### The path for an API request

When you run `kubectl apply`, the request typically goes through:

1. **Authentication**: who are you?
   - client certs, OIDC, tokens, etc.
2. **Authorization**: are you allowed?
   - RBAC rules are evaluated.
3. **Admission**: should this object be allowed and/or modified?
   - validation admission (reject)
   - mutating admission (inject sidecars, add defaults)
4. **Validation + schema**: does it match the API type?
5. **Persist to etcd**: store desired state.
6. **Controllers react**: reconciliation loops act on it.

### Admission controllers in real life

Admission is where enterprises enforce rules like:

- “Every pod must have resource requests.”
- “No privileged containers.”
- “Only signed images are allowed.”

Two big categories:

- **Mutating**: changes objects (adds labels, injects sidecars)
- **Validating**: allows or rejects

### Why objects sometimes look different than your YAML

Kubernetes adds defaults, and mutating admission can inject fields.

That’s why you should inspect the live object:

```bash
kubectl get deploy api -n <ns> -o yaml
```

---

## Deep dive: Controllers, reconciliation, and watches

The “Kubernetes magic” is mostly a set of control loops.

### Reconciliation loop (simple explanation)

Controllers run a loop that compares:

- desired state (spec)
- current state (status + what exists)

Then they take action to reduce the difference.

### Watches (how controllers react quickly)

Controllers use **watches** on the API server so they don’t need to poll constantly.

If a watch stream breaks (network issues, overload), controllers reconnect.

### Why deleting pods is normal

People new to Kubernetes often panic when pods are deleted.

In many cases, it is normal:

- Deployment replacing old ReplicaSet
- node drain during upgrade
- autoscaler scaling down nodes

The key is whether the controller maintains desired replicas.

---

## Deep dive: CRDs and Operators (extending Kubernetes)

Kubernetes is extensible. This is how “databases on Kubernetes”, “certificate automation”, and many platform features are implemented.

### CRD (CustomResourceDefinition)

A CRD adds a new API type to your cluster.

Example idea (not a real built-in kind):

- `kind: Database`
- `kind: KafkaCluster`

Once installed, you can create those resources like native ones.

### Operator (controller for a custom resource)

An **operator** is just a controller that understands a domain.

Example: `Certificate` resources managed by cert-manager.

Typical operator responsibilities:

- create/update underlying Deployments/StatefulSets/Secrets
- handle upgrades
- handle recovery
- expose status

### When operators are a win

- you want a self-managing platform component
- you want a consistent API for app teams

### When operators become risky

- unclear ownership (who maintains upgrades?)
- operator bugs can affect many namespaces
- CRD versioning and breaking changes

Operational advice:

- treat operators like critical dependencies
- pin versions
- test upgrades in non-prod

---

## Deep dive: Services and load balancing internals

### Endpoints and EndpointSlices

When a Service selects pods, Kubernetes creates endpoints that represent “where traffic should go.”

Modern clusters use **EndpointSlices** to scale better than classic Endpoints.

If a Service exists but traffic goes nowhere, check:

```bash
kubectl get endpointslices -n <ns> -l kubernetes.io/service-name=<svc>
```

### How kube-proxy load balancing works (mental model)

At a high level:

- client connects to Service virtual IP
- node rules translate that to one backend pod IP

Two common kube-proxy implementations:

- **iptables**: widely used, simple, but large rule sets can be heavy at scale
- **ipvs**: more efficient in some scenarios

Some CNIs (like eBPF-based ones) can implement Service handling differently.

### sessionAffinity

If you need “sticky sessions” (often better to avoid, but sometimes required):

```yaml
spec:
  sessionAffinity: ClientIP
```

Know the tradeoff: stickiness can create uneven load.

### External traffic and source IP

With LoadBalancers/NodePorts, source IP behavior depends on:

- cloud load balancer type
- `externalTrafficPolicy` (`Cluster` vs `Local`)

If you need real client IPs, you often need `externalTrafficPolicy: Local` and correct health checks.

---

## Deep dive: NetworkPolicy patterns

NetworkPolicy is one of the biggest “enterprise maturity” signals.

### First, know what enforces it

NetworkPolicy objects are just rules. Whether they work depends on your CNI.

If your CNI doesn’t enforce NetworkPolicy, the objects won’t have effect.

### A practical starting model

1) Default allow (early stage): no policies
2) Default deny in sensitive namespaces
3) Gradually move to default deny everywhere

### Default deny (ingress) example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

This blocks all inbound traffic to pods in `payments` unless you add allow policies.

### Allow traffic from one app to another

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
```

### Don’t forget DNS

When you start locking down egress, pods still need DNS (usually to CoreDNS in `kube-system`). Many teams break clusters by blocking DNS.

---

## Deep dive: Storage details you’ll need in production

### Access modes

- **ReadWriteOnce (RWO)**: mounted read-write by a single node (common for block storage)
- **ReadOnlyMany (ROX)**: many nodes, read-only
- **ReadWriteMany (RWX)**: many nodes can mount read-write (common for shared filesystems)

Which modes are supported depends on the storage backend.

### Reclaim policy

Defines what happens to a PV when the PVC is deleted:

- **Delete**: storage is deleted (common for dynamic provisioning)
- **Retain**: storage is kept for manual recovery

In production, be very clear which storage classes delete data.

### VolumeBindingMode

Some storage classes delay binding until scheduling so that storage topology (zone) matches the chosen node.

This matters a lot in multi-zone clusters.

### Volume expansion and snapshots

Many CSI drivers support:

- expanding a PVC size
- volume snapshots

Snapshots are not backups by themselves, but they’re a common building block for backups.

### StatefulSet storage pattern

StatefulSets often create a PVC per replica.

If you delete a StatefulSet, those PVCs may remain (depending on settings). That is often what you want.

---

## Deep dive: Runtime and node security

Kubernetes security is not only YAML. Nodes and runtimes matter.

### SecurityContext (pod/container level)

Common settings you should learn:

- run as non-root
- drop Linux capabilities
- read-only root filesystem

Example:

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
containers:
- name: api
  image: myrepo/api:1.0.0
  securityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities:
      drop: ["ALL"]
```

### seccomp and AppArmor

These are Linux features that reduce what processes can do.

If you’re running regulated workloads, this becomes important.

### Privileged pods: treat as “break glass”

If a pod is privileged or mounts host paths, it can often escape normal containment.

Only allow this when there is a strong, reviewed reason.

### Node hardening basics

At a minimum:

- minimal OS images
- regular patching
- restrict SSH access
- separate node pools for sensitive workloads

### RuntimeClass and sandboxing

Some environments use extra isolation runtimes like gVisor or Kata Containers.

This can reduce risk for untrusted workloads, at a performance cost.

---

## Enterprise runbooks (what operators actually do)

This section is written like an operator’s notebook.

### Runbook: A deployment is failing to roll out

Symptoms:

- rollout stuck
- pods not becoming ready

Checklist:

1. `kubectl rollout status deployment/<name> -n <ns>`
2. `kubectl get rs -n <ns>` (is a new ReplicaSet created?)
3. `kubectl get pods -n <ns> -l app=<label> -o wide`
4. `kubectl describe pod ...` (events)
5. Check readiness probe and dependency connectivity
6. If urgent: rollback with `kubectl rollout undo ...`

### Runbook: Node is NotReady

Checklist:

1. `kubectl describe node <node>` (look for conditions)
2. Is it network, disk, memory pressure?
3. Check kubelet and runtime logs on the node
4. Check CNI pods and kube-proxy on that node
5. If node is unhealthy, cordon + drain, then replace

### Runbook: Pods stuck Pending

Checklist:

1. `kubectl describe pod <pod> -n <ns>`
2. Look for “0/… nodes are available” reasons
3. Verify requests/limits vs available nodes
4. Check taints and required tolerations
5. Check PVC binding and storage class

### Runbook: Production upgrade (safe, boring upgrades)

Core principles:

- upgrade frequently enough that each upgrade is small
- do it in non-prod first
- watch real health signals (latency, error rate)

Typical flow:

1. Confirm PDBs exist for critical workloads.
2. Confirm alerting is quiet and you have rollback plan.
3. Upgrade control plane.
4. Upgrade one node pool at a time.
5. Drain nodes gradually.
6. Validate core add-ons (DNS, ingress, CNI).
7. Validate critical apps.

### Runbook: etcd (why it matters even on managed Kubernetes)

If you self-manage control planes, etcd care is non-negotiable.

Operator basics:

- monitor etcd latency
- regular backups
- avoid overloading API server with huge watch loads

Even on managed services, you should understand that etcd performance affects API responsiveness.

---

## Deep dive: Kubernetes object model (metadata, deletion, finalizers)

This section explains behaviors that confuse even experienced teams: “Why is this object stuck terminating?” “Why did Kubernetes delete that?” “Why is my apply fighting someone else’s apply?”

### The object fields you should recognize

Almost every Kubernetes object has:

- `metadata.name` and (usually) `metadata.namespace`
- `metadata.labels` and `metadata.annotations`
- `spec` (desired state)
- `status` (observed state)

You should also know these important metadata fields:

- **ownerReferences**: tells Kubernetes that one object owns another.
- **finalizers**: “do not delete until cleanup is done.”
- **deletionTimestamp**: set when deletion starts.
- **generation / resourceVersion**: used for change tracking and concurrency.

### Owner references and garbage collection

If object A owns object B, Kubernetes can delete B when A is deleted.

Example: a Deployment owns ReplicaSets; ReplicaSets own Pods.

So when you delete a Deployment, the system cleans up related ReplicaSets and Pods.

### Deletion is not always immediate

When you delete an object, Kubernetes typically:

1. sets `deletionTimestamp`
2. waits for finalizers to finish
3. removes the object

If a finalizer never completes, the object can stay “Terminating” forever.

### Finalizers (why “Terminating” gets stuck)

Finalizers are common with:

- cloud load balancers
- storage volumes
- custom resources managed by operators

If you see a resource stuck, inspect finalizers:

```bash
kubectl get <kind> <name> -n <ns> -o jsonpath='{.metadata.finalizers}'
```

In an emergency, you *can* remove a finalizer manually, but treat it like breaking glass: you may leak cloud resources or skip cleanup.

### Apply conflicts and “field ownership” (server-side apply)

In modern clusters, server-side apply tracks who owns which fields.

If two tools both manage the same field (Helm vs kubectl apply vs operator), you can see conflicts.

Enterprise advice:

- decide which tool owns which resources
- avoid mixing Helm-managed resources and raw apply for the same fields

---

## Deep dive: RBAC in practice (patterns that scale)

RBAC is where “secure by default” lives.

### RBAC building blocks

- **Role**: permissions within one namespace
- **ClusterRole**: permissions cluster-wide (or reusable)
- **RoleBinding**: attaches Role/ClusterRole to a subject in a namespace
- **ClusterRoleBinding**: attaches ClusterRole to a subject cluster-wide

Subjects can be:

- users
- groups
- service accounts

### A safe default model for enterprises

1) Platform admins (small group)

- cluster-admin-like powers
- tightly controlled

2) Namespace admins (per team)

- can manage most resources in their namespaces
- cannot change cluster-wide add-ons

3) App runtime identities (service accounts)

- minimal API permissions
- often none at all

### Example: read-only access to a namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: view-workloads
  namespace: payments
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "services", "deployments", "replicasets", "jobs", "cronjobs", "configmaps"]
  verbs: ["get", "list", "watch"]
```

Bind it:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-workloads-binding
  namespace: payments
subjects:
- kind: Group
  name: payments-readonly
roleRef:
  kind: Role
  name: view-workloads
  apiGroup: rbac.authorization.k8s.io
```

### Common RBAC mistakes

- giving developers `cluster-admin` “temporarily” (it never stays temporary)
- giving pods permissions they don’t need
- forgetting that `list/watch` can reveal sensitive info (names, labels)

### Service account tokens (short-lived is better)

Modern Kubernetes uses projected service account tokens that can be short-lived.

Try to avoid long-lived tokens and manual secret-based token handling.

---

## Deep dive: TLS, certificates, and cert-manager

If you run HTTPS (you do), you need a plan for certificates.

### TLS endpoints in Kubernetes

Where TLS can terminate:

- at Ingress controller / Gateway
- at a service mesh proxy
- in the app itself (less common)

Most enterprises terminate TLS at the edge (ingress/gateway) and use mTLS internally if needed.

### cert-manager (common automation)

cert-manager is an operator that manages certificates as Kubernetes resources.

You typically define:

- an **Issuer** / **ClusterIssuer** (where certs come from)
- a **Certificate** resource (what you want)

Then cert-manager keeps the secret updated and renews automatically.

Operational advice:

- monitor certificate expiration
- test renewal in non-prod
- ensure time synchronization on nodes

---

## Deep dive: Advanced scheduling (priority, preemption, descheduler)

Basic scheduling is “fit the pod on a node.” Advanced scheduling is “fit the right pods on the right nodes, even under pressure.”

### PriorityClass

PriorityClass lets you say which workloads are more important.

- critical platform pods should be higher priority
- batch jobs should be lower priority

### Preemption

If the cluster is full and a high-priority pod cannot be scheduled, Kubernetes can evict lower-priority pods.

This is powerful but dangerous if misused.

### Descheduler

Over time, clusters get “unbalanced.” The descheduler can evict pods to improve distribution.

Use cases:

- after scaling node pools
- after changing affinity rules

Treat it carefully; it causes voluntary disruptions.

---

## Deep dive: Progressive delivery (canary, blue-green)

Rolling updates are good, but sometimes you need finer control.

### Canary

Canary means:

- send a small percentage of traffic to the new version
- observe metrics
- gradually increase

This requires traffic control (ingress/gateway/mesh) and good observability.

### Blue-green

Blue-green means:

- run old (blue) and new (green) side by side
- switch traffic all at once when ready

It gives fast rollback but uses more capacity.

### Tooling

Many teams use:

- Argo Rollouts (progressive delivery controller)
- service mesh traffic splitting
- Gateway API advanced routing

---

## Deep dive: Multi-cluster and fleet basics

Large enterprises rarely run one cluster.

### Why multi-cluster happens

- isolation (prod vs non-prod)
- blast radius control
- regional latency
- regulatory boundaries
- scaling organization and ownership

### Fleet management goals

- consistent add-ons across clusters
- consistent policies
- consistent upgrades

Common patterns:

- GitOps per cluster (same repo structure)
- centralized policy + audit
- standard “platform baseline” chart/manifest

### Multi-cluster networking

Options include:

- public routing (least complex, sometimes acceptable)
- private peering (VNet/VPC peering)
- service mesh multi-cluster (more complex)

Choose the simplest option that meets security and latency requirements.

---

## Appendix: kubectl cheat sheet

### Get basics

```bash
kubectl get ns
kubectl get all -n <ns>
kubectl get pods -n <ns> -o wide
```

### Describe and events

```bash
kubectl describe pod -n <ns> <pod>
kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp
```

### Logs

```bash
kubectl logs -n <ns> pod/<pod> --tail=200
kubectl logs -n <ns> pod/<pod> -c <container> --previous
```

### Exec and debug

```bash
kubectl exec -n <ns> -it pod/<pod> -- sh
kubectl debug -n <ns> -it pod/<pod> --image=busybox:1.36
```

### Rollouts

```bash
kubectl rollout status -n <ns> deployment/<name>
kubectl rollout undo -n <ns> deployment/<name>
```

### Apply and diff

```bash
kubectl diff -f ./manifests/
kubectl apply -f ./manifests/
```


