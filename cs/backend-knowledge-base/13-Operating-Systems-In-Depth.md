# Operating Systems In Depth

Language: English | [中文](../后端知识库/13-操作系统深入.md)

---

## Table of Contents

### Foundations
0. [Operating System Big Picture](#0-operating-system-big-picture)

### Process and Memory
1. [Processes and Threads](#1-processes-and-threads)
2. [Memory Management](#2-memory-management) (reclaim chain, THP, cache/MESI)

### I/O and Files
3. [IO Models and Async IO](#3-io-models-and-async-io) (epoll LT/ET, io_uring, zero copy)
4. [File Systems](#4-file-systems)

### Network and Isolation
5. [Network Stack (Kernel Path)](#5-network-stack-kernel-path) → protocol detail in [06](./06-Networking-Fundamentals-and-Protocols.md)
6. [Namespace and Cgroup](#6-namespace-and-cgroup-resource-isolation)
7. [System Calls and Performance](#7-system-calls-and-performance)

### Kernel and Production
8. [Linux Kernel Topics](#8-linux-kernel-topics)
9. [Production Troubleshooting SOP](#9-production-troubleshooting-sop)

### Self-Check
10. [Interview Self-Check](#10-interview-self-check)
11. [Summary](#summary)

---

## 0. Operating System Big Picture

An operating system is the control layer between hardware and applications. It solves four problems:

1. **Abstract hardware**: expose CPU, memory, disks, and NICs as processes, virtual memory, files, and sockets.
2. **Allocate resources**: decide who gets CPU, memory, IO, and network bandwidth.
3. **Isolate and protect**: keep processes from stomping each other's memory or driving hardware directly.
4. **Expose kernel services**: files, networking, process control, time, and devices via system calls.

### 0.1 System Structure

```text
Applications
  |  normal calls: business logic, serialization, GC, algorithms
  |
  |- syscalls: read/write/open/socket/epoll_wait/futex/mmap
  |
Operating System Kernel
  |- scheduling: processes, threads, context switch, CFS/EEVDF
  |- memory: virtual memory, page table, TLB, page fault, page cache
  |- filesystem: inode, directory, page cache, fsync, block device
  |- networking: socket, TCP/IP, softirq, congestion control
  |- isolation: namespace (view) + cgroup (quota)
  |- drivers: disk, NIC, timer, interrupts
  |
Hardware: CPU / memory / disk / NIC
```

| Topic | Problem | Section |
|-------|---------|---------|
| Process and thread | how CPU runs many tasks | [Processes and Threads](#1-processes-and-threads) |
| Memory management | how virtual addresses map to physical memory | [Memory Management](#2-memory-management) |
| IO model | how applications wait for file/network IO | [IO Models and Async IO](#3-io-models-and-async-io) |
| Filesystem | names, inodes, page cache, disks | [File Systems](#4-file-systems) |
| Network stack | how packets enter/leave this host | [Network Stack](#5-network-stack-kernel-path) |
| Namespace/Cgroup | container views and quotas | [Namespace and Cgroup](#6-namespace-and-cgroup-resource-isolation) |
| System call | user mode requests kernel services | [System Calls and Performance](#7-system-calls-and-performance) |
| Kernel mechanisms | interrupts, locks, preemption, D state | [Linux Kernel Topics](#8-linux-kernel-topics) |

### 0.2 User Mode and Kernel Mode

| Dimension | User Mode | Kernel Mode |
|-----------|-----------|-------------|
| Who runs | application processes | kernel, drivers, interrupt handlers |
| Privilege | restricted | full hardware / all APIs |
| Typical work | business logic, GC, serialization | syscalls, scheduling, networking, FS |
| Failure blast radius | usually one process | may take down the machine |

Common paths into kernel mode: syscall, page fault, hardware interrupt, signal/trap/exception — then return to user mode.

### 0.3 System Calls as the Boundary

A `read()` is not a normal function call: privilege transition, fd/buffer validation, VFS/device work, then return.

| Syscall | Scenario |
|---------|----------|
| `openat/read/write/close` | file IO |
| `socket/connect/accept/sendto/recvfrom` | network IO |
| `epoll_wait` | IO multiplexing |
| `clone/fork/execve/wait4` | process/thread lifecycle |
| `mmap/munmap/brk` | mappings and heap growth |
| `futex` | user-space lock slow path |

```bash
strace -c -p <PID>
strace -tt -T -p <PID> -e trace=network,file,process
perf trace -p <PID>
```

### 0.4 From Symptoms to Subsystems

| Question | Signal | Commands |
|----------|--------|----------|
| user vs kernel CPU? | `us/sy/wa/si` | `top`, `pidstat -u -p <PID> 1` |
| running / sleep / D / zombie? | `STAT`, `wchan` | `ps -p <PID> -o pid,ppid,stat,wchan:32,cmd` |
| too many syscalls? | count/latency | `strace -c`, `perf trace` |
| leak or OOM? | RSS, page cache, cgroup | `free -h`, `pmap -x`, `smaps`, `dmesg -T` |
| slow disk? | await, util, queue | `iostat -x 1`, `pidstat -d`, `iotop` |
| bad network? | state, retransmit, queues | `ss -s`, `ss -lntp`, `netstat -s`, `tcpdump` |
| container throttle? | quota/limit/pressure | cgroup files, `kubectl top` |

```bash
# us: user-mode; sy: kernel-mode; wa: IO wait; hi/si: hard/soft IRQ (si often NIC RX)
ss -lntp       # preferred listen check on modern Linux
netstat -lntp  # older hosts / interviews
ss -s; netstat -s; tcpdump -i eth0 port 8080 -nn
```

### 0.5 Learning Order

```text
1. user/kernel mode, syscall cost
2. process/thread/scheduling (CFS → EEVDF note) / context switch (μs)
3. virtual memory, reclaim chain, page cache, page fault
4. epoll/io_uring and zero-copy numbers
5. host RX/TX path; protocol detail → 06
6. Namespace + Cgroup
7. interrupt/softirq, kernel locks, preemption, D state
8. map top/vmstat/strace/perf back to modules
```

---

## 1. Processes and Threads

### 1.1 Process vs Thread

Linux schedules **tasks**. Process = isolation/resource boundary; thread = scheduling unit sharing that address space.

| Dimension | Process | Thread |
|-----------|---------|--------|
| Definition | resource allocation unit | CPU scheduling unit |
| Memory | private address space | shared address space |
| Communication | IPC | shared globals (+ sync) |
| **Create** cost | higher (tens of μs–ms; depends on working set/fds) | lower (often a few μs) |
| **Switch** cost | same-CPU process switch ~ few μs (+ CR3/TLB) | same-process thread ~ **1–3 μs** (no page-table switch) |
| Failure | crash usually stays in-process | one bad thread can take the process |

> **Canonical numbers**: context **switch** is measured in **μs**. **Create** is heavier and workload-dependent. Never merge “create cost” and “switch cost” into one sentence.

### 1.2 Process States

```text
create (fork) → ready → running ⇄ blocked (IO/lock) → terminated
```

```bash
ps -eo pid,ppid,stat,wchan:32,cmd | head
# R runnable; S interruptible sleep; D uninterruptible (often IO);
# Z zombie; T stopped
pidstat -u -r -d -p <PID> 1
```

### 1.3 Scheduling: CFS and EEVDF Note

For a long time the default fair class was **CFS**: `vruntime` tracks who is owed CPU.

```text
vruntime += delta_exec * (NICE_0_WEIGHT / weight)
# lower nice → higher weight → vruntime grows slower → scheduled sooner
```

**Evolution**: Linux **6.6+** gradually replaces the fair-class CFS story with **EEVDF** (Earliest Eligible Virtual Deadline First)—still fairness-oriented, with better latency for lagging tasks.

Interview answer:

- Explain CFS/vruntime for classic questions / older kernels.
- Add: mainline fair class moved toward EEVDF; observe the kernel you actually run.

```bash
uname -r
# /sys/kernel/debug/sched* (debugfs), perf sched
nice -n 10 ./myapp; renice -n 5 -p 1234
chrt -r -p 50 1234   # RT class when needed
```

### 1.4 IPC (Compact)

- **Pipe / FIFO**: related processes or named FIFO (`mkfifo`).
- **Shared memory**: fastest; needs sync. `IPC_PRIVATE` means “create a new unnamed SysV segment”, **not** a magic key for arbitrary processes—child inherits `shmid` via `fork`, or use `ftok` / POSIX `shm_open`.
- **Signals**: `SIGTERM` graceful, `SIGKILL` uncatchable; Go: `signal.Notify`.
- Also: sockets, message queues, `mmap`.

### 1.5 Thread Models: User Threads vs Go M:N

| Type | Implementation | Pros | Cons |
|------|----------------|------|------|
| Pure user-level (N:1) | user scheduler; kernel sees one thread | cheap create/switch | **one blocking syscall stalls all**; no multi-core |
| Kernel threads (1:1) | pthread / kernel schedule | true multi-core | higher create/switch cost |
| Hybrid (M:N) | Go G:P:M | light tasks + multi-core | complex runtime (block, preempt, netpoll) |

> **Pitfall**: do **not** equate “user-level threads” with “Go runtime”. Go is **M:N**. “Cannot use multiple cores” applies to classic N:1, **not** to goroutines. Also: goroutines are **not** “multi-process safe”—they share one address space; a crash still kills the process. Isolation is weaker than multi-process.

```text
G (goroutine, ~2KB stack) → scheduled on P → bound to M (OS thread)
Work-stealing: idle P steals half of a busy P's run queue
```

### 1.6 Zombies and Orphans

- **Zombie**: child exited; parent never `wait`ed → PID slot held. Fix parent, or kill parent so init reaps.
- **Orphan**: parent died; init/subreaper adopts—**normal**, not a leak by itself.

### 1.7 Context Switch (μs)

```text
Same-CPU process switch (~ few μs typical):
  save regs/PC/SP → switch CR3 (TLB flush without PCID) → restore new process

Same-process thread switch (~ 1–3 μs typical):
  shared mm; no page-table switch; regs/stack + scheduler

Create (fork/clone) ≠ context switch. Create is a one-shot heavier operation.
```

```bash
vmstat 1          # cs = context switches/s
pidstat -w -p <PID> 1
# cswch/s voluntary (wait IO/lock); nvcswch/s involuntary (preempt/timeslice)
```

Reduce: fewer threads/goroutines, less lock churn, batch work, avoid busy-wait.

---

## 2. Memory Management

### 2.1 Virtual Memory

Why: isolation, overcommit/swap expansion, sharing (e.g. libc pages).

```bash
cat /proc/<PID>/maps   # code, data, heap, stack, libs
```

### 2.2 Page Table and TLB

x86_64: 4-level walk (PGD→PUD→PMD→PTE) + 12-bit offset. TLB caches translations; miss → page walk.

```bash
perf stat -e dTLB-load-misses,dTLB-loads ./myapp
```

Prefer sequential access; PCID reduces full TLB flush on CR3 switch.

### 2.3 Page Fault

- **Minor**: page already in RAM; mapping needs fix-up.
- **Major**: must read storage/swap.

Page fault is an **exception**, not a syscall. `strace -c` will **not** show a `page_fault` row.

```bash
ps -o pid,min_flt,maj_flt,comm -p <PID>
perf stat -e page-faults,minor-faults,major-faults ./myapp
sar -B 1 5
```

### 2.4 fork, COW, mmap

`fork` is fast because of **Copy-On-Write**: share pages until a write. `mmap` often only creates a VMA; first touch faults in pages. Heavy writes by both parent and child turn COW into a fault storm. Static file → socket: prefer `sendfile` over mythic mmap-as-zero-copy.

### 2.5 Reclaim Chain: Watermark → kswapd → Direct Reclaim → Swap → OOM

```text
Memory pressure / below low watermark
  ├─ async: wake kswapd (reclaim file pages, inactive anon)
  └─ urgent: direct reclaim on allocation path (hurts latency)
        ├─ clean file pages first; dirty may writeback
        ├─ anon → swap if enabled
        └─ still short → OOM Killer (oom_score)
```

```bash
grep -E 'low|min|high' /proc/zoneinfo | head
grep -E 'pgsteal|pgscan|oom_kill' /proc/vmstat
cat /proc/<PID>/oom_score /proc/<PID>/oom_score_adj
vmstat 1 5   # sustained si/so → memory pressure
```

Cgroup `memory.max` can OOM a group while the host still has free RAM.

### 2.6 Transparent Huge Pages (THP)

Huge pages cut page-table depth / TLB misses; may add compact/split latency.

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# always / madvise / never — latency-sensitive services often madvise or never
grep -E 'thp_|compact_' /proc/vmstat
```

Measure for DB/JVM large heaps: throughput up, tail latency may jitter.

### 2.7 CPU Cache, MESI, Barriers

```text
core → L1 → L2 → L3 (shared) → DRAM; cache line usually 64B
```

**False sharing**: two threads write different fields on the same line → MESI Invalid ping-pong. Pad to line size. **MESI**: Modified / Exclusive / Shared / Invalid. Barriers constrain reorder; language atomics/locks usually suffice—rarely hand-roll fences in Go.

```bash
perf stat -e cache-references,cache-misses ./myapp
```

### 2.8 Allocators

- **Buddy**: power-of-two blocks (`/proc/buddyinfo`).
- **Thread-cache lineage** (TCMalloc idea, jemalloc, Go mcache/mcentral/mheap): per-thread fast path → central → page heap. Go has its **own** allocator—not “Go uses TCMalloc”.

### 2.9 Leak Diagnosis

Go: pprof heap diff + goroutine count. C/C++: Valgrind/ASan. Always compare two snapshots; one RSS sample is weak.

```bash
go tool pprof -base heap1.prof heap2.prof
valgrind --leak-check=full ./myapp
```

---

## 3. IO Models and Async IO

### 3.1 Five Models

Blocking / non-blocking / multiplexing (select/epoll) / signal-driven / async (kernel completes then notifies). epoll is usually **sync multiplex + non-blocking read/write**, not the same as io_uring/AIO.

### 3.2 epoll vs select: Complexity and Event Delivery

| Property | select / poll | epoll |
|----------|---------------|-------|
| fd scale | select often `FD_SETSIZE` (~1024); poll linear | practical limit = system fd limit |
| add/mod | resubmit full set | `epoll_ctl`: RB-tree interest set, **O(log N)** |
| wait return | scan all interested fds, **O(N)** | return ready only, **O(R)** |
| user↔kernel | copy fd sets each call | ready events copied to user buffer; **not** an mmap'd shared event table |
| fit | few fds, portability | high concurrency (C10K+) |

> **Myth busted**: epoll is **not** “O(1) + shared-memory events”. Accurate: manage interest set **O(log N)**; return ready **O(R)**; delivery is copy-to-user style, **not** mmap of the ready queue.

### 3.3 epoll Modes: LT / ET / ONESHOT, Thundering Herd, Go netpoller

| Mode | Behavior | Caveat |
|------|----------|--------|
| **LT** (default) | keep reporting while readable/writable | easy; forgotten drain still retriggers |
| **ET** | notify on state change | must nonblock + loop until `EAGAIN` |
| **EPOLLONESHOT** | disarm after one fire | avoids multi-worker races; re-arm with `epoll_ctl` |

**Accept thundering herd**: many waiters on one listen fd wake; only one `accept` wins. Mitigations: `SO_REUSEPORT`, kernel exclusive wakeups in some paths—still avoid blind multi-process stampede on one listen.

Go netpoller (Linux): goroutine blocks on `conn.Read` → G parks; M runs other Gs; epoll readiness requeues G. Business code rarely picks LT/ET; C libraries must.

### 3.4 io_uring

SQ/CQ rings batch async ops; fewer “one syscall per op” round-trips. Great for high-IOPS storage; harder ops/kernel requirements. epoll waits for readiness then sync IO; io_uring asyncs the IO itself. Go ecosystem still mostly netpoller+epoll.

### 3.5 Zero Copy

#### Traditional `read` + `write`: **4 copies + 4 context switches**

Two syscalls; each has enter+return → **4 user/kernel transitions**. Data path: **4 copies** (2× DMA + 2× CPU):

```text
read:  disk --DMA--> page cache --CPU--> user buf     (enter + return)
write: user buf --CPU--> socket buf --DMA--> NIC      (enter + return)
```

> Older slides said “4 copies, 2 switches” and undercounted returns. Interview answer for this path: **4 copies + 4 switches**.

#### `sendfile` (~2 copies)

Kernel moves file pages toward the socket (DMA gather possible); no user buffer bounce.

#### mmap vs Nginx static path

`mmap` maps pages; faults still happen; may avoid one explicit `read` but is **not** the same claim as “Nginx zero-copies via mmap”.

**Nginx** static main path: **`sendfile on`** (+ `tcp_nopush` etc.), not “mmap as the primary zero-copy story”.

```nginx
location /static/ {
    sendfile on;
    tcp_nopush on;
}
```

### 3.6 Direct IO

`O_DIRECT` bypasses page cache; alignment/size constraints. Use when the app owns cache (DB) or for large streaming that should not pollute cache.

---

## 4. File Systems

### 4.1 inode and Links

inode holds metadata + block pointers; directory maps name → inode. Hard link: same inode; symlink: new inode storing path.

```bash
ls -li /etc/hosts; stat /etc/hosts
```

### 4.2 ext4 Layout (Sketch)

Superblock → group descriptors → inode/block bitmaps → inode table → data blocks. Journal for crash consistency.

```bash
dumpe2fs /dev/sda1 | head -20
```

### 4.3 Buffering

stdio buffer → `write` → **page cache** → later writeback. `fsync`/`fdatasync` force durability (latency trade-off).

```bash
free -h   # buff/cache; MemAvailable is the real headroom
# drop_caches for tests only
```

### 4.4 Performance Habits

Batch writes (fewer syscalls); avoid per-record `fsync`; async/periodic sync for logs; watch util/await/queue and random vs sequential patterns.

---

## 5. Network Stack (Kernel Path)

> **Scope of this chapter**: how packets enter/leave **this host**—socket, softirq, NAPI, queues, host knobs.
> TCP/HTTP/TLS/congestion detail, packet decode, protocol choice → [`06-Networking-Fundamentals-and-Protocols.md`](./06-Networking-Fundamentals-and-Protocols.md). Do not memorize two handshake novels.

### 5.1 RX/TX with softirq / NAPI

Hard IRQ rings the bell; bulk work runs in **softirq** (Linux bottom half)—not the same thing as x86 `int` “software interrupt”.

```text
[RX] NIC DMA → ring → hard IRQ (short) → NET_RX softirq
     → NAPI poll (batch, mitigate interrupt storm)
     → IP/TCP → socket recv queue → wake recv/epoll waiters

[TX] send → socket send queue → stack → qdisc (e.g. fq_codel)
     → driver TX ring → DMA → often NET_TX softirq completion
```

**NAPI**: under load, switch from per-packet IRQ to polling batches; idle falls back to interrupts.

```bash
cat /proc/interrupts | grep -i eth
cat /proc/softirqs          # NET_RX / NET_TX
mpstat -P ALL 1             # rising si% often RX softirq
ss -s; nstat -az | grep -E 'Tcp|Ip|Udp|TcpExt'
```

### 5.2 Host Queues: SYN / Accept Backlog

Handshake state machines live in 06; host triage starts with queue overflow:

```bash
ss -lnt          # Recv-Q backlog depth vs Send-Q listen backlog
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
# nstat/netstat -s: ListenOverflows, ListenDrops, SYNFIFO
```

### 5.3 Initial Congestion Window

Textbooks often draw `cwnd=1`. **Modern Linux default initcwnd is about 10 MSS** (RFC 6928-style practice). Do not answer “always start at 1”.

```bash
ss -i    # ... cubic ... cwnd:10 ...
sysctl net.ipv4.tcp_congestion_control
# Cubic vs BBR comparison/tuning → 06
```

### 5.4 Socket Knobs (Host Side)

`SO_REUSEADDR`/`SO_REUSEPORT`, `TCP_NODELAY`, keepalive, buffer sizes—tune with evidence; oversized buffers can raise latency/memory.

---

## 6. Namespace and Cgroup Resource Isolation

> Container ≈ **Namespace (what you see)** + **Cgroup (how much you may use)**.

### 6.1 Namespace

| Namespace | Isolates | Typical effect |
|-----------|----------|----------------|
| **pid** | PID view | container PID 1 ≠ host init |
| **mnt** | mounts | separate root FS |
| **net** | NICs, routes, iptables, sockets | own eth0/ports |
| **uts** | hostname | different hostname |
| **ipc** | SysV IPC / POSIX MQ | no cross-talk |
| **user** | UID/GID map | container root → host unprivileged |
| **cgroup** | cgroup root view | virtualized hierarchy |

```bash
lsns; ls -l /proc/<PID>/ns
nsenter -t <PID> -n ss -lntp
```

`--net=host` / `hostPID` deliberately weaken isolation for performance or observability.

### 6.2 Cgroup Overview

| Subsystem | Controls | Notes |
|-----------|----------|-------|
| CPU | time, cpuset | quota vs shares |
| Memory | RSS+cache limits, OOM | soft/hard |
| BlkIO / io | bandwidth, IOPS | v1 buffered IO often weak |
| Net | via net_cls + tc | not a simple single file |

```bash
mount | grep cgroup    # v1 separate mounts
mount | grep cgroup2   # unified hierarchy
```

### 6.3 CPU Isolation

- **cpuset**: pin cores (watch hyperthread sibling pairs).
- **quota** (`cpu.cfs_quota_us` / v2 `cpu.max`): hard cap even if idle cores exist.
- **shares/weight**: proportional under contention; can borrow when idle.

Throttling can create high latency with “low average CPU”.

### 6.4 Memory Isolation

Hard/soft limits; memsw; `oom_control` / `memory.oom.group`. Host reclaim chain is §2.5; here only **group limits**. Group OOM can fire while host `MemAvailable` looks fine.

### 6.5 IO Isolation and Schedulers

```text
app → VFS → FS → page cache → block layer (throttle) → IO scheduler → driver → device
```

| Scheduler | Era | Fit |
|-----------|-----|-----|
| **none** | multi-queue | NVMe/SSD common default |
| **mq-deadline** | multi-queue | general / mixed / many HDD |
| **bfq** | multi-queue | desktop / shared HDD latency |
| cfq / deadline / noop | **legacy single-queue** | old kernels/devices only |

```bash
cat /sys/block/nvme0n1/queue/scheduler
# modern: [none] mq-deadline bfq
# legacy: noop deadline [cfq]
```

Do **not** recite cfq as today's default truth. v1 `blkio.weight` often **does nothing** on mq+none; prefer throttle / v2 `io.max` / `io.weight`. v1 buffered IO throttle is frequently ineffective; v2 improves this.

### 6.6 Network Isolation

`net_cls` classid + `tc` (HTB / fq_codel). fq_codel fights bufferbloat by controlling sojourn time, not only queue length.

### 6.7 Cgroup v2 and PSI

Unified hierarchy; better buffered IO control; **PSI** (`cpu.pressure` / `memory.pressure` / `io.pressure`): `some` = at least one task stalled; `full` = all tasks stalled (no cpu.full).

```bash
echo '+cpu +memory +io +pids' > /sys/fs/cgroup/cgroup.subtree_control
echo '50000 100000' > /sys/fs/cgroup/mygroup/cpu.max
echo 512M > /sys/fs/cgroup/mygroup/memory.max
cat /sys/fs/cgroup/mygroup/memory.pressure
```

---

## 7. System Calls and Performance

A syscall is a **privilege round-trip + kernel path**, not a “slow function”. Split cost, then batch, vDSO, or user-space bypass.

### 7.1 Cost Breakdown

```text
prepare args → trap to kernel → save/restore → validate
  → real work (VFS/stack/schedule/sleep) → return → I-cache/TLB noise
```

| Cost piece | Intuition | Note |
|------------|-----------|------|
| bare trap round-trip | tens–hundreds of ns | empty syscall microbench |
| validate/lookup | path-dependent | fd → file |
| real work | μs–ms | disk/net/locks dominate |
| sleep/schedule | unbounded | wait ≫ switch itself |

Do not memorize “syscall = 100ns”—that is empty-path scale. Production slow `read` is usually waiting.

### 7.2 How to Measure

```bash
strace -c -p <PID>                 # syscalls only; not faults/softirq
strace -tt -T -p <PID> -e trace=network,file,desc
perf trace -p <PID>
perf stat -e raw_syscalls:sys_enter ./myapp
pidstat -u -p <PID> 1              # high %system + hot syscalls → kernel path
```

### 7.3 Batch IO

Aggregate user buffers; `writev`/`readv`; `sendmmsg`/`recvmmsg`; epoll returns many events; storage may use io_uring batch submit.

### 7.4 vDSO

Kernel maps a read-only page; libc implements some “syscalls” in user space (`gettimeofday`, `clock_gettime`, often `getcpu`).

```bash
ldd /bin/ls | grep vdso
cat /proc/self/maps | grep vdso
```

### 7.5 futex

Uncontended mutex/channel stays in user CAS; contention sleeps in **futex**—common Go/pthread kernel exit.

```bash
strace -c -e futex -p <PID>
perf top -p <PID>   # look for futex_wait
```

Shrink critical sections, shard locks, avoid broadcast wake storms.

### 7.6 Optimization Checklist

1. Measure: count problem vs wait problem.
2. Batch.
3. Prefer vDSO-friendly time APIs.
4. Treat futex as lock contention, not “mystery IO”.
5. Only then consider io_uring / XDP / DPDK (ops cost).

---

## 8. Linux Kernel Topics

### 8.1 Address Types

Logical (selector:offset) → linear/virtual → physical via paging. On Linux x86_64 flat model, segment bases are 0 → logical ≈ virtual; **paging** does the real work. Process switch loads CR3; without PCID, TLB flushes.

### 8.2 Interrupts, Exceptions, softirq

| Kind | Source | Examples |
|------|--------|----------|
| Hardware IRQ | devices | NIC, disk, timer |
| **softirq** | Linux bottom half | `NET_RX_SOFTIRQ`, `TIMER_SOFTIRQ` |
| Exception | CPU instruction | `#PF`, divide-by-zero |

> **Terminology trap**: textbook “software interrupt” (`int n`) ≠ Linux **softirq**. In this doc “soft interrupt” means softirq bottom half.

Flow: device → APIC → CPU → IDT → **top half** (short, IRQs off) → **bottom half** (softirq / tasklet / workqueue). softirq/tasklet: no sleep; workqueue: can sleep.

```bash
cat /proc/interrupts
cat /proc/softirqs
```

### 8.3 Kernel Locks

| Lock | Sleep? | Use |
|------|--------|-----|
| spinlock | no (spin) | short, IRQ-safe variants |
| mutex | yes | process context, longer |
| rwlock / rwsem | no / yes | read-heavy |
| RCU | read almost free | read-mostly structures |

Never sleep while holding a spinlock. Never use mutex in hardirq context.

### 8.4 Kernel Preemption

| Model | Config | Bias |
|-------|--------|------|
| None | `PREEMPT_NONE` | throughput |
| Voluntary | `PREEMPT_VOLUNTARY` | common general/server |
| Preempt / dynamic | `PREEMPT*` / `PREEMPT_DYNAMIC` | desktop / low latency |

**Myth**: “server kernels are always `PREEMPT_NONE`”—false. Check the running kernel:

```bash
grep -E '^CONFIG_PREEMPT' /boot/config-$(uname -r)
uname -v
```

### 8.5 Soft Lockup vs Hard Lockup

- **Soft lockup**: CPU stuck in kernel without scheduling (~20s default); timer + watchdog thread.
- **Hard lockup**: IRQs also stuck (often IRQ-off spin); **NMI** watchdog.

```bash
dmesg | grep -i lockup
cat /proc/sys/kernel/nmi_watchdog
```

### 8.6 D State and Load Average

**D** = `TASK_UNINTERRUPTIBLE` (disk/NFS/kernel wait)—`kill -9` often ineffective until wait ends. User-space mutex wait is usually **S** + futex, **not** D.

Linux load average ≈ EMA of **R + D** tasks—so high load + idle CPU often means IO/NFS D, not “need more cores”.

```bash
ps aux | awk '$8 ~ /D/'
cat /proc/<pid>/stack
uptime; cat /proc/loadavg
```

### 8.7 Syscall Calling Convention (x86_64)

`syscall` instruction; args: rdi, rsi, rdx, **r10**, r8, r9 (4th is r10 because `syscall` clobbers rcx with return RIP). Number in rax.

---

## 9. Production Troubleshooting SOP

Pattern: confirm symptom → narrow dimension → root cause → fix + verify.

### 9.1 CPU 100%

```bash
top; pidstat -u -p <PID> 1 5
# us high → app/runtime profile (Go pprof / Java jstack + top -Hp)
# sy high → perf top -g; strace -c; softirqs (NET_RX storm, short connections)
# si high → NIC softirq / RSS-RPS
```

Go: CPU profile + goroutine dump + GC. Java: TID→hex `nid=` match, multi-sample stacks. Kernel: `perf record -g`, hot syscalls, softirq distribution.

### 9.2 OOM

```bash
dmesg -T | grep -iE 'oom|killed process'
free -h; cat /proc/meminfo   # trust MemAvailable, not MemFree alone
ps aux --sort=-%mem | head
# cgroup memory.events / memory.max
# Go heap diff; Java heap dump (careful STW); smaps / slabtop / vmstat si-so
```

Preserve evidence before mass restart. Distinguish leak vs cache vs traffic spike vs bad limit. Protect critical PIDs with `oom_score_adj`; prefer cgroup caps over “more swap forever”.

### 9.3 Slow Disk IO

```bash
iostat -x 1 5     # util, await, avgqu-sz, merge rates
iotop -oP; pidstat -d 1
df -h; df -i; lsof | grep deleted
cat /sys/block/<dev>/queue/scheduler   # none / mq-deadline / bfq
```

High `wa` → sync logging, random DB IO, swap thrash. Full disk / inode exhaustion / FS remounted RO are separate FS failures.

---

## 10. Interview Self-Check

> Dual track: **S** = quick oral; **Q** = deep follow-up. Format: **points → pitfalls → observe → follow-ups**. Details live in the body.

### Quick Track (S)

#### S1: User vs kernel mode; why syscalls feel “slow”?

**Points**: privilege levels; trap + validate + kernel path + return; may sleep.
**Pitfalls**: blaming “complex function” only; ignoring wait/cache/TLB.
**Observe**: `top`/`pidstat` `us/sy/wa/si`; `strace -c`; `perf trace`.
**Follow-up**: high `sy` → hot syscall + kernel stack; high `wa` → IO wait, not lack of cores.

#### S2: Process vs thread; create vs switch?

**Points**: isolation boundary vs sched entity; switch in **μs**; create heavier.
**Pitfalls**: “process switch 3–5 ms”; mixing create with switch.
**Observe**: `pidstat -w`; `vmstat` `cs`.
**Follow-up**: why same-process thread switch is lighter? → usually no CR3/TLB shootdown.

#### S3: Page cache?

**Points**: file pages in RAM; hits never touch disk.
**Pitfalls**: microbench without cold cache.
**Observe**: `free -h`; `Cached` in meminfo.
**Follow-up**: dirty + `fsync` latency spikes → writeback/device queue.

#### S4: mmap vs read/write?

**Points**: explicit copy vs map+fault.
**Pitfalls**: “mmap always faster”.
**Observe**: minflt/majflt; `perf` page-faults.
**Follow-up**: static file to socket → often **sendfile**.

#### S5: Blocking/nonblocking vs sync/async?

**Points**: hang-or-not vs who drives completion. epoll ≈ sync multiplex.
**Pitfalls**: calling epoll “async IO” like io_uring.
**Observe**: `strace -e epoll_wait,read,write`.
**Follow-up**: does Go `conn.Read` block an OS thread? → blocks G; M can run others.

#### S6: Triage command chain?

**Points**: pick resource dimension first.
**Pitfalls**: jump to `tcpdump` or only `%CPU`.
**Observe**: ps/top → vmstat/iostat → ss/nstat → strace/perf → cgroup/dmesg.
**Follow-up**: low CPU, high latency → queue, futex, IO, faults, retransmit, steal.

#### S7: fd leak?

**Points**: `EMFILE`, connect/open failures.
**Pitfalls**: restart without finding leak.
**Observe**: `ls /proc/<PID>/fd | wc -l`; `lsof`; `ulimit -n`; container nofile/pids.

#### S8: Why fork looks fast?

**Points**: COW.
**Pitfalls**: thinking full address-space memcpy.
**Observe**: post-fork write storm → min_flt spike.

#### S9: User mutex wait → D state?

**Points**: usually S + futex; D is uninterruptible kernel wait.
**Pitfalls**: calling lock wait “D”.
**Observe**: `ps -o stat,wchan`; `/proc/<PID>/stack`.

#### S10: Slow with low CPU?

**Points**: latency from queue/wait.
**Observe**: vmstat r/b/cs; pidstat -w; iostat; `ss -i`; mpstat steal.

---

### Deep Track (Q)

#### Q1: Process/thread/goroutine choice?

**Points**: isolation vs cheap sharing; goroutine = **M:N**, not multi-process safety.
**Pitfalls**: “goroutines give multi-process safety”.
**Observe**: NumGoroutine/pprof; `ps -eLf`; fault domains of multi-process.
**Follow-up**: why N:1 cannot multi-core? why Go can? → many Ms.

#### Q2: Zombie vs orphan?

**Points**: unreaped exit vs adopted child. Orphans are normal.
**Observe**: `ps` Z; `pstree -p`.

#### Q3: Virtual memory / TLB?

**Points**: multi-level walk + TLB; process switch TLB cost; PCID helps.

#### Q4: Minor vs major fault; `strace -c`?

**Points**: fault ≠ syscall; no `page_fault` in strace.
**Observe**: `ps` min/maj flt; `perf` faults; `sar -B`.

#### Q5: epoll scalability—O(1)+mmap?

**Points**: `epoll_ctl` **O(log N)**; wait **O(R)**; events **copied**, not mmap shared table.
**Pitfalls**: “shared memory O(1)”.
**Follow-up**: LT vs ET vs ONESHOT; thundering herd.

#### Q6: Zero copy counts?

**Points**: read+write → **4 copies + 4 switches**; sendfile/splice reduce user bounce.
**Pitfalls**: “4 copies 2 switches”; “Nginx zeros with mmap”—main path is **sendfile**.

#### Q7: False sharing?

**Points**: same cache line, different fields; MESI ping-pong ≠ data race.

#### Q8: Three-way handshake? (point to 06)

**Points**: mutual capability + old SYN mitigation; this doc watches backlog.
**Pitfalls**: writing congestion essays in the OS chapter. Detail → **06**.

#### Q9: Context switch reduce?

**Points**: μs units; goroutine schedule ≠ kernel thread switch.
**Observe**: vmstat `cs`; voluntary vs involuntary.

#### Q10: Cgroup CPU three knobs?

**Points**: cpuset / quota / shares. Shares idle-borrow. Namespace = view; cgroup = quota.

#### Q11: Modern IO schedulers?

**Points**: **none / mq-deadline / bfq**; cfq/noop are legacy narrative.
**Observe**: `/sys/block/.../scheduler`.

#### Q12: CFS vs EEVDF?

**Points**: vruntime fairness → 6.6+ EEVDF. Don't swear “always CFS”.

#### Q13: mmap / sendfile / read-write choice?

**Points**: general IO; file→socket prefer sendfile; random/shared consider mmap. DB often Direct IO.

#### Q14: OOM and reclaim chain?

**Points**: watermark → kswapd/direct reclaim → swap → OOM (+ cgroup limit).
**Observe**: dmesg OOM; oom_score; zoneinfo; si/so.

#### Q15: GMP design?

**Points**: G light, M executes, P holds queues/mcache; work-stealing. Not multi-process isolation.

#### Q16: Direct IO when?

**Points**: app-managed cache; alignment rules; hurts small random reads needing page cache.

#### Q17: initcwnd still 1?

**Points**: modern Linux ≈ **10 MSS**. Algorithms → 06.

#### Q18: Memory leak Go/C?

**Points**: heap diff + goroutine; Valgrind/ASan. Don't trust one RSS sample.

#### Q19: SIGTERM graceful exit?

**Points**: stop admit → drain → persist → exit; SIGKILL last resort; K8s grace period.

#### Q20: Cgroup v2 vs v1?

**Points**: unified tree, better buffered IO, PSI, simpler knobs.

#### Q21: Soft vs hard lockup?

**Points**: no schedule vs no IRQ; NMI for hard. ≠ softirq storm (busy bottom half).

#### Q22: Kernel preemption—always NONE on servers?

**Points**: check config; distributions differ; dynamic preempt exists.

#### Q23: D state and load?

**Points**: load includes D; don't add CPUs for NFS D storms.

#### Q24: Top/bottom half; is softirq “software interrupt”?

**Points**: softirq ≠ x86 software interrupt. RX: hardIRQ → NET_RX softirq → NAPI → stack → socket.

#### Q25: Kernel locks / RCU intuition?

**Points**: spin short; mutex sleepable; RCU read-mostly with deferred free.

#### Q26: Namespaces list?

**Points**: pid/mnt/net/uts/ipc/user/cgroup; Docker is ns+cgroup, not cgroup alone. `hostNetwork` drops net ns.

#### Q27: Reclaim chain narrative?

**Points**: don't jump straight to OOM; direct reclaim hurts tail latency.

#### Q28: futex in perf?

**Points**: user lock escalation to kernel wait; shrink critical sections / shard / avoid broadcast.

---

### Open Design (D)

#### D1: User-space network stack (kernel bypass)?

**Points**: UIO/VFIO + poll rings, huge pages, CPU isolation; lose iptables/tcpdump ecosystem. When XDP/DPDK vs kernel+io_uring—PPS vs team cost.

#### D2: SSH/Ping dead, power on?

**Points**: BMC/IPMI → panic/lockup; kdump/vmcore; don't wait on the dead SSH session.

---

## Summary

1. **Process/thread**: create vs switch (**μs**), CFS/EEVDF note, IPC (`IPC_PRIVATE`+fork), GMP (**not** multi-process safety).
2. **Memory**: faults ≠ syscalls, reclaim chain, COW/mmap, THP, MESI, allocator ideas.
3. **IO**: epoll **O(log N)/O(R)** (not mmap events), LT/ET/ONESHOT/herd/io_uring, zero copy **4+4**, Nginx **sendfile**.
4. **Filesystem**: inode, page cache, fsync.
5. **Network (this doc)**: softirq/NAPI path; protocol detail → **06** (**initcwnd ≈ 10**).
6. **Isolation**: seven namespaces + cgroup quotas; v2/PSI; IO schedulers **none/mq-deadline/bfq**.
7. **Syscall performance**: cost, measure, batch, vDSO, futex.
8. **Kernel**: softirq ≠ software interrupt insn; locks; preemption from **running config**; D/load; lockup.

**Optimization order**: observe → cut wait / cut syscall count → then zero-copy and bypass.
