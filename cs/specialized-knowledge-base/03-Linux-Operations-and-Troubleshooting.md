# Linux Operations and Troubleshooting

Language: English | [中文](../专项知识库/03-Linux运维与排障.md)

---

## Table of Contents

### Commands and Troubleshooting SOPs
1. [Common Command Quick Reference](#1-common-command-quick-reference)
2. [CPU Spike Troubleshooting SOP](#2-cpu-spike-troubleshooting-sop)
3. [Memory Troubleshooting SOP](#3-memory-troubleshooting-sop)
4. [Network Troubleshooting SOP](#4-network-troubleshooting-sop)
5. [Disk I/O Troubleshooting SOP](#5-disk-io-troubleshooting-sop)

### Practice and Self-Check
6. [Interview Self-Check](#6-interview-self-check)
7. [Production Cases](#7-production-cases)

---

## 1. Common Command Quick Reference

### 1.1 Process Management

```bash
# Process list and hierarchy
ps aux | grep java
ps -ef | grep nginx
ps -Lf <pid>
pstree -p <pid>

# Real-time process view
top
top -Hp <pid>
htop

# Process resources and open files
lsof -p <pid>
ls -l /proc/<pid>/fd
cat /proc/<pid>/status
cat /proc/<pid>/limits
```

Key judgment:

- `top` tells you which process or thread is hot.
- `/proc/<pid>` tells you what the kernel knows about the process.
- `lsof` often explains file descriptor leaks, deleted files still occupying disk, and unexpected network sockets.

### 1.2 Network

```bash
# Connections
ss -antp
ss -s
netstat -antp

# Packet capture
tcpdump -i eth0 port 8080 -w capture.pcap
tcpdump -i eth0 host 10.0.0.1 and port 3306
tcpdump -i eth0 -nn -s0 -c 100

# DNS and routes
dig example.com
nslookup example.com
host example.com
traceroute example.com
mtr example.com
ip route show
```

Prefer `ss` over `netstat` on modern Linux. For network issues, always separate name resolution, connection establishment, TLS handshake, request processing, and downstream dependency latency.

### 1.3 System Resources

```bash
# CPU
mpstat 1 10
pidstat -u 1
sar -u 1 10

# Memory
free -h
vmstat 1
pmap -x <pid>
cat /proc/meminfo

# Disk I/O and capacity
iostat -x 1 10
iotop
df -h
du -sh *

# Load
uptime
sar -q 1 10
```

Interpretation tips:

- High `us/sys`: CPU is actively busy.
- High `wa`: CPUs are waiting for I/O.
- High `st`: a VM is losing CPU time to the host, often caused by overcommit or noisy neighbors.

### 1.4 Troubleshooting Tools

```bash
# System calls
strace -p <pid>
strace -c -p <pid>
strace -T -p <pid>

# Profiling
perf top
perf record -g -p <pid>
perf report

# Kernel logs
dmesg -T
journalctl -xe
```

Production rule of thumb:

- Use `strace` carefully because `ptrace` can slow a syscall-heavy process heavily.
- Prefer `perf` for low-overhead sampling when you need production evidence.
- Use `tcpdump` with narrow filters and bounded capture size.

---

## 2. CPU Spike Troubleshooting SOP

### Step 1: Confirm CPU State

```bash
top
mpstat -P ALL 1 5
pidstat -u 1
```

Look at:

- `%us`: user-space code, often business logic or compute-heavy code.
- `%sy`: kernel work, often syscalls, networking, file system, or lock paths.
- `%wa`: I/O wait, not pure CPU saturation.
- `%st`: stolen time inside VMs.
- Load average versus CPU cores.

Load average includes runnable tasks and uninterruptible sleep tasks. A high load with low CPU often means many tasks are blocked in `D` state.

### Step 2: Locate the Process

```bash
ps aux | sort -nrk 3 | head -10
ps -p <pid> -o pid,ppid,cmd,%mem,%cpu
```

Confirm whether the hot process is the expected service. In containerized environments, map the host PID to the container or pod before escalating to the service owner.

### Step 3: Locate the Hot Thread

```bash
top -Hp <pid>
ps -Lf <pid> | sort -nrk 10 | head -10
```

Record the thread ID. For Java, convert it to hexadecimal:

```bash
printf "%x\n" <tid>
```

### Step 4: Capture Runtime Stack Evidence

Go:

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2
go tool pprof http://localhost:6060/debug/pprof/profile
```

Java:

```bash
jstack <pid> | grep -A 50 <hex_tid>
jstack <pid> > jstack.log
```

Python:

```bash
py-spy top --pid <pid>
py-spy dump --pid <pid>
```

Native or unknown process:

```bash
perf top -p <pid>
perf record -g -p <pid> -- sleep 30
perf report
```

### Step 5: Classify the Root Cause

Common causes:

- Infinite loop or busy polling.
- CPU-heavy encryption, compression, parsing, image processing, or serialization.
- Excessive garbage collection.
- Lock contention and spin loops.
- Kernel overhead from excessive syscalls or packet processing.

Senior-level answer pattern:

1. Identify whether the CPU is really busy or waiting.
2. Find process, thread, and stack.
3. Connect stack evidence to recent changes, traffic shape, or dependency behavior.
4. Mitigate first: rollback, rate-limit, reduce traffic, disable expensive feature, or scale out.
5. Verify P99 latency, error rate, and CPU state after mitigation.

---

## 3. Memory Troubleshooting SOP

### Step 1: Confirm Memory Pressure

```bash
free -h
cat /proc/meminfo
vmstat 1
```

Important fields:

- `available`: practical available memory including reclaimable cache.
- `buff/cache`: page cache and buffer cache.
- `si/so`: swap in/out; sustained non-zero values mean memory pressure.
- `Slab`, `SReclaimable`, `SUnreclaim`: kernel memory use.

Thresholds are workload-dependent, but `available < 10%` deserves attention and `available < 5%` is usually urgent.

### Step 2: Locate the Process

```bash
ps aux --sort=-rss | head
pmap -x <pid>
cat /proc/<pid>/smaps
```

Differentiate:

- Heap growth.
- Off-heap/native memory.
- File mappings.
- Thread stacks.
- Kernel memory not attributed to a user process.

### Step 3: Analyze Runtime Memory

Go:

```bash
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof
```

Java:

```bash
jmap -heap <pid>
jmap -histo <pid> | head -20
jmap -dump:format=b,file=heap.hprof <pid>
```

Python:

```python
import objgraph
objgraph.show_most_common_types(limit=10)
```

Be careful with heap dumps in production. Some dump operations can pause the process and expose sensitive data.

### Step 4: Check Common Leak Patterns

Go:

- Goroutines never exit because they lack `context` cancellation.
- Unbounded maps or slices used as caches.
- HTTP clients or response bodies are not closed.
- Timers or tickers are not stopped.

Java:

- Static maps retain request or session objects.
- ThreadLocal values are not removed in thread pools.
- Excessive direct buffers or off-heap caches.
- ClassLoader leaks in application servers.

Python:

- Global references keep objects alive.
- LRU caches have no practical size bound.
- Cycles involving finalizers are retained.

### Kernel Memory Leak Direction

If user processes do not explain the pressure, compare `/proc/meminfo` against a healthy machine:

```bash
cat /proc/meminfo
slabtop -s c
cat /proc/slabinfo
cat /proc/vmallocinfo
```

Classification:

- `Slab` high: inspect `slabtop`, `SUnreclaim`, dentry, inode, or kmalloc caches.
- `VmallocUsed` high: aggregate `/proc/vmallocinfo` by allocation function.
- Slab and vmalloc normal but memory missing: investigate page allocation leaks.

---

## 4. Network Troubleshooting SOP

### Scenario 1: Connection Count Explosion

```bash
ss -s
ss -antp | awk '{print $1}' | sort | uniq -c
ss -antp | grep ESTABLISHED | wc -l
ss -antp | grep TIME_WAIT | wc -l
```

Common causes:

- Excessive short-lived connections.
- Connection pool misconfiguration.
- Downstream timeout causing connection buildup.
- Accept queue or SYN queue overflow.
- Client-side ephemeral port exhaustion.

Mitigation:

- Enable keep-alive and connection pooling.
- Reduce unnecessary retries.
- Tune `ip_local_port_range`, `tcp_tw_reuse`, and backlog parameters carefully.
- Scale or protect the overloaded dependency.

Avoid `tcp_tw_recycle`; it is unsafe in NAT environments and removed from modern kernels.

### Scenario 2: High Latency

```bash
ping -c 10 example.com
mtr example.com
tcpdump -i eth0 -s 0 -w capture.pcap host example.com
```

Split the path:

1. DNS resolution.
2. TCP connect.
3. TLS handshake.
4. Application handling.
5. Downstream RPC or database access.

If retransmissions are not high but P99 is high, look for queueing, connection pool wait, thread pool saturation, GC, DNS stalls, or dependency long tail.

### Scenario 3: Packet Loss

```bash
ethtool -S eth0 | grep -E "drop|error"
ifconfig eth0 | grep -E "drop|error"
netstat -s | grep -E "drop|error"
ss -lnt
```

Interpretation:

- NIC errors or drops: check driver, ring buffer, interrupt balance, link quality, and host overload.
- Kernel TCP drops: check backlog, conntrack, socket buffers, and application accept speed.
- Queue buildup: `Recv-Q` or `Send-Q` may reveal application or network backpressure.

DNS intermittent failure:

1. Run `dig` from the failing node or pod.
2. Inspect `/etc/resolv.conf`.
3. Capture port 53 traffic.
4. If packets never leave, check local policy, iptables, CNI, NetworkPolicy, or security groups.
5. If packets leave but responses are slow, inspect CoreDNS capacity, upstream DNS, and conntrack.

---

## 5. Disk I/O Troubleshooting SOP

### Step 1: Confirm Disk Pressure

```bash
iostat -x 1 10
iotop -o
pidstat -d 1
```

Key fields:

- `%util`: device busy time; sustained high values suggest saturation.
- `await`: average I/O wait time.
- `r/s`, `w/s`: operation rate.
- `aqu-sz`: queue depth.

High `%util` with high `await` usually means real device pressure. High `await` with remote storage may indicate network storage or backend throttling.

### Step 2: Locate the Process and Files

```bash
lsof -p <pid>
lsof -p <pid> | grep -E "REG|DIR"
cat /proc/<pid>/io
```

Common causes:

- Excessive synchronous logging.
- Slow database queries causing random I/O.
- Cache miss storm.
- Swap activity.
- Remote disk or network storage jitter.
- File system almost full, causing fragmentation or metadata pressure.

### Step 3: Optimize by Pattern

- Frequent small writes: batch, buffer, or use async logging.
- Database I/O: fix indexes, reduce random reads, increase buffer pool, or separate hot data.
- Log storms: sample logs, lower log level, rotate correctly.
- Full disk: clean safely, expand online, or move cold data.

### Deleted File Still Occupies Disk

```bash
lsof | grep deleted
echo "" > /proc/<pid>/fd/<fd>
```

This explains the common `df` and `du` mismatch: `df` sees blocks still referenced by an open file descriptor, while `du` cannot see the deleted path.

---

## 6. Interview Self-Check

### Q1: How do you locate CPU 100% on a production Linux server?

**Answer**:

1. Use `top` or `mpstat` to confirm whether CPU is truly busy and which CPU state dominates.
2. Use `ps` or `top` to identify the hot process.
3. Use `top -Hp <pid>` to identify the hot thread.
4. Capture a stack: `jstack`, Go `pprof`, `py-spy`, or `perf`.
5. Map the stack to code, recent changes, traffic, and dependencies.
6. Mitigate first if users are affected, then complete root cause analysis.

### Q2: Why can load average be high while CPU usage is low?

**Answer**:

Load average counts runnable tasks and uninterruptible sleep tasks. If many processes are stuck in `D` state waiting for disk, NFS, or hardware, load can be high while CPU remains idle. Check `ps` for `D` state and inspect `/proc/<pid>/stack`.

### Q3: How do you distinguish CPU busy, CPU waiting, and CPU steal?

**Answer**:

Use `mpstat`, `sar -u`, and `top`:

- `us/sys` high means real CPU work.
- `wa` high means I/O wait.
- `st` high means the VM is being preempted by the host.

Then correlate with `pidstat -u -w`, `vmstat`, disk metrics, and hypervisor or cloud host signals.

### Q4: How do you locate a memory leak in Go and Java?

**Answer**:

Common flow:

1. Confirm sustained growth using `free`, RSS, and runtime metrics.
2. Identify the process and memory type.
3. For Go, capture heap profiles and compare before/after.
4. For Java, use `jmap -histo`, heap dump, and GC logs.
5. Confirm whether the growth is heap, off-heap, thread stack, mmap, or kernel memory.

### Q5: What should you do when TIME_WAIT is excessive?

**Answer**:

First confirm who actively closes connections and whether short connections are expected. Prefer connection pooling and keep-alive over kernel tuning. Tune ephemeral port range and safe TIME_WAIT reuse only after understanding the traffic pattern. Never rely on unsafe NAT-breaking options.

### Q6: What is the difference between `strace` and `perf`?

**Answer**:

`strace` uses `ptrace` to intercept syscalls and can create heavy overhead. It is useful for syscall-level debugging. `perf` samples CPU or kernel events with lower overhead and is generally better for production profiling.

### Q7: How do SYN queue and accept queue differ?

**Answer**:

The SYN queue stores half-open connections during handshake. The accept queue stores fully established connections waiting for the application to call `accept()`. SYN queue overflow causes connection timeouts; accept queue overflow may cause refused connections or retransmission behavior depending on kernel settings.

### Q8: Why is Page Cache useful, and why do databases sometimes bypass it?

**Answer**:

Page Cache accelerates file reads and buffers writes. Databases often have their own buffer pool. Using both can cause double caching and reduce control over eviction, so systems such as MySQL may use `O_DIRECT` for data files.

### Q9: DNS intermittently fails from one node. How do you isolate the layer?

**Answer**:

Run `dig` on the node, inspect resolver config, capture port 53, and classify the failure: request not sent, request sent without response, slow response, or wrong answer. Then investigate local firewall or network policy, DNS service capacity, upstream resolver, and conntrack.

### Q10: Readiness is healthy but P99 latency is high. What do you check first?

**Answer**:

Readiness only proves the probe path works. Check user-path RED metrics, application queues, thread pools, connection pools, GC, downstream latency, CPU states, disk `await`, network drops, and DNS timing.

### Q11: How do you build a minimal root-cause loop during an incident?

**Answer**:

Define the symptom, split it by layer, collect one or two decisive pieces of evidence, mitigate, and validate recovery. A senior answer should form a causal chain rather than list commands.

### Q12: How do you handle a Linux host that is fully unresponsive?

**Answer**:

If SSH is unavailable, use out-of-band management. If kdump is configured, trigger crash dump through SysRq or NMI, collect `vmcore`, and analyze with `crash`. For production fleets, automate host fencing, VM or pod evacuation, and crash evidence collection.

---

### Open-Ended Design Questions

**D1: A server suddenly has 100k+ TIME_WAIT connections and new connections fail. What do you do?**

Reference answer:

- Confirm state distribution and whether failures are due to port exhaustion, accept queue overflow, or downstream slowness.
- Stop the bleeding with connection pooling, rate limiting, retry reduction, and capacity expansion.
- Apply safe kernel tuning only after validating traffic shape.
- Fix the root cause: short connection design, load balancer behavior, or client SDK reuse.

**D2: The root partition is almost full and the service cannot stop. How do you expand safely?**

Reference answer:

- Free emergency space safely: truncate active logs through file descriptors, clean old artifacts, avoid deleting files still held open.
- Expand cloud disk, grow partition, and grow file system using the correct tool for ext4 or XFS.
- Move large cold directories to a new mount if online expansion is not enough.
- Add alerts, log rotation, and deployment guards afterward.

---

### Deep Dive Topics

Senior candidates should be able to explain:

- Load average calculation and why `D` state matters.
- TCP state transitions, TIME_WAIT, and 2MSL.
- Page Cache read/write path and direct I/O trade-offs.
- TCP backlog, SYN queue, accept queue, and socket buffer tuning.
- `strace`, `perf`, and `tcpdump` implementation principles.
- Crash dump, SysRq, NMI, and kdump workflow.
- Kernel memory leak categories: slab, vmalloc, page allocation.

---

## 7. Production Cases

### Case 1: CPU 100%

Symptom: a Java service CPU jumps to 100%.

Process:

```bash
top
top -Hp <pid>
printf "%x\n" <tid>
jstack <pid> | grep -A 50 <hex_tid>
```

Root cause: a loop in `processData()` had no blocking or exit condition.

Resolution: rollback or hotfix the loop, then add metrics and tests around the path.

### Case 2: Go Memory Leak

Symptom: Go service RSS grows for three days and finally OOMs.

Process:

```bash
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof
```

Root cause: a new HTTP client was created per request and response bodies were not consistently closed.

Resolution: reuse clients, close bodies, set connection pool limits, and add heap profile comparison to the incident review.

### Case 3: Connection Exhaustion

Symptom: new requests fail while the service process remains alive.

Process:

```bash
ss -s
ss -antp | awk '{print $1}' | sort | uniq -c
lsof -p <pid> | wc -l
```

Root cause: short connections and retry amplification created a TIME_WAIT and file descriptor surge.

Resolution: enable keep-alive, reduce retry fan-out, increase connection pool reuse, and alert on connection states and FD usage.

---

## Summary

Linux troubleshooting is a disciplined narrowing process:

1. Observe the symptom with user impact in mind.
2. Locate the layer: application, runtime, OS, network, storage, or host infrastructure.
3. Gather decisive evidence, not every possible metric.
4. Mitigate first when users are affected.
5. Validate recovery and turn the incident into a runbook, alert, or design improvement.
