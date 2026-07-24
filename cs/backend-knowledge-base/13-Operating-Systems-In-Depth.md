# Operating Systems In Depth

Language: English | [中文](../后端知识库/13-操作系统深入.md)

---

## Table of Contents

0. [Operating System Big Picture](#0-operating-system-big-picture)
1. [Processes and Threads](#1-processes-and-threads)
2. [Memory Management](#2-memory-management)
3. [IO Models and Async IO](#3-io-models-and-async-io)
4. [File Systems](#4-file-systems)
5. [Network Stack](#5-network-stack)
6. [Cgroup Resource Isolation](#6-cgroup-resource-isolation)
7. [System Calls and Performance](#7-system-calls-and-performance)
8. [Linux Kernel Topics](#8-linux-kernel-topics)
9. [Production Troubleshooting SOP](#9-production-troubleshooting-sop)
10. [Interview Self-Check](#10-interview-self-check)

---

## 0. Operating System Big Picture

An operating system is not a bag of unrelated concepts. It is the control layer between hardware and applications. It solves four core problems:

1. **Abstract hardware**: expose CPU, memory, disks, and NICs as processes, virtual memory, files, and sockets.
2. **Allocate resources**: decide who gets CPU time, memory, IO, and network bandwidth.
3. **Protect and isolate**: prevent processes from corrupting each other's memory or directly controlling hardware.
4. **Expose kernel services**: provide files, networking, process control, time, devices, and memory mappings through system calls.

### 0.1 System Structure

```text
Applications
  |  normal function calls: business logic, serialization, GC, algorithms
  |
  |- syscalls: read/write/open/socket/epoll_wait/futex/mmap
  |
Operating System Kernel
  |- scheduling: processes, threads, context switch, CFS
  |- memory: virtual memory, page table, TLB, page fault, page cache
  |- filesystem: inode, directory, page cache, fsync, block device
  |- networking: socket, TCP/IP, softirq, congestion control
  |- isolation: cgroup, namespace, container limits
  |- drivers: disk, NIC, timer, interrupts
  |
Hardware: CPU / memory / disk / NIC
```

How later sections map to the system:

| Topic | Problem | Section |
|-------|---------|---------|
| Process and thread | how CPU runs many tasks | [Processes and Threads](#1-processes-and-threads) |
| Memory management | how virtual addresses map to physical memory | [Memory Management](#2-memory-management) |
| IO model | how applications wait for file/network IO | [IO Models and Async IO](#3-io-models-and-async-io) |
| Filesystem | how names, inodes, page cache, and disks work together | [File Systems](#4-file-systems) |
| Network stack | how packets move between application and NIC | [Network Stack](#5-network-stack) |
| Cgroup | how containers limit CPU, memory, and IO | [Cgroup Resource Isolation](#6-cgroup-resource-isolation) |
| System call | how user mode requests kernel services | [System Calls and Performance](#7-system-calls-and-performance) |
| Kernel mechanisms | interrupts, locks, preemption, D state | [Linux Kernel Topics](#8-linux-kernel-topics) |

### 0.2 User Mode and Kernel Mode

CPUs use privilege levels to separate applications from the kernel. Applications run in user mode and cannot directly access hardware or kernel memory. The kernel runs in kernel mode and can operate page tables, schedulers, filesystems, devices, and network stacks.

| Dimension | User Mode | Kernel Mode |
|-----------|-----------|-------------|
| Who runs there | normal application code | kernel, drivers, interrupt handlers |
| Privilege | restricted | highest privilege |
| Typical work | business logic, runtime, GC, serialization | syscalls, scheduling, networking, filesystem |
| Failure impact | usually one process | may affect the whole system |

Common transitions into kernel mode:

```text
application code
  |- syscall: read/write/open/socket/epoll_wait
  |- page fault: virtual page not mapped yet
  |- interrupt: NIC, disk, timer
  |- signal/trap/exception
      v
kernel handles it
      v
return to user mode
```

### 0.3 System Calls as the Boundary

A system call is not an ordinary function call. A `read()` switches from user mode to kernel mode, validates fd and buffers, executes VFS/filesystem/device logic, then returns to user mode.

| Syscall | Scenario |
|---------|----------|
| `openat/read/write/close` | file IO |
| `socket/connect/accept/sendto/recvfrom` | network IO |
| `epoll_wait` | IO multiplexing |
| `clone/fork/execve/wait4` | process/thread lifecycle |
| `mmap/munmap/brk` | memory mapping and heap growth |
| `futex` | lock contention after user-space fast path fails |

Observe syscalls:

```bash
strace -c -p <PID>
strace -tt -T -p <PID> -e trace=network,file,process
perf trace -p <PID>
```

### 0.4 From Symptoms to Subsystems

Linux commands should answer a question. First classify the resource dimension, then go to the relevant subsystem.

| Question | Signal | Commands |
|----------|--------|----------|
| user CPU or kernel CPU? | `us/sy/wa/si` | `top`, `pidstat -u -p <PID> 1` |
| process running, sleeping, D state, zombie? | `STAT`, `wchan` | `ps -p <PID> -o pid,ppid,stat,wchan:32,cmd`, `top -Hp <PID>` |
| too many syscalls? | syscall count/latency | `strace -c -p <PID>`, `perf trace -p <PID>` |
| memory leak or OOM? | RSS, page cache, cgroup | `free -h`, `pmap -x <PID>`, `cat /proc/<PID>/smaps`, `dmesg -T` |
| slow disk? | await, util, queue depth | `iostat -x 1`, `pidstat -d -p <PID> 1`, `iotop` |
| abnormal network? | TCP state, retransmit, queues | `ss -s`, `ss -lntp`, `netstat -s`, `tcpdump` |
| container throttling? | cgroup quota/limit/pressure | `cat /sys/fs/cgroup/*`, `kubectl top`, `crictl stats` |

CPU fields:

```bash
top
# us: user-mode CPU, application/runtime/GC
# sy: kernel-mode CPU, syscalls/kernel networking/filesystem/locks
# wa: IO wait
# hi/si: hardware/software interrupt CPU, often network related
```

Network command choice:

```bash
ss -lntp       # listening ports, preferred on modern Linux
netstat -lntp  # common on older machines and interviews
ss -s          # TCP state summary
netstat -s     # protocol statistics: retransmits, resets, failed connects
lsof -iTCP:8080 -sTCP:LISTEN
tcpdump -i eth0 port 8080 -nn
```

### 0.5 Learning Order

For a beginner, follow this path:

```text
1. user mode / kernel mode / system calls
2. process / thread / scheduling / context switch
3. virtual memory / page cache / page fault
4. file IO and IO multiplexing
5. network stack and sockets
6. cgroup / namespace and container resource limits
7. interrupts, kernel locks, preemption, D state
8. map Linux symptoms back to the modules above
```

The rest of the document follows this path: build concepts first, then explain kernel mechanisms, then apply them to production troubleshooting.

---

## 1. Processes and Threads

### 1.1 Process vs Thread

Linux schedules tasks. A process and a thread can be understood as tasks with different resource sharing. In application-level terminology, process is the isolation boundary and thread is the CPU scheduling unit inside a process.

Process:

- Own address space.
- Strong isolation.
- Higher context switch cost.

Thread:

- Shares process address space.
- Lower cost.
- Requires synchronization for shared data.

### 1.2 Process States

Common states:

```text
new -> ready -> running -> waiting -> terminated
```

Linux also has states such as sleeping, uninterruptible sleep, stopped, and zombie.

Commands:

```bash
ps aux | head
ps -eo pid,ppid,stat,wchan:32,cmd | head
ps -p <PID> -o pid,ppid,stat,etime,%cpu,%mem,cmd
top
pidstat -u -r -d -p <PID> 1
```

`STAT` values:

```text
R: running/runnable
S: interruptible sleep
D: uninterruptible sleep, usually kernel IO
Z: zombie
T: stopped/traced
```

### 1.3 Scheduling

Scheduling goals:

- fairness.
- throughput.
- latency.
- CPU utilization.

Common algorithms:

- FCFS.
- Round-robin.
- Priority scheduling.
- CFS in Linux.

### 1.4 IPC

Common IPC mechanisms:

- pipe.
- signal.
- shared memory.
- message queue.
- socket.
- mmap.

Shared memory is fast but needs synchronization.

### 1.5 Context Switching

Context switch cost includes:

- saving/restoring CPU registers.
- switching address space.
- TLB/cache effects.
- scheduler overhead.

High context switching can indicate too many threads, lock contention, or frequent blocking.

### 1.6 CPU Cache and Memory Barriers

Modern CPUs reorder and cache memory operations. Memory barriers constrain ordering and visibility. This is the hardware foundation behind language memory models such as C++ atomics and Java JMM.

---

## 2. Memory Management

### 2.1 Virtual Memory

Virtual memory gives each process an independent address space.

Benefits:

- isolation.
- simplified programming model.
- demand paging.
- memory overcommit.

### 2.2 Page Table and TLB

Page tables map virtual pages to physical frames. TLB caches recent translations.

TLB miss causes page table walk and increases latency.

### 2.3 Page Fault

Page fault happens when a virtual page is not currently mapped or accessible.

Types:

- minor page fault: page exists in memory but mapping needs update.
- major page fault: requires disk IO.

### 2.4 Memory Allocator

Allocators manage heap memory and reduce syscall frequency through arenas, free lists, and caches.

### 2.5 Memory Leak Diagnosis

Workflow:

```text
observe RSS/heap growth -> identify process -> capture heap/profile -> find retaining paths -> verify fix
```

Tools:

- `top`, `ps`, `pmap`.
- `valgrind`, `heaptrack`.
- language-specific profilers.

---

## 3. IO Models and Async IO

### 3.1 Five IO Models

Common IO models:

- blocking IO.
- non-blocking IO.
- IO multiplexing.
- signal-driven IO.
- asynchronous IO.

### 3.2 epoll vs select

`select` scans file descriptor sets and has size limitations. `epoll` uses event notification and scales better with many connections.

Important modes:

- LT: level-triggered.
- ET: edge-triggered.

ET requires draining the socket until `EAGAIN`.

### 3.3 Zero Copy

Zero copy reduces data copies between kernel and user space.

Examples:

- `sendfile`.
- `mmap`.
- `splice`.

It is useful for high-throughput file/network transfer.

### 3.4 Direct IO

Direct IO bypasses page cache. It is useful when the application manages caching itself, such as databases.

---

## 4. File Systems

### 4.1 inode and Directory

inode stores file metadata and block pointers. Directory maps names to inode numbers.

### 4.2 ext4 Layout

Key ideas:

- block groups.
- inode table.
- data blocks.
- journal.

### 4.3 File IO Buffering

Write path may involve:

```text
application buffer -> page cache -> disk scheduler -> device
```

`fsync` forces durability but increases latency.

### 4.4 Performance Optimization

Check:

- IO wait.
- disk utilization.
- queue depth.
- random vs sequential IO.
- page cache hit rate.
- filesystem mount options.

---

## 5. Network Stack

### 5.1 Packet Flow

Simplified receive path:

```text
NIC -> driver -> kernel network stack -> socket receive buffer -> application
```

Send path is the reverse.

Network stack commands:

```bash
netstat -s
ss -s
nstat -az
ip -s link
```

### 5.2 TCP Connection

TCP handshake establishes sequence numbers and connection state.

`TIME_WAIT` ensures delayed packets do not corrupt future connections and allows final ACK retransmission.

Connection inspection:

```bash
ss -lntp
netstat -lntp
ss -antp '( sport = :8080 or dport = :8080 )'
netstat -ant | awk '{print $6}' | sort | uniq -c | sort -nr
lsof -iTCP:8080 -sTCP:LISTEN
tcpdump -i eth0 port 8080 -nn
```

### 5.3 Congestion Control

Congestion control protects network stability. Algorithms include Reno, CUBIC, and BBR.

### 5.4 Socket Tuning

Common parameters:

- backlog.
- receive/send buffer.
- keepalive.
- ephemeral port range.
- `tcp_tw_reuse` depending on kernel/version/context.

Tune only with evidence.

---

## 6. Cgroup Resource Isolation

### 6.1 Overview

cgroups limit and account resources for process groups. Containers rely heavily on namespaces and cgroups.

### 6.2 CPU

Controls:

- shares.
- quota.
- cpuset.

CPU throttling can cause high latency even when average CPU usage looks low.

### 6.3 Memory

Memory cgroups enforce limits. Exceeding limit can trigger OOM kill.

### 6.4 IO and Network

IO and network controls limit device and network resource consumption.

### 6.5 Cgroup v2

Cgroup v2 uses unified hierarchy and more consistent resource control semantics.

---

## 7. System Calls and Performance

System calls switch from user mode to kernel mode.

Cost comes from:

- privilege transition.
- validation.
- scheduler/kernel work.
- cache effects.

vDSO lets some operations avoid full syscall overhead, such as reading time.

Commands:

```bash
strace -c -p <PID>
strace -tt -T -p <PID> -e trace=network,file,process
perf trace -p <PID>
ldd /bin/ls | grep vdso
```

---

## 8. Linux Kernel Topics

### 8.1 Address Types

Concepts:

- virtual address.
- physical address.
- kernel virtual address.
- user virtual address.

### 8.2 Interrupts and Exceptions

Interrupts respond to external events. Exceptions are triggered by CPU execution events such as page faults or divide-by-zero.

### 8.3 Kernel Locks

Kernel uses spinlocks, mutexes, RCU, seqlocks, and atomic operations depending on context.

### 8.4 Kernel Preemption

Preemption improves latency but adds synchronization complexity.

### 8.5 D State and Load Average

Uninterruptible sleep usually indicates waiting for IO. Many D-state processes can raise load average without high CPU usage.

---

## 9. Production Troubleshooting SOP

### 9.1 CPU 100%

```bash
top
pidstat -u -p <pid> 1
perf top
perf record -g -p <pid>
```

For Java/Go/Python, also use runtime profilers and thread dumps.

### 9.2 OOM

Check:

- dmesg OOM logs.
- cgroup memory limit.
- RSS vs heap.
- page cache.
- leak profiles.
- recent traffic/deploy changes.

### 9.3 Slow Disk IO

```bash
iostat -x 1
pidstat -d 1
iotop
```

Look at utilization, await, queue size, and read/write pattern.

---

## 10. Interview Self-Check

### Q0: What is the difference between user mode and kernel mode, and how do you tell which one is consuming CPU?

**Answer:** User mode runs normal application code with restricted privileges. Kernel mode runs the OS kernel, drivers, interrupt handlers, and syscall handlers with full privileges. Use `top` first: high `us` points to application/runtime/GC hot paths; high `sy` points to syscalls, kernel networking, filesystem work, or kernel locks; high `wa` points to IO wait; high `si` often points to network softirq work. Then use `pidstat -u -p <PID> 1`, `strace -c -p <PID>`, and `perf top -g` to narrow down the path.

### Q0.1: Which Linux commands do you commonly use for process, network, and disk troubleshooting?

**Answer:** For processes: `ps aux --sort=-%cpu`, `ps -p <PID> -o pid,ppid,stat,%cpu,%mem,cmd`, `top -Hp <PID>`, `lsof -p <PID>`, `pmap -x <PID>`. For networking: `ss -lntp`, `ss -s`, `netstat -ant`, `netstat -s`, `lsof -iTCP:<PORT>`, `tcpdump -i eth0 port <PORT> -nn`. For disk and system view: `vmstat 1`, `iostat -x 1`, `pidstat -d -p <PID> 1`, `iotop`, `dmesg -T`. The goal is to quickly classify the bottleneck as CPU, memory, disk, network, syscall, or cgroup limit.

### Q1: Process vs thread?

**Answer:** Processes have separate address spaces and stronger isolation. Threads share process memory and are cheaper but require synchronization.

### Q2: What is virtual memory?

**Answer:** Virtual memory maps process-visible addresses to physical memory through page tables, enabling isolation, demand paging, and simpler memory management.

### Q3: What is a page fault?

**Answer:** A page fault occurs when a virtual page is missing or inaccessible. It may be minor if no disk IO is needed, or major if data must be loaded from disk.

### Q4: Why is epoll more scalable than select?

**Answer:** `select` scans descriptor sets. `epoll` maintains interest lists and returns ready events, making it more efficient for many connections.

### Q5: What is zero copy?

**Answer:** Zero copy reduces copying between kernel and user space, improving throughput for file/network transfer.

### Q6: Why does high load average not always mean high CPU?

**Answer:** Load average includes runnable tasks and tasks in uninterruptible sleep. Many IO-blocked D-state tasks can raise load.

### Q7: How do containers isolate resources?

**Answer:** Namespaces isolate views such as PID/network/mount. cgroups limit and account CPU, memory, IO, and other resources.

### Q8: How do you troubleshoot CPU spikes?

**Answer:** Identify process/thread, capture profile or stack, inspect hot functions, correlate with traffic/deploy, then verify with metrics after mitigation.

### Q9: What is page cache?

**Answer:** Page cache stores file data in memory to speed up file IO. It can make free memory look low but is reclaimable.

### Q10: What is `fsync`?

**Answer:** `fsync` flushes file data and metadata needed for durability to storage, trading latency for stronger persistence guarantees.

### Senior Interview Follow-Ups

### Q11: A Linux server shows high load average but low CPU usage. How do you investigate?

**Answer:** Load average includes runnable tasks and uninterruptible sleep. Check `top`, `vmstat`, `pidstat`, `iostat`, blocked tasks, and `dmesg`. If many processes are in D state, focus on disk, network filesystem, device drivers, or storage latency. Also check cgroup throttling, container limits, and recent IO-heavy deploys. The answer should separate CPU saturation from IO wait and scheduler pressure.

### Q12: How do page cache and application memory interact in production?

**Answer:** Linux uses free memory for page cache, so low free memory is not automatically a leak. Databases and search engines often rely on page cache for file-backed data, while JVM/Go/Python heaps compete for RAM. Capacity planning should reserve memory for OS cache, avoid excessive heap sizing, and monitor RSS, cache, swap, major page faults, OOM events, and cgroup limits together.

### Q13: What is your first response to an OOM incident?

**Answer:** Preserve evidence before restarting everything: OOM killer logs, cgroup memory stats, process RSS, heap profiles, container limits, traffic/deploy timeline, and core dumps if available. Mitigate by scaling, rolling back, reducing traffic, or raising limits only when safe. Root cause analysis should distinguish true leaks, cache growth, payload spikes, fragmentation, page cache pressure, and memory limit misconfiguration.
