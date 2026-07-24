# IaaS Infrastructure Technologies

Language: English | [中文](../专项知识库/06-IaaS基础设施技术.md)

---

## Table of Contents

### Part 1: Virtualization
1. [Virtualization Overview](#1-virtualization-overview)
2. [KVM Architecture and Principles](#2-kvm-architecture-and-principles)
3. [QEMU and Device Emulation](#3-qemu-and-device-emulation)
4. [libvirt Management Framework](#4-libvirt-management-framework)
5. [Virtual Machine Image Formats](#5-virtual-machine-image-formats)
6. [Deep Dive: CPU, Memory, and I/O Virtualization](#6-deep-dive-cpu-memory-and-io-virtualization)

### Part 2: Virtual Machine Networking
7. [Virtual Machine Network Modes](#7-virtual-machine-network-modes)
8. [Linux Network Virtualization Components](#8-linux-network-virtualization-components)
9. [VM Network Configuration Practice](#9-vm-network-configuration-practice)
10. [VM Migration](#10-vm-migration)
11. [VNC Remote Management](#11-vnc-remote-management)

### Part 3: Physical and Virtualized Networking
12. [Physical Network Foundations](#12-physical-network-foundations)
13. [VLAN](#13-vlan)
14. [Network Tunneling](#14-network-tunneling)
15. [VXLAN Layer-2 Overlay](#15-vxlan-layer-2-overlay)
16. [VPC](#16-vpc)

### Part 4: Server-Level Technologies
17. [Server Management](#17-server-management)
18. [High-Performance Networking with DPDK](#18-high-performance-networking-with-dpdk)
19. [High-Performance Storage with SPDK](#19-high-performance-storage-with-spdk)
20. [RDMA](#20-rdma)
21. [LVM](#21-lvm)

### Part 5: Linux Operations Deep Dive
22. [Linux Kernel Parameter Tuning](#22-linux-kernel-parameter-tuning)
23. [System Troubleshooting Tools](#23-system-troubleshooting-tools)
24. [Performance Analysis and Monitoring](#24-performance-analysis-and-monitoring)
25. [File System Troubleshooting](#25-file-system-troubleshooting)
26. [Network Troubleshooting Tools](#26-network-troubleshooting-tools)
27. [Kernel Analysis Tools](#27-kernel-analysis-tools)
28. [Interview Self-Check](#28-interview-self-check)

---

# Part 1: Virtualization

## 1. Virtualization Overview

| Type | Representative Technologies | Performance | Isolation | Use |
|------|-----------------------------|-------------|-----------|-----|
| Full virtualization | KVM, Xen HVM | high | strong | multi-tenant production |
| Paravirtualization | Xen PV, virtio | very high | strong | optimized guest drivers |
| Container virtualization | Docker, LXC | near native | medium | microservices, CI |
| Hardware-assisted virtualization | Intel VT-x, AMD-V | high | strong | modern cloud platforms |

### 1.1 Type 1 vs Type 2

Type 1 hypervisors run close to hardware:

```text
VMs -> Hypervisor -> Hardware
```

Examples: KVM, Xen, ESXi, Hyper-V.

Type 2 hypervisors run on a host operating system:

```text
VMs -> Hosted Hypervisor -> Host OS -> Hardware
```

Examples: VirtualBox, VMware Workstation.

Cloud platforms prefer Type 1 or Type-1-like architectures because they need performance, isolation, and operational control.

---

## 2. KVM Architecture and Principles

KVM turns the Linux kernel into a hypervisor. It provides CPU and memory virtualization through `/dev/kvm`, while QEMU runs in user space to emulate or paravirtualize devices.

```text
Guest VM
  -> Guest kernel and applications
  -> vCPU, vMemory, vIO
Host user space
  -> QEMU process
Host kernel
  -> KVM module
Hardware
  -> VT-x / AMD-V / EPT / NPT
```

### 2.1 Core Components

| Component | Role |
|-----------|------|
| KVM kernel module | vCPU execution, memory virtualization, interrupt virtualization |
| QEMU | device emulation, disk and network backends, monitor, migration |
| `/dev/kvm` | ioctl interface between QEMU and KVM |
| libvirt | management API and lifecycle orchestration |
| virtio | paravirtualized high-performance device model |

### 2.2 CPU Virtualization

Key concepts:

- VMX root and non-root mode.
- VM Entry and VM Exit.
- VMCS stores guest and host execution state.
- Certain privileged instructions cause VM Exit.

Simplified vCPU loop:

```c
for (;;) {
    prepare_guest_state();
    vm_entry();
    reason = read_vm_exit_reason();
    handle_exit(reason);
}
```

VM Exit is expensive. Performance work often means reducing unnecessary exits through virtio, APICv, posted interrupts, huge pages, and better device models.

### 2.3 Memory Virtualization

Address path:

```text
GVA -> Guest page table -> GPA -> EPT/NPT -> HPA
```

Terms:

| Term | Meaning |
|------|---------|
| GVA | guest virtual address |
| GPA | guest physical address |
| HVA | host virtual address in QEMU |
| HPA | host physical address |

Important technologies:

- EPT/NPT: second-level address translation.
- Huge pages: reduce TLB pressure.
- KSM: deduplicate identical memory pages.
- Ballooning: reclaim guest memory under host pressure.
- NUMA pinning: keep vCPU and memory close.

Trade-off: overcommit improves utilization but increases tail latency and noisy-neighbor risk.

### 2.4 Interrupt Virtualization

Traditional interrupt injection may require VM Exit and VM Entry. Optimizations include:

- MSI-X
- irqfd
- APICv
- posted interrupts
- vCPU and interrupt affinity

High network PPS workloads are very sensitive to interrupt handling overhead.

---

## 3. QEMU and Device Emulation

QEMU provides device models and VM process management. With KVM acceleration, QEMU delegates guest CPU execution to KVM and handles devices, I/O, migration, and management interfaces.

### 3.1 QEMU, KVM, and qemu-kvm

| Name | Meaning |
|------|---------|
| QEMU | generic emulator and virtualizer |
| KVM | Linux kernel virtualization module |
| qemu-kvm | QEMU built with KVM acceleration |

Without KVM, QEMU can emulate CPU in software, but performance is far below production requirements.

### 3.2 virtio

virtio is the standard paravirtualized device framework:

- `virtio-net`: network.
- `virtio-blk`: block storage.
- `virtio-scsi`: SCSI storage.
- `virtio-balloon`: memory ballooning.
- `virtio-serial`: guest-host communication.

Vring structure:

1. Descriptor table.
2. Available ring: guest writes requests.
3. Used ring: backend writes completed requests.

Notification:

- guest to host: kick through I/O port, MMIO, or eventfd.
- host to guest: interrupt through irqfd or MSI-X.

### 3.3 QEMU Thread Model

Typical threads:

- main thread: event loop and management.
- vCPU threads: one per vCPU.
- I/O threads: optional dedicated device I/O loops.
- worker threads: background tasks such as qcow2 operations.

If the main thread blocks, vCPU threads may still execute guest code, but device operations requiring the main thread can stall. Use I/O threads for high-performance disk and network paths.

---

## 4. libvirt Management Framework

libvirt provides a stable management API over KVM/QEMU, Xen, LXC, and other virtualization backends.

### 4.1 Core Concepts

| Concept | Meaning |
|---------|---------|
| Domain | virtual machine |
| Network | virtual network object |
| Storage pool | source of storage volumes |
| Volume | disk image |
| XML definition | VM configuration |

### 4.2 virsh Commands

```bash
# lifecycle
virsh list --all
virsh start <vm>
virsh shutdown <vm>
virsh destroy <vm>

# configuration
virsh dumpxml <vm>
virsh edit <vm>

# snapshots
virsh snapshot-create-as <vm> snap1
virsh snapshot-list <vm>

# migration
virsh migrate --live <vm> qemu+ssh://host/system
```

### 4.3 Production Concerns

- CPU model compatibility across hosts.
- Storage availability during migration.
- Network identity continuity.
- Guest agent availability.
- NUMA and huge page settings.
- Snapshot consistency.

---

## 5. Virtual Machine Image Formats

| Format | Advantages | Disadvantages | Use |
|--------|------------|---------------|-----|
| raw | simplest, fastest | no snapshots or compression | performance-sensitive workloads |
| qcow2 | snapshots, compression, sparse allocation | more overhead | general-purpose cloud images |
| vmdk | VMware ecosystem | portability concerns | VMware migration |

### 5.1 raw

```bash
qemu-img create -f raw disk.raw 10G
qemu-img convert -f qcow2 -O raw disk.qcow2 disk.raw
```

### 5.2 qcow2

```bash
qemu-img create -f qcow2 disk.qcow2 10G
qemu-img create -f qcow2 -b base.qcow2 vm1.qcow2
qemu-img info disk.qcow2
qemu-img snapshot -c snap1 disk.qcow2
```

qcow2 is operationally convenient, but performance-sensitive workloads may prefer raw or tuned qcow2 with appropriate cache and async I/O settings.

### 5.3 Image Optimization

```bash
# compact after zero-filling inside guest
qemu-img convert -O qcow2 old.qcow2 compact.qcow2

# extend image
qemu-img resize disk.qcow2 +10G

# check and repair
qemu-img check disk.qcow2
```

Senior answer: image format choice is a trade-off between performance, snapshot needs, storage efficiency, migration workflow, and operational tooling.

---

## 6. Deep Dive: CPU, Memory, and I/O Virtualization

### 6.1 CPU Pinning and NUMA

For latency-sensitive VMs:

- pin vCPU threads to physical CPUs
- align VM memory to NUMA node
- avoid cross-NUMA memory access
- isolate host CPUs for noisy workloads

Example concepts:

```bash
virsh vcpupin <vm> <vcpu> <host-cpu>
numactl --hardware
```

### 6.2 Huge Pages

Huge pages reduce page table overhead and TLB misses.

Trade-offs:

- better performance and more stable latency
- less flexible memory allocation
- requires capacity reservation
- can fragment host memory if not managed early

### 6.3 I/O Optimization

Storage:

- use virtio-blk or virtio-scsi
- use I/O threads
- choose cache mode carefully
- align guest file system and block size
- avoid snapshot chains for high-write workloads

Network:

- use virtio-net
- enable multiqueue
- tune RSS/RPS/XPS
- place IRQs and vCPUs carefully
- use SR-IOV or passthrough for extreme performance

---

# Part 2: Virtual Machine Networking

## 7. Virtual Machine Network Modes

Common modes:

| Mode | Meaning | Use |
|------|---------|-----|
| NAT | VM exits through host translation | development and simple labs |
| Bridge | VM appears on the same L2 network as host | production-like networking |
| Routed | host routes VM subnet | controlled L3 networks |
| Isolated | VM-only private network | testing and security isolation |
| SR-IOV | VF passed to VM | high-performance networking |

Bridge mode is common in private cloud platforms. SR-IOV improves performance but reduces migration flexibility and network observability.

---

## 8. Linux Network Virtualization Components

Important primitives:

- `tap`: virtual NIC endpoint used by QEMU.
- `veth`: pair of virtual Ethernet devices.
- `bridge`: L2 forwarding switch in Linux.
- `iptables` / `nftables`: filtering and NAT.
- `tc`: traffic shaping and qdisc.
- `ip netns`: network namespace isolation.
- OVS: programmable virtual switch for cloud networking.

Typical VM path:

```text
Guest virtio-net
  -> QEMU tap
  -> Linux bridge or OVS
  -> physical NIC or tunnel
```

Troubleshooting:

```bash
ip link
bridge link
bridge fdb show
ovs-vsctl show
tcpdump -i <tap>
tcpdump -i <bridge>
```

---

## 9. VM Network Configuration Practice

### 9.1 Bridge Example

```bash
ip link add br0 type bridge
ip link set br0 up
ip link set eth0 master br0
ip tuntap add tap0 mode tap
ip link set tap0 master br0
ip link set tap0 up
```

### 9.2 Checklist

When a VM cannot reach the network:

1. Guest NIC and IP.
2. Guest route and DNS.
3. tap device on host.
4. bridge or OVS forwarding.
5. security group or firewall.
6. physical NIC.
7. upstream network or tunnel.

Senior troubleshooting answer: prove the packet at each hop instead of guessing from the top.

---

## 10. VM Migration

### 10.1 Live Migration Flow

1. Pre-copy memory while VM is running.
2. Dirty pages are copied repeatedly.
3. Brief stop-and-copy phase transfers final dirty pages and device state.
4. Destination resumes the VM.

### 10.2 Requirements

- compatible CPU model
- shared storage or block migration
- network reachability
- consistent VM configuration
- migration bandwidth
- controlled dirty page rate

### 10.3 Failure Modes

- high memory dirty rate prevents convergence
- incompatible CPU flags
- storage latency spike
- network interruption
- passthrough device cannot migrate

Mitigations:

- throttle workload briefly
- use post-copy carefully
- pin CPU models across clusters
- avoid passthrough for workloads requiring live migration

---

## 11. VNC Remote Management

VNC provides console access when network access inside the guest is broken. Use it for:

- boot failures
- kernel panic inspection
- network misconfiguration
- rescue mode

Security concerns:

- restrict VNC binding address
- require authentication
- tunnel over SSH or management network
- audit access

---

# Part 3: Physical and Virtualized Networking

## 12. Physical Network Foundations

Core concepts:

- L2 switching and MAC learning.
- L3 routing and ARP/ND.
- MTU and fragmentation.
- ECMP.
- TCP congestion control.
- NIC offloads.
- RSS and interrupt distribution.

Cloud networking failures often come from a mismatch between overlay MTU, physical MTU, security policy, and routing state.

---

## 13. VLAN

VLAN uses 802.1Q tags to split L2 networks.

Use cases:

- tenant or environment isolation
- management network separation
- storage network separation
- physical switch segmentation

Limitations:

- VLAN ID space is limited to 4096.
- L2 domains can become too large.
- Multi-tenant cloud needs more scalable overlays.

---

## 14. Network Tunneling

Tunnels encapsulate packets to create overlays:

| Tunnel | Use |
|--------|-----|
| GRE | simple L3 tunnel |
| IPIP | IP over IP |
| VXLAN | scalable L2 over L3 |
| Geneve | extensible cloud overlay |

Key concerns:

- MTU overhead.
- encapsulation CPU cost.
- packet visibility.
- underlay reachability.
- offload support.

---

## 15. VXLAN Layer-2 Overlay

VXLAN creates L2 segments over L3 underlay.

Key terms:

- VNI: VXLAN network identifier, 24-bit.
- VTEP: VXLAN tunnel endpoint.
- underlay: physical routed network.
- overlay: virtual tenant network.

Packet path:

```text
VM packet
  -> local VTEP
  -> VXLAN encapsulation
  -> underlay IP network
  -> remote VTEP
  -> remote VM
```

Advantages:

- larger tenant ID space than VLAN.
- L2 adjacency over L3 network.
- good fit for cloud multi-tenancy.

Risks:

- MTU issues.
- troubleshooting complexity.
- control plane design.
- broadcast, unknown unicast, multicast handling.

---

## 16. VPC

A VPC is a tenant-isolated virtual network abstraction.

Typical components:

- subnets
- route tables
- security groups
- network ACLs
- NAT gateway
- VPN or direct connect
- load balancer
- peering or transit gateway

### 16.1 Security Group vs Network ACL

| Feature | Security Group | Network ACL |
|---------|----------------|-------------|
| Scope | instance or ENI | subnet |
| State | stateful | stateless |
| Rule type | allow-focused | allow and deny |
| Order | evaluated as a set | ordered |
| Use | fine-grained workload control | coarse subnet boundary |

---

# Part 4: Server-Level Technologies

## 17. Server Management

Important technologies:

- BMC/IPMI/Redfish for out-of-band management.
- BIOS/UEFI settings.
- RAID and HBA configuration.
- firmware lifecycle.
- hardware health monitoring.
- CE/UE memory error detection.
- power and thermal telemetry.

When a host has increasing correctable memory errors:

1. Assess rate and DIMM location.
2. Evacuate VMs by priority.
3. Handle non-migratable passthrough workloads through maintenance windows.
4. Mark host unschedulable.
5. Replace hardware and run validation.

---

## 18. High-Performance Networking with DPDK

DPDK moves packet processing to user space and bypasses much of the kernel network stack.

Benefits:

- high packet per second throughput
- reduced syscall overhead
- polling model
- huge page memory
- CPU affinity control

Costs:

- dedicated CPU cores
- more complex observability
- bypasses kernel tools in some paths
- operational complexity

Use cases:

- virtual switches
- NFV
- high-performance gateways
- packet processing appliances

---

## 19. High-Performance Storage with SPDK

SPDK applies a user-space, polling, zero-copy approach to NVMe and storage services.

Benefits:

- lower latency
- higher IOPS
- better CPU cache behavior

Costs:

- dedicated polling cores
- huge page management
- specialized debugging
- higher operational requirements

Use cases:

- NVMe-oF targets.
- high-performance cloud block storage.
- storage gateways.

---

## 20. RDMA

RDMA allows direct memory access across hosts with low CPU overhead.

Types:

- InfiniBand.
- RoCE.
- iWARP.

Benefits:

- low latency.
- high bandwidth.
- low CPU cost.

Challenges:

- lossless or carefully tuned networks for RoCE.
- congestion control.
- PFC and ECN tuning.
- difficult troubleshooting.
- strict hardware and driver compatibility.

Use cases:

- distributed storage.
- HPC.
- low-latency databases.
- AI training clusters.

---

## 21. LVM

LVM concepts:

- PV: physical volume.
- VG: volume group.
- LV: logical volume.

Common operations:

```bash
pvcreate /dev/sdb
vgcreate vg_data /dev/sdb
lvcreate -L 100G -n lv_app vg_data
mkfs.xfs /dev/vg_data/lv_app

lvextend -L +50G /dev/vg_data/lv_app
xfs_growfs /mountpoint
```

Strengths:

- flexible resizing
- snapshots
- volume grouping

Risks:

- snapshot performance overhead
- metadata corruption risk
- operational mistakes can affect multiple volumes

---

# Part 5: Linux Operations Deep Dive

## 22. Linux Kernel Parameter Tuning

Tune only with a clear hypothesis and rollback.

Common areas:

### 22.1 TCP Queues

```bash
sysctl -w net.core.somaxconn=2048
sysctl -w net.ipv4.tcp_max_syn_backlog=4096
sysctl -w net.ipv4.tcp_syncookies=1
```

Concepts:

- SYN queue stores half-open connections.
- Accept queue stores established connections waiting for application accept.
- Queue tuning cannot fix a slow application forever; it only adds buffer.

### 22.2 Socket Buffers

```bash
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
```

Large buffers help high bandwidth-delay product paths but can hide backpressure.

### 22.3 File Descriptors

```bash
ulimit -n
cat /proc/sys/fs/file-max
cat /proc/<pid>/limits
```

FD tuning must be paired with leak detection and connection lifecycle management.

### 22.4 BBR

```bash
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

BBR can help high-latency or lossy networks by modeling bandwidth and RTT instead of relying only on packet loss. It is not a cure for application bottlenecks.

---

## 23. System Troubleshooting Tools

| Tool | Use |
|------|-----|
| `top` / `htop` | process and CPU overview |
| `mpstat` | per-CPU state |
| `pidstat` | per-process CPU, memory, I/O, context switches |
| `vmstat` | run queue, memory, swap, CPU |
| `iostat` | disk utilization and latency |
| `iotop` | per-process I/O |
| `lsof` | open files and sockets |
| `strace` | syscall tracing |
| `perf` | low-overhead profiling |
| `tcpdump` | packet capture |

Senior principle: choose the tool based on the hypothesis, not habit.

---

## 24. Performance Analysis and Monitoring

### 24.1 CPU

Check:

- user vs system CPU
- run queue
- context switches
- interrupts
- steal time
- hotspots through perf

### 24.2 Memory

Check:

- RSS versus cache
- swap activity
- slab growth
- page faults
- OOM logs

### 24.3 Disk

Check:

- `%util`
- `await`
- queue depth
- read/write mix
- dirty pages
- file system fullness

### 24.4 Network

Check:

- packet drops
- retransmissions
- connection state distribution
- NIC errors
- DNS behavior
- conntrack saturation

---

## 25. File System Troubleshooting

### 25.1 `df` and `du` Mismatch

Most common cause: deleted files still held open.

```bash
lsof | grep deleted
echo "" > /proc/<pid>/fd/<fd>
```

### 25.2 File System Full

Checklist:

- largest directories
- deleted-but-open files
- log rotation
- inode exhaustion
- container logs
- snapshots
- temporary files

```bash
df -h
df -i
du -xh /var | sort -h | tail
```

### 25.3 ext4 Deletion Recovery

If a large file is deleted:

1. Stop writes immediately.
2. Remount read-only if possible.
3. Use `debugfs` to inspect inode and extent data.
4. Recover blocks with `dd` if metadata remains.

Recovery is never guaranteed. The best fix is backup and immutability for critical data.

---

## 26. Network Troubleshooting Tools

```bash
ss -antp
ss -s
ip addr
ip route
ip neigh
ethtool -S eth0
tcpdump -i any port 53
conntrack -S
mtr <host>
```

Layered workflow:

1. Is the name resolved?
2. Is the route correct?
3. Is ARP or neighbor discovery working?
4. Is the packet leaving the host?
5. Is the packet returning?
6. Is the socket accepted by the application?

---

## 27. Kernel Analysis Tools

### 27.1 kdump and crash

Use for hard hangs and kernel panics:

```bash
systemctl enable kdump
systemctl start kdump
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/*/vmcore
```

Inside `crash`:

```text
bt
log
ps
files
vm
```

### 27.2 SysRq and NMI

SysRq can trigger controlled kernel actions when the system partially responds. NMI can trigger panic through out-of-band management when the OS is unresponsive and configured for crash dump collection.

### 27.3 ftrace and perf

Use `ftrace` for targeted kernel event tracing and `perf` for sampling:

```bash
perf record -e 'kmem:kmem_cache_alloc' -a -g -- sleep 30
perf report
```

---

## 28. Interview Self-Check

### Q1: What is the relationship between QEMU and KVM?

**Answer**: KVM is the kernel virtualization module that runs vCPUs with hardware assistance. QEMU is the user-space process that provides device models, I/O backends, management, and migration. In production they work together.

### Q2: What is VM Exit and why does it matter?

**Answer**: VM Exit is a transition from guest execution to host handling. It is necessary for privileged operations and device emulation but expensive, so high-performance virtualization reduces unnecessary exits.

### Q3: Explain GVA, GPA, HVA, and HPA.

**Answer**: GVA is guest virtual address, GPA is guest physical address, HVA is QEMU host virtual address, and HPA is real host physical address. Guest memory access typically follows GVA -> guest page table -> GPA -> EPT/NPT -> HPA.

### Q4: How does virtio improve performance?

**Answer**: virtio avoids expensive full hardware emulation by using paravirtualized front-end and back-end drivers with shared vrings and event notifications.

### Q5: What are QEMU's main threads?

**Answer**: main thread, vCPU threads, optional I/O threads, and worker threads. Device operations tied to the main thread can stall if it blocks.

### Q6: raw versus qcow2: how do you choose?

**Answer**: raw is simpler and faster; qcow2 offers snapshots, sparse allocation, backing files, and compression. Use raw for performance-sensitive workloads and qcow2 for operational flexibility.

### Q7: How does live migration work?

**Answer**: pre-copy memory while the VM runs, repeatedly copy dirty pages, briefly stop the VM, transfer final state, then resume on the destination. It needs compatible CPU, storage, network, and manageable dirty page rate.

### Q8: What is VXLAN and why is it used?

**Answer**: VXLAN is an L2 overlay over an L3 underlay using VNI and VTEP. It gives cloud platforms scalable tenant networks beyond VLAN limitations.

### Q9: Security Group versus Network ACL?

**Answer**: security groups are stateful and apply to instances or ENIs. Network ACLs are stateless and apply to subnets with ordered allow and deny rules.

### Q10: When would you use SR-IOV?

**Answer**: use SR-IOV for very high network performance and low latency. Avoid it when live migration, flexible scheduling, or deep virtual network observability is more important.

### Q11: What are DPDK's benefits and costs?

**Answer**: DPDK provides high PPS through user-space polling and kernel bypass, but it consumes dedicated CPU, complicates debugging, and bypasses some kernel tooling.

### Q12: What is RDMA good for and why is it hard?

**Answer**: RDMA is excellent for low-latency, high-throughput storage, HPC, and AI clusters. It is hard because RoCE often needs careful lossless network tuning, congestion control, and hardware compatibility.

### Q13: How do you troubleshoot high iowait?

**Answer**: confirm `wa`, identify the busy disk with `iostat`, identify processes with `iotop` and `pidstat`, inspect files with `lsof`, check dirty pages and swap, then classify logging, database, remote storage, or disk hardware as the root direction.

### Q14: Why can `df` and `du` disagree?

**Answer**: a deleted file can still be held by an open file descriptor. `df` sees allocated blocks; `du` cannot see the deleted path. Use `lsof | grep deleted`.

### Q15: How do you handle a host with increasing memory CE errors while VMs are running?

**Answer**: evaluate error rate and DIMM location, live-migrate critical VMs first, coordinate cold migration for passthrough workloads, mark the host unschedulable, replace hardware, and validate before returning it to the pool.

### Q16: How would you design a scheduler for millions of VMs?

**Answer**: use a two-stage scheduler: filter eligible hosts by hard constraints, then score by resource fit, anti-affinity, AZ balance, fragmentation, and risk. Maintain indexes to avoid scanning all hosts, and use migration to defragment over time.

---

### Open-Ended Design Questions

**D1: Design a cloud scheduler for a million VMs.**

Reference answer:

- Hard filters: CPU, memory, storage, network, host health, tenant constraints.
- Scoring: best fit, fragmentation reduction, AZ balance, anti-affinity, performance class.
- Scale: host indexes, resource caches, asynchronous reconciliation.
- Safety: reserve buffers, avoid overcommit for critical workloads, detect noisy neighbors.
- Defragmentation: live migration and maintenance windows.

**D2: A physical server reports increasing correctable memory errors while carrying production VMs. What do you do?**

Reference answer:

- Assess CE rate and risk of UE.
- Prioritize VM evacuation by business criticality.
- Live-migrate migratable VMs.
- Coordinate cold migration for passthrough or pinned workloads.
- Mark host unschedulable and open hardware replacement workflow.
- Verify host health before rejoining the pool.

---

## Summary

IaaS engineering connects kernel virtualization, network overlays, storage, hardware, and operations. Senior-level understanding means explaining not only each component, but also the trade-offs: performance versus mobility, utilization versus isolation, automation versus safety, and abstraction versus debuggability.
