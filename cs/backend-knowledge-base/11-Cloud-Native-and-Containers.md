# Cloud Native and Containers

Language: English | [中文](../后端知识库/11-云原生与容器.md)

---

## Table of Contents

1. [Docker Containers](#1-docker-containers)
2. [Kubernetes Core Principles](#2-kubernetes-core-principles)
3. [Service Mesh](#3-service-mesh)
4. [Cloud Native Best Practices](#4-cloud-native-best-practices)
5. [GitOps and Continuous Delivery](#5-gitops-and-continuous-delivery)
6. [Kubernetes Troubleshooting](#6-kubernetes-troubleshooting)
7. [Interview Self-Check](#7-interview-self-check)

---

## 1. Docker Containers

### 1.1 What Is a Container?

A container packages an application with its dependencies and runs it with OS-level isolation.

Containers are not lightweight virtual machines. They share the host kernel and rely on namespaces, cgroups, and layered filesystems.

### 1.2 Docker Internals ⭐⭐⭐

Core mechanisms:

- Namespace: isolates PID, network, mount, IPC, UTS, and user views.
- Cgroup: limits and accounts CPU, memory, IO, and pids.
- Union filesystem: implements image layers.
- Capabilities, seccomp, and AppArmor: reduce kernel attack surface.

Image layers make builds reusable. Containers add a writable layer on top of read-only image layers.

### 1.3 Docker Networking

Common modes:

- bridge.
- host.
- none.
- overlay.

Bridge networking uses veth pairs, a Linux bridge, iptables, and NAT.

### 1.4 Docker Best Practices

- Use small base images.
- Use multi-stage builds.
- Do not run as root when possible.
- Pin image versions.
- Keep secrets out of images.
- Add health checks.
- Make containers stateless.

---

## 2. Kubernetes Core Principles

### 2.1 Architecture

Control plane:

- API Server.
- Scheduler.
- Controller Manager.
- etcd.

Worker node:

- kubelet.
- kube-proxy.
- container runtime.
- pods.

Kubernetes is declarative: users submit desired state, and controllers reconcile actual state toward it.

### 2.2 Pod Scheduling ⭐⭐⭐

Scheduler considers:

- resource requests.
- node selectors and affinity.
- taints and tolerations.
- topology spread.
- volume constraints.
- pod affinity/anti-affinity.

Requests are used for scheduling. Limits are enforced at runtime.

### 2.3 Service Networking ⭐⭐⭐

Service provides a stable virtual IP and load balancing for pods.

Types:

- ClusterIP.
- NodePort.
- LoadBalancer.
- ExternalName.

`kube-proxy` on every node watches Services and Endpoints/EndpointSlices, then programs the local datapath. The ClusterIP is usually not a process listening on a real NIC; packets are rewritten in the kernel and forwarded to a ready Pod.

**iptables mode**

- Uses Linux **Netfilter** rules configured via `iptables`.
- Match ClusterIP:Port in long NAT chains (`KUBE-SERVICES` → `KUBE-SVC-*` → `KUBE-SEP-*`), then **DNAT** to a PodIP:Port.
- Balancing is typically random probability via the `statistic` match.
- Cost grows roughly with rule count: many Services make chain walks and rule sync expensive.

**IPVS mode**

- Uses Linux **IPVS** (IP Virtual Server), the kernel L4 load balancer from LVS, managed with `ipvsadm`.
- Each Service becomes a Virtual Server (VIP:Port); each Endpoint is a Real Server.
- Lookup is hash-based (~O(1)); schedulers include `rr`, `lc`, `sh`, etc.
- Still may use some iptables for masquerade / helpers; needs `ip_vs*` kernel modules and often a dummy interface such as `kube-ipvs0`.

| | iptables mode | IPVS mode |
|--|---------------|-----------|
| Kernel mechanism | Netfilter chains + DNAT | IPVS VS/RS scheduling |
| Scale behavior | ~O(n) with rule count | ~O(1) VIP lookup |
| Algorithms | mainly random | rr / lc / sh / ... |
| Best fit | small/medium clusters | large Service counts |

Newer clusters may also use `nftables` mode, or eBPF CNIs (e.g. Cilium) that replace kube-proxy entirely. See the Chinese note for a fuller walkthrough and commands.

### 2.4 Pod Network and CNI

CNI plugins provide pod networking.

Examples:

- Calico.
- Flannel.
- Cilium.

Kubernetes expects pods to communicate with each other without NAT inside the cluster network model.

### 2.5 Deployment Rolling Update

Deployment manages ReplicaSets and supports rolling updates.

Important fields:

- `maxSurge`.
- `maxUnavailable`.
- readiness probe.
- liveness probe.
- startup probe.

Readiness decides traffic eligibility. Liveness decides restart.

---

## 3. Service Mesh

Service mesh moves traffic governance into a data plane proxy.

Capabilities:

- mTLS.
- retries and timeouts.
- traffic splitting.
- observability.
- policy enforcement.

Istio typically uses Envoy as data plane and control-plane components to distribute configuration.

Trade-offs:

- operational complexity.
- extra latency.
- more debugging layers.

---

## 4. Cloud Native Best Practices

### 4.1 Twelve-Factor App

Key ideas:

- config through environment.
- stateless processes.
- logs as event streams.
- disposable processes.
- strict separation of build, release, and run.

### 4.2 Container Security

Checklist:

- run as non-root.
- drop unnecessary capabilities.
- use read-only root filesystem where possible.
- scan images.
- pin dependencies.
- restrict egress.
- use network policies.

### 4.3 Kubernetes Storage

Core concepts:

- PV.
- PVC.
- StorageClass.
- StatefulSet.

Stateless services are easier to run. Stateful workloads need careful backup, recovery, and scheduling design.

### 4.4 Resource Management

Requests and limits:

- CPU request affects scheduling.
- CPU limit can cause throttling.
- Memory limit can cause OOM kill.

QoS classes:

- Guaranteed.
- Burstable.
- BestEffort.

### 4.5 Kubernetes High Availability

HA requires:

- multiple control-plane nodes.
- reliable etcd backup.
- multi-zone worker nodes.
- pod disruption budgets.
- anti-affinity.
- load balancer health checks.

---

## 5. GitOps and Continuous Delivery

### 5.1 GitOps

GitOps uses Git as the source of truth for desired cluster state.

Flow:

```text
Git change -> review -> merge -> controller syncs cluster -> drift detection
```

Benefits:

- auditability.
- rollback.
- reproducibility.
- consistent deployment process.

### 5.2 Argo CD

Argo CD continuously compares Git desired state with cluster actual state.

Key concepts:

- Application.
- Sync.
- Health.
- Drift.
- Rollback.

### 5.3 Helm

Helm packages Kubernetes manifests into charts.

Use values files to manage environment-specific configuration, but avoid overly complex templating that hides final manifests.

---

## 6. Kubernetes Troubleshooting

### 6.1 Pod States

Useful commands:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get events --sort-by=.lastTimestamp
```

Common issues:

- ImagePullBackOff.
- CrashLoopBackOff.
- Pending.
- OOMKilled.
- readiness probe failure.

### 6.2 Network Troubleshooting

Check:

- Service selector.
- EndpointSlice.
- DNS resolution.
- NetworkPolicy.
- CNI status.
- kube-proxy/eBPF datapath.

### 6.3 Performance Troubleshooting

Check:

- CPU throttling.
- memory limit and OOM.
- pod scheduling.
- node pressure.
- container restarts.
- noisy neighbors.

---

## 7. Interview Self-Check

### Q1: Container vs VM?

**Answer:** Containers share the host kernel and use OS-level isolation, so they are lighter. VMs virtualize hardware and run separate kernels, giving stronger isolation at higher cost.

### Q2: What are namespaces and cgroups?

**Answer:** Namespaces isolate views of system resources. Cgroups limit and account resource usage.

### Q3: What is a Pod?

**Answer:** A Pod is the smallest deployable unit in Kubernetes. Containers in a Pod share network namespace and can share volumes.

### Q4: How does Kubernetes scheduling work?

**Answer:** Scheduler filters and scores nodes based on resources, constraints, affinity, taints/tolerations, topology, and volume requirements.

### Q5: Service vs Ingress?

**Answer:** Service provides stable access to pods inside or outside the cluster. Ingress manages HTTP/HTTPS routing from outside into services.

### Q6: Readiness vs liveness probe?

**Answer:** Readiness controls whether a pod receives traffic. Liveness controls whether kubelet restarts the container.

### Q7: What is GitOps?

**Answer:** GitOps treats Git as the source of truth and uses controllers to reconcile cluster state with Git.

### Q8: Why can CPU limits hurt latency?

**Answer:** CPU limits can trigger throttling when a container exceeds quota, causing request latency spikes even if the node has idle CPU.

### Q9: What is service mesh?

**Answer:** A service mesh provides traffic management, security, and observability through sidecar or data-plane proxies.

### Q10: How do you debug CrashLoopBackOff?

**Answer:** Check logs, previous logs, describe pod events, command/args, env/config, probes, resource limits, dependencies, and recent image changes.

### Senior Interview Follow-Ups

### Q11: A Kubernetes rollout caused elevated 5xx but all pods are Running. What do you check?

**Answer:** `Running` only means containers started. Check readiness probes, endpoint membership, service selectors, ingress/gateway routing, recent config changes, dependency errors, and application logs. Compare old and new ReplicaSets by version, zone, node, and traffic share. If impact is active, pause or roll back the rollout first, then inspect traces, metrics, and events to identify whether the issue is startup readiness, bad config, incompatible schema, or downstream saturation.

### Q12: How do requests and limits affect production latency?

**Answer:** Requests drive scheduling and capacity planning. Limits enforce runtime ceilings. Memory limits can cause OOMKilled; CPU limits can cause throttling and P99 spikes even when average CPU looks fine. A practical setup gives predictable services enough requests, avoids unnecessary CPU limits for latency-sensitive workloads when policy allows, monitors throttling, and uses HPA/VPA carefully with load tests.

### Q13: How do you design Kubernetes reliability across zones?

**Answer:** Use multiple worker zones, topology spread constraints, pod anti-affinity, PodDisruptionBudgets, readiness gates, replicated dependencies, and zone-aware load balancing. etcd and control-plane HA need backups and tested restore. The trade-off is cost and operational complexity, so the design should be driven by RTO/RPO, traffic criticality, and dependency topology.


## 8. Production Kubernetes Updates

### CRI, OCI, containerd, and runc

Modern Kubernetes does not rely on dockershim. kubelet talks to container runtimes through CRI; containerd or CRI-O manages images and containers; an OCI runtime such as runc creates the actual Linux container. Docker is still common for local builds, but production Kubernetes should be described as CRI + OCI. Prefer immutable image digests over mutable tags when supply-chain reproducibility matters.

### Workloads and Pod Lifecycle

Use Deployment for stateless services, StatefulSet for stable identity and PVC binding, DaemonSet for node agents, Job for one-off tasks, and CronJob for scheduled jobs. `startupProbe` protects slow startup, `readinessProbe` controls traffic admission, and `livenessProbe` should only restart a process that cannot recover. Graceful shutdown requires readiness removal, `preStop`, `terminationGracePeriodSeconds`, and connection draining.

### Service, Ingress, Gateway API, and CNI

Service provides stable L4 service discovery. Ingress expresses HTTP routing and TLS through an ingress controller. Gateway API splits responsibilities across GatewayClass, Gateway, and Route resources. CNI decides Pod IP allocation, cross-node routing, and whether NetworkPolicy is enforced. Flannel is simple, Calico is mature for policy/BGP, and Cilium uses eBPF for policy, observability, and datapath acceleration.

### Security, Configuration, Autoscaling, and Operators

RBAC answers who can call which Kubernetes API; ServiceAccount is the workload identity; Admission enforces policy before persistence. ConfigMap/Secret updates do not automatically restart Deployments; use checksum annotations or reloaders. HPA scales replicas, VPA adjusts requests, and KEDA connects scaling to event sources such as Kafka lag or queue length. Operators extend Kubernetes with CRDs plus a controller loop that continuously reconciles state.

### Additional Senior Questions

### Q26: Does Kubernetes still use Docker directly?
No. kubelet uses CRI to call containerd or CRI-O, which then uses OCI runtimes such as runc. Docker remains useful for development and image building.

### Q27: Ingress vs Gateway API?
Ingress is a simpler HTTP routing resource. Gateway API is more expressive and separates infrastructure ownership from application routing through GatewayClass, Gateway, and Route objects.

### Q28: Why can a ConfigMap change fail to affect running pods?
Environment variables do not update after process start, mounted files may update but the app must reload them, and Deployments do not roll automatically unless you trigger them.
