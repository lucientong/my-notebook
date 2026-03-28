# IaaS基础设施技术

---

## 📑 目录

### Part 1：虚拟化技术
1. [虚拟化技术概述](#1-虚拟化技术概述)
2. [KVM架构与原理](#2-kvm架构与原理)
3. [QEMU与设备模拟](#3-qemu与设备模拟)
4. [libvirt管理框架](#4-libvirt管理框架)
5. [虚拟机镜像格式](#5-虚拟机镜像格式)
6. [虚拟化深入：CPU/内存/IO](#6-虚拟化深入cpuio)

### Part 2：虚拟机网络
6. [虚拟机网络模式](#1-虚拟机网络模式)
7. [Linux网络虚拟化组件](#2-linux网络虚拟化组件)
8. [虚拟机网络配置实战](#3-虚拟机网络配置实战)
9. [虚拟机迁移技术](#4-虚拟机迁移技术)
10. [VNC远程管理](#5-vnc远程管理)

### Part 3：物理网络与虚拟化网络
11. [物理网络基础](#1-物理网络基础)
12. [VLAN虚拟局域网](#2-vlan虚拟局域网)
13. [网络隧道技术](#3-网络隧道技术)
14. [VXLAN大二层网络](#4-vxlan大二层网络)
15. [VPC虚拟私有云](#5-vpc虚拟私有云)

### Part 4：服务器底层技术
16. [服务器管理技术](#1-服务器管理技术)
17. [高性能网络DPDK](#2-高性能网络dpdk)
18. [高性能存储SPDK](#3-高性能存储spdk)
19. [RDMA远程直接内存访问](#4-rdma远程直接内存访问)
20. [LVM逻辑卷管理](#5-lvm逻辑卷管理)

### Part 5：操作系统运维深入
21. [Linux内核参数调优](#1-linux内核参数调优)
22. [系统排障工具](#2-系统排障工具)
23. [性能分析与监控](#3-性能分析与监控)
24. [文件系统排障](#4-文件系统排障)
25. [网络排障工具](#5-网络排障工具)
26. [内核分析工具](#6-内核分析工具)

### 面试题自查
26. [面试题自查](#面试题自查)

---

# Part 1: 虚拟化技术

---

## 1. 虚拟化技术概述

### 1.1 虚拟化类型

| 类型 | 代表技术 | 性能 | 隔离性 | 使用场景 |
|------|---------|------|--------|---------|
| **全虚拟化** | KVM、Xen HVM | 95-98% | 强 | 生产环境、多租户 |
| **半虚拟化** | Xen PV | 98-99% | 强 | 需修改Guest内核 |
| **容器虚拟化** | Docker、LXC | 99-100% | 中 | 微服务、CI/CD |
| **硬件辅助虚拟化** | Intel VT-x、AMD-V | 98-99% | 强 | 现代云平台 |

### 1.2 虚拟化架构对比

#### Type 1（裸金属虚拟化）
```
┌─────────────────────────────────────────┐
│  VM1        VM2        VM3              │
│ ┌─────┐   ┌─────┐   ┌─────┐            │
│ │Guest│   │Guest│   │Guest│            │
│ │ OS  │   │ OS  │   │ OS  │            │
│ └─────┘   └─────┘   └─────┘            │
├─────────────────────────────────────────┤
│        Hypervisor（VMM）                │
│     KVM/Xen/ESXi/Hyper-V               │
├─────────────────────────────────────────┤
│        物理硬件（CPU/内存/IO）          │
└─────────────────────────────────────────┘
```

**优点**：性能最优，资源直接管理  
**代表**：KVM、VMware ESXi、Xen

#### Type 2（宿主机虚拟化）
```
┌─────────────────────────────────────────┐
│  VM1        VM2                         │
│ ┌─────┐   ┌─────┐                      │
│ │Guest│   │Guest│                      │
│ │ OS  │   │ OS  │                      │
│ └─────┘   └─────┘                      │
├─────────────────────────────────────────┤
│   Hypervisor（VirtualBox/VMware）      │
├─────────────────────────────────────────┤
│        Host OS（Windows/Linux）        │
├─────────────────────────────────────────┤
│        物理硬件                         │
└─────────────────────────────────────────┘
```

**优点**：易用性好，适合开发测试  
**代表**：VirtualBox、VMware Workstation

---

## 2. KVM架构与原理

### 2.1 KVM核心组件

```
┌───────────────────────────────────────────────────┐
│                Guest VM                           │
│  ┌─────────────────────────────────────┐         │
│  │    Guest User Space                 │         │
│  │    Applications                     │         │
│  ├─────────────────────────────────────┤         │
│  │    Guest Kernel Space               │         │
│  │    驱动、文件系统、网络栈           │         │
│  └─────────────────────────────────────┘         │
│           ↕ (vCPU/vMem/vIO)                      │
└───────────────────────────────────────────────────┘
            ↕ VM Exit/Entry（特权指令陷入）
┌───────────────────────────────────────────────────┐
│              Host User Space                      │
│  ┌──────────────────────────────────┐            │
│  │  QEMU Process                    │            │
│  │  ├─ 设备模拟（硬盘/网卡/显卡）   │            │
│  │  ├─ IO处理                       │            │
│  │  └─ VNC/监控                     │            │
│  └──────────────────────────────────┘            │
│           ↕ ioctl(/dev/kvm)                      │
├───────────────────────────────────────────────────┤
│              Host Kernel Space                    │
│  ┌──────────────────────────────────┐            │
│  │  KVM Module (kvm.ko/kvm-intel.ko)│            │
│  │  ├─ vCPU调度                     │            │
│  │  ├─ 内存虚拟化（EPT/NPT）        │            │
│  │  └─ 中断虚拟化                   │            │
│  └──────────────────────────────────┘            │
├───────────────────────────────────────────────────┤
│           物理硬件（Intel VT-x/AMD-V）            │
└───────────────────────────────────────────────────┘
```

### 2.2 KVM关键技术

#### 2.2.1 CPU虚拟化

**Intel VT-x关键特性**：
- **VMX（Virtual Machine Extensions）**：引入VMX root/non-root两种操作模式
- **VMCS（VM Control Structure）**：保存虚拟机状态
- **EPT（Extended Page Table）**：硬件辅助的内存地址转换

**vCPU调度流程**：
```c
// 简化的KVM vCPU运行循环
int kvm_arch_vcpu_ioctl_run(struct kvm_vcpu *vcpu) {
    for (;;) {
        // 1. 准备进入Guest模式
        if (signal_pending(current)) {
            return -EINTR;  // 处理宿主机信号
        }
        
        // 2. 加载VMCS（Guest状态）
        vmx_load_host_state();
        
        // 3. VM Entry（进入Guest）
        vmx_vcpu_run(vcpu);  // 执行VMLAUNCH/VMRESUME指令
        
        // 4. VM Exit（退出到Host）
        exit_reason = vmcs_read32(VM_EXIT_REASON);
        
        // 5. 处理VM Exit原因
        switch (exit_reason) {
            case EXIT_REASON_IO_INSTRUCTION:
                handle_io();  // IO指令 -> 交给QEMU
                break;
            case EXIT_REASON_CPUID:
                handle_cpuid();  // CPUID指令
                break;
            case EXIT_REASON_EPT_VIOLATION:
                handle_ept_violation();  // 缺页异常
                break;
            // ... 更多Exit原因
        }
    }
}
```

**VM Exit触发条件**（性能关键）：
```bash
# 查看VM Exit统计
cat /sys/kernel/debug/kvm/exits

# 常见Exit原因及优化
EXIT_REASON_IO_INSTRUCTION        # IO端口访问（使用virtio减少）
EXIT_REASON_EPT_VIOLATION         # 内存访问异常（预分配内存）
EXIT_REASON_EXTERNAL_INTERRUPT    # 外部中断（使用中断亲和性）
EXIT_REASON_PAUSE_INSTRUCTION     # 自旋锁（使用paravirt_spinlocks）
```

#### 2.2.2 内存虚拟化

**三层地址转换**：
```
Guest虚拟地址（GVA）
    ↓ Guest页表
Guest物理地址（GPA）
    ↓ EPT/NPT（硬件加速）
Host物理地址（HPA）
```

**EPT（Extended Page Table）工作流程**：
```c
// EPT页表结构（4级）
EPT PML4 Table
  ↓
EPT Page Directory Pointer Table
  ↓
EPT Page Directory Table
  ↓
EPT Page Table
  ↓
物理页帧（HPA）

// EPT权限位
#define EPT_READ    (1 << 0)
#define EPT_WRITE   (1 << 1)
#define EPT_EXEC    (1 << 2)

// EPT Violation处理
int handle_ept_violation(struct kvm_vcpu *vcpu, gpa_t gpa) {
    // 1. 检查是否是缺页
    if (!gpa_present(gpa)) {
        // 分配物理页
        hpa = alloc_page();
        // 建立EPT映射
        ept_map(gpa, hpa, EPT_READ | EPT_WRITE);
    }
    // 2. 检查权限错误
    else if (access_violation(gpa)) {
        // 处理权限错误（如写时复制）
        handle_cow(gpa);
    }
}
```

**内存超分配（Memory Overcommitment）**：
```bash
# KVM内存管理技术
1. KSM（Kernel Samepage Merging）：合并相同内存页
echo 1 > /sys/kernel/mm/ksm/run

2. Memory Ballooning：动态调整内存
# QEMU启动参数
-device virtio-balloon-pci,id=balloon0

# 在Guest内调整
virsh qemu-monitor-command vm1 --hmp balloon 2048

3. Swap：内存交换到磁盘
# 性能影响大，生产环境需谨慎
```

#### 2.2.3 中断虚拟化

**中断注入流程**：
```
物理中断
  ↓
Host Kernel（KVM）
  ↓ 判断中断目标
Guest VCPU
  ↓ 虚拟中断控制器（vAPIC）
Guest Interrupt Handler
```

**PCI设备中断优化**：
```bash
# 传统方式：中断共享，性能差
# 优化方式：MSI-X（每个设备独立中断向量）

# 查看设备中断模式
lspci -vv | grep -i msi
Capabilities: [50] MSI-X: Enable+ Count=10 Masked-

# QEMU配置MSI-X
-device virtio-net-pci,msi=on,vectors=4
```

---

## 3. QEMU与设备模拟

### 3.1 QEMU架构

```
┌────────────────────────────────────────┐
│         QEMU Process                   │
│                                        │
│  ┌──────────────────────────────┐     │
│  │   Device Emulation           │     │
│  │   ├─ virtio-blk (硬盘)       │     │
│  │   ├─ virtio-net (网卡)       │     │
│  │   ├─ VGA (显卡)              │     │
│  │   └─ USB/Serial              │     │
│  └──────────────────────────────┘     │
│              ↕                         │
│  ┌──────────────────────────────┐     │
│  │   Main Loop (事件循环)       │     │
│  │   select/epoll on fds        │     │
│  └──────────────────────────────┘     │
│              ↕                         │
│  ┌──────────────────────────────┐     │
│  │   KVM Interface              │     │
│  │   ioctl(/dev/kvm)            │     │
│  └──────────────────────────────┘     │
└────────────────────────────────────────┘
```

### 3.2 QEMU vs KVM vs qemu-kvm

| 组件 | 作用 | 关系 |
|------|------|------|
| **QEMU** | 设备模拟器，可独立运行（纯软件模拟，慢） | 用户空间程序 |
| **KVM** | 内核模块，CPU/内存虚拟化 | 内核模块 |
| **qemu-kvm** | QEMU + KVM，硬件加速的虚拟机 | QEMU调用KVM API |

**组合方式**：
```bash
# 1. QEMU单独使用（纯软件模拟，性能差）
qemu-system-x86_64 -hda disk.img  # 速度约为物理机的5-10%

# 2. QEMU + KVM（推荐，生产环境）
qemu-system-x86_64 -enable-kvm -hda disk.img  # 速度约为物理机的95-98%

# 验证KVM是否启用
qemu-system-x86_64 -enable-kvm -nographic
# 在Guest内执行
dmesg | grep -i kvm
[    0.000000] Hypervisor detected: KVM
```

### 3.3 virtio半虚拟化驱动

**传统全虚拟化 vs virtio**：

```
传统方式（e1000网卡模拟）：
Guest网络栈 → e1000驱动 → VM Exit → QEMU模拟e1000 → Host网络栈
              (每次IO都触发VM Exit，开销大)

virtio方式：
Guest网络栈 → virtio-net前端驱动 → virtio环形缓冲区 → virtio-net后端 → Host网络栈
              (批量处理，减少VM Exit)
```

**virtio设备类型**：
```bash
# 块设备（硬盘）
-drive file=disk.img,if=virtio

# 网络设备
-device virtio-net-pci,netdev=net0

# 其他virtio设备
virtio-balloon  # 内存气球
virtio-scsi     # SCSI控制器
virtio-serial   # 串口
virtio-rng      # 随机数生成器
```

**virtio性能对比**：
```bash
# 测试环境：虚拟机网络性能
# 工具：iperf3

# e1000模拟网卡
iperf3 -c server
[  5]   0.00-10.00  sec  1.10 GBytes  945 Mbits/sec  (VM Exit频繁)

# virtio-net
iperf3 -c server
[  5]   0.00-10.00  sec  9.31 GBytes  8.00 Gbits/sec  (接近物理机)
```

### 3.4 QEMU启动完整示例

```bash
qemu-system-x86_64 \
  -enable-kvm \                        # 启用KVM加速
  -m 4096 \                            # 4GB内存
  -smp 4,cores=2,threads=2,sockets=1 \ # 4个vCPU
  -cpu host \                          # 透传Host CPU特性
  -drive file=vm.qcow2,if=virtio,cache=none,aio=native \ # virtio磁盘
  -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \ # TAP网络
  -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:56 \
  -vnc :1 \                            # VNC远程桌面（5901端口）
  -monitor unix:/tmp/vm.sock,server,nowait \ # 监控接口
  -daemonize \                         # 后台运行
  -pidfile /var/run/vm.pid             # PID文件
```

---

## 4. libvirt管理框架

### 4.1 libvirt架构

```
┌─────────────────────────────────────────────┐
│       管理工具                               │
│  virsh / virt-manager / OpenStack Nova     │
└─────────────────────────────────────────────┘
                    ↕ libvirt API
┌─────────────────────────────────────────────┐
│           libvirtd (守护进程)               │
│  ┌───────────────────────────────────┐     │
│  │  Driver Layer                     │     │
│  │  ├─ QEMU Driver                   │     │
│  │  ├─ LXC Driver                    │     │
│  │  └─ Xen Driver                    │     │
│  └───────────────────────────────────┘     │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│       Hypervisor                            │
│  QEMU/KVM / LXC / Xen                      │
└─────────────────────────────────────────────┘
```

### 4.2 libvirt核心概念

| 概念 | 说明 | 示例 |
|------|------|------|
| **Domain** | 虚拟机实例 | `virsh list` |
| **Network** | 虚拟网络（NAT/桥接） | `virsh net-list` |
| **Storage Pool** | 存储池 | `virsh pool-list` |
| **Volume** | 存储卷（虚拟磁盘） | `virsh vol-list default` |

### 4.3 virsh常用命令

```bash
# === 虚拟机生命周期管理 ===
virsh list --all                  # 列出所有虚拟机
virsh start vm1                   # 启动虚拟机
virsh shutdown vm1                # 正常关机（ACPI）
virsh destroy vm1                 # 强制关机（拔电源）
virsh reboot vm1                  # 重启
virsh suspend vm1                 # 暂停（内存保留）
virsh resume vm1                  # 恢复
virsh autostart vm1               # 设置开机自启

# === 虚拟机信息查看 ===
virsh dominfo vm1                 # 虚拟机详细信息
virsh vcpuinfo vm1                # vCPU信息
virsh dommemstat vm1              # 内存统计
virsh domifstat vm1 vnet0         # 网络流量统计
virsh domblkstat vm1 vda          # 磁盘IO统计

# === 虚拟机配置修改 ===
virsh edit vm1                    # 编辑XML配置
virsh setmem vm1 2048M            # 动态调整内存
virsh setvcpus vm1 4 --live       # 动态调整CPU

# === 快照管理 ===
virsh snapshot-create-as vm1 snap1 "First snapshot"  # 创建快照
virsh snapshot-list vm1                              # 列出快照
virsh snapshot-revert vm1 snap1                      # 恢复快照
virsh snapshot-delete vm1 snap1                      # 删除快照

# === 虚拟机迁移 ===
virsh migrate --live vm1 qemu+ssh://host2/system  # 热迁移
```

### 4.4 libvirt XML配置示例

```xml
<domain type='kvm'>
  <name>centos7-vm</name>
  <memory unit='GiB'>4</memory>
  <currentMemory unit='GiB'>4</currentMemory>
  <vcpu placement='static'>4</vcpu>
  
  <!-- CPU配置 -->
  <cpu mode='host-passthrough'>
    <topology sockets='1' cores='2' threads='2'/>
  </cpu>
  
  <!-- 操作系统启动 -->
  <os>
    <type arch='x86_64' machine='pc-i440fx-2.12'>hvm</type>
    <boot dev='hd'/>
  </os>
  
  <!-- 设备配置 -->
  <devices>
    <!-- virtio磁盘 -->
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='native'/>
      <source file='/var/lib/libvirt/images/centos7.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    
    <!-- virtio网卡（桥接模式） -->
    <interface type='bridge'>
      <source bridge='br0'/>
      <model type='virtio'/>
      <mac address='52:54:00:6b:3c:58'/>
    </interface>
    
    <!-- VNC图形界面 -->
    <graphics type='vnc' port='5900' autoport='yes' listen='0.0.0.0'/>
    
    <!-- Serial控制台 -->
    <serial type='pty'>
      <target port='0'/>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
  </devices>
</domain>
```

---

## 5. 虚拟机镜像格式

### 5.1 镜像格式对比

| 格式 | 特性 | 性能 | 快照 | 压缩 | 适用场景 |
|------|------|------|------|------|---------|
| **raw** | 原始格式，1:1映射 | ⭐⭐⭐⭐⭐ | ❌ | ❌ | 生产环境、性能优先 |
| **qcow2** | QEMU写时复制 | ⭐⭐⭐⭐ | ✅ | ✅ | 开发测试、节省空间 |
| **qcow** | qcow2前身 | ⭐⭐⭐ | ✅ | ❌ | 已废弃 |
| **vmdk** | VMware格式 | ⭐⭐⭐⭐ | ✅ | ✅ | VMware兼容 |
| **vdi** | VirtualBox格式 | ⭐⭐⭐⭐ | ✅ | ✅ | VirtualBox |
| **vhd/vhdx** | Hyper-V格式 | ⭐⭐⭐⭐ | ✅ | ✅ | Azure、Hyper-V |

### 5.2 raw格式

**特点**：
- 裸格式，磁盘空间预分配
- 性能最佳，无元数据开销
- 不支持快照、压缩

```bash
# 创建raw镜像（预分配10GB）
qemu-img create -f raw disk.raw 10G
ls -lh disk.raw
-rw-r--r-- 1 root root 10G  disk.raw  # 立即占用10GB

# 转换为raw格式
qemu-img convert -f qcow2 -O raw source.qcow2 output.raw

# 使用raw格式启动虚拟机
qemu-system-x86_64 -drive file=disk.raw,format=raw,if=virtio
```

### 5.3 qcow2格式（推荐）

**特点**：
- 写时复制（Copy-On-Write）
- 支持快照、压缩、加密
- 支持精简配置（thin provisioning）

**qcow2文件结构**：
```
┌────────────────────────────────────┐
│  Header（512字节）                 │
│  - Magic: 'QFI\xfb'                │
│  - Version: 3                      │
│  - Backing file offset             │
│  - Cluster size (默认64KB)         │
├────────────────────────────────────┤
│  L1 Table（一级映射表）            │
│  指向L2 Table的指针数组            │
├────────────────────────────────────┤
│  L2 Tables（二级映射表）           │
│  指向数据Cluster的指针             │
├────────────────────────────────────┤
│  Refcount Table（引用计数表）      │
│  用于写时复制和快照                │
├────────────────────────────────────┤
│  Data Clusters（实际数据）         │
│  按需分配                          │
└────────────────────────────────────┘
```

**qcow2高级特性**：

```bash
# 1. 创建qcow2镜像（精简配置）
qemu-img create -f qcow2 disk.qcow2 100G
ls -lh disk.qcow2
-rw-r--r-- 1 root root 193K disk.qcow2  # 实际只占用193KB！

# 2. 基于backing file创建镜像（链式快照）
qemu-img create -f qcow2 -b base.qcow2 -F qcow2 vm1.qcow2
# vm1.qcow2只存储与base.qcow2的差异

# 3. 压缩镜像
qemu-img convert -c -O qcow2 source.qcow2 compressed.qcow2

# 4. 查看镜像信息
qemu-img info disk.qcow2
image: disk.qcow2
file format: qcow2
virtual size: 100 GiB (107374182400 bytes)
disk size: 2.1 GiB              # 实际占用空间
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: true
    refcount bits: 16
    corrupt: false

# 5. 内部快照（qcow2特性）
qemu-img snapshot -c snap1 disk.qcow2       # 创建快照
qemu-img snapshot -l disk.qcow2             # 列出快照
qemu-img snapshot -a snap1 disk.qcow2       # 恢复快照
qemu-img snapshot -d snap1 disk.qcow2       # 删除快照

# 6. 加密镜像
qemu-img create -f qcow2 -o encryption=on encrypted.qcow2 10G
# 启动时需要输入密码
```

### 5.4 镜像转换与优化

```bash
# === 格式转换 ===
# qcow2 -> raw
qemu-img convert -f qcow2 -O raw source.qcow2 output.raw

# vmdk -> qcow2
qemu-img convert -f vmdk -O qcow2 vm.vmdk vm.qcow2

# === 镜像压缩（回收未使用空间）===
# Step 1: 在Guest内执行零填充
dd if=/dev/zero of=/zerofile bs=1M
rm -f /zerofile

# Step 2: 在Host上压缩
qemu-img convert -O qcow2 old.qcow2 new.qcow2

# === 镜像扩容 ===
# 增加10GB
qemu-img resize disk.qcow2 +10G

# 在Guest内扩展分区
fdisk /dev/vda          # 扩展分区
resize2fs /dev/vda1     # 扩展文件系统（ext4）
xfs_growfs /dev/vda1    # 扩展文件系统（xfs）

# === 镜像检查与修复 ===
qemu-img check disk.qcow2           # 检查镜像
qemu-img check -r all disk.qcow2    # 修复镜像
```

### 5.5 镜像性能对比测试

```bash
# 测试环境：fio随机4K读写
# 物理磁盘：NVMe SSD

# 1. raw格式
fio --name=randwrite --ioengine=libaio --iodepth=16 \
    --rw=randwrite --bs=4k --size=1G --numjobs=4 \
    --filename=/dev/vda
IOPS: 78,000 (接近物理磁盘的85,000)

# 2. qcow2格式（默认）
IOPS: 68,000 (约为raw的87%)

# 3. qcow2格式（cache=none，aio=native）
-drive file=disk.qcow2,cache=none,aio=native,if=virtio
IOPS: 73,000 (约为raw的93%)

# 结论：
# - 性能差异：raw > qcow2 (10-15%)
# - 生产环境推荐：性能敏感用raw，其他用qcow2
```

---

---

## 6. 虚拟化深入：CPU/内存/IO

> 本节深入探讨KVM/QEMU虚拟化的核心机制，包括QEMU线程模型、CPU虚拟化细节、内存虚拟化数据结构、virtio通信原理等，对应从源码层面理解虚拟化行为。

### 6.1 QEMU线程模型

**QEMU进程中的线程组成**：

```
QEMU进程（一个虚拟机 = 一个QEMU进程）
├── Main Thread（主线程）
│   ├── 事件循环（Event Loop / Main Loop）
│   ├── 管理操作（QMP命令处理、热迁移控制）
│   ├── 非dataplane的设备模拟
│   └── select/epoll监听各类fd（monitor socket、VNC等）
│
├── vCPU Thread × N（每个vCPU一个线程）
│   ├── KVM_RUN ioctl → 进入Guest执行
│   ├── VM Exit处理（KVM内核无法处理的部分回到此线程）
│   └── 信号处理（SIG_IPI用于vCPU间中断）
│
├── IO Thread（可选，iothread=on）
│   ├── 独立事件循环，处理virtio-blk/virtio-scsi的IO
│   └── 避免IO处理阻塞主线程或vCPU线程
│
└── Worker Thread Pool
    └── 处理异步任务（如qcow2 cluster分配、镜像压缩等）
```

```bash
# 查看QEMU进程的线程
ps -T -p $(pidof qemu-system-x86_64)
#   PID    SPID TTY          TIME CMD
# 12345   12345 ?        00:00:05 qemu-system-x86  ← 主线程
# 12345   12346 ?        00:01:23 CPU 0/KVM        ← vCPU 0
# 12345   12347 ?        00:01:20 CPU 1/KVM        ← vCPU 1
# 12345   12348 ?        00:00:10 IO iothread1     ← IO线程
# 12345   12349 ?        00:00:01 worker           ← 工作线程

# 使用taskset绑定vCPU线程到物理核心（减少缓存miss）
taskset -p 0x01 12346   # vCPU 0 → 物理核心0
taskset -p 0x02 12347   # vCPU 1 → 物理核心1
```

**主线程阻塞对vCPU的影响**：
- vCPU线程与主线程是**独立**的，主线程阻塞时vCPU**仍可继续执行**Guest代码
- 但如果VM Exit需要主线程处理（如某些设备模拟），则vCPU线程会阻塞等待
- 这就是为什么推荐使用独立IO线程（`-object iothread`）：让IO处理不经过主线程

```bash
# 启用独立IO线程
qemu-system-x86_64 \
  -object iothread,id=iothread0 \
  -device virtio-blk-pci,drive=drive0,iothread=iothread0 \
  -drive id=drive0,file=disk.qcow2,if=none
```

### 6.2 CPU虚拟化深入

#### vCPU创建与运行流程

```c
// 1. QEMU创建vCPU（用户空间）
int vcpu_fd = ioctl(vm_fd, KVM_CREATE_VCPU, vcpu_id);

// 2. 获取vCPU的共享内存区域（kvm_run结构）
struct kvm_run *run = mmap(NULL, mmap_size, ..., vcpu_fd, 0);

// 3. 设置vCPU寄存器（初始状态）
struct kvm_regs regs = { .rip = 0xFFF0, .rflags = 0x2 };
ioctl(vcpu_fd, KVM_SET_REGS, &regs);

// 4. vCPU运行循环（在vCPU线程中）
while (true) {
    // 进入Guest执行（阻塞直到VM Exit）
    ioctl(vcpu_fd, KVM_RUN, NULL);
    
    // VM Exit，检查退出原因
    switch (run->exit_reason) {
        case KVM_EXIT_IO:
            // IO端口访问 → QEMU模拟设备
            handle_io(run->io.port, run->io.data, ...);
            break;
        case KVM_EXIT_MMIO:
            // 内存映射IO → QEMU处理
            handle_mmio(run->mmio.phys_addr, run->mmio.data, ...);
            break;
        case KVM_EXIT_HLT:
            // HLT指令 → vCPU空闲
            break;
        case KVM_EXIT_SHUTDOWN:
            // 关机
            return;
    }
}
```

#### VMCS与上下文保存恢复

```
VMCS（Virtual Machine Control Structure）：
每个vCPU有一个VMCS，存储Guest和Host的完整CPU状态

VMCS包含6个区域：
┌─────────────────────────────────────┐
│ Guest State Area                     │
│  - CR0/CR3/CR4（控制寄存器）         │
│  - RSP/RIP/RFLAGS                   │
│  - CS/DS/SS/ES/FS/GS（段寄存器）     │
│  - GDTR/IDTR/LDTR/TR               │
│  - MSR（如IA32_EFER）               │
├─────────────────────────────────────┤
│ Host State Area                      │
│  - CR0/CR3/CR4                      │
│  - RSP/RIP                          │
│  - CS/DS/SS/ES/FS/GS               │
├─────────────────────────────────────┤
│ VM-Execution Control                 │
│  - Pin-Based（中断控制）             │
│  - Processor-Based（指令拦截）       │
│  - Exception Bitmap（异常拦截）       │
│  - EPT Pointer（EPT页表基址）        │
├─────────────────────────────────────┤
│ VM-Exit Control（Exit时的行为）      │
├─────────────────────────────────────┤
│ VM-Entry Control（Entry时的行为）    │
├─────────────────────────────────────┤
│ VM-Exit Information（Exit原因信息）  │
└─────────────────────────────────────┘

VM Entry流程：
1. VMLAUNCH/VMRESUME指令
2. 从VMCS加载Guest State → CPU寄存器
3. CPU进入VMX non-root模式（Guest模式）

VM Exit流程：
1. 触发Exit条件（特权指令、EPT violation等）
2. CPU保存Guest State → VMCS
3. 从VMCS加载Host State → CPU寄存器
4. CPU进入VMX root模式（Host模式）
```

#### 特权指令与Exit原因

```
常见触发VM Exit的指令/事件：
┌────────────────────────┬──────────────────────────┬───────────────────┐
│ 类别                    │ 具体指令/事件             │ 处理位置          │
├────────────────────────┼──────────────────────────┼───────────────────┤
│ 特权指令                │ CPUID                    │ KVM内核处理       │
│                        │ MOV to/from CR           │ KVM内核处理       │
│                        │ HLT                      │ KVM/QEMU          │
│                        │ WRMSR/RDMSR              │ KVM内核处理       │
├────────────────────────┼──────────────────────────┼───────────────────┤
│ IO访问                  │ IN/OUT（IO端口）          │ QEMU设备模拟      │
│                        │ MMIO访问                  │ QEMU设备模拟      │
├────────────────────────┼──────────────────────────┼───────────────────┤
│ 内存事件                │ EPT Violation             │ KVM内核处理       │
│                        │ EPT Misconfig             │ KVM内核处理       │
├────────────────────────┼──────────────────────────┼───────────────────┤
│ 中断/异常               │ 外部中断                  │ KVM内核处理       │
│                        │ NMI                       │ KVM内核处理       │
├────────────────────────┼──────────────────────────┼───────────────────┤
│ 其他                    │ PAUSE（自旋等待）         │ KVM内核处理       │
│                        │ VMCALL（超级调用）         │ KVM内核处理       │
└────────────────────────┴──────────────────────────┴───────────────────┘

# 使用perf/ftrace查看VM Exit统计
perf kvm stat live
# Event        Samples   Percent
# EPT_VIOLATION   1234    45.2%
# IO_INSTRUCTION   890    32.6%
# EXTERNAL_INT     456    16.7%
# HLT              100     3.7%
# PAUSE             50     1.8%
```

#### Posted Interrupt（中断直接注入）

```
传统中断注入流程：
外部中断 → Host KVM → 设置VMCS中断信息 → VM Entry时注入 → Guest处理
问题：需要一次VM Exit + VM Entry，开销大

Posted Interrupt原理：
外部中断 → APIC直接写入PIR（Posted Interrupt Request）→ 发送通知向量
         → CPU自动将PIR合并到虚拟APIC → Guest直接处理
         → 无需VM Exit！

PIR（Posted Interrupt Descriptor）结构：
┌──────────────────────────────────┐
│ PIR[0-255]：256位中断请求向量     │ ← APIC直接写入
│ ON (Outstanding Notification)    │ ← 标记有新中断
│ SN (Suppress Notification)       │ ← 抑制通知
│ NV (Notification Vector)         │ ← 通知使用的物理中断向量
│ NDST (Notification Destination)  │ ← 目标物理CPU
└──────────────────────────────────┘

硬件要求：
- Intel VT-x with APICv（Virtual APIC）
- CPU支持 Posted Interrupt Processing
- 需要启用 Virtual Interrupt Delivery
```

```bash
# 检查是否支持APICv / Posted Interrupt
cat /sys/module/kvm_intel/parameters/enable_apicv
# Y → 已启用

# 查看中断注入统计
cat /sys/kernel/debug/kvm/irq_injections
```

### 6.3 内存虚拟化深入

#### 地址空间关系

```
四种地址：
GVA（Guest Virtual Address）  ← Guest进程使用的虚拟地址
  ↓ Guest页表（Guest CR3）
GPA（Guest Physical Address） ← Guest看到的"物理地址"
  ↓ EPT页表（EPTP）
HPA（Host Physical Address）  ← 真实的物理地址
  
HVA（Host Virtual Address）   ← QEMU进程的虚拟地址

关系：
- GPA → HPA：通过EPT硬件转换（Guest内存访问路径）
- GPA → HVA：通过QEMU的memory region映射（QEMU访问Guest内存）
- HVA → HPA：通过Host页表（QEMU进程的正常虚拟→物理转换）
```

#### QEMU与KVM内存数据结构

```
QEMU侧（用户空间）：
┌────────────────────────────────────────────┐
│ AddressSpace（地址空间）                     │
│  └── MemoryRegion Tree（内存区域树）         │
│       ├── RAM MemoryRegion（普通内存）       │
│       │   - offset_in_ram: GPA起始地址      │
│       │   - size: 区域大小                   │
│       │   - ram_block: 指向RAMBlock          │
│       ├── IO MemoryRegion（MMIO设备）        │
│       │   - read_fn/write_fn: MMIO回调函数   │
│       └── Alias MemoryRegion（别名映射）      │
└────────────────────────────────────────────┘
              ↓ 转换为
┌────────────────────────────────────────────┐
│ FlatView（扁平化视图）                       │
│  - 将树形MemoryRegion展平为线性地址范围      │
│  - 每个范围对应一个FlatRange                 │
└────────────────────────────────────────────┘
              ↓ 同步到KVM
KVM侧（内核空间）：
┌────────────────────────────────────────────┐
│ kvm_memory_slot（内存槽）                    │
│  - base_gfn: GPA的页框号                    │
│  - npages: 页数                             │
│  - userspace_addr: HVA（QEMU mmap的地址）   │
│  - flags: 只读/可写/日志脏页等              │
└────────────────────────────────────────────┘

# QEMU通过 ioctl(KVM_SET_USER_MEMORY_REGION) 注册mem slot
# KVM据此建立 GPA → HPA 的EPT映射
```

**QEMU与GPA的关系**：
```
QEMU进程用mmap分配一块连续的HVA空间作为VM的"物理内存"
GPA 0x0000_0000 ~ 0x8000_0000  ←→  HVA 0x7f0000000000 ~ 0x7f0080000000

QEMU访问Guest内存内容：
  cpu_physical_memory_rw(gpa, buf, len, is_write)
  → 将GPA转换为HVA（查MemoryRegion/RAMBlock）
  → 直接通过HVA读写（因为QEMU进程的虚拟地址空间包含这块内存）
```

#### EPT缺页处理完整流程

```
1. Guest进程访问虚拟地址（GVA）
   ↓
2. Guest MMU通过Guest页表将GVA → GPA
   ↓
3. CPU查EPT页表将GPA → HPA
   ↓ 如果EPT项不存在（EPT Violation）
4. VM Exit（exit_reason = EPT_VIOLATION）
   ↓
5. KVM的handle_ept_violation()处理：
   ├── 获取GPA（从VMCS读取 Guest Physical Address字段）
   ├── 查找对应的kvm_memory_slot
   │   ├── 如果GPA在合法范围内：
   │   │   ├── 计算HVA = slot->userspace_addr + (gpa - slot->base_gfn * PAGE_SIZE)
   │   │   ├── 通过Host页表获取HPA（可能触发Host缺页）
   │   │   └── 建立EPT映射：GPA → HPA
   │   └── 如果GPA不在任何slot中：
   │       └── 视为MMIO访问 → 返回QEMU处理
   ↓
6. VM Entry，Guest继续执行
```

#### MMIO访问处理

```
MMIO（Memory-Mapped IO）：设备寄存器映射到GPA地址空间

当Guest访问MMIO地址时：
1. EPT中该GPA区域标记为无映射（或标记为MMIO）
2. 触发EPT Violation → VM Exit
3. KVM发现GPA不属于RAM slot → 标记为KVM_EXIT_MMIO
4. 返回QEMU用户空间
5. QEMU查MemoryRegion树，找到对应的IO MemoryRegion
6. 调用设备的read/write回调函数
7. 返回KVM，VM Entry继续执行

示例：Guest读取virtio设备状态寄存器
GPA 0xFEBF1000 → EPT miss → VM Exit → QEMU →
virtio_pci_config_read() → 返回设备状态 → VM Entry
```

#### Fast Page Fault与Async Page Fault

**Fast Page Fault**：
```
场景：EPT权限错误（如写时复制COW）而非真正缺页

传统处理：EPT Violation → 获取mmu_lock → 修改EPT → 释放锁
问题：mmu_lock是全局锁，多vCPU并发时成为瓶颈

Fast Page Fault优化：
1. 检查是否是简单的权限更新（如COW后只需添加写权限）
2. 如果是，直接用原子操作（cmpxchg）修改EPT PTE
3. 无需获取mmu_lock → 大幅减少锁竞争

适用条件：
- 仅权限位变化（如只读→可写）
- 不涉及页框号变化
```

**Async Page Fault（异步缺页）**：
```
场景：Guest访问的内存页被Host换出到Swap

传统处理：
Guest访问 → EPT Violation → KVM处理 → Host缺页（从Swap读回）
→ vCPU阻塞等待磁盘IO → Guest CPU空闲浪费

Async PF优化：
1. Guest访问被swap的页 → EPT Violation
2. KVM向Guest注入#PF异常（async page fault类型）
3. Guest收到异步缺页通知 → 将当前进程设为睡眠，调度其他进程
4. Host后台将页从Swap读回
5. 页准备好后 → KVM通知Guest（通过中断）
6. Guest唤醒之前睡眠的进程继续执行

效果：vCPU不会因Host的Swap IO而空闲，可以执行其他Guest进程
```

```bash
# 启用async page fault
echo 1 > /sys/module/kvm/parameters/async_pf

# 查看统计
cat /sys/kernel/debug/kvm/async_pf_completed
```

#### EPT反向映射与页表回收

```
EPT反向映射（Reverse Mapping）：
目的：从HPA快速找到所有映射到它的EPT PTE

数据结构：
kvm_memory_slot中每个页框维护一个rmap链表
slot->arch.rmap[level] → rmap_head → pte_list → EPT PTE指针

用途：
1. 页面迁移/回收时，需要解除所有EPT映射
2. 脏页追踪（热迁移时标记脏页）
3. 内存超分时回收Guest内存

EPT页表回收：
当Host内存紧张时，KVM可以回收EPT页表本身占用的内存
- kvm_mmu_zap_all()：清除所有EPT映射
- Guest下次访问会重新建立（类似TLB miss后的page walk）
- 通过MMU notifier机制与Host MM子系统协调
```

### 6.4 virtio深入：数据结构与通信原理

#### virtio整体架构

```
┌──────────────────────────────────────────────┐
│                 Guest                         │
│  ┌─────────────────────────────────┐         │
│  │  virtio前端驱动                  │         │
│  │  (virtio-blk/virtio-net/...)    │         │
│  └──────────┬──────────────────────┘         │
│             │ 共享内存                        │
│  ┌──────────↓──────────────────────┐         │
│  │  Virtqueue（virtio环形队列）     │         │
│  │  ├── Descriptor Table           │         │
│  │  ├── Available Ring             │         │
│  │  └── Used Ring                  │         │
│  └──────────┬──────────────────────┘         │
└─────────────│────────────────────────────────┘
              │ 通知机制（IO端口/MMIO/eventfd）
┌─────────────↓────────────────────────────────┐
│  ┌──────────────────────────────────┐        │
│  │  virtio后端                      │        │
│  │  QEMU / vhost-kernel / vhost-user│        │
│  └──────────────────────────────────┘        │
│                 Host                          │
└──────────────────────────────────────────────┘
```

#### Vring数据结构

```c
// Virtqueue由三个部分组成（共享内存，前端和后端都可访问）

// 1. Descriptor Table（描述符表）
//    - 描述数据缓冲区的物理地址和长度
//    - 前端和后端都可读取，前端负责填写
struct vring_desc {
    __le64 addr;    // 缓冲区的GPA地址
    __le32 len;     // 缓冲区长度
    __le16 flags;   // VRING_DESC_F_NEXT：链式描述符
                    // VRING_DESC_F_WRITE：设备可写（用于接收）
                    // VRING_DESC_F_INDIRECT：间接描述符表
    __le16 next;    // 下一个描述符索引（链式时使用）
};

// 2. Available Ring（可用环）
//    - 前端写入，后端读取
//    - 前端放入"请后端处理"的描述符链头部索引
struct vring_avail {
    __le16 flags;       // VRING_AVAIL_F_NO_INTERRUPT：抑制中断
    __le16 idx;         // 下一个可写位置（单调递增）
    __le16 ring[];      // 环形数组，存放描述符链的head index
};

// 3. Used Ring（已用环）
//    - 后端写入，前端读取
//    - 后端放入"已处理完毕"的描述符链信息
struct vring_used {
    __le16 flags;       // VRING_USED_F_NO_NOTIFY：抑制前端通知
    __le16 idx;         // 下一个可写位置（单调递增）
    struct vring_used_elem ring[];
};
struct vring_used_elem {
    __le32 id;          // 描述符链的head index
    __le32 len;         // 后端写入的数据长度
};
```

#### 前后端通知机制

```
前端通知后端（Guest → Host）：
1. IO端口写入（传统方式）：写virtio PCI设备的Queue Notify寄存器
   → 触发VM Exit → KVM → QEMU处理
2. ioeventfd（优化方式）：写入映射的eventfd地址
   → KVM直接触发eventfd，无需返回QEMU主线程
   → 绑定到IO线程/vhost，减少延迟

后端通知前端（Host → Guest）：
1. 虚拟中断（传统方式）：QEMU注入MSI-X中断
   → KVM设置VMCS → 下次VM Entry时Guest收到中断
2. irqfd（优化方式）：后端写入eventfd
   → KVM直接注入中断，无需经过QEMU
   → 配合Posted Interrupt可避免VM Exit
```

#### virtio设备/队列初始化流程

```
Guest驱动探测到virtio PCI设备后的初始化流程：

1. 重置设备
   → 写 Status Register = 0

2. 设置ACKNOWLEDGE位
   → Status |= VIRTIO_CONFIG_S_ACKNOWLEDGE

3. 设置DRIVER位
   → Status |= VIRTIO_CONFIG_S_DRIVER

4. 协商特性（Feature Negotiation）
   → 读取设备支持的feature bits
   → 驱动选择自己支持的子集
   → 写回guest_features

5. 初始化Virtqueue
   → 读取Queue Size（描述符表大小）
   → 分配共享内存（Descriptor Table + Available Ring + Used Ring）
   → 将GPA写入Queue Address寄存器
   → 后端据此映射共享内存

6. 设置DRIVER_OK位
   → Status |= VIRTIO_CONFIG_S_DRIVER_OK
   → 设备可以开始工作
```

### 6.5 virtio-blk IO请求完整路径

```
Guest应用调用write()
  ↓
Guest VFS → 文件系统（ext4）→ Block Layer → virtio-blk前端驱动
  ↓
1. 分配Descriptor：
   desc[0] = { addr=请求头GPA, len=sizeof(header), flags=0 }      # 设备读
   desc[1] = { addr=数据缓冲区GPA, len=4096, flags=0 }            # 设备读
   desc[2] = { addr=状态字节GPA, len=1, flags=WRITE }             # 设备写
   desc[0].next=1, desc[1].next=2（链式）

2. 放入Available Ring：
   avail.ring[avail.idx % queue_size] = desc[0]的index
   avail.idx++
   wmb()  // 写内存屏障，确保后端能看到

3. 通知后端（kick）：
   写Queue Notify寄存器 → ioeventfd → 后端收到通知
  ↓
后端（QEMU或vhost）处理：
4. 读取Available Ring新条目 → 获取描述符链
5. 解析请求头（读/写/flush）
6. 根据后端类型执行IO：
   ├── native AIO（aio=native）：使用Linux io_submit直接提交
   └── thread pool（aio=threads）：分发到工作线程执行pread/pwrite

7. IO完成后，填写Used Ring：
   used.ring[used.idx % queue_size] = { id=head_index, len=written_bytes }
   used.idx++
   wmb()

8. 通知前端（中断注入）：
   irqfd → KVM → 注入MSI-X中断 → Guest中断处理
  ↓
Guest virtio-blk中断处理：
9. 读取Used Ring → 获取已完成的请求
10. 检查状态字节 → 完成Block Layer的request
11. 通知上层（文件系统/应用）IO完成
```

**QEMU的native IO vs aio threads**：

| 特性 | native AIO (io_submit) | aio threads |
|------|----------------------|-------------|
| **机制** | Linux异步IO系统调用 | QEMU线程池+同步IO |
| **要求** | 需要O_DIRECT | 支持所有缓存模式 |
| **性能** | 更高（无线程开销） | 较低（线程切换开销） |
| **适用** | 生产环境、高性能 | 兼容性好、cache=writeback |

### 6.6 virtio-net收发包路径（vhost-kernel）

```
vhost-kernel：IO数据路径在内核中处理，绕过QEMU用户空间

发包路径（Guest → 外部网络）：
Guest应用 send()
  ↓
Guest TCP/IP栈 → virtio-net前端驱动
  ↓ 将skb数据放入TX Virtqueue的Descriptor
  ↓ 更新Available Ring → kick
  ↓
Host vhost-net内核线程
  ↓ 从TX Virtqueue取出数据
  ↓ 写入TAP设备的fd
  ↓
Host TCP/IP栈或Bridge
  ↓
物理网卡 → 外部网络

收包路径（外部网络 → Guest）：
物理网卡
  ↓
Host Bridge/TAP设备
  ↓ TAP fd可读
vhost-net内核线程
  ↓ 从RX Virtqueue的Descriptor获取空闲缓冲区
  ↓ 将网络数据写入缓冲区
  ↓ 更新Used Ring → irqfd注入中断
  ↓
Guest virtio-net驱动中断处理
  ↓ 从RX Used Ring获取接收到的数据
  ↓ 构建skb → 送入Guest TCP/IP栈
  ↓
Guest应用 recv()
```

**多队列控制**：
```bash
# 启用多队列virtio-net（队列数建议 = vCPU数）
# libvirt XML
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
  <driver name='vhost' queues='4'/>  # 4个队列
</interface>

# Guest内启用多队列
ethtool -L eth0 combined 4

# 查看队列绑定
cat /proc/interrupts | grep virtio
#  40: 1234 0 0 0  PCI-MSI  virtio0-input.0    ← 队列0绑定CPU0
#  41: 0 5678 0 0  PCI-MSI  virtio0-input.1    ← 队列1绑定CPU1

# 手动绑定中断到CPU
echo 1 > /proc/irq/40/smp_affinity  # 队列0 → CPU0
echo 2 > /proc/irq/41/smp_affinity  # 队列1 → CPU1
```

---

## Part 1 总结
### 核心知识点
1. **虚拟化类型**：全虚拟化、半虚拟化、容器虚拟化
2. **KVM架构**：Type 1裸金属虚拟化，CPU/内存/中断虚拟化
3. **QEMU角色**：设备模拟器，virtio半虚拟化驱动
4. **libvirt管理**：统一API，virsh命令行工具
5. **镜像格式**：raw（性能）vs qcow2（功能）
6. **虚拟化深入**：QEMU线程模型、VMCS上下文切换、EPT缺页处理、Posted Interrupt、virtio vring通信、IO请求完整路径

### 本章自查
1. **KVM vs QEMU vs qemu-kvm的区别？**
   - KVM：内核模块，CPU/内存虚拟化
   - QEMU：用户空间，设备模拟
   - qemu-kvm：两者结合，硬件加速

2. **VM Exit为什么影响性能？**
   - 每次Exit需要保存/恢复上下文（开销约1-5μs）
   - 优化方法：使用virtio减少Exit次数

3. **qcow2 backing file的应用场景？**
   - 快速创建虚拟机（链式克隆）
   - 节省磁盘空间（共享base镜像）

4. **如何选择镜像格式？**
   - 性能敏感：raw + 预分配
   - 功能需求：qcow2（快照、压缩）

### 下一步
- **Part 2**：虚拟机网络（桥接、NAT、vSwitch）
- **Part 3**：虚拟机迁移（热迁移、冷迁移）
- **Part 4**：物理网络与虚拟化网络
- **Part 5**：服务器底层技术（BMC、BIOS、DPDK）

---

**关键工具速查**：
```bash
# KVM验证
lsmod | grep kvm
cat /sys/module/kvm_intel/parameters/nested

# QEMU启动
qemu-system-x86_64 -enable-kvm -m 4G -smp 4

# libvirt管理
virsh list --all
virsh dominfo vm1

# 镜像操作
qemu-img create -f qcow2 disk.qcow2 10G
qemu-img info disk.qcow2
```# IaaS基础设施技术 - Part 2：虚拟机网络

> **核心能力**：虚拟机网络模式、虚拟交换机、网络性能优化  
> **文件规模**：Part2约600行，虚拟机网络技术

---

## 📑 Part 2 目录

1. [虚拟机网络模式](#1-虚拟机网络模式)
2. [Linux网络虚拟化组件](#2-linux网络虚拟化组件)
3. [虚拟机网络配置实战](#3-虚拟机网络配置实战)
4. [虚拟机迁移技术](#4-虚拟机迁移技术)
5. [VNC远程管理](#5-vnc远程管理)

---

## 1. 虚拟机网络模式

### 1.1 三种网络模式对比

| 模式 | 原理 | 网络可见性 | 性能 | 适用场景 |
|------|------|-----------|------|---------|
| **桥接（Bridge）** | 虚拟机直接连接物理网络 | 与Host在同一网段 | ⭐⭐⭐⭐⭐ | 生产环境、需要外部访问 |
| **NAT** | Host做网络地址转换 | 虚拟机在私有网段 | ⭐⭐⭐⭐ | 开发测试、节省IP |
| **用户模式（User）** | QEMU内部网络栈模拟 | 完全隔离 | ⭐⭐ | 简单测试、无需root |

### 1.2 桥接模式（Bridge）

#### 工作原理
```
┌──────────────────────────────────────────┐
│         Physical Network (192.168.1.0/24) │
│                                            │
│  Router (192.168.1.1)                     │
└──────────────────────────────────────────┘
                    │
┌───────────────────┴──────────────────────┐
│         Host Machine                      │
│                                           │
│  ┌──────────────────────────────────┐    │
│  │  br0 (Linux Bridge)              │    │
│  │  IP: 192.168.1.100               │    │
│  └──────────────────────────────────┘    │
│         │              │                  │
│    ┌────┴────┐    ┌────┴────┐           │
│    │  eth0   │    │  vnet0  │           │
│    │ 物理网卡│    │ TAP设备 │           │
│    └─────────┘    └─────────┘           │
│                         │                 │
│                    ┌─────────┐           │
│                    │   VM1   │           │
│                    │ 192.168.1.101       │
│                    └─────────┘           │
└───────────────────────────────────────────┘
```

#### 配置步骤

**1. 创建Linux Bridge**
```bash
# 安装bridge工具
apt-get install bridge-utils   # Debian/Ubuntu
yum install bridge-utils        # CentOS

# 创建网桥
brctl addbr br0

# 将物理网卡加入网桥
brctl addif br0 eth0

# 配置网桥IP（原eth0 IP移到br0）
ip addr del 192.168.1.100/24 dev eth0
ip addr add 192.168.1.100/24 dev br0

# 启动网桥和物理网卡
ip link set dev br0 up
ip link set dev eth0 up

# 验证
brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.000c29abcdef       no              eth0
                                                        vnet0
```

**2. 永久配置（Ubuntu）**
```bash
# /etc/network/interfaces
auto eth0
iface eth0 inet manual

auto br0
iface br0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    bridge_ports eth0
    bridge_stp off
    bridge_fd 0
```

**3. 启动虚拟机（桥接模式）**
```bash
# QEMU命令行
qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -drive file=vm1.qcow2,if=virtio \
  -netdev tap,id=net0,ifname=tap0,script=/etc/qemu-ifup,downscript=/etc/qemu-ifdown \
  -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:56

# /etc/qemu-ifup（TAP设备启动脚本）
#!/bin/bash
ip link set $1 up
brctl addif br0 $1

# /etc/qemu-ifdown（TAP设备关闭脚本）
#!/bin/bash
brctl delif br0 $1
ip link set $1 down
```

**4. libvirt桥接配置**
```xml
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
  <mac address='52:54:00:12:34:56'/>
</interface>
```

### 1.3 NAT模式

#### 工作原理
```
┌──────────────────────────────────────────┐
│         Physical Network (192.168.1.0/24) │
└──────────────────────────────────────────┘
                    │
┌───────────────────┴──────────────────────┐
│         Host Machine (192.168.1.100)     │
│                                           │
│  ┌──────────────────────────────────┐    │
│  │  virbr0 (NAT Bridge)             │    │
│  │  IP: 192.168.122.1               │    │
│  │  DHCP: 192.168.122.2-254         │    │
│  └──────────────────────────────────┘    │
│         │              │                  │
│    ┌────┴────┐    ┌────┴────┐           │
│    │ iptables│    │  vnet0  │           │
│    │  SNAT   │    │         │           │
│    └─────────┘    └─────────┘           │
│                         │                 │
│                    ┌─────────┐           │
│                    │   VM1   │           │
│                    │ 192.168.122.100     │
│                    └─────────┘           │
└───────────────────────────────────────────┘

# 数据包流向（VM1 -> 外网）：
192.168.122.100:1234 -> 8.8.8.8:53 (DNS查询)
              ↓ SNAT (iptables)
192.168.1.100:5678 -> 8.8.8.8:53 (源地址转换)
```

#### 配置步骤

**1. libvirt默认NAT网络**
```bash
# libvirt自动创建default网络（NAT模式）
virsh net-list
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes

# 查看default网络配置
virsh net-dumpxml default
<network>
  <name>default</name>
  <uuid>xxx</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>

# iptables规则（自动生成）
iptables -t nat -L -n -v
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  192.168.122.0/24    !192.168.122.0/24
```

**2. 手动配置NAT网络**
```bash
# 创建网桥
brctl addbr virbr1
ip addr add 10.0.0.1/24 dev virbr1
ip link set virbr1 up

# 启用IP转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 配置iptables SNAT
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE

# 启动dnsmasq提供DHCP和DNS
dnsmasq --interface=virbr1 \
        --bind-interfaces \
        --dhcp-range=10.0.0.100,10.0.0.200,12h
```

**3. 端口转发（让外部访问VM）**
```bash
# 将Host的8080端口转发到VM的80端口
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.122.100:80

# 验证
curl http://192.168.1.100:8080  # 访问Host IP，转发到VM
```

### 1.4 用户模式（User Mode）

#### 特点
- 无需root权限
- QEMU内部实现网络栈（libslirp）
- 性能最差（纯软件实现）
- 只能VM访问外网，外网无法主动访问VM

```bash
# 启动虚拟机（用户模式）
qemu-system-x86_64 \
  -m 2048 \
  -drive file=vm.qcow2,if=virtio \
  -net nic,model=virtio \
  -net user,hostfwd=tcp::2222-:22  # 端口转发：Host 2222 -> VM 22

# 从Host SSH到VM
ssh -p 2222 user@localhost
```

---

## 2. Linux网络虚拟化组件

### 2.1 TAP/TUN设备

| 设备类型 | 工作层次 | 数据包格式 | 用途 |
|---------|---------|-----------|------|
| **TUN** | 三层（IP层） | IP数据包 | VPN（OpenVPN） |
| **TAP** | 二层（链路层） | 以太网帧 | 虚拟机网络、网桥 |

#### TAP设备工作原理
```
┌────────────────────────────────────┐
│       User Space (QEMU)            │
│  ┌──────────────────────────┐     │
│  │  virtio-net backend      │     │
│  └──────────────────────────┘     │
│              ↕                     │
│       read/write fd                │
└────────────────────────────────────┘
                ↕
┌────────────────────────────────────┐
│       Kernel Space                 │
│  ┌──────────────────────────┐     │
│  │  tap0 (TAP Device)       │     │
│  └──────────────────────────┘     │
│              ↕                     │
│  ┌──────────────────────────┐     │
│  │  br0 (Bridge)            │     │
│  └──────────────────────────┘     │
└────────────────────────────────────┘
```

#### 创建与管理TAP设备
```bash
# 创建TAP设备
ip tuntap add mode tap tap0
ip link set dev tap0 up

# 加入网桥
brctl addif br0 tap0

# 查看TAP设备
ip tuntap list
tap0: tap

# 删除TAP设备
ip tuntap del mode tap tap0

# 查看TAP设备统计
ifconfig tap0
tap0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        RX packets 123456  bytes 89012345 (84.9 MiB)
        TX packets 234567  bytes 123456789 (117.7 MiB)
```

### 2.2 veth pair（虚拟网线）

**概念**：成对出现的虚拟网卡，一端发送的数据从另一端接收

```
┌────────────────┐          ┌────────────────┐
│  Container A   │          │  Container B   │
│                │          │                │
│  eth0 (vethA)  │          │  eth0 (vethB)  │
└────────────────┘          └────────────────┘
        ↕                           ↕
┌─────────────────────────────────────────────┐
│            Host Network Stack               │
│  vethA-host ←──────→ vethB-host            │
└─────────────────────────────────────────────┘
```

```bash
# 创建veth pair
ip link add veth0 type veth peer name veth1

# 将一端放入容器
ip link set veth1 netns container_ns

# 配置IP
ip addr add 10.0.0.1/24 dev veth0
ip netns exec container_ns ip addr add 10.0.0.2/24 dev veth1

# 启用
ip link set veth0 up
ip netns exec container_ns ip link set veth1 up

# 测试连通性
ping 10.0.0.2
```

### 2.3 macvlan/macvtap

**macvlan**：在一个物理网卡上创建多个虚拟网卡，每个有独立MAC地址

```
┌─────────────────────────────────────────┐
│       Physical NIC (eth0)               │
│       MAC: 00:11:22:33:44:55            │
└─────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
    macvlan1    macvlan2    macvlan3
    MAC: :01    MAC: :02    MAC: :03
        ↓           ↓           ↓
      VM1         VM2         VM3
```

**四种模式**：
| 模式 | 特点 | 用途 |
|------|------|------|
| **Bridge** | 同父接口的macvlan间可通信 | 容器网络 |
| **VEPA** | 流量必须经过外部交换机 | 数据中心 |
| **Private** | 完全隔离 | 安全隔离 |
| **Passthru** | 直通模式 | SR-IOV |

```bash
# 创建macvlan接口
ip link add macvlan1 link eth0 type macvlan mode bridge

# 配置IP
ip addr add 192.168.1.101/24 dev macvlan1
ip link set macvlan1 up

# 用于容器
docker run --net=macvlan --ip=192.168.1.102 myapp
```

---

## 3. 虚拟机网络配置实战

### 3.1 libvirt网络类型

| 类型 | 配置 | 说明 |
|------|------|------|
| **NAT网络** | `<forward mode='nat'/>` | 默认，私有网段+SNAT |
| **路由网络** | `<forward mode='route'/>` | 需手动配置路由 |
| **桥接网络** | `<forward mode='bridge'/>` | 直接连接物理网络 |
| **隔离网络** | 无`<forward>`标签 | 虚拟机间通信 |
| **SR-IOV** | `<forward mode='hostdev'/>` | 网卡直通 |

### 3.2 创建自定义libvirt网络

**示例1：隔离网络（VM间互通，不能上外网）**
```xml
<!-- isolated-net.xml -->
<network>
  <name>isolated</name>
  <bridge name='virbr-isolated' stp='on' delay='0'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.10.10.100' end='10.10.10.200'/>
    </dhcp>
  </ip>
</network>
```

```bash
# 定义网络
virsh net-define isolated-net.xml

# 启动网络
virsh net-start isolated

# 设置自动启动
virsh net-autostart isolated

# 虚拟机使用该网络
virsh edit vm1
<interface type='network'>
  <source network='isolated'/>
</interface>
```

**示例2：多个DHCP主机预留**
```xml
<network>
  <name>dhcp-reserved</name>
  <bridge name='virbr1'/>
  <forward mode='nat'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.100' end='192.168.100.200'/>
      <!-- MAC地址预留固定IP -->
      <host mac='52:54:00:11:22:33' ip='192.168.100.10' name='vm-web'/>
      <host mac='52:54:00:11:22:44' ip='192.168.100.11' name='vm-db'/>
    </dhcp>
  </ip>
</network>
```

### 3.3 网络性能优化

**1. 多队列virtio-net**
```xml
<!-- 启用多队列（队列数=vCPU数） -->
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
  <driver name='vhost' queues='4'/>
</interface>
```

```bash
# 在Guest内启用多队列
ethtool -L eth0 combined 4

# 验证
ethtool -l eth0
Channel parameters for eth0:
Pre-set maximums:
Combined:       4
Current hardware settings:
Combined:       4
```

**2. vhost-net（内核加速）**
```bash
# 加载vhost-net模块
modprobe vhost-net

# QEMU启动参数
-netdev tap,id=net0,vhost=on

# 或libvirt XML
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
  <driver name='vhost'/>
</interface>
```

**3. 网卡offload功能**
```bash
# 在Guest内启用TSO/GSO/GRO
ethtool -K eth0 tso on
ethtool -K eth0 gso on
ethtool -K eth0 gro on

# 查看offload状态
ethtool -k eth0
```

**4. 网络性能测试**
```bash
# Host -> VM 网络带宽测试（iperf3）
# 在VM内启动服务端
iperf3 -s

# 在Host启动客户端
iperf3 -c 192.168.122.100 -P 4 -t 30
[  5]   0.00-30.00  sec  34.2 GBytes  9.80 Gbits/sec  (接近10Gbps网卡)

# 延迟测试（ping）
ping -c 100 192.168.122.100
100 packets transmitted, 100 received, 0% packet loss
rtt min/avg/max/mdev = 0.112/0.234/0.456/0.089 ms
```

---

## 4. 虚拟机迁移技术

### 4.1 冷迁移 vs 热迁移

| 特性 | 冷迁移 | 热迁移（Live Migration） |
|------|--------|-------------------------|
| **虚拟机状态** | 关机 | 运行中 |
| **业务中断** | 完全中断 | 几乎无感（<1秒） |
| **迁移时间** | 短（只传输磁盘） | 长（传输内存+磁盘） |
| **复杂度** | 简单 | 复杂 |
| **适用场景** | 维护窗口、非关键业务 | 7x24业务、负载均衡 |

### 4.2 热迁移原理

```
源Host                          目标Host
┌─────────────┐                ┌─────────────┐
│    VM       │                │             │
│  运行中     │                │             │
└─────────────┘                └─────────────┘

第一阶段：预复制（Pre-copy）
┌─────────────┐                ┌─────────────┐
│    VM       │ ──内存数据───→ │  VM (准备)  │
│  继续运行   │                │             │
└─────────────┘                └─────────────┘
  ↓ 脏页继续产生                 ↓ 接收内存

第二阶段：迭代复制
┌─────────────┐                ┌─────────────┐
│    VM       │ ──脏页数据───→ │  VM (准备)  │
│  运行变慢   │                │             │
└─────────────┘                └─────────────┘
  ↓ 脏页速度降低                 ↓ 接收脏页

第三阶段：Stop-and-Copy（黑障期）
┌─────────────┐                ┌─────────────┐
│    VM       │ ──剩余脏页───→ │  VM (准备)  │
│  **暂停**   │                │             │
└─────────────┘                └─────────────┘

第四阶段：切换
┌─────────────┐                ┌─────────────┐
│   VM (销毁) │                │    VM       │
│             │                │  **运行**   │
└─────────────┘                └─────────────┘
```

### 4.3 热迁移实战

**前提条件**：
1. 共享存储（NFS/Ceph）或块迁移
2. 相同CPU架构
3. 网络互通
4. libvirt版本一致

**1. 共享存储热迁移**
```bash
# 挂载共享存储（两台Host都执行）
mount -t nfs 192.168.1.200:/vms /var/lib/libvirt/images

# 执行热迁移
virsh migrate --live --persistent \
  vm1 qemu+ssh://host2/system

# 带宽限制（避免占满网络）
virsh migrate --live --persistent \
  --bandwidth 500 \
  vm1 qemu+ssh://host2/system

# 查看迁移进度
virsh domjobinfo vm1
Job type:         Unbounded
Time elapsed:     5234 ms
Data processed:   2.1 GiB
Data remaining:   512 MiB
Memory processed: 2.0 GiB
Memory remaining: 500 MiB
```

**2. 块迁移（无共享存储）**
```bash
# 同时迁移磁盘和内存
virsh migrate --live --persistent \
  --copy-storage-all \
  vm1 qemu+ssh://host2/system
```

**3. 热迁移参数优化**
```bash
# 调整迁移参数（减少黑障时间）
virsh migrate-setmaxdowntime vm1 500   # 最大黑障时间500ms
virsh migrate-setspeed vm1 1000        # 迁移带宽1000 MiB/s

# 自动收敛（自动降低VM速度）
virsh migrate --live --auto-converge vm1 qemu+ssh://host2/system
```

### 4.4 冷迁移

```bash
# 1. 关闭虚拟机
virsh shutdown vm1

# 2. 导出XML配置
virsh dumpxml vm1 > vm1.xml

# 3. 传输磁盘镜像
rsync -avz --progress /var/lib/libvirt/images/vm1.qcow2 \
  host2:/var/lib/libvirt/images/

# 4. 在目标Host定义虚拟机
virsh define vm1.xml

# 5. 启动虚拟机
virsh start vm1

# 6. 删除源Host上的虚拟机
virsh undefine vm1 --remove-all-storage
```

---

## 5. VNC远程管理

### 5.1 VNC架构

```
┌────────────────────────────────────┐
│       VNC Client                   │
│  (VNC Viewer/TigerVNC)             │
└────────────────────────────────────┘
                ↕ RFB协议（5900+端口）
┌────────────────────────────────────┐
│       Host Machine                 │
│  ┌──────────────────────────┐     │
│  │  QEMU Process            │     │
│  │  ├─ VNC Server           │     │
│  │  │  监听 127.0.0.1:5901   │     │
│  │  └─ 屏幕缓冲区           │     │
│  └──────────────────────────┘     │
│              ↕                     │
│  ┌──────────────────────────┐     │
│  │  Guest VM                │     │
│  │  显卡输出                │     │
│  └──────────────────────────┘     │
└────────────────────────────────────┘
```

### 5.2 配置VNC

**QEMU命令行**：
```bash
qemu-system-x86_64 \
  -vnc :1                # 监听5901端口（5900+1）
  # 或
  -vnc 0.0.0.0:1         # 允许外部访问
  # 或
  -vnc :1,password       # 启用VNC密码

# 设置VNC密码（QEMU Monitor）
virsh qemu-monitor-command vm1 --hmp "change vnc password"
Password: ********
```

**libvirt XML**：
```xml
<graphics type='vnc' port='5901' autoport='no' listen='0.0.0.0'>
  <listen type='address' address='0.0.0.0'/>
</graphics>

<!-- 或自动分配端口 -->
<graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'/>
```

### 5.3 VNC客户端连接

```bash
# 查看VNC端口
virsh vncdisplay vm1
:1  # 表示5901端口

# 使用vncviewer连接
vncviewer 192.168.1.100:1
# 或
vncviewer 192.168.1.100:5901

# TigerVNC（性能更好）
vncviewer -SecurityTypes VncAuth 192.168.1.100:5901
```

### 5.4 VNC安全加固

**1. 使用SSH隧道**
```bash
# 在客户端建立SSH隧道
ssh -L 5901:localhost:5901 user@host

# 然后连接本地端口
vncviewer localhost:5901
```

**2. TLS加密**
```xml
<graphics type='vnc' port='5901'>
  <listen type='address' address='0.0.0.0'/>
  <tls port='5902'/>
</graphics>
```

---

## Part 2 总结
### 核心知识点
1. **三种网络模式**：桥接（生产）、NAT（测试）、用户模式（开发）
2. **TAP/veth/macvlan**：Linux网络虚拟化基础组件
3. **热迁移原理**：预复制 + 迭代复制 + Stop-and-Copy
4. **VNC远程管理**：虚拟机图形界面访问

### 性能优化要点
- 使用virtio-net + vhost + 多队列
- 启用网卡offload（TSO/GSO/GRO）
- 桥接模式 > NAT > 用户模式

### 本章自查
1. **桥接 vs NAT的区别？**
   - 桥接：VM与Host同网段，外部可直接访问
   - NAT：VM在私有网段，通过SNAT访问外网

2. **热迁移如何保证业务不中断？**
   - 预复制阶段：VM继续运行，传输大部分内存
   - Stop-and-Copy：暂停<1秒，传输剩余脏页
   - 目标Host启动VM，网络切换

3. **TAP vs veth的区别？**
   - TAP：用于虚拟机，连接到Bridge
   - veth：用于容器，成对出现

### 下一步
- **Part 3**：物理网络与虚拟化网络（VLAN/VPC/GRE）
- **Part 4**：服务器底层技术（BMC/BIOS/DPDK）
- **Part 5**：存储虚拟化（LVM/Ceph）

---

**关键命令速查**：
```bash
# 网桥管理
brctl addbr br0
brctl addif br0 eth0

# TAP设备
ip tuntap add mode tap tap0

# 热迁移
virsh migrate --live vm1 qemu+ssh://host2/system

# VNC
virsh vncdisplay vm1
vncviewer 192.168.1.100:5901
```# IaaS基础设施技术 - Part 3：物理网络与虚拟化网络

> **核心能力**：VLAN、VPC、GRE隧道、VXLAN、网卡绑定  
> **文件规模**：Part3约600行，网络虚拟化深度技术

---

## 📑 Part 3 目录

1. [物理网络基础](#1-物理网络基础)
2. [VLAN虚拟局域网](#2-vlan虚拟局域网)
3. [网络隧道技术](#3-网络隧道技术)
4. [VXLAN大二层网络](#4-vxlan大二层网络)
5. [VPC虚拟私有云](#5-vpc虚拟私有云)

---

## 1. 物理网络基础

### 1.1 网卡绑定（Bonding）

#### 七种Bonding模式

| 模式 | 名称 | 特性 | 容错 | 负载均衡 | 用途 |
|------|------|------|------|---------|------|
| **Mode 0** | balance-rr | 轮询 | ✅ | ✅ | 高吞吐量 |
| **Mode 1** | active-backup | 主备 | ✅ | ❌ | 高可用（推荐） |
| **Mode 2** | balance-xor | XOR哈希 | ✅ | ✅ | 需要交换机支持 |
| **Mode 3** | broadcast | 广播 | ✅ | ❌ | 特殊场景 |
| **Mode 4** | 802.3ad (LACP) | 动态链路聚合 | ✅ | ✅ | 企业级（推荐） |
| **Mode 5** | balance-tlb | 传输负载均衡 | ✅ | ✅（单向） | 无需交换机配置 |
| **Mode 6** | balance-alb | 自适应负载均衡 | ✅ | ✅（双向） | 无需交换机配置 |

#### 配置Bonding

**Mode 1（主备模式）配置**：
```bash
# 1. 加载bonding模块
modprobe bonding

# 2. 创建bonding接口
cat > /etc/sysconfig/network-scripts/ifcfg-bond0 <<EOF
DEVICE=bond0
TYPE=Bond
BONDING_MASTER=yes
BOOTPROTO=static
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
BONDING_OPTS="mode=1 miimon=100"
EOF

# 3. 配置从接口
cat > /etc/sysconfig/network-scripts/ifcfg-eth0 <<EOF
DEVICE=eth0
TYPE=Ethernet
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
EOF

cat > /etc/sysconfig/network-scripts/ifcfg-eth1 <<EOF
DEVICE=eth1
TYPE=Ethernet
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
EOF

# 4. 重启网络
systemctl restart network

# 5. 验证
cat /proc/net/bonding/bond0
Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: eth0
MII Status: up
MII Polling Interval (ms): 100

Slave Interface: eth0
MII Status: up
Speed: 10000 Mbps
Duplex: full

Slave Interface: eth1
MII Status: up
Speed: 10000 Mbps
Duplex: full
```

**Mode 4（LACP）配置**：
```bash
# Bonding配置
BONDING_OPTS="mode=4 miimon=100 lacp_rate=fast xmit_hash_policy=layer3+4"

# 交换机配置（Cisco示例）
interface Port-channel1
  description Server-Bond
  switchport mode trunk
  
interface GigabitEthernet1/0/1
  channel-group 1 mode active
  
interface GigabitEthernet1/0/2
  channel-group 1 mode active
```

**测试故障切换**：
```bash
# 模拟网卡故障
ifconfig eth0 down

# 查看切换情况
cat /proc/net/bonding/bond0
Currently Active Slave: eth1  # 自动切换到eth1

# 恢复
ifconfig eth0 up
```

### 1.2 交换机堆叠（Stacking）

**堆叠 vs 非堆叠**：
```
非堆叠（传统）：
┌─────────┐   ┌─────────┐
│ Switch1 │   │ Switch2 │
│  单独   │   │  单独   │
│  管理   │   │  管理   │
└─────────┘   └─────────┘
管理IP: .1     管理IP: .2

堆叠（Stacking）：
┌─────────────────────────┐
│  Virtual Switch Stack   │
│  ┌─────────┬─────────┐  │
│  │ Switch1 │ Switch2 │  │
│  │ (Master)│ (Slave) │  │
│  └─────────┴─────────┘  │
│  统一管理IP: .1         │
└─────────────────────────┘
    ↕ 堆叠线缆（480Gbps）
```

**优势**：
- 统一管理（一个管理IP）
- 跨交换机链路聚合
- 故障自动切换
- 简化配置

**配置示例（华为）**：
```bash
# Switch1（主交换机）
[Huawei] irf member 1 priority 32
[Huawei] irf-port 1/1
[Huawei-irf-port1/1] port group interface Ten-GigabitEthernet 1/0/49

# Switch2（从交换机）
[Huawei] irf member 2 priority 1
[Huawei] irf-port 2/2
[Huawei-irf-port2/2] port group interface Ten-GigabitEthernet 2/0/49

# 验证
[Huawei] display irf
MemberID    Role    Priority  CPU-Mac         Status
1           Master  32        00e0-fc12-3456  Normal
2           Slave   1         00e0-fc12-7890  Normal
```

---

## 2. VLAN虚拟局域网

### 2.1 VLAN原理

**VLAN的作用**：
1. 隔离广播域（减少广播风暴）
2. 逻辑划分网络（安全隔离）
3. 灵活的网络规划

**VLAN标签（802.1Q）**：
```
以太网帧结构（无VLAN）：
┌────────┬────────┬──────┬─────────┬─────┐
│ Dest   │ Source │ Type │ Payload │ FCS │
│ MAC    │ MAC    │      │         │     │
│ 6字节  │ 6字节  │2字节 │46-1500B │4字节│
└────────┴────────┴──────┴─────────┴─────┘

以太网帧结构（带VLAN）：
┌────────┬────────┬──────┬─────────┬──────┬─────────┬─────┐
│ Dest   │ Source │ TPID │ TCI     │ Type │ Payload │ FCS │
│ MAC    │ MAC    │0x8100│VLAN ID  │      │         │     │
│ 6字节  │ 6字节  │2字节 │2字节    │2字节 │         │     │
└────────┴────────┴──────┴─────────┴──────┴─────────┴─────┘
                          ↑
                    VLAN ID: 12位（1-4094）
```

### 2.2 VLAN端口类型

| 端口类型 | 说明 | VLAN标签处理 | 用途 |
|---------|------|-------------|------|
| **Access** | 接入端口 | 入：打标签<br>出：去标签 | 连接终端设备 |
| **Trunk** | 干道端口 | 透传多个VLAN | 连接交换机/路由器 |
| **Hybrid** | 混合端口 | 灵活配置 | 特殊场景 |

### 2.3 Linux VLAN配置

```bash
# 1. 加载8021q模块
modprobe 8021q

# 2. 创建VLAN接口
ip link add link eth0 name eth0.10 type vlan id 10
ip link add link eth0 name eth0.20 type vlan id 20

# 3. 配置IP
ip addr add 10.0.0.1/24 dev eth0.10
ip addr add 10.0.0.2/24 dev eth0.20

# 4. 启用接口
ip link set eth0.10 up
ip link set eth0.20 up

# 5. 验证
ip -d link show eth0.10
5: eth0.10@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 00:0c:29:12:34:56 brd ff:ff:ff:ff:ff:ff
    vlan protocol 802.1Q id 10 <REORDER_HDR>

# 永久配置（CentOS）
cat > /etc/sysconfig/network-scripts/ifcfg-eth0.10 <<EOF
DEVICE=eth0.10
BOOTPROTO=static
IPADDR=10.0.0.1
NETMASK=255.255.255.0
VLAN=yes
EOF
```

### 2.4 交换机VLAN配置

**Cisco交换机**：
```bash
# 创建VLAN
Switch(config)# vlan 10
Switch(config-vlan)# name WEB-Servers
Switch(config-vlan)# exit

Switch(config)# vlan 20
Switch(config-vlan)# name DB-Servers
Switch(config-vlan)# exit

# 配置Access端口
Switch(config)# interface GigabitEthernet1/0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10

# 配置Trunk端口
Switch(config)# interface GigabitEthernet1/0/24
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30

# 验证
Switch# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi1/0/2, Gi1/0/3
10   WEB-Servers                      active    Gi1/0/1
20   DB-Servers                       active    Gi1/0/5
```

### 2.5 虚拟机使用VLAN

**libvirt配置**：
```xml
<interface type='bridge'>
  <source bridge='br0'/>
  <vlan>
    <tag id='10'/>
  </vlan>
  <model type='virtio'/>
</interface>
```

**Open vSwitch配置**：
```bash
# 创建OVS网桥
ovs-vsctl add-br ovs-br0

# 添加物理网卡（trunk口）
ovs-vsctl add-port ovs-br0 eth0

# 添加虚拟机端口（access口，VLAN 10）
ovs-vsctl add-port ovs-br0 vnet0 tag=10

# 验证
ovs-vsctl show
Bridge "ovs-br0"
    Port "eth0"
        Interface "eth0"
    Port "vnet0"
        tag: 10
        Interface "vnet0"
```

---

## 3. 网络隧道技术

### 3.1 GRE隧道

**GRE（Generic Routing Encapsulation）**：三层隧道协议

```
原始数据包：
┌──────┬──────┬─────────┐
│ IP   │ TCP  │ Payload │
│ Hdr  │ Hdr  │         │
└──────┴──────┴─────────┘

GRE封装后：
┌───────────┬──────┬──────┬──────┬─────────┐
│ Outer IP  │ GRE  │ IP   │ TCP  │ Payload │
│ Hdr       │ Hdr  │ Hdr  │ Hdr  │         │
└───────────┴──────┴──────┴──────┴─────────┘
    ↑           ↑
  隧道端点    GRE协议(47)
```

**应用场景**：
- 连接异地数据中心
- VPN
- 云平台跨可用区互联

**配置GRE隧道**：
```bash
# === Site A (192.168.1.100) ===
# 创建GRE隧道
ip tunnel add gre1 mode gre \
  local 192.168.1.100 \
  remote 203.0.113.20 \
  ttl 255

# 配置隧道IP
ip addr add 10.0.0.2/30 dev gre1
ip link set gre1 up

# 添加路由（Site B内网通过隧道）
ip route add 192.168.100.1/24 dev gre1

# === Site B (203.0.113.20) ===
ip tunnel add gre1 mode gre \
  local 203.0.113.20 \
  remote 192.168.1.100 \
  ttl 255

ip addr add 4.26.149.182/30 dev gre1
ip link set gre1 up

ip route add 10.0.0.1/24 dev gre1

# === 验证 ===
# Site A ping Site B内网
ping 192.168.100.10  # 通过GRE隧道
traceroute 192.168.100.10
1  4.26.149.182 (GRE隧道对端)  1.234 ms
2  192.168.100.10 (目标主机)  2.456 ms
```

**GRE over IPsec（加密隧道）**：
```bash
# 先建立IPsec SA，再创建GRE隧道
# /etc/ipsec.conf
conn site-to-site
    left=192.168.1.100
    right=203.0.113.20
    authby=secret
    auto=start
    type=transport  # 传输模式（GRE over IPsec）
```

### 3.2 IPIP隧道

**IPIP vs GRE**：
| 特性 | IPIP | GRE |
|------|------|-----|
| **协议号** | 4 | 47 |
| **开销** | 20字节（IP头） | 24字节（IP+GRE头） |
| **承载协议** | 仅IP | IP/IPX/AppleTalk |
| **性能** | 略优 | 略差 |

```bash
# 创建IPIP隧道
ip tunnel add ipip1 mode ipip \
  local 192.168.1.100 \
  remote 203.0.113.20

ip addr add 10.0.0.2/30 dev ipip1
ip link set ipip1 up
```

### 3.3 VxLAN vs GRE

| 特性 | GRE | VxLAN |
|------|-----|-------|
| **工作层次** | 三层 | 二层（大二层网络） |
| **封装开销** | 24字节 | 50字节（UDP+VxLAN） |
| **VLAN数量** | 无限制 | 1600万个 |
| **适用场景** | 点对点隧道 | 数据中心大二层 |

---

## 4. VXLAN大二层网络

### 4.1 VXLAN原理

**为什么需要VXLAN？**
1. VLAN只支持4096个（12位），不够用
2. 数据中心需要大二层网络（虚拟机迁移）
3. 传统三层网络无法满足云平台需求

**VXLAN封装格式**：
```
原始以太网帧：
┌────────┬────────┬──────┬─────────┬─────┐
│ Dest   │ Source │ Type │ Payload │ FCS │
│ MAC    │ MAC    │      │         │     │
└────────┴────────┴──────┴─────────┴─────┘

VXLAN封装后：
┌──────┬──────┬──────┬────────┬────────┬────────┬──────┬─────────┬─────┐
│Outer │Outer │ UDP  │ VXLAN  │ Inner  │ Inner  │ Type │ Payload │ FCS │
│ IP   │ MAC  │ Hdr  │ Header │ Dest   │ Source │      │         │     │
│      │      │      │(VNI 24)│ MAC    │ MAC    │      │         │     │
└──────┴──────┴──────┴────────┴────────┴────────┴──────┴─────────┴─────┘
                        ↑
                   VNI: 24位（1600万个网络）
```

### 4.2 VXLAN组件

```
┌─────────────────────────────────────────────┐
│            Overlay Network                  │
│     ┌─────────┐         ┌─────────┐        │
│     │  VM1    │         │  VM2    │        │
│     │ VXLAN10 │         │ VXLAN10 │        │
│     └─────────┘         └─────────┘        │
└─────────────────────────────────────────────┘
        ↕ VTEP               ↕ VTEP
┌─────────────────────────────────────────────┐
│         Underlay Network (三层网络)         │
│       Host1 (192.168.1.100) ←──→ Host2 (192.168.1.101)      │
└─────────────────────────────────────────────┘

VTEP (VXLAN Tunnel Endpoint)：
- 封装/解封装VXLAN数据包
- 维护MAC-IP映射表
```

### 4.3 Linux VXLAN配置

```bash
# === Host1 (192.168.1.100) ===
# 创建VXLAN接口（VNI 10）
ip link add vxlan10 type vxlan \
  id 10 \
  dstport 4789 \
  local 192.168.1.100 \
  dev eth0

# 创建网桥
brctl addbr br-vxlan10
brctl addif br-vxlan10 vxlan10

# 将虚拟机网卡加入网桥
brctl addif br-vxlan10 vnet0

ip link set vxlan10 up
ip link set br-vxlan10 up

# 添加远端VTEP（Host2）
bridge fdb append 00:00:00:00:00:00 dev vxlan10 dst 192.168.1.101

# === Host2 (192.168.1.101) ===
ip link add vxlan10 type vxlan \
  id 10 \
  dstport 4789 \
  local 192.168.1.101 \
  dev eth0

brctl addbr br-vxlan10
brctl addif br-vxlan10 vxlan10
brctl addif br-vxlan10 vnet0

ip link set vxlan10 up
ip link set br-vxlan10 up

bridge fdb append 00:00:00:00:00:00 dev vxlan10 dst 192.168.1.100

# === 验证 ===
# 在VM1中ping VM2（同一VXLAN）
ping 192.168.122.100  # 通过VXLAN隧道

# 查看FDB表（转发数据库）
bridge fdb show dev vxlan10
00:00:00:00:00:00 dst 192.168.1.101 self permanent
52:54:00:12:34:56 dst 192.168.1.101 self  # VM2的MAC学习到的
```

### 4.4 Open vSwitch + VXLAN

```bash
# 创建OVS网桥
ovs-vsctl add-br br-int

# 添加VXLAN隧道
ovs-vsctl add-port br-int vxlan1 \
  -- set interface vxlan1 type=vxlan \
     options:remote_ip=192.168.1.101 \
     options:key=10

# 添加虚拟机端口
ovs-vsctl add-port br-int vnet0

# 查看端口
ovs-vsctl show
Bridge "br-int"
    Port "vxlan1"
        Interface "vxlan1"
            type: vxlan
            options: {key="10", remote_ip="192.168.1.101"}
    Port "vnet0"
        Interface "vnet0"
```

### 4.5 VXLAN性能测试

```bash
# 测试环境：
# - 物理机：10Gbps网卡
# - CPU：Intel Xeon（支持offload）

# 1. 无VXLAN（直接桥接）
iperf3 -c 192.168.122.100
[  5]   0.00-10.00  sec  11.2 GBytes  9.62 Gbits/sec

# 2. VXLAN（默认）
iperf3 -c 192.168.122.100
[  5]   0.00-10.00  sec  6.8 GBytes  5.84 Gbits/sec  (约60%性能)

# 3. VXLAN + Offload（网卡硬件加速）
ethtool -K eth0 tx-udp_tnl-segmentation on
iperf3 -c 192.168.122.100
[  5]   0.00-10.00  sec  10.1 GBytes  8.67 Gbits/sec  (约90%性能)

# 结论：启用硬件offload可显著提升VXLAN性能
```

---

## 5. VPC虚拟私有云

### 5.1 VPC架构

```
┌───────────────────────────────────────────────────────┐
│                    VPC (82.158.87.228/16)                       │
│                                                       │
│  ┌───────────────────────────┐  ┌────────────────┐  │
│  │  Subnet-1 (公有子网)       │  │ Subnet-2 (私有)│  │
│  │  82.158.87.2280/24               │  │ 82.158.87.2281/24    │  │
│  │                           │  │                │  │
│  │  ┌─────┐  ┌─────┐        │  │  ┌─────┐      │  │
│  │  │ VM1 │  │ VM2 │        │  │  │ VM3 │      │  │
│  │  │Web  │  │Web  │        │  │  │ DB  │      │  │
│  │  └─────┘  └─────┘        │  │  └─────┘      │  │
│  │      ↓                    │  │                │  │
│  │  ┌──────────────┐        │  │                │  │
│  │  │ EIP (弹性IP) │        │  │                │  │
│  │  │ 203.0.113.10 │        │  │                │  │
│  │  └──────────────┘        │  │                │  │
│  └───────────────────────────┘  └────────────────┘  │
│                ↓                         ↓           │
│  ┌─────────────────────────────────────────┐        │
│  │        Virtual Router                   │        │
│  │  - NAT Gateway                          │        │
│  │  - Security Group                       │        │
│  │  - Route Table                          │        │
│  └─────────────────────────────────────────┘        │
└───────────────────────────────────────────────────────┘
                    ↓
        ┌────────────────────┐
        │  Internet Gateway  │
        └────────────────────┘
```

### 5.2 VPC核心组件

| 组件 | 作用 | 示例 |
|------|------|------|
| **VPC** | 隔离的虚拟网络 | 82.158.87.228/16 |
| **Subnet** | VPC内的子网 | 公有子网、私有子网 |
| **Route Table** | 路由表 | 0.0.0.0/0 -> IGW |
| **Internet Gateway** | 公网网关 | 访问互联网 |
| **NAT Gateway** | NAT网关 | 私有子网访问外网 |
| **Security Group** | 安全组（状态防火墙） | 允许80/443端口 |
| **Network ACL** | 网络ACL（无状态） | 子网级别过滤 |
| **EIP** | 弹性公网IP | 可动态绑定 |

### 5.3 VPC网络隔离实现

**方案1：VXLAN + 路由表**
```bash
# VPC-1（VXLAN 100）
ip link add vxlan100 type vxlan id 100 dstport 4789
ip addr add 82.158.87.2280/16 dev vxlan100

# VPC-2（VXLAN 200）
ip link add vxlan200 type vxlan id 200 dstport 4789
ip addr add 192.168.100.10/16 dev vxlan200

# 路由隔离
ip route add 82.158.87.228/16 dev vxlan100 table 100
ip route add 192.168.100.1/16 dev vxlan200 table 200

# 虚拟机根据VPC使用不同路由表
ip rule add from 82.158.87.2280/24 table 100
ip rule add from 192.168.100.10/24 table 200
```

**方案2：Network Namespace + OVS**
```bash
# 为每个VPC创建独立的Network Namespace
ip netns add vpc-100
ip netns add vpc-200

# 创建veth pair连接Namespace和OVS
ip link add veth-vpc100 type veth peer name veth-ns100
ip link set veth-ns100 netns vpc-100

# 在Namespace内配置网络
ip netns exec vpc-100 ip addr add 82.158.87.2280/16 dev veth-ns100
ip netns exec vpc-100 ip link set veth-ns100 up

# OVS配置（不同VPC使用不同VXLAN VNI）
ovs-vsctl add-port br-int veth-vpc100 tag=100
```

### 5.4 Security Group实现

**iptables实现安全组**：
```bash
# Security Group: web-sg
# 规则：允许入80/443，允许所有出流量

# 为每个VM创建iptables链
iptables -N SG-WEB-IN
iptables -N SG-WEB-OUT

# 入方向规则
iptables -A SG-WEB-IN -p tcp --dport 80 -j ACCEPT
iptables -A SG-WEB-IN -p tcp --dport 443 -j ACCEPT
iptables -A SG-WEB-IN -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A SG-WEB-IN -j DROP

# 出方向规则
iptables -A SG-WEB-OUT -j ACCEPT

# 应用到VM接口
iptables -A FORWARD -i vnet0 -j SG-WEB-OUT
iptables -A FORWARD -o vnet0 -j SG-WEB-IN

# 查看规则命中数
iptables -L SG-WEB-IN -v -n
Chain SG-WEB-IN (1 references)
 pkts bytes target     prot opt in     out     source               destination
  123  45K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80
   45  20K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:443
```

**OVS + OpenFlow实现安全组**：
```bash
# 允许入80/443端口
ovs-ofctl add-flow br-int \
  "table=0,priority=100,tcp,tp_dst=80,actions=normal"
ovs-ofctl add-flow br-int \
  "table=0,priority=100,tcp,tp_dst=443,actions=normal"

# 默认拒绝
ovs-ofctl add-flow br-int \
  "table=0,priority=1,actions=drop"
```

### 5.5 NAT Gateway实现

```bash
# 创建NAT网关（在VPC路由器上）
# 私有子网：82.158.87.2281/24
# 公网IP：203.0.113.50

# 启用IP转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# SNAT规则（私有子网访问外网）
iptables -t nat -A POSTROUTING \
  -s 82.158.87.2281/24 \
  -o eth0 \
  -j SNAT --to-source 203.0.113.50

# 或使用MASQUERADE（动态IP）
iptables -t nat -A POSTROUTING \
  -s 82.158.87.2281/24 \
  -o eth0 \
  -j MASQUERADE

# DNAT规则（端口转发，让外部访问私有VM）
iptables -t nat -A PREROUTING \
  -d 203.0.113.50 -p tcp --dport 8080 \
  -j DNAT --to-destination 82.158.87.22810:80

# 查看NAT连接跟踪
conntrack -L | grep 82.158.87.22810
tcp 6 431998 ESTABLISHED src=82.158.87.22810 dst=8.8.8.8 sport=54321 dport=443 \
                         src=8.8.8.8 dst=203.0.113.50 sport=443 dport=54321
```

---

## Part 3 总结
### 核心知识点
1. **Bonding模式**：Mode 1（主备）、Mode 4（LACP）生产环境常用
2. **VLAN**：二层隔离，4096个网络（12位）
3. **GRE/VXLAN**：隧道技术，VXLAN支持1600万网络（24位）
4. **VPC架构**：Subnet、路由表、安全组、NAT网关

### 技术选型建议
| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| **网卡高可用** | Bonding Mode 1 | 简单可靠 |
| **网卡带宽聚合** | Bonding Mode 4 (LACP) | 需交换机支持 |
| **点对点隧道** | GRE | 开销小 |
| **数据中心大二层** | VXLAN | 支持1600万网络 |
| **云平台网络隔离** | VPC (VXLAN + Namespace) | 完全隔离 |

### 本章自查
1. **VLAN vs VXLAN？**
   - VLAN：4096个（12位），二层隔离
   - VXLAN：1600万（24位），支持大二层

2. **GRE vs VXLAN？**
   - GRE：三层隧道，点对点
   - VXLAN：二层隧道（UDP封装），多点

3. **Security Group vs Network ACL？**
   - SG：状态防火墙，VM级别
   - ACL：无状态，Subnet级别

4. **Bonding Mode 1 vs Mode 4？**
   - Mode 1：主备，无需交换机配置
   - Mode 4：LACP，带宽聚合，需交换机支持

### 下一步
- **Part 4**：服务器底层技术（BMC/BIOS/DPDK/SPDK/RDMA）
- **Part 5**：存储虚拟化（LVM/Ceph）

---

**关键命令速查**：
```bash
# Bonding
cat /proc/net/bonding/bond0

# VLAN
ip link add link eth0 name eth0.10 type vlan id 10

# GRE
ip tunnel add gre1 mode gre local 192.168.1.100 remote 203.0.113.20

# VXLAN
ip link add vxlan10 type vxlan id 10 dstport 4789

# Security Group
iptables -A FORWARD -i vnet0 -p tcp --dport 80 -j ACCEPT
```# IaaS基础设施技术 - Part 4：服务器底层技术

> **核心能力**：BMC、BIOS、DPDK、SPDK、RDMA、LVM  
> **文件规模**：Part4约600行，服务器底层与高性能技术

---

## 📑 Part 4 目录

1. [服务器管理技术](#1-服务器管理技术)
2. [高性能网络DPDK](#2-高性能网络dpdk)
3. [高性能存储SPDK](#3-高性能存储spdk)
4. [RDMA远程直接内存访问](#4-rdma远程直接内存访问)
5. [LVM逻辑卷管理](#5-lvm逻辑卷管理)

---

## 1. 服务器管理技术

### 1.1 BMC（Baseboard Management Controller）

**BMC是什么？**
- 服务器主板上的独立芯片
- 即使服务器关机也能运行
- 提供带外管理（Out-of-Band Management）

```
┌───────────────────────────────────────┐
│         Server (关机状态)             │
│                                       │
│  ┌─────────────────────────────┐     │
│  │  CPU  Memory  Disk          │     │
│  │  (Power OFF)                │     │
│  └─────────────────────────────┘     │
│                                       │
│  ┌─────────────────────────────┐     │
│  │  BMC Chip (独立运行)        │     │
│  │  ├─ IPMI协议               │     │
│  │  ├─ 传感器监控             │     │
│  │  ├─ 远程电源控制           │     │
│  │  └─ KVM over IP             │     │
│  └─────────────────────────────┘     │
│              ↕                        │
│        独立网口（带外网口）           │
└───────────────────────────────────────┘
              ↕
    ┌──────────────────┐
    │  管理员电脑      │
    │  ipmitool/Web UI │
    └──────────────────┘
```

**BMC功能**：
| 功能 | 说明 | 用途 |
|------|------|------|
| **远程电源管理** | 开机/关机/重启 | 故障恢复 |
| **传感器监控** | 温度/风扇/电压 | 硬件健康检查 |
| **日志记录** | SEL（System Event Log） | 故障诊断 |
| **KVM over IP** | 远程桌面（图形界面） | 操作系统安装/维护 |
| **虚拟媒体** | 挂载ISO镜像 | 远程安装系统 |
| **固件升级** | BIOS/BMC固件更新 | 功能更新/安全补丁 |

### 1.2 IPMI（Intelligent Platform Management Interface）

**ipmitool常用命令**：
```bash
# === 基础信息 ===
# 查看BMC信息
ipmitool bmc info
Device ID                 : 32
Device Revision           : 1
Firmware Revision         : 1.23
IPMI Version              : 2.0
Manufacturer ID           : 19046

# 查看传感器
ipmitool sensor list
CPU Temp         | 45.000     | degrees C  | ok
System Temp      | 32.000     | degrees C  | ok
Fan1             | 3600.000   | RPM        | ok
Fan2             | 3750.000   | RPM        | ok
12V              | 12.096     | Volts      | ok
5V               | 5.024      | Volts      | ok

# === 电源管理 ===
# 查看电源状态
ipmitool power status
Chassis Power is on

# 电源操作
ipmitool power on          # 开机
ipmitool power off         # 强制关机（拔电源）
ipmitool power cycle       # 重启（硬重启）
ipmitool power soft        # 正常关机（ACPI）

# === 日志管理 ===
# 查看系统事件日志
ipmitool sel list
   1 | 01/15/2026 | 10:23:45 | Temperature CPU | Upper Critical going high
   2 | 01/15/2026 | 10:24:00 | Fan #1 | Lower Critical going low

# 清空日志
ipmitool sel clear

# === SOL（Serial Over LAN）===
# 激活串口重定向
ipmitool sol activate
[SOL Session operational]
# 按 ~. 退出

# === 虚拟媒体 ===
# 挂载ISO镜像（通过网络）
ipmitool -I lanplus -H 187.36.96.249 -U admin -P password \
  raw 0x00 0x08 0x05 0xe0 0x00 0x00 0x00 0x05 0x00

# === 网络配置 ===
# 查看BMC IP配置
ipmitool lan print 1
IP Address              : 187.36.96.249
Subnet Mask             : 255.255.255.0
Default Gateway IP      : 245.196.103.2

# 设置BMC IP（静态）
ipmitool lan set 1 ipsrc static
ipmitool lan set 1 ipaddr 187.36.96.249
ipmitool lan set 1 netmask 255.255.255.0
ipmitool lan set 1 defgw ipaddr 245.196.103.2

# 设置BMC密码
ipmitool user set password 2 newpassword
```

### 1.3 BIOS与UEFI

**BIOS vs UEFI**：
| 特性 | Legacy BIOS | UEFI |
|------|------------|------|
| **启动速度** | 慢 | 快 |
| **磁盘分区** | MBR（最大2TB） | GPT（支持9.4ZB） |
| **界面** | 文本 | 图形+鼠标 |
| **安全启动** | 不支持 | Secure Boot |
| **网络启动** | PXE | HTTP Boot |

**BIOS配置要点**：
```bash
# === 虚拟化相关 ===
Intel VT-x/AMD-V      : Enabled  # CPU虚拟化
Intel VT-d/AMD-Vi     : Enabled  # IO虚拟化（PCI直通）
SR-IOV                : Enabled  # 单根IO虚拟化

# === 性能相关 ===
Hyper-Threading       : Enabled  # 超线程
Turbo Boost           : Enabled  # 睿频加速
C-States              : Disabled # 省电模式（性能场景禁用）
NUMA                  : Enabled  # 非统一内存访问

# === 电源管理 ===
Power Profile         : Maximum Performance

# === 启动设备顺序 ===
Boot Mode             : UEFI
Boot Order            : 1. Hard Drive  2. PXE  3. USB
```

**远程修改BIOS设置（通过BMC）**：
```bash
# Dell服务器（racadm）
racadm set BIOS.ProcSettings.LogicalProc Enabled
racadm jobqueue create BIOS.Setup.1-1

# HP服务器（hponcfg）
hponcfg -w config.xml
```

### 1.4 带外管理最佳实践

**网络架构**：
```
┌──────────────────────────────────┐
│      Management Network          │
│      (187.36.96.240/24)  VLAN 100 │
│                                  │
│  管理员 ←──→ BMC1, BMC2, BMC3   │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│      Production Network          │
│      (93.107.109.1/24)     VLAN 200 │
│                                  │
│  用户 ←──→ Server1, Server2     │
└──────────────────────────────────┘

原则：
1. 带外网络与业务网络物理隔离
2. 带外网络不连接互联网
3. 使用VPN访问带外网络
```

**安全加固**：
```bash
# 1. 修改默认密码
ipmitool user set password 2 StrongPassword123!

# 2. 禁用不必要的用户
ipmitool user disable 3

# 3. 限制访问IP
# 在BMC Web界面配置IP白名单

# 4. 启用加密（IPMI 2.0）
ipmitool lan set 1 cipher_privs 3,4

# 5. 定期更新固件
# 下载最新BMC固件并升级
```

---

## 2. 高性能网络DPDK

### 2.1 DPDK原理

**传统网络 IO vs DPDK**：

```
传统网络栈（Kernel Space）：
应用程序 (User Space)
    ↓ System Call
┌─────────────────────────────┐
│  Socket API                 │
│  ↓                          │
│  TCP/IP Stack (Kernel)      │
│  ↓                          │
│  Network Driver             │
│  ↓ 中断 + 上下文切换         │
│  NIC Hardware               │
└─────────────────────────────┘
问题：
- 中断频繁（每个包都触发）
- 上下文切换开销大
- 内存拷贝多次

DPDK（User Space）：
应用程序 (User Space)
    ↓ 直接调用DPDK API
┌─────────────────────────────┐
│  DPDK PMD (Poll Mode Driver)│
│  ├─ 轮询（无中断）          │
│  ├─ 零拷贝                  │
│  └─ CPU亲和性绑定           │
│  ↓ UIO/VFIO（绕过内核）     │
│  NIC Hardware               │
└─────────────────────────────┘
优势：
- 无中断（轮询）
- 零拷贝（DMA）
- 性能：14.88 Mpps（64字节小包）
```

### 2.2 DPDK核心技术

**1. UIO/VFIO（用户空间驱动）**
```bash
# 加载VFIO模块（推荐，支持IOMMU）
modprobe vfio-pci

# 绑定网卡到VFIO
# 1. 查看网卡PCI地址
lspci | grep Ethernet
02:00.0 Ethernet controller: Intel Corporation 82599ES

# 2. 解绑内核驱动
echo "0000:02:00.0" > /sys/bus/pci/drivers/ixgbe/unbind

# 3. 绑定DPDK驱动
dpdk-devbind.py --bind=vfio-pci 02:00.0

# 验证
dpdk-devbind.py --status
Network devices using DPDK-compatible driver
============================================
0000:02:00.0 '82599ES' drv=vfio-pci unused=ixgbe
```

**2. 巨页（Hugepages）**
```bash
# 配置2MB巨页（1024个 = 2GB）
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# 挂载hugetlbfs
mkdir -p /mnt/huge
mount -t hugetlbfs nodev /mnt/huge

# 或配置1GB巨页（需要CPU支持）
echo 4 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# 永久配置（/etc/sysctl.conf）
vm.nr_hugepages = 1024

# 验证
cat /proc/meminfo | grep Huge
HugePages_Total:    1024
HugePages_Free:     1020
Hugepagesize:       2048 kB
```

**3. CPU亲和性**
```bash
# 绑定DPDK进程到特定CPU核心
# testpmd示例（DPDK测试工具）
dpdk-testpmd -l 0-3 -n 4 -- -i \
  --rxq=2 --txq=2
# -l 0-3：使用CPU 0-3
# --rxq=2：2个接收队列
# --txq=2：2个发送队列

# 查看CPU亲和性
taskset -cp <pid>
pid 12345's current affinity list: 0-3
```

### 2.3 DPDK性能测试

```bash
# === 环境准备 ===
# 1. 安装DPDK
apt-get install dpdk dpdk-dev

# 2. 绑定网卡
dpdk-devbind.py --bind=vfio-pci 02:00.0 02:00.1

# === 性能测试（testpmd）===
# 发送端
dpdk-testpmd -l 0-3 -n 4 -- \
  --forward-mode=txonly \
  --tx-pkt-size=64 \
  --txq=4 --rxq=4

# 接收端
dpdk-testpmd -l 0-3 -n 4 -- \
  --forward-mode=rxonly \
  --rxq=4 --txq=4

# 测试结果（10GbE网卡，64字节小包）
Tx-pps:      14,880,952  # 14.88 Mpps（接近线速）
Tx-bps:   7,618,566,144  # 7.6 Gbps

# 对比：Linux内核网络栈
# 性能：~1.5 Mpps（仅为DPDK的10%）
```

### 2.4 DPDK应用场景

| 场景 | 说明 | 示例 |
|------|------|------|
| **NFV（网络功能虚拟化）** | 虚拟路由器/防火墙 | OVS-DPDK、VPP |
| **高频交易** | 低延迟网络通信 | 金融交易系统 |
| **负载均衡** | 高性能LB | DPVS、Katran |
| **CDN** | 内容分发网络 | Nginx+DPDK |

**OVS-DPDK配置**：
```bash
# 安装OVS-DPDK
apt-get install openvswitch-switch-dpdk

# 启用DPDK
ovs-vsctl set Open_vSwitch . other_config:dpdk-init=true
systemctl restart openvswitch-switch

# 创建DPDK网桥
ovs-vsctl add-br br0 -- set bridge br0 datapath_type=netdev

# 添加DPDK端口
ovs-vsctl add-port br0 dpdk0 \
  -- set Interface dpdk0 type=dpdk \
     options:dpdk-devargs=0000:02:00.0

# 性能测试
# 传统OVS：~2 Mpps
# OVS-DPDK：~10 Mpps（5倍提升）
```

---

## 3. 高性能存储SPDK

### 3.1 SPDK原理

**传统存储IO vs SPDK**：

```
传统块设备IO：
应用程序
    ↓ System Call
┌─────────────────────┐
│  VFS                │
│  ↓                  │
│  Block Layer        │
│  ↓                  │
│  SCSI Layer         │
│  ↓ 中断             │
│  NVMe Driver        │
│  ↓                  │
│  NVMe SSD           │
└─────────────────────┘
延迟：~10-20μs

SPDK（用户空间）：
应用程序
    ↓ SPDK API
┌─────────────────────┐
│  SPDK NVMe Driver   │
│  ├─ 轮询            │
│  ├─ 零拷贝          │
│  └─ 无锁队列        │
│  ↓ UIO/VFIO         │
│  NVMe SSD           │
└─────────────────────┘
延迟：~2-3μs（减少70%）
```

### 3.2 SPDK核心组件

```
┌────────────────────────────────────┐
│       SPDK Application             │
└────────────────────────────────────┘
                ↓
┌────────────────────────────────────┐
│      SPDK Blockdev Layer           │
│  ├─ bdev_nvme (NVMe)               │
│  ├─ bdev_aio (Linux AIO)           │
│  ├─ bdev_raid (RAID)               │
│  └─ bdev_lvol (逻辑卷)             │
└────────────────────────────────────┘
                ↓
┌────────────────────────────────────┐
│      SPDK NVMe Driver              │
│  ├─ I/O Queue Pair                 │
│  ├─ Poll Mode (轮询)               │
│  └─ DMA (直接内存访问)             │
└────────────────────────────────────┘
                ↓
┌────────────────────────────────────┐
│      NVMe SSD                      │
└────────────────────────────────────┘
```

### 3.3 SPDK配置与使用

```bash
# === 安装SPDK ===
git clone https://github.com/spdk/spdk
cd spdk
./scripts/pkgdep.sh
./configure
make

# === 配置巨页 ===
./scripts/setup.sh
# 自动配置：
# - 巨页（2GB）
# - 绑定NVMe设备到UIO/VFIO

# === 查看NVMe设备 ===
./scripts/setup.sh status
NVMe devices
0000:03:00.0 (144d a808): nvme0n1 (Samsung SSD 970 EVO)

# === 启动SPDK target（iSCSI） ===
./app/spdk_tgt/spdk_tgt &

# 创建块设备
./scripts/rpc.py bdev_nvme_attach_controller \
  -b Nvme0 -t PCIe -a 0000:03:00.0

# 创建iSCSI target
./scripts/rpc.py iscsi_create_portal_group 1 0.0.0.0:3260
./scripts/rpc.py iscsi_create_initiator_group 2 ANY 0.0.0.0/0
./scripts/rpc.py iscsi_create_target_node \
  iqn.2026-01.com.example:target0 \
  "Nvme0n1:0" "1:2" 64 -d

# 客户端挂载
iscsiadm -m discovery -t st -p 187.36.96.249
iscsiadm -m node --login
```

### 3.4 SPDK性能测试

```bash
# fio测试（4K随机读）
# 传统NVMe驱动
fio --name=randread --ioengine=libaio --direct=1 \
    --bs=4k --iodepth=128 --rw=randread \
    --filename=/dev/nvme0n1 --runtime=60
IOPS: 450,000
Latency: 28μs

# SPDK fio插件
fio --name=randread --ioengine=spdk --direct=1 \
    --bs=4k --iodepth=128 --rw=randread \
    --filename=trtype=PCIe traddr=0000:03:00.0 ns=1 \
    --runtime=60
IOPS: 720,000 (提升60%)
Latency: 17μs (减少40%)
```

---

## 4. RDMA远程直接内存访问

### 4.1 RDMA原理

**传统网络通信 vs RDMA**：

```
传统TCP/IP（需CPU参与）：
App A                  App B
  ↓                      ↑
Socket                Socket
  ↓                      ↑
TCP/IP Stack          TCP/IP Stack
  ↓                      ↑
Network Driver        Network Driver
  ↓                      ↑
   NIC ←──────网络──────→ NIC
   
CPU参与：拷贝、协议处理、中断

RDMA（CPU零参与）：
App A                  App B
  ↓                      ↑
RDMA Verbs            RDMA Verbs
  ↓                      ↑
   RNIC ←─────网络──────→ RNIC
          (DMA直接读写远端内存)
          
特点：
- 零拷贝（Zero-Copy）
- 零CPU（Kernel Bypass）
- 低延迟（<1μs）
```

### 4.2 RDMA技术栈

| 技术 | 说明 | 性能 | 成本 |
|------|------|------|------|
| **InfiniBand** | 专用RDMA协议 | 最高（200Gbps） | 最高 |
| **RoCE v2** | RDMA over Ethernet | 高（100Gbps） | 中 |
| **iWARP** | RDMA over TCP | 中（40Gbps） | 低 |

### 4.3 RDMA配置（RoCE）

```bash
# === 安装RDMA软件栈 ===
apt-get install rdma-core libibverbs1 ibverbs-utils

# === 加载驱动 ===
modprobe mlx5_core  # Mellanox网卡
modprobe rdma_rxe   # 软件RoCE（测试用）

# === 查看RDMA设备 ===
ibv_devices
device                 node GUID
------              ----------------
mlx5_0              248a0703006a1234
mlx5_1              248a0703006a5678

# === 查看设备详细信息 ===
ibv_devinfo
hca_id: mlx5_0
        transport:                      InfiniBand (0)
        fw_ver:                         16.28.2006
        node_guid:                      248a:0703:006a:1234
        sys_image_guid:                 248a:0703:006a:1234
        vendor_id:                      0x02c9
        vendor_part_id:                 4119
        hw_ver:                         0x0
        phys_port_cnt:                  1
        port:   1
                state:                  PORT_ACTIVE (4)
                max_mtu:                4096 (5)
                active_mtu:             1024 (3)
                sm_lid:                 0
                port_lid:               0
                port_lmc:               0x00
                link_layer:             Ethernet

# === 配置网卡IP ===
ip addr add 192.168.1.100/24 dev mlx5_0
ip link set mlx5_0 up

# === 配置RoCE优先级（QoS）===
# 启用PFC（Priority Flow Control）
mlnx_qos -i mlx5_0 --pfc 0,0,0,1,0,0,0,0

# 配置DSCP to Priority映射
echo "0 1 2 3 4 5 6 7" > /sys/class/infiniband/mlx5_0/tc/1/traffic_class
```

### 4.4 RDMA性能测试

```bash
# === ib_send_bw（带宽测试）===
# 服务端
ib_send_bw -d mlx5_0 -i 1

# 客户端
ib_send_bw -d mlx5_0 -i 1 192.168.1.100

#                    Send BW Test
# Dual-port       : OFF          Device         : mlx5_0
# Number of qps   : 1            Transport type : IB
# Connection type : RC           Using SRQ      : OFF
# PCIe relax order: ON
# ibv_wr* API     : ON
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      5000           11728.63            11725.42             0.187606
---------------------------------------------------------------------------------------
# 结果：~94 Gbps（100Gbps网卡）

# === ib_send_lat（延迟测试）===
# 服务端
ib_send_lat -d mlx5_0

# 客户端
ib_send_lat -d mlx5_0 192.168.1.100

#                    Send Latency Test
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]
 2       1000          0.98           5.23         1.02               1.05
---------------------------------------------------------------------------------------
# 结果：~1μs延迟
```

### 4.5 RDMA应用场景

```bash
# 1. 存储（NVMe over Fabrics）
modprobe nvme-rdma
nvme connect -t rdma -n nqn.2026-01.com.example:nvme:1 \
  -a 192.168.1.100 -s 4420

# 2. 文件系统（NFS over RDMA）
mount -o rdma,port=20049 192.168.1.100:/export /mnt/nfs

# 3. 数据库（Oracle RAC）
# 使用RDMA替代传统TCP，延迟降低90%

# 4. 机器学习（分布式训练）
# TensorFlow/PyTorch使用RDMA进行梯度同步
# 性能提升：2-3倍
```

---

## 5. LVM逻辑卷管理

### 5.1 LVM架构

```
┌─────────────────────────────────────┐
│    Logical Volume (LV)              │
│    /dev/vg1/lv_data (100GB)         │
└─────────────────────────────────────┘
                ↓
┌─────────────────────────────────────┐
│    Volume Group (VG)                │
│    vg1 (200GB)                      │
└─────────────────────────────────────┘
                ↓
┌──────────┬──────────┬──────────────┐
│   PV1    │   PV2    │     PV3      │
│ /dev/sda │ /dev/sdb │  /dev/sdc    │
│  50GB    │  100GB   │   50GB       │
└──────────┴──────────┴──────────────┘
                ↓
┌──────────┬──────────┬──────────────┐
│  Disk 1  │  Disk 2  │   Disk 3     │
└──────────┴──────────┴──────────────┘
```

**核心概念**：
- **PV（Physical Volume）**：物理卷，硬盘或分区
- **VG（Volume Group）**：卷组，多个PV的集合
- **LV（Logical Volume）**：逻辑卷，从VG分配的逻辑磁盘
- **PE（Physical Extent）**：物理扩展单元，默认4MB

### 5.2 LVM创建与管理

```bash
# === 1. 创建物理卷 ===
pvcreate /dev/sdb /dev/sdc
  Physical volume "/dev/sdb" successfully created.
  Physical volume "/dev/sdc" successfully created.

# 查看PV
pvdisplay
  --- Physical volume ---
  PV Name               /dev/sdb
  VG Name               vg1
  PV Size               100.00 GiB
  Allocatable           yes
  PE Size               4.00 MiB
  Total PE              25599
  Free PE               12799

# === 2. 创建卷组 ===
vgcreate vg1 /dev/sdb /dev/sdc
  Volume group "vg1" successfully created

# 查看VG
vgdisplay vg1
  --- Volume group ---
  VG Name               vg1
  System ID
  Format                lvm2
  VG Size               199.99 GiB
  PE Size               4.00 MiB
  Total PE              51198
  Alloc PE / Size       25600 / 100.00 GiB
  Free  PE / Size       25598 / 99.99 GiB

# 扩展VG（添加新磁盘）
vgextend vg1 /dev/sdd

# === 3. 创建逻辑卷 ===
# 固定大小
lvcreate -L 100G -n lv_data vg1

# 使用百分比
lvcreate -l 100%FREE -n lv_backup vg1

# 查看LV
lvdisplay /dev/vg1/lv_data
  --- Logical volume ---
  LV Path                /dev/vg1/lv_data
  LV Name                lv_data
  VG Name                vg1
  LV Size                100.00 GiB

# === 4. 格式化并挂载 ===
mkfs.ext4 /dev/vg1/lv_data
mount /dev/vg1/lv_data /mnt/data

# 永久挂载（/etc/fstab）
echo "/dev/vg1/lv_data  /mnt/data  ext4  defaults  0 2" >> /etc/fstab
```

### 5.3 LVM高级功能

**1. 在线扩容**：
```bash
# 扩展LV（增加50GB）
lvextend -L +50G /dev/vg1/lv_data

# 或扩展到指定大小
lvextend -L 150G /dev/vg1/lv_data

# 扩展文件系统
resize2fs /dev/vg1/lv_data  # ext4
xfs_growfs /mnt/data        # xfs

# 验证
df -h /mnt/data
Filesystem                Size  Used Avail Use% Mounted on
/dev/mapper/vg1-lv_data  148G   10G  131G   8% /mnt/data
```

**2. 在线缩容（仅ext4，xfs不支持）**：
```bash
# 先卸载
umount /mnt/data

# 检查文件系统
e2fsck -f /dev/vg1/lv_data

# 缩小文件系统
resize2fs /dev/vg1/lv_data 80G

# 缩小LV
lvreduce -L 80G /dev/vg1/lv_data

# 重新挂载
mount /dev/vg1/lv_data /mnt/data
```

**3. LVM快照**：
```bash
# 创建快照（10GB空间，COW）
lvcreate -L 10G -s -n lv_data_snap /dev/vg1/lv_data

# 挂载快照
mount /dev/vg1/lv_data_snap /mnt/snap

# 恢复快照
umount /mnt/data
lvconvert --merge /dev/vg1/lv_data_snap
# 重启后生效

# 删除快照
lvremove /dev/vg1/lv_data_snap
```

**4. LVM精简配置（Thin Provisioning）**：
```bash
# 创建精简池
lvcreate -L 100G --thinpool vg1/thin_pool

# 创建精简卷（虚拟200GB，实际按需分配）
lvcreate -V 200G --thin -n lv_thin vg1/thin_pool

# 查看使用率
lvs -a
  LV              VG  Attr       LSize   Pool       Origin Data%
  lv_thin         vg1 Vwi-a-tz-- 200.00g thin_pool         12.34
  thin_pool       vg1 twi-aotz-- 100.00g                   24.68
```

### 5.4 LVM性能优化

```bash
# 1. 条带化（Striping，类似RAID 0）
# 将数据分散到多个PV，提高吞吐量
lvcreate -L 100G -i 3 -I 64 -n lv_stripe vg1
# -i 3：3个条带（3个磁盘）
# -I 64：条带大小64KB

# 性能对比（fio顺序读）
# 单磁盘：500 MB/s
# 3磁盘条带：1400 MB/s（接近3倍）

# 2. 镜像（Mirroring，类似RAID 1）
lvcreate -L 100G -m 1 -n lv_mirror vg1
# -m 1：1个镜像（2份数据）

# 3. RAID（LVM RAID）
lvcreate --type raid5 -L 300G -i 3 -n lv_raid5 vg1
# RAID5：3个数据盘 + 1个校验盘

# 4. 缓存（LVM Cache）
# 使用SSD加速HDD
lvcreate -L 500G -n lv_hdd vg_hdd
lvcreate -L 50G -n lv_cache vg_ssd
lvconvert --type cache --cachepool vg_ssd/lv_cache vg_hdd/lv_hdd
```

---

## Part 4 总结
### 核心知识点
1. **BMC/IPMI**：带外管理，独立于操作系统
2. **DPDK**：用户空间网络，性能提升10倍
3. **SPDK**：用户空间存储，延迟降低70%
4. **RDMA**：零拷贝零CPU，延迟<1μs
5. **LVM**：灵活的存储管理，支持快照/扩容

### 性能对比

| 技术 | 传统方案 | 新方案 | 性能提升 |
|------|---------|--------|---------|
| **网络** | Kernel网络栈 | DPDK | 10倍（1.5 -> 14.88 Mpps） |
| **存储** | Kernel块设备 | SPDK | 60%（450K -> 720K IOPS） |
| **RDMA** | TCP/IP | RoCE | 延迟降低90%（10μs -> 1μs） |

### 本章自查
1. **BMC vs BIOS？**
   - BMC：独立芯片，带外管理，服务器关机也能运行
   - BIOS：固件，启动时配置硬件

2. **DPDK为什么快？**
   - 无中断（轮询）
   - 零拷贝（DMA）
   - 用户空间驱动（绕过内核）

3. **RDMA适用场景？**
   - 低延迟：高频交易、HPC
   - 高带宽：分布式存储、机器学习

4. **LVM vs 分区？**
   - LVM：灵活，支持在线扩容/快照
   - 分区：简单，性能略优

### 下一步
- **Part 5**：操作系统运维深入（内核参数/排障工具）

---

**关键命令速查**：
```bash
# BMC/IPMI
ipmitool power status
ipmitool sensor list

# DPDK
dpdk-devbind.py --bind=vfio-pci 02:00.0
dpdk-testpmd -l 0-3 -n 4

# RDMA
ibv_devices
ib_send_bw -d mlx5_0

# LVM
pvcreate /dev/sdb
vgcreate vg1 /dev/sdb
lvcreate -L 100G -n lv_data vg1
lvextend -L +50G /dev/vg1/lv_data
```# IaaS基础设施技术 - Part 5：操作系统运维深入

> **核心能力**：内核参数调优、系统排障工具、性能分析  
> **文件规模**：Part5约600行，操作系统运维实战

---

## 📑 Part 5 目录

1. [Linux内核参数调优](#1-linux内核参数调优)
2. [系统排障工具](#2-系统排障工具)
3. [性能分析与监控](#3-性能分析与监控)
4. [文件系统排障](#4-文件系统排障)
5. [网络排障工具](#5-网络排障工具)
6. [内核分析工具](#6-内核分析工具)（ftrace、kprobe、kdump/crash、eBPF）

---

## 1. Linux内核参数调优

### 1.1 网络相关参数

```bash
# /etc/sysctl.conf 或 /etc/sysctl.d/99-custom.conf

# === TCP性能优化 ===
# TCP缓冲区大小（接收）
net.ipv4.tcp_rmem = 4096 87380 16777216
# 最小4KB，默认85KB，最大16MB

# TCP缓冲区大小（发送）
net.ipv4.tcp_wmem = 4096 65536 16777216

# TCP窗口缩放（支持大窗口）
net.ipv4.tcp_window_scaling = 1

# TCP时间戳（RTTM）
net.ipv4.tcp_timestamps = 1

# TCP快速打开（TFO）
net.ipv4.tcp_fastopen = 3
# 1: 作为客户端  2: 作为服务端  3: 两者

# === 连接队列 ===
# SYN队列大小（半连接）
net.ipv4.tcp_max_syn_backlog = 8192

# Accept队列大小（全连接）
net.core.somaxconn = 4096

# === TIME_WAIT优化 ===
# 允许TIME_WAIT socket重用（客户端）
net.ipv4.tcp_tw_reuse = 1

# TIME_WAIT socket最大数量
net.ipv4.tcp_max_tw_buckets = 5000

# === 拥塞控制算法 ===
# 使用BBR（推荐，需要内核4.9+）
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# === 其他优化 ===
# TCP keepalive参数
net.ipv4.tcp_keepalive_time = 600      # 10分钟
net.ipv4.tcp_keepalive_intvl = 10      # 探测间隔10秒
net.ipv4.tcp_keepalive_probes = 3      # 探测次数3次

# 最大文件句柄数
fs.file-max = 1000000

# 应用配置
sysctl -p  # 立即生效
```

**BBR vs Cubic性能对比**：
```bash
# 测试环境：100ms RTT，1% 丢包率

# Cubic（默认）
iperf3 -c server
[  5]   0.00-10.00  sec  2.1 GBytes  1.80 Gbits/sec

# BBR
iperf3 -c server
[  5]   0.00-10.00  sec  8.3 GBytes  7.13 Gbits/sec
# 性能提升约4倍
```

### 1.2 内存相关参数

```bash
# === 内存管理 ===
# Swappiness（0-100，越小越少使用swap）
vm.swappiness = 10
# 0: 仅在内存不足时swap
# 10: 数据库/缓存服务器推荐值
# 60: 默认值
# 100: 积极使用swap

# 脏页刷盘参数
vm.dirty_ratio = 10                # 脏页占总内存10%时阻塞写入
vm.dirty_background_ratio = 5      # 后台刷盘阈值5%
vm.dirty_expire_centisecs = 3000   # 脏页30秒后刷盘
vm.dirty_writeback_centisecs = 500 # 每5秒唤醒刷盘线程

# OOM Killer参数
vm.overcommit_memory = 0
# 0: 启发式（默认）
# 1: 允许过度分配（危险）
# 2: 严格控制（CommitLimit）

# 透明大页（THP）
# 数据库场景建议禁用（会导致延迟抖动）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### 1.3 文件系统参数

```bash
# === 文件描述符限制 ===
# /etc/security/limits.conf
*  soft  nofile  1000000
*  hard  nofile  1000000
*  soft  nproc   65535
*  hard  nproc   65535

# 验证
ulimit -n
1000000

# === inode缓存 ===
# inode和dentry缓存回收优先级
vm.vfs_cache_pressure = 50
# 默认100，越小越倾向于保留缓存

# === 文件系统挂载选项优化 ===
# ext4优化（/etc/fstab）
/dev/sda1  /data  ext4  noatime,nodiratime,data=writeback  0 2
# noatime: 不更新访问时间（性能提升20%）
# nodiratime: 不更新目录访问时间
# data=writeback: 异步写入（性能最高，安全性低）

# xfs优化
/dev/sdb1  /data  xfs  noatime,nodiratime,logbufs=8,logbsize=256k  0 2
# logbufs: 日志缓冲区数量
# logbsize: 日志缓冲区大小
```

### 1.4 CPU调度参数

```bash
# === CPU调度器参数 ===
# 进程调度粒度
kernel.sched_min_granularity_ns = 10000000  # 10ms
kernel.sched_wakeup_granularity_ns = 15000000  # 15ms

# CPU亲和性
# 将进程绑定到特定CPU核心
taskset -cp 0-3 <pid>  # 绑定到CPU 0-3

# === 中断亲和性（IRQ affinity）===
# 查看网卡中断
cat /proc/interrupts | grep eth0
 125:   12345678          PCI-MSI-edge      eth0-TxRx-0
 126:   23456789          PCI-MSI-edge      eth0-TxRx-1

# 绑定中断125到CPU 0
echo 1 > /proc/irq/125/smp_affinity_list  # CPU 0
echo 2 > /proc/irq/126/smp_affinity_list  # CPU 1

# 或使用irqbalance自动均衡
systemctl start irqbalance
```

---

## 2. 系统排障工具

### 2.1 mcelog - 硬件错误日志

**作用**：检测和记录CPU、内存硬件错误

```bash
# 安装mcelog
apt-get install mcelog  # Debian/Ubuntu
yum install mcelog      # CentOS

# 启动服务
systemctl start mcelog
systemctl enable mcelog

# 查看硬件错误
mcelog --client
# 无输出表示无错误

# 示例输出（有错误时）
Hardware event. This is not a software error.
CPU 2 BANK 4
TIME 1705392123 Mon Jan 15 10:15:23 2026
MCG status: MCIP
MCi status:
Corrected error, no action required.
MCA: MEMORY CONTROLLER RD_CHANNELunspecified_ERR
Transaction: Memory read error
Memory error in DIMM_A1 (Slot 1)
  Correctable ECC error
  
# 配置告警（/etc/mcelog/mcelog.conf）
[dimm]
dimm-tracking-enabled = yes
dmi-prepopulate = yes
uc-error-threshold = 5 / 24h
ce-error-threshold = 10 / 24h

# 内存错误超过阈值时，mcelog会触发告警
```

**常见硬件错误**：
| 错误类型 | 说明 | 处理方法 |
|---------|------|---------|
| **CE（Correctable Error）** | 可纠正错误（ECC自动修复） | 监控频率，频繁则更换内存 |
| **UCE（Uncorrectable Error）** | 不可纠正错误 | 立即更换内存/CPU |
| **Cache Error** | CPU缓存错误 | 检查CPU温度，可能需要更换 |

### 2.2 fsck - 文件系统检查修复

```bash
# === ext4文件系统检查 ===
# 必须先卸载文件系统
umount /dev/sda1

# 检查文件系统
fsck.ext4 -n /dev/sda1  # -n: 只读模式，不修复

# 自动修复
fsck.ext4 -y /dev/sda1  # -y: 自动回答yes

# 强制检查（即使标记为clean）
fsck.ext4 -f /dev/sda1

# 常见输出
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information

# 如果发现错误
/dev/sda1: ***** FILE SYSTEM WAS MODIFIED *****
/dev/sda1: 12345/655360 files (1.2% non-contiguous), 234567/2621440 blocks

# === xfs文件系统检查 ===
# xfs_repair必须在卸载状态下运行
umount /dev/sdb1

# 检查（只读）
xfs_repair -n /dev/sdb1

# 修复
xfs_repair /dev/sdb1

# === 启动时自动检查 ===
# 设置/dev/sda1下次启动时检查
tune2fs -c 1 /dev/sda1  # ext4

# 查看检查间隔
tune2fs -l /dev/sda1 | grep -i "mount count"
Mount count:              5
Maximum mount count:      30  # 每30次挂载检查一次
```

**文件系统损坏场景**：
1. **突然断电**：日志未刷盘，元数据不一致
2. **硬盘坏道**：数据读写错误
3. **内核bug**：文件系统驱动错误
4. **人为误操作**：直接修改块设备

### 2.3 ethtool - 网卡诊断

```bash
# === 查看网卡基本信息 ===
ethtool eth0
Settings for eth0:
        Supported ports: [ TP ]
        Supported link modes:   1000baseT/Full
                                10000baseT/Full
        Speed: 10000Mb/s
        Duplex: Full
        Port: Twisted Pair
        Link detected: yes

# === 查看网卡统计 ===
ethtool -S eth0
NIC statistics:
     rx_packets: 123456789
     tx_packets: 234567890
     rx_bytes: 100000000000
     tx_bytes: 200000000000
     rx_errors: 0         # 接收错误（重要！）
     tx_errors: 0         # 发送错误
     rx_dropped: 0        # 接收丢包
     tx_dropped: 0        # 发送丢包
     rx_crc_errors: 0     # CRC错误（线缆问题）
     rx_fifo_errors: 0    # FIFO溢出（CPU处理不过来）

# === 网卡offload功能 ===
ethtool -k eth0
Features for eth0:
rx-checksumming: on      # 接收校验和offload
tx-checksumming: on      # 发送校验和offload
scatter-gather: on       # 分散聚合
tcp-segmentation-offload: on  # TSO
generic-segmentation-offload: on  # GSO
generic-receive-offload: on  # GRO

# 启用/禁用offload
ethtool -K eth0 tso on
ethtool -K eth0 gro on

# === 网卡环形缓冲区 ===
ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:             4096
TX:             4096
Current hardware settings:
RX:             512      # 接收队列长度
TX:             512      # 发送队列长度

# 调大缓冲区（减少丢包）
ethtool -G eth0 rx 2048 tx 2048

# === 中断合并（Interrupt Coalescing）===
ethtool -c eth0
Coalesce parameters for eth0:
rx-usecs: 50             # 接收中断延迟50μs
tx-usecs: 50             # 发送中断延迟50μs

# 调整中断合并（降低中断频率，提高吞吐量）
ethtool -C eth0 rx-usecs 100 tx-usecs 100

# === 查看驱动版本 ===
ethtool -i eth0
driver: ixgbe
version: 5.1.0-k
firmware-version: 0x800009e0
bus-info: 0000:02:00.0
```

**网卡问题排查流程**：
```
1. 检查物理链路
   ethtool eth0 | grep "Link detected"
   
2. 检查错误统计
   ethtool -S eth0 | grep -i error
   
3. 检查丢包情况
   ethtool -S eth0 | grep -i drop
   
4. 检查硬件问题（CRC错误）
   如果rx_crc_errors增长 -> 检查网线/光模块
   
5. 检查性能瓶颈
   如果rx_fifo_errors增长 -> CPU处理不过来，调整IRQ亲和性
```

### 2.4 smartctl - 硬盘健康检查

```bash
# 安装smartmontools
apt-get install smartmontools

# === 查看硬盘信息 ===
smartctl -i /dev/sda
Model Family:     Samsung 860 EVO Series
Device Model:     Samsung SSD 860 EVO 500GB
Serial Number:    S3Z1NB0K123456
Firmware Version: RVT02B6Q
User Capacity:    500,107,862,016 bytes [500 GB]
Sector Size:      512 bytes logical/physical
SMART support is: Available - device has SMART capability.
SMART support is: Enabled

# === SMART健康检查 ===
smartctl -H /dev/sda
SMART overall-health self-assessment test result: PASSED

# === 详细SMART属性 ===
smartctl -A /dev/sda
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  5 Reallocated_Sector_Ct   0x0033   100   100   010    Pre-fail  Always       -       0
  9 Power_On_Hours          0x0032   099   099   000    Old_age   Always       -       1234
 12 Power_Cycle_Count       0x0032   099   099   000    Old_age   Always       -       567
177 Wear_Leveling_Count     0x0013   095   095   000    Pre-fail  Always       -       5
187 Reported_Uncorrect      0x0032   100   100   000    Old_age   Always       -       0
194 Temperature_Celsius     0x0022   042   050   000    Old_age   Always       -       42
199 UDMA_CRC_Error_Count    0x003e   100   100   000    Old_age   Always       -       0

# 关键指标：
# Reallocated_Sector_Ct: 重新分配扇区数（>0表示有坏道）
# Power_On_Hours: 通电时间
# Wear_Leveling_Count: SSD磨损程度（越低越旧）
# Temperature_Celsius: 温度
# UDMA_CRC_Error_Count: 数据传输错误（线缆问题）

# === SMART自检 ===
# 短自检（2分钟）
smartctl -t short /dev/sda

# 长自检（1-2小时）
smartctl -t long /dev/sda

# 查看自检结果
smartctl -l selftest /dev/sda
SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%      1234         -

# === 配置smartd监控 ===
# /etc/smartd.conf
/dev/sda -a -o on -S on -s (S/../.././02|L/../../6/03) -m admin@example.com
# -a: 监控所有属性
# -s: 定期自检（每天2点短自检，每周六3点长自检）
# -m: 告警邮件

systemctl enable smartd
systemctl start smartd
```

---

## 3. 性能分析与监控

### 3.1 perf - 性能分析工具

```bash
# 安装perf
apt-get install linux-tools-$(uname -r)

# === CPU性能分析 ===
# 统计系统调用
perf stat -e syscalls:* sleep 1

# CPU事件统计
perf stat -e cpu-cycles,instructions,cache-misses ./myapp
Performance counter stats for './myapp':
    1,234,567,890      cpu-cycles
    2,345,678,901      instructions              #    1.90  insn per cycle
       12,345,678      cache-misses              #    1.234 % of all cache refs

# === 火焰图（Flame Graph）===
# 1. 采集数据
perf record -F 99 -a -g -- sleep 60
# -F 99: 采样频率99Hz
# -a: 所有CPU
# -g: 调用栈

# 2. 生成报告
perf report
#    95.23%  myapp  [.] hot_function
#     2.34%  myapp  [.] cold_function

# 3. 生成火焰图
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flamegraph.svg

# === 实时监控 ===
# 实时查看热点函数
perf top
Samples: 12K of event 'cpu-clock', 4000 Hz, Event count (approx.): 123456789
Overhead  Shared Object       Symbol
  15.23%  myapp               [.] hot_function
   8.45%  [kernel]            [k] _raw_spin_lock
   5.67%  libc-2.31.so        [.] __memcpy_avx_unaligned
```

### 3.2 sar - 系统活动报告

```bash
# 安装sysstat
apt-get install sysstat

# 启用数据收集
systemctl enable sysstat
systemctl start sysstat

# === CPU使用率 ===
sar -u 1 10  # 每秒采样，共10次
12:00:01 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
12:00:02 PM     all     25.34      0.00     10.23      2.34      0.00     62.09

# === 内存使用 ===
sar -r 1 10
12:00:01 PM kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached
12:00:02 PM   1234567   8901234   6543210     65.43    123456   3456789

# === 磁盘IO ===
sar -d 1 10
12:00:01 PM       DEV       tps     rkB/s     wkB/s   await    svctm     %util
12:00:02 PM    dev8-0    123.45   5678.90   1234.56    2.34     1.23     15.23

# === 网络IO ===
sar -n DEV 1 10
12:00:01 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s
12:00:02 PM      eth0  12345.67  23456.78  12345.67  23456.78

# === 查看历史数据 ===
sar -u -f /var/log/sysstat/sa15  # 查看15号的CPU数据
```

### 3.3 iostat - IO统计

```bash
# === 磁盘IO统计 ===
iostat -x 1 10
Device   r/s   w/s   rkB/s   wkB/s  await  svctm  %util
sda    123.4 234.5 5678.9 12345.6   2.34   1.23  15.23

# 关键指标：
# r/s, w/s: 每秒读写次数
# rkB/s, wkB/s: 每秒读写KB数
# await: 平均IO等待时间（ms）
#   < 10ms: 正常
#   10-50ms: 较慢
#   > 50ms: 瓶颈
# svctm: 平均服务时间（ms）
# %util: 设备繁忙程度
#   < 60%: 正常
#   > 80%: 瓶颈

# === CPU统计 ===
iostat -c 1 10
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          25.34    0.00   10.23    2.34    0.00   62.09

# iowait高表示CPU等待IO完成
```

### 3.4 vmstat - 虚拟内存统计

```bash
vmstat 1 10  # 每秒采样，共10次
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 123456  12345 345678    0    0   123   456  789 1234 25 10 63  2  0

# 关键指标：
# r: 运行队列长度（> CPU核心数 -> CPU瓶颈）
# b: 不可中断睡眠进程数（等待IO）
# si, so: 每秒swap in/out（> 0 -> 内存不足）
# bi, bo: 每秒块设备读写KB数
# in: 每秒中断次数
# cs: 每秒上下文切换次数（> 10000 -> 上下文切换频繁）
# us: 用户空间CPU%
# sy: 内核空间CPU%
# id: 空闲CPU%
# wa: iowait CPU%（> 20% -> IO瓶颈）
```

---

## 4. 文件系统排障

### 4.1 df vs du

```bash
# === df（磁盘空间使用）===
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G   30G   18G  63% /
/dev/sdb1       500G  450G   26G  95% /data

# df -i（inode使用）
df -i
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/sda1      3276800 234567 3042233    8% /
/dev/sdb1     32768000 32765000    3000  100% /data  # inode用尽！

# === du（目录空间占用）===
du -sh /data/*
10G     /data/logs
5G      /data/backup
435G    /data/files

# 查找大文件
du -h /data | sort -rh | head -20

# === df与du不一致排查 ===
# 场景：df显示磁盘满，du显示空间很小
# 原因：文件被删除但进程仍持有文件句柄

# 查找已删除但仍被占用的文件
lsof | grep deleted
nginx     1234 root   5w   REG    8,1  10737418240  12345 /var/log/nginx/access.log (deleted)

# 解决方法：重启进程或清空文件
echo "" > /proc/1234/fd/5  # 清空文件描述符5
```

### 4.2 lsof - 列出打开的文件

```bash
# === 查看进程打开的文件 ===
lsof -p 1234
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   1234 root  cwd    DIR    8,1     4096    2 /
nginx   1234 root  rtd    DIR    8,1     4096    2 /
nginx   1234 root  txt    REG    8,1  1234567   12 /usr/sbin/nginx
nginx   1234 root    5w   REG    8,1 10737418240  34 /var/log/nginx/access.log

# === 查看文件被哪些进程打开 ===
lsof /var/log/nginx/access.log
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   1234 root    5w   REG    8,1 10737418240  34 /var/log/nginx/access.log

# === 查看网络连接 ===
lsof -i :80
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   1234 root    6u  IPv4  12345      0t0  TCP *:http (LISTEN)

# 查看所有TCP连接
lsof -i TCP

# === 查看用户打开的文件 ===
lsof -u root

# === 查看目录下打开的文件 ===
lsof +D /data

# === 恢复被删除的文件 ===
# 如果进程仍持有文件句柄
lsof | grep deleted
nginx   1234 root    5w   REG    8,1  12345  /var/log/nginx/access.log (deleted)

# 从/proc恢复
cp /proc/1234/fd/5 /var/log/nginx/access.log.recovered
```

### 4.3 ext4数据恢复⭐⭐⭐

**背景**：ext4文件系统上误删除文件后的数据恢复，需要深入理解ext4文件系统内部结构。

#### 4.3.1 核心工具：debugfs

```bash
# debugfs是ext4文件系统的交互式调试工具（只读模式打开，避免二次损坏）
debugfs -R "ls -d /path/to/dir" /dev/sda1
# -d: 显示已删除的文件（会用<>标记）
# 输出示例：
# 12345  (100644) .
# 12346  (100755) ..
# <12347> (100644) deleted_file.txt    ← 已删除

# 常用debugfs命令
debugfs /dev/sda1
debugfs:  stat <12347>          # 查看inode详细信息
debugfs:  ex <12347>            # 查看extent映射（文件数据块位置）
debugfs:  imap <12347>          # 查看inode在磁盘上的位置
debugfs:  block_dump 12345      # 查看指定块的原始内容
debugfs:  cat <12347>           # 直接输出文件内容（小文件）
debugfs:  dump <12347> /tmp/recovered_file  # 导出文件
```

#### 4.3.2 ext4 inode结构

```c
// ext4 inode核心字段（128字节 / 256字节）
struct ext4_inode {
    __le16  i_mode;         // 文件类型和权限（如0100644 = 普通文件）
    __le16  i_uid;          // 用户ID（低16位）
    __le32  i_size_lo;      // 文件大小（低32位）
    __le32  i_atime;        // 最后访问时间
    __le32  i_ctime;        // inode修改时间
    __le32  i_mtime;        // 文件内容修改时间
    __le32  i_dtime;        // 删除时间（非零表示已删除）⭐
    __le16  i_gid;          // 组ID
    __le16  i_links_count;  // 硬链接数（删除后=0）⭐
    __le32  i_blocks_lo;    // 占用块数（删除后=0）⭐
    __le32  i_flags;        // 标志位
    __le32  i_block[15];    // 数据块指针 / extent树⭐
    // i_block[15] 在使用extent时存储extent树（60字节）
};
```

#### 4.3.3 ext4 extent树结构

```c
// ext4使用extent树管理文件数据块映射（比传统间接块更高效）

// 1. Extent Header（每个extent节点的头部，12字节）
struct ext4_extent_header {
    __le16  eh_magic;       // 魔数 0xF30A
    __le16  eh_entries;     // 当前entries数量
    __le16  eh_max;         // 最大entries数量
    __le16  eh_depth;       // 树深度（0=叶子节点，>0=索引节点）⭐
    __le32  eh_generation;  // 版本号
};

// 2. Extent Index（索引节点，指向下级extent块，12字节）
struct ext4_extent_idx {
    __le32  ei_block;       // 逻辑块号
    __le32  ei_leaf_lo;     // 子节点物理块号（低32位）⭐ 删除后保留！
    __le16  ei_leaf_hi;     // 子节点物理块号（高16位）
    __le16  ei_unused;      // 保留
};

// 3. Extent（叶子节点，实际的数据块映射，12字节）
struct ext4_extent {
    __le32  ee_block;       // 逻辑块号
    __le16  ee_len;         // 连续块数（最高位=1表示未初始化）
    __le16  ee_start_hi;    // 物理块号（高16位）
    __le32  ee_start_lo;    // 物理块号（低32位）
};
```

**extent树结构示意**：

```
深度0（小文件，extent直接存在inode中）：
inode.i_block[60字节]
├── ext4_extent_header (eh_depth=0)
├── ext4_extent (映射1)   ← 逻辑块0-99 → 物理块1000-1099
├── ext4_extent (映射2)   ← 逻辑块100-199 → 物理块2000-2099
└── ext4_extent (映射3)

深度1+（大文件，需要extent索引）：
inode.i_block[60字节]
├── ext4_extent_header (eh_depth=1)
├── ext4_extent_idx → 指向物理块X（存储二级extent）
│   └── 物理块X:
│       ├── ext4_extent_header (eh_depth=0)
│       ├── ext4_extent (映射1)
│       ├── ext4_extent (映射2)
│       └── ...
└── ext4_extent_idx → 指向物理块Y
```

#### 4.3.4 删除后inode的变化

```bash
# 删除操作对inode的影响：
# 1. i_size_lo → 0           （文件大小清零）
# 2. i_links_count → 0       （硬链接数清零）
# 3. i_blocks_lo → 0         （占用块数清零）
# 4. i_dtime → 当前时间       （记录删除时间）
# 5. eh_depth → 0            （extent header的树深度清零）⭐

# 关键发现：
# ✅ ext4_extent_idx 中的 ei_leaf_lo（子节点物理块号）不会被清除！
# ✅ 对于大文件（extent深度>0），虽然inode中的eh_depth被清零，
#    但索引节点的物理块号仍然保留在inode的i_block字段中
# ❌ 小文件/目录/符号链接：extent完全存储在inode中，删除后被清零，无法恢复
```

#### 4.3.5 大文件恢复实战

```bash
# 场景：误删除一个2GB大文件，extent深度>=1

# Step 1: 找到被删除文件的inode号
debugfs -R "ls -d /data" /dev/sda1
# <12347> (100644) big_file.dat

# Step 2: 查看inode中残留的extent信息
debugfs -R "stat <12347>" /dev/sda1
# 注意：size/blocks已被清零，但i_block字段中的数据仍然存在

# Step 3: 用debugfs的block_dump查看inode原始数据
# 先找到inode在磁盘上的位置
debugfs -R "imap <12347>" /dev/sda1
# 输出：Inode 12347 is part of block group 1, block 8193, offset 256

# Step 4: 从inode的i_block字段中提取extent索引
# i_block字段偏移：inode起始 + 0x28（40字节）
# 读取 ext4_extent_header + ext4_extent_idx 结构
# 找到 ei_leaf_lo 值（索引指向的物理块号）

# Step 5: 读取索引指向的物理块，获取实际的extent映射
debugfs -R "block_dump <ei_leaf_lo值>" /dev/sda1
# 解析 ext4_extent_header + ext4_extent 结构
# 获取 ee_start_lo 和 ee_len（数据的物理起始块和连续块数）

# Step 6: 使用dd恢复数据
# 假设：物理起始块=100000，连续块数=524288（2GB / 4KB块大小）
dd if=/dev/sda1 of=/tmp/recovered_file bs=4096 skip=100000 count=524288

# Step 7: 验证恢复的数据
md5sum /tmp/recovered_file
file /tmp/recovered_file   # 检查文件类型
```

#### 4.3.6 ext4分区结构要点

```bash
# flex_bg特性：将多个块组的metadata集中到第一个块组
# 好处：减少seek操作，提高metadata访问效率

# 查看ext4超级块信息
dumpe2fs /dev/sda1 | head -50
# Filesystem features: ... flex_bg ...
# Flex block group size: 16   ← 每16个块组为一个flex_bg

# 目录索引（dir_index）：使用hash tree加速目录查找
# dx_root → dx_entry → 叶子块（目录项）
# 查看目录是否使用htree
debugfs -R "htree_dump /data" /dev/sda1
```

**恢复可行性总结**：

| 文件类型 | extent深度 | 删除后extent残留 | 恢复可能性 |
|---------|-----------|----------------|-----------|
| **大文件（>几MB）** | ≥1 | ei_leaf_lo保留 | ✅ 高（通过残留索引定位数据块） |
| **小文件（<几MB）** | 0 | extent在inode中被清零 | ❌ 极低（数据块位置信息丢失） |
| **目录** | 0 | 清零 | ❌ 不可恢复 |
| **符号链接** | 0 | 清零 | ❌ 不可恢复 |

> ⚠️ **关键提醒**：发现误删除后，**立即卸载文件系统**（或remount为只读），避免新数据覆盖被删除文件的数据块。

> 📖 **关联知识**：文件系统基础检查修复（fsck/xfs_repair）请参考本文 [Section 2.2 fsck](#22-fsck---文件系统检查修复)。

---

## 5. 网络排障工具

### 5.1 ss - 网络连接查看

```bash
# === 查看所有连接 ===
ss -tunapl
# -t: TCP  -u: UDP  -n: 数字显示  -a: 所有  -p: 进程  -l: 监听

# 示例输出
State      Recv-Q Send-Q Local Address:Port   Peer Address:Port
LISTEN     0      128    0.0.0.0:22           0.0.0.0:*         users:(("sshd",pid=1234,fd=3))
ESTAB      0      0      192.168.1.100:22    203.0.113.20:54321   users:(("sshd",pid=5678,fd=3))

# === TCP连接状态统计 ===
ss -s
Total: 1234
TCP:   567 (estab 123, closed 234, orphaned 5, timewait 200)

# === 查看监听端口 ===
ss -tlnp
State   Recv-Q Send-Q Local Address:Port  Peer Address:Port
LISTEN  0      128    0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=1234))
LISTEN  0      128    0.0.0.0:80          0.0.0.0:*          users:(("nginx",pid=5678))

# === 查看连接详细信息 ===
ss -ti
State      Recv-Q Send-Q Local Address:Port   Peer Address:Port
ESTAB      0      0      192.168.1.100:22    203.0.113.20:54321
         cubic rto:204 rtt:0.234/0.123 ato:40 mss:1460 pmtu:1500 rcvmss:1460
         advmss:1460 cwnd:10 bytes_acked:12345 bytes_received:67890
         send 493.7Mbps lastsnd:1234 lastrcv:5678 lastack:5678
         pacing_rate 987.4Mbps delivery_rate 720.0Mbps app_limited
         rcv_rtt:123 rcv_space:29200 rcv_ssthresh:29200 minrtt:0.123

# === 查看特定端口 ===
ss -tlnp | grep :80

# === 按状态过滤 ===
ss -tan state established  # 已建立连接
ss -tan state time-wait    # TIME_WAIT连接
ss -tan state syn-sent     # SYN_SENT状态
```

### 5.2 tcpdump - 抓包分析

```bash
# === 基本抓包 ===
tcpdump -i eth0
# 显示所有经过eth0的数据包

# === 过滤条件 ===
# 按主机
tcpdump host 192.168.1.100

# 按端口
tcpdump port 80

# 按协议
tcpdump tcp
tcpdump udp
tcpdump icmp

# 组合条件
tcpdump 'tcp port 80 and host 192.168.1.100'
tcpdump 'tcp port 443 or tcp port 80'

# === 保存到文件 ===
tcpdump -i eth0 -w capture.pcap

# 读取文件
tcpdump -r capture.pcap

# === 详细输出 ===
tcpdump -i eth0 -nn -vv
# -nn: 不解析主机名和端口名
# -vv: 详细输出

# === 抓取HTTP请求 ===
tcpdump -i eth0 -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
# -A: ASCII输出
# -s 0: 抓取完整数据包

# === 抓取SYN包（半连接攻击排查）===
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# === 抓取RST包（连接异常排查）===
tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0'
```

### 5.3 netstat - 网络统计（传统工具）

```bash
# === 查看网络连接 ===
netstat -tunapl
# 功能类似ss，但性能较差（已逐渐被ss替代）

# === 路由表 ===
netstat -rn
Kernel IP routing table
Destination     Gateway         Genmask         Flags   Iface
0.0.0.0         192.168.1.1      0.0.0.0         UG      eth0
82.158.87.228     0.0.0.0         255.255.255.0       U       eth0

# === 接口统计 ===
netstat -i
Kernel Interface table
Iface   MTU Met   RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
eth0   1500   0 12345678      0      0      0  23456789      0      0      0 BMRU
```

---

## 6. 内核分析工具

> 虚拟化和底层系统开发中，内核分析工具是定位问题的关键手段。本节介绍ftrace、kprobe、kdump/crash、eBPF等工具的原理与使用。

### 6.1 ftrace - 内核函数追踪

**ftrace**是Linux内核内置的追踪框架，通过debugfs接口操作，无需安装额外工具。

```bash
# ftrace工作目录
cd /sys/kernel/debug/tracing/

# === 基本使用 ===

# 1. 查看可用追踪器
cat available_tracers
# function function_graph nop

# 2. 函数追踪（function tracer）
echo function > current_tracer
echo 1 > tracing_on
# 等待几秒
echo 0 > tracing_on
cat trace | head -20
#  tracer: function
#    TASK-PID    CPU#  ||||   TIMESTAMP  FUNCTION
#    bash-1234   [002]  ....  1234.567:  mutex_lock <-schedule
#    bash-1234   [002]  ....  1234.568:  __schedule <-schedule

# 3. 函数调用图（function_graph）
echo function_graph > current_tracer
echo 1 > tracing_on
cat trace
#  CPU  DURATION                  FUNCTION CALLS
#  2)               |  schedule() {
#  2)               |    __schedule() {
#  2)   0.215 us    |      rcu_note_context_switch();
#  2)               |      pick_next_task_fair() {
#  2)   0.123 us    |        update_curr();
#  2)   0.987 us    |      }
#  2)   3.456 us    |    }
#  2)   4.567 us    |  }

# 4. 过滤特定函数
echo 'schedule' > set_ftrace_filter    # 只追踪schedule函数
echo 'ext4_*' > set_ftrace_filter      # 追踪所有ext4_开头的函数
echo '*lock*' > set_ftrace_filter      # 追踪所有含lock的函数
cat available_filter_functions | wc -l  # 查看可追踪函数数量

# 5. 过滤特定进程
echo <pid> > set_ftrace_pid

# 6. 清理
echo nop > current_tracer
echo > set_ftrace_filter
```

**虚拟化场景应用**：
```bash
# 追踪KVM相关函数（分析VM Exit原因）
echo 'kvm_*' > set_ftrace_filter
echo function_graph > current_tracer
echo 1 > tracing_on
# 触发VM操作
echo 0 > tracing_on
cat trace | grep -E 'kvm_vcpu_|handle_'
```

### 6.2 kprobe/kretprobe - 内核动态探针

**kprobe**：在任意内核函数入口处插入探针，获取参数和上下文
**kretprobe**：在函数返回时插入探针，获取返回值

```bash
# === 使用ftrace接口设置kprobe ===

# 1. 在函数入口设置探针
echo 'p:myprobe do_sys_openat2 dfd=%di filename=%si flags=%dx' > /sys/kernel/debug/tracing/kprobe_events
# p: probe类型
# myprobe: 探针名称
# do_sys_openat2: 目标函数
# %di/%si/%dx: x86_64寄存器（对应函数参数）

# 2. 启用探针
echo 1 > /sys/kernel/debug/tracing/events/kprobes/myprobe/enable

# 3. 查看输出
cat /sys/kernel/debug/tracing/trace
#  bash-1234 [001] .... 1234.567: myprobe: (do_sys_openat2+0x0/0x100) dfd=0xffffff9c filename=0x7ffd1234 flags=0x0

# 4. 设置kretprobe（捕获返回值）
echo 'r:myretprobe do_sys_openat2 ret=$retval' >> /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/myretprobe/enable

# 5. 清理
echo > /sys/kernel/debug/tracing/kprobe_events
```

**虚拟化场景：追踪EPT Violation**：
```bash
# 追踪handle_ept_violation的调用和返回值
echo 'p:ept_probe handle_ept_violation' > /sys/kernel/debug/tracing/kprobe_events
echo 'r:ept_ret handle_ept_violation ret=$retval' >> /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/enable
```

### 6.3 kdump与crash - 内核崩溃分析

**kdump**：内核崩溃时自动生成内存转储（vmcore）
**crash**：分析vmcore的交互式工具

```bash
# === kdump配置 ===

# 1. 安装
yum install kexec-tools crash kernel-debuginfo

# 2. 配置预留内存（grub引导参数）
grubby --update-kernel=ALL --args="crashkernel=256M"
# 重启生效

# 3. 启动kdump服务
systemctl enable kdump
systemctl start kdump

# 4. 验证
kdumpctl status
# kdump: operational  ← 正常

# 5. 配置转储路径（/etc/kdump.conf）
path /var/crash
core_collector makedumpfile -l --message-level 7 -d 31
# -d 31: 排除不必要的页（零页、缓存页等），减小vmcore体积

# === 手动触发崩溃（测试用！） ===
echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger  # 触发panic
# 系统会自动重启并生成 /var/crash/127.0.0.1-2026-01-15-10:15:23/vmcore
```

```bash
# === crash分析 ===

# 打开vmcore
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/*/vmcore

# 常用crash命令
crash> bt              # 查看崩溃时的调用栈（最重要！）
PID: 1234  TASK: ffff8800abcd1234  CPU: 2  COMMAND: "myapp"
 #0 [ffff8800def01000] machine_kexec at ffffffff81060000
 #1 [ffff8800def01050] __crash_kexec at ffffffff81100000
 #2 [ffff8800def01100] panic at ffffffff81800000
 #3 [ffff8800def01200] nmi_panic at ffffffff81010000
 #4 [ffff8800def01300] watchdog_overflow_callback ← 这里是root cause

crash> log             # 查看内核日志（dmesg）
crash> ps              # 查看崩溃时所有进程状态
crash> files 1234      # 查看某进程打开的文件
crash> vm 1234         # 查看某进程的虚拟内存映射
crash> rd -d <addr> 10 # 读取内存地址内容
crash> dis <func>      # 反汇编函数
crash> struct task_struct <addr> # 查看数据结构
```

### 6.4 eBPF - 内核可编程观测

**eBPF（Extended Berkeley Packet Filter）** 是现代Linux内核提供的安全、高效的可编程框架，可在内核中运行沙盒程序，无需修改内核代码或加载模块。

```
eBPF工作原理：
用户空间编写eBPF程序（C/Python）
    ↓ 编译为eBPF字节码
    ↓ 通过bpf()系统调用加载到内核
    ↓ 内核验证器（Verifier）检查安全性
    ↓ JIT编译为本机指令
    ↓ 挂载到Hook点（kprobe/tracepoint/XDP等）
    ↓ 事件触发时执行eBPF程序
    ↓ 通过Map与用户空间交换数据
```

**bcc（BPF Compiler Collection）工具集**：
```bash
# 安装bcc
apt-get install bpfcc-tools  # Debian/Ubuntu
yum install bcc-tools        # CentOS

# === 常用bcc工具 ===

# 1. 追踪打开的文件
opensnoop-bpfcc
# PID    COMM      FD ERR PATH
# 1234   nginx      5   0 /etc/nginx/nginx.conf
# 5678   mysql      8   0 /var/lib/mysql/ibdata1

# 2. 追踪TCP连接
tcpconnect-bpfcc
# PID    COMM   IP SADDR          DADDR          DPORT
# 1234   curl   4  10.0.1.2       1.70.84.48       80

# 3. 追踪块设备IO延迟
biolatency-bpfcc
# usecs     : count    distribution
#   1 -> 1  : 0       |                    |
#   2 -> 3  : 15      |**                  |
#   4 -> 7  : 234     |**********************|
#   8 -> 15 : 123     |************          |

# 4. 追踪ext4文件系统慢操作
ext4slower-bpfcc 1  # 大于1ms的操作
# TIME     COMM   PID  T BYTES   OFF_KB   LAT(ms) FILENAME
# 10:15:23 mysql  1234 W 16384   1234     12.5    ibdata1

# 5. 追踪CPU调度延迟
runqlat-bpfcc
# usecs     : count    distribution
#   0 -> 1  : 1234    |**********************|
#   2 -> 3  : 567     |**********            |
#   4 -> 7  : 89      |*                     |

# 6. 追踪函数调用和返回值
funccount-bpfcc 'vfs_*'    # 统计vfs_*函数调用次数
trace-bpfcc 'kvm:*'        # 追踪KVM相关tracepoint
```

**bpftrace（高级单行脚本）**：
```bash
# 安装
apt-get install bpftrace

# 1. 追踪系统调用耗时
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
              tracepoint:syscalls:sys_exit_read  /@start[tid]/ {
                @usecs = hist((nsecs - @start[tid]) / 1000);
                delete(@start[tid]);
              }'

# 2. 追踪进程创建
bpftrace -e 'tracepoint:sched:sched_process_exec {
  printf("%-6d %-16s %s\n", pid, comm, str(args->filename));
}'

# 3. 追踪KVM VM Exit原因
bpftrace -e 'tracepoint:kvm:kvm_exit {
  @exit_reason[args->exit_reason] = count();
}'
# 输出：
# @exit_reason[48]: 12345   (EPT_VIOLATION)
# @exit_reason[30]: 8901    (IO_INSTRUCTION)
# @exit_reason[1]:  4567    (EXTERNAL_INTERRUPT)

# 4. 内存分配热点
bpftrace -e 'tracepoint:kmem:kmalloc {
  @bytes[kstack] = sum(args->bytes_alloc);
}'
```

**eBPF vs 传统工具对比**：

| 特性 | ftrace/kprobe | SystemTap | eBPF |
|------|-------------|-----------|------|
| **安全性** | 中（可能崩溃） | 低（内核模块） | 高（验证器保证） |
| **性能开销** | 低-中 | 高 | 极低 |
| **学习成本** | 低 | 高 | 中 |
| **内核支持** | 所有版本 | 需要编译 | 4.x+（推荐5.x+） |
| **使用场景** | 简单追踪 | 复杂脚本 | 生产环境观测 |

---

## Part 5 总结 & 整体总结
### Part 5 核心知识点
1. **内核参数调优**：TCP/内存/文件系统/CPU参数
2. **系统排障工具**：mcelog/fsck/ethtool/smartctl
3. **性能分析**：perf/sar/iostat/vmstat
4. **文件系统排障**：df/du/lsof
5. **网络排障**：ss/tcpdump/netstat
6. **内核分析工具**：ftrace（函数追踪/调用图）、kprobe（动态探针）、kdump/crash（崩溃分析）、eBPF（安全可编程观测）

### IaaS全系列知识体系

```
IaaS基础设施技术（5个Part，约3000行）

Part 1: 虚拟化技术
├── KVM架构（CPU/内存/中断虚拟化）
├── QEMU设备模拟（virtio）
├── libvirt管理（virsh）
├── 虚拟机镜像格式（raw/qcow2）
└── 虚拟化深入（QEMU线程模型/VMCS/EPT/Posted Interrupt/virtio vring/IO路径）

Part 2: 虚拟机网络
├── 三种网络模式（桥接/NAT/用户模式）
├── Linux网络虚拟化（TAP/veth/macvlan）
├── 虚拟机迁移（热迁移/冷迁移）
└── VNC远程管理

Part 3: 物理网络与虚拟化网络
├── 网卡Bonding（7种模式）
├── VLAN（4096个网络）
├── 隧道技术（GRE/IPIP）
├── VXLAN（1600万网络）
└── VPC架构（Subnet/安全组/NAT网关）

Part 4: 服务器底层技术
├── BMC/IPMI（带外管理）
├── DPDK（用户空间网络，10倍性能）
├── SPDK（用户空间存储，60%提升）
├── RDMA（零拷贝，<1μs延迟）
└── LVM（逻辑卷管理）

Part 5: 操作系统运维深入
├── 内核参数调优
├── 系统排障工具
├── 性能分析与监控
├── 文件系统排障
├── 网络排障工具
└── 内核分析工具（ftrace/kprobe/kdump/crash/eBPF）
```

### 技术深度评级

| 模块 | 深度评级 | 适用岗位 |
|------|---------|---------|
| **虚拟化** | ⭐⭐⭐⭐⭐ | 云平台开发/虚拟化架构师 |
| **网络** | ⭐⭐⭐⭐⭐ | 网络工程师/SDN开发 |
| **底层技术** | ⭐⭐⭐⭐⭐ | 系统架构师/性能优化专家 |
| **运维** | ⭐⭐⭐⭐⭐ | SRE/Linux运维专家 |

### 全文自查提纲
**虚拟化**：
1. KVM vs Xen vs VMware？
2. qcow2写时复制原理？
3. virtio为什么比e1000快？

**网络**：
1. VLAN vs VXLAN？
2. 热迁移如何保证业务不中断？
3. VPC如何实现租户隔离？

**高性能**：
1. DPDK为什么快10倍？
2. RDMA适用场景？
3. BBR vs Cubic性能对比？

**运维**：
1. iowait高如何排查？
2. df与du不一致原因？
3. 如何定位CPU热点函数？

### 关键命令速查表

```bash
# === 虚拟化 ===
virsh list --all
qemu-img create -f qcow2 disk.qcow2 10G

# === 网络 ===
brctl addbr br0
ip link add vxlan10 type vxlan id 10

# === 高性能 ===
dpdk-devbind.py --bind=vfio-pci 02:00.0
ibv_devices

# === 运维 ===
ethtool -S eth0
smartctl -H /dev/sda
perf top
ss -tunapl
tcpdump -i eth0 port 80
```

### 学习路径建议

**初级（1-2年经验）**：
1. 掌握虚拟化基础（KVM/QEMU）
2. 熟悉Linux网络配置（网桥/VLAN）
3. 掌握基础运维工具（df/top/netstat）

**中级（3-5年经验）**：
1. 深入虚拟化原理（EPT/vCPU调度）
2. 掌握VXLAN/VPC架构
3. 熟练使用性能分析工具（perf/sar）

**高级（5年以上）**：
1. 掌握DPDK/SPDK/RDMA
2. 能够进行内核参数调优
3. 具备系统级性能优化能力

---

---

## 面试题自查

### Q1: KVM、QEMU、qemu-kvm三者的关系是什么？

**答案**：
- **KVM**：Linux内核模块（kvm.ko），利用CPU的硬件虚拟化扩展（Intel VT-x/AMD-V）提供CPU和内存虚拟化
- **QEMU**：用户空间程序，模拟硬件设备（磁盘、网卡、显卡等），可独立运行但性能差（纯软件模拟）
- **qemu-kvm**：QEMU + KVM的组合，QEMU负责设备模拟，KVM负责CPU/内存虚拟化

**架构关系**：
```
Guest VM
   ↕ VM Exit/Entry
KVM（内核模块，/dev/kvm）
   ↕ ioctl
QEMU（用户空间，设备模拟）
```

性能对比：纯QEMU约5-10%物理机性能，qemu-kvm约95-98%。

---

### Q2: VM Exit为什么影响性能？如何优化？

**答案**：
**影响原因**：
- 每次VM Exit需要保存Guest状态、切换到Host上下文，开销约1-5μs
- 频繁Exit会显著降低虚拟机性能

**常见Exit原因及优化**：
| Exit原因 | 触发场景 | 优化方案 |
|---------|---------|---------|
| IO_INSTRUCTION | IO端口访问 | 使用virtio减少Exit |
| EPT_VIOLATION | 内存访问异常 | 预分配内存、启用大页 |
| EXTERNAL_INTERRUPT | 外部中断 | 中断亲和性绑定 |
| PAUSE_INSTRUCTION | 自旋锁 | paravirt_spinlocks |

**查看Exit统计**：
```bash
cat /sys/kernel/debug/kvm/exits
```

---

### Q3: virtio为什么比传统设备模拟（如e1000）性能好？

**答案**：
**传统设备模拟（e1000）**：
```
Guest → e1000驱动 → VM Exit → QEMU模拟e1000 → Host网络栈
每次IO都触发VM Exit，开销大
```

**virtio半虚拟化**：
```
Guest → virtio前端驱动 → virtio环形缓冲区 → virtio后端 → Host网络栈
批量处理，减少VM Exit
```

**关键优化**：
1. **Vring环形队列**：批量传输，减少上下文切换
2. **vhost内核模块**：数据路径不经过QEMU，直接在内核处理
3. **多队列（Multi-queue）**：利用多CPU并行处理

**性能对比**（网络）：
- e1000：~1 Gbps
- virtio-net：~8-9 Gbps（接近物理网卡）

---

### Q4: qcow2镜像格式的写时复制（COW）原理是什么？有什么应用场景？

**答案**：
**COW原理**：
- qcow2支持backing file（基础镜像）
- 新创建的镜像只存储与基础镜像的差异
- 读取时：先查当前镜像，未找到则读backing file
- 写入时：只写入当前镜像，不修改backing file

```bash
# 创建基于base.qcow2的差异镜像
qemu-img create -f qcow2 -b base.qcow2 vm1.qcow2
```

**应用场景**：
1. **快速创建虚拟机**：多个VM共享一个base镜像，每个只存储差异
2. **快照**：创建快照时，原镜像变为backing file
3. **节省存储空间**：100个VM可共享一个30GB的系统镜像

**性能影响**：链式backing file会影响读性能，生产环境建议定期合并。

---

### Q5: 虚拟机热迁移的原理是什么？如何保证业务不中断？

**答案**：
**热迁移三阶段**：

1. **预复制阶段（Pre-copy）**：
   - VM继续运行，后台传输内存到目标Host
   - 同时记录脏页（被修改的内存页）

2. **迭代复制阶段**：
   - 反复传输脏页，直到脏页速度小于传输速度
   - VM运行可能变慢（内存带宽被占用）

3. **Stop-and-Copy阶段**：
   - 暂停VM（黑障期，通常<1秒）
   - 传输剩余脏页和CPU状态
   - 在目标Host启动VM

**前提条件**：
- 共享存储（NFS/Ceph）或块迁移
- 相同CPU架构
- 网络互通

**黑障时间优化**：
```bash
virsh migrate-setmaxdowntime vm1 500  # 最大500ms
virsh migrate --live --auto-converge vm1 qemu+ssh://host2/system
```

---

### Q6: 桥接模式和NAT模式的区别是什么？如何选择？

**答案**：
| 特性 | 桥接（Bridge） | NAT |
|-----|--------------|-----|
| **网络地址** | 与Host同网段 | 私有网段 |
| **外部访问VM** | 直接访问 | 需要端口转发 |
| **IP消耗** | 占用公网/内网IP | 不占用 |
| **性能** | 最佳 | 略低（地址转换） |
| **适用场景** | 生产环境 | 开发测试 |

**选择建议**：
- 需要外部直接访问VM → 桥接
- IP紧张、隔离需求 → NAT
- 简单测试 → 用户模式（无需root）

---

### Q7: VLAN和VXLAN的区别是什么？为什么需要VXLAN？

**答案**：
| 特性 | VLAN | VXLAN |
|-----|------|-------|
| **网络数量** | 4096（12位ID） | 1600万（24位VNI） |
| **工作层次** | 二层（L2） | 三层封装二层 |
| **封装开销** | 4字节 | 50字节（UDP+VXLAN头） |
| **跨数据中心** | 困难 | 支持 |

**为什么需要VXLAN**：
1. **规模限制**：云平台租户多，4096个VLAN不够
2. **虚拟机迁移**：需要大二层网络，VLAN跨机房困难
3. **隔离需求**：多租户环境需要更多隔离网络

**VXLAN封装**：
```
原始帧 → UDP(4789) → VXLAN头(VNI) → 外层IP → 外层MAC
```

---

### Q8: DPDK为什么能达到10倍于内核网络栈的性能？

**答案**：
**性能瓶颈分析（内核网络栈）**：
1. 中断频繁：每个包触发硬中断+软中断
2. 上下文切换：用户态/内核态切换开销
3. 内存拷贝：数据从网卡→内核缓冲区→用户空间

**DPDK优化手段**：
| 优化 | 机制 | 收益 |
|-----|------|------|
| **轮询替代中断** | PMD（Poll Mode Driver） | 消除中断开销 |
| **用户空间驱动** | UIO/VFIO绕过内核 | 消除上下文切换 |
| **大页内存** | Hugepages | 减少TLB miss |
| **CPU亲和性** | 专用核心 | 减少缓存miss |
| **零拷贝** | DMA直接到用户空间 | 消除内存拷贝 |

**性能数据**：
- 内核网络栈：~1.5 Mpps（64字节小包）
- DPDK：~14.88 Mpps（接近10GbE线速）

---

### Q9: RDMA的核心优势是什么？适用于哪些场景？

**答案**：
**核心优势**：
1. **零拷贝（Zero-Copy）**：数据直接从网卡DMA到应用内存
2. **零CPU（Kernel Bypass）**：绕过操作系统内核
3. **低延迟**：端到端延迟<1μs（TCP约10-20μs）

**三种技术**：
| 技术 | 说明 | 成本 |
|-----|------|-----|
| InfiniBand | 专用协议，性能最高 | 最高 |
| RoCE v2 | RDMA over Ethernet | 中等 |
| iWARP | RDMA over TCP | 最低 |

**适用场景**：
- **高频交易**：延迟敏感
- **分布式存储**：NVMe over Fabrics
- **HPC**：MPI通信
- **机器学习**：分布式训练梯度同步

---

### Q10: LVM的核心概念是什么？如何实现在线扩容？

**答案**：
**核心概念**：
```
PV（Physical Volume） → 物理卷（磁盘/分区）
    ↓
VG（Volume Group） → 卷组（PV的集合）
    ↓
LV（Logical Volume） → 逻辑卷（从VG分配）
```

**在线扩容步骤**：
```bash
# 1. 扩展VG（如果需要新磁盘）
pvcreate /dev/sdc
vgextend vg1 /dev/sdc

# 2. 扩展LV
lvextend -L +50G /dev/vg1/lv_data

# 3. 扩展文件系统（在线）
resize2fs /dev/vg1/lv_data  # ext4
xfs_growfs /mnt/data        # xfs
```

**优势**：灵活的存储管理，支持快照、在线扩容，比分区更易管理。

---

### Q11: BMC/IPMI的作用是什么？生产环境如何保证安全？

**答案**：
**BMC作用**：
- 独立于操作系统的带外管理芯片
- 即使服务器关机也能远程管理

**核心功能**：
| 功能 | 说明 |
|-----|------|
| 远程电源 | 开机/关机/重启 |
| 传感器监控 | 温度/风扇/电压 |
| KVM over IP | 远程图形界面 |
| 虚拟媒体 | 挂载ISO安装系统 |
| 日志（SEL） | 硬件事件记录 |

**安全最佳实践**：
1. **网络隔离**：带外网络与业务网络物理隔离
2. **修改默认密码**：ipmitool user set password 2 StrongPass
3. **IP白名单**：限制BMC访问来源
4. **禁用不必要服务**：如SNMP、Telnet
5. **定期更新固件**：修复安全漏洞

---

### Q12: 如何排查Linux系统iowait高的问题？

**答案**：
**排查流程**：

```bash
# 1. 确认iowait高
top
# 看%wa列，>20%表示IO瓶颈

# 2. 找出哪个磁盘繁忙
iostat -x 1
# 看%util列，>80%表示瓶颈

# 3. 找出哪些进程在做IO
iotop -o
# -o只显示有IO的进程

# 4. 分析IO模式
pidstat -d 1
# 看读写KB/s

# 5. 查看进程打开的文件
lsof -p <pid>

# 6. 查看文件系统状态
cat /proc/meminfo | grep -i dirty
# Dirty页多说明写入积压
```

**常见原因及解决**：
| 原因 | 现象 | 解决方案 |
|-----|------|---------|
| 日志写入过多 | 某进程写入量大 | 调整日志级别/异步写入 |
| 数据库查询 | MySQL进程IO高 | 优化SQL/增加索引/增加内存 |
| Swap | si/so不为0 | 增加物理内存 |
| 磁盘性能差 | await高 | 更换SSD |

---

### Q13: TCP BBR拥塞控制算法相比Cubic有什么优势？

**答案**：
**Cubic（默认）**：
- 基于丢包的拥塞控制
- 丢包后才降低发送速率
- 在高延迟/高丢包网络表现差

**BBR（Bottleneck Bandwidth and RTT）**：
- 基于带宽和RTT的模型
- 主动探测瓶颈带宽
- 不依赖丢包信号

**性能对比**（100ms RTT，1%丢包）：
- Cubic：~1.8 Gbps
- BBR：~7.1 Gbps（4倍提升）

**启用BBR**：
```bash
# 需要内核4.9+
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

---

### Q14: df和du显示的空间不一致，可能是什么原因？如何排查？

**答案**：
**常见原因**：文件被删除但进程仍持有文件句柄

```bash
# 场景
rm /var/log/huge.log  # 删除大文件
df -h                  # 磁盘空间未释放
du -sh /var           # 显示空间很小
```

**排查步骤**：
```bash
# 1. 查找被删除但仍占用的文件
lsof | grep deleted
# nginx 1234 root 5w REG 8,1 10737418240 /var/log/nginx/access.log (deleted)

# 2. 解决方法
# 方法1：重启进程
systemctl restart nginx

# 方法2：清空文件描述符（不重启）
echo "" > /proc/1234/fd/5
```

**预防**：使用logrotate + copytruncate或postrotate重载进程。

---

### Q15: ext4文件系统误删除大文件后，如何尝试恢复？

**答案**：
**关键点**：大文件（extent深度≥1）删除后，extent索引的物理块号不会被清除。

**恢复步骤**：
```bash
# 1. 立即卸载或remount只读（防止覆盖）
mount -o remount,ro /data

# 2. 找到删除文件的inode
debugfs -R "ls -d /data" /dev/sda1
# <12347> deleted_file.dat

# 3. 查看inode信息
debugfs -R "stat <12347>" /dev/sda1
# 查看i_block字段中的extent索引

# 4. 定位数据块
debugfs -R "imap <12347>" /dev/sda1
# 从extent索引找到物理块号

# 5. 使用dd恢复
dd if=/dev/sda1 of=/tmp/recovered bs=4096 skip=<物理块号> count=<块数>
```

**可恢复性**：
| 文件类型 | 可恢复性 | 原因 |
|---------|---------|------|
| 大文件（>几MB） | 高 | extent索引保留 |
| 小文件 | 低 | extent信息被清零 |
| 目录/符号链接 | 不可恢复 | 信息完全清除 |

---

### Q16: Security Group和Network ACL的区别是什么？

**答案**：
| 特性 | Security Group | Network ACL |
|-----|---------------|-------------|
| **作用范围** | VM/ENI级别 | Subnet级别 |
| **状态** | 有状态（自动允许返回流量） | 无状态（需要显式配置） |
| **规则类型** | 只有Allow | Allow + Deny |
| **规则顺序** | 全部评估 | 按序号评估 |
| **默认行为** | 拒绝所有入，允许所有出 | 允许所有 |

**使用场景**：
- Security Group：精细化控制单个VM
- Network ACL：Subnet级别粗粒度控制

**示例**：
```bash
# Security Group（有状态）
# 只需允许入80端口，返回流量自动允许
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Network ACL（无状态）
# 需要同时允许入80和出临时端口
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 80 -j ACCEPT
```

---

### Q17: QEMU有哪些线程？主线程阻塞会影响vCPU执行吗？

**答案**：
**QEMU主要线程**：
1. **Main Thread**：事件循环、QMP命令处理、非dataplane的设备模拟
2. **vCPU Thread × N**：每个vCPU一个线程，执行KVM_RUN
3. **IO Thread**（可选）：独立事件循环处理virtio IO
4. **Worker Thread Pool**：异步任务（qcow2分配、压缩等）

**主线程阻塞的影响**：
- vCPU线程独立运行，主线程阻塞时Guest代码**仍可执行**
- 但如果VM Exit需要主线程模拟设备（如PIO），则vCPU会等待
- 推荐使用独立IO线程（iothread）避免IO处理阻塞主线程

---

### Q18: 什么是GVA/GPA/HVA/HPA？一次Guest内存访问的完整路径是怎样的？

**答案**：
| 地址 | 含义 | 管理者 |
|------|------|--------|
| GVA | Guest进程虚拟地址 | Guest MMU |
| GPA | Guest "物理"地址 | Guest内核页表 |
| HVA | Host进程虚拟地址 | QEMU进程空间 |
| HPA | 真实物理地址 | Host MMU + EPT |

**完整访问路径**：
GVA → Guest页表 → GPA → EPT页表 → HPA

如果EPT缺页（EPT Violation）：VM Exit → KVM分配物理页 → 建立EPT映射 → VM Entry

---

### Q19: Posted Interrupt的原理是什么？有什么硬件要求？

**答案**：
**原理**：传统中断注入需要VM Exit+Entry，开销大。Posted Interrupt让APIC直接写入PIR（Posted Interrupt Request），通过通知向量让CPU自动合并到虚拟APIC，Guest无需VM Exit即可收到中断。

**硬件要求**：
1. Intel VT-x with APICv（Virtual APIC）
2. CPU支持Posted Interrupt Processing
3. 需要启用Virtual Interrupt Delivery

**效果**：高频中断场景（如网络密集型）性能显著提升。

---

### Q20: virtio的vring通信原理是什么？前端和后端如何互相通知？

**答案**：
**Vring三部分**：
1. **Descriptor Table**：描述数据缓冲区（GPA地址、长度、读写标记）
2. **Available Ring**：前端写、后端读，放入"待处理"的描述符链
3. **Used Ring**：后端写、前端读，放入"已完成"的描述符链

**通知机制**：
- 前端→后端（kick）：写IO端口/ioeventfd → 通知后端有新请求
- 后端→前端（中断）：irqfd → KVM注入MSI-X中断 → 通知前端有完成结果

**优化**：ioeventfd避免退到QEMU主线程，irqfd配合Posted Interrupt可零VM Exit注入中断。