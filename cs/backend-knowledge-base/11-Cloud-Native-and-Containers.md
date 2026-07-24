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

Service provides stable virtual IP and load balancing for pods.

Types:

- ClusterIP.
- NodePort.
- LoadBalancer.
- ExternalName.

kube-proxy implements service routing through iptables, IPVS, or eBPF depending on environment.

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
