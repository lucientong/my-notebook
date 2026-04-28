# Linux 运维与排障

---

## 📑 目录

### 命令与排障SOP
1. [常用命令速查](#1-常用命令速查)
2. [CPU飙高排查SOP](#2-cpu飙高排查sop)
3. [内存问题排查SOP](#3-内存问题排查sop)
4. [网络问题排查SOP](#4-网络问题排查sop)
5. [磁盘I/O排查SOP](#5-磁盘io排查sop)

### 实战与自查
6. [面试题自查](#6-面试题自查)
7. [实战案例](#7-实战案例)

---

## 1. 常用命令速查

### 1.1 进程管理

```bash
# 查看进程
ps aux | grep java
ps -ef | grep nginx
ps -Lf <pid>  # 查看线程

# 实时监控
top               # 整体负载
top -Hp <pid>     # 进程内线程
htop              # 更友好的top

# 进程树
pstree -p <pid>

# 查看进程文件
lsof -p <pid>              # 进程打开的文件
ls -l /proc/<pid>/fd       # 文件描述符
cat /proc/<pid>/status     # 进程状态
cat /proc/<pid>/limits     # 资源限制
```

### 1.2 网络

```bash
# 连接状态
ss -antp          # TCP连接（推荐）
ss -s             # 统计信息
netstat -antp     # 老版本

# 抓包
tcpdump -i eth0 port 8080 -w capture.pcap
tcpdump -i eth0 host 10.0.0.1 and port 3306
tcpdump -i eth0 -nn -s0 -c 100  # 抓100个包

# DNS
dig example.com
nslookup example.com
host example.com

# 路由
traceroute example.com
mtr example.com  # 更强大
ip route show
```

### 1.3 系统资源

```bash
# CPU
mpstat 1 10       # CPU使用率（1秒刷新，10次）
pidstat -u 1      # 进程CPU
sar -u 1 10       # 系统CPU历史

# 内存
free -h           # 内存使用
vmstat 1          # 虚拟内存统计
pmap -x <pid>     # 进程内存映射
cat /proc/meminfo  # 详细内存信息

# 磁盘I/O
iostat -x 1 10    # I/O统计
iotop             # 实时I/O
df -h             # 磁盘空间
du -sh *          # 目录大小

# 系统负载
uptime            # 负载均衡
sar -q 1 10       # 负载历史
```

### 1.4 排障工具

```bash
# 追踪系统调用
strace -p <pid>
strace -c -p <pid>  # 统计调用
strace -T -p <pid>  # 显示耗时

# 性能分析
perf top          # 实时性能
perf record -g -p <pid>  # 记录调用栈
perf report       # 查看报告

# 内核日志
dmesg | tail
dmesg -T          # 带时间戳
journalctl -xe    # systemd日志
```

> 💡 **本文定位**：排障SOP流程 + 底层原理深度剖析。各工具的完整用法手册（ethtool网卡诊断、smartctl硬盘检查、perf火焰图、sar历史数据、iostat阈值判断等）请参考 《专项知识库/06-IaaS基础设施技术》Part 5。

---

## 2. CPU飙高排查SOP

### Step 1: 确认CPU使用情况

```bash
top
# 观察指标：
# - %us: 用户态CPU（业务代码）
# - %sy: 内核态CPU（系统调用）
# - %wa: I/O等待
# - %id: 空闲CPU
# - load average: 1/5/15分钟平均负载
```

**Load Average判断标准**：
- 小于CPU核心数：正常
- 等于CPU核心数：满载
- 大于CPU核心数：过载

### Step 2: 定位进程

```bash
# 方法1：top排序
top，按P（按CPU排序）

# 方法2：ps排序
ps aux | sort -nrk 3 | head -10

# 查看进程详情
ps -p <pid> -o pid,ppid,cmd,%mem,%cpu
```

### Step 3: 定位线程

```bash
# 查看进程内哪个线程CPU高
top -Hp <pid>

# 或使用ps
ps -Lf <pid> | sort -nrk 10 | head -10

# 记录线程ID（TID），例如：12345
```

### Step 4: 查看线程堆栈

**Go程序**：
```bash
# 查看goroutine堆栈
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# 或使用pprof
go tool pprof http://localhost:6060/debug/pprof/profile
```

**Java程序**：
```bash
# 1. 将TID转为16进制
printf "%x\n" 12345  # 输出：3039

# 2. 查看线程堆栈
jstack <pid> | grep -A 50 3039

# 3. 查看整体堆栈
jstack <pid> > jstack.log
```

**Python程序**：
```bash
# 使用py-spy
py-spy top --pid <pid>
py-spy dump --pid <pid>
```

### Step 5: 分析原因

**常见原因**：
1. **死循环**：代码逻辑错误
2. **CPU密集计算**：加解密、压缩、编码
3. **频繁GC**：内存不足，Full GC
4. **锁竞争**：大量线程竞争锁，CPU自旋

**示例：Go死循环**
```go
// 错误代码：CPU 100%
for {
    // 没有任何阻塞操作
}

// 正确做法
for {
    select {
    case <-time.After(10 * time.Millisecond):
        // 处理
    case <-ctx.Done():
        return
    }
}
```

---

## 3. 内存问题排查SOP

### Step 1: 确认内存使用

```bash
free -h
# 关注：
# - used: 已使用
# - available: 可用（包含cache）
# - buff/cache: 页缓存（可回收）
```

**判断标准**：
- available < 10%：内存紧张
- available < 5%：内存严重不足

### Step 2: 定位进程

```bash
# 按内存排序
ps aux | sort -nrk 4 | head -10

# 查看进程内存详情
pmap -x <pid>
cat /proc/<pid>/smaps  # 详细内存映射
```

### Step 3: 分析内存分布

**Go程序**：
```bash
# heap dump
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof

# 在pprof交互模式：
(pprof) top          # 查看内存占用Top
(pprof) list <func>  # 查看具体函数
(pprof) web          # 生成调用图
```

**Java程序**：
```bash
# 查看堆内存
jmap -heap <pid>

# 查看对象统计
jmap -histo <pid> | head -20

# heap dump（慎用，会暂停应用）
jmap -dump:format=b,file=heap.hprof <pid>

# 使用MAT分析heap.hprof
```

**Python程序**：
```bash
# 使用memory_profiler
python -m memory_profiler script.py

# 或使用objgraph
import objgraph
objgraph.show_most_common_types(limit=10)
```

### Step 4: 常见内存泄漏场景

**Go程序**：
```go
// 1. goroutine泄漏
func leak() {
    go func() {
        for {
            // 永远不会退出
            time.Sleep(1 * time.Second)
        }
    }()
}

// 正确做法：传入context
func noLeak(ctx context.Context) {
    go func() {
        for {
            select {
            case <-time.After(1 * time.Second):
                // 处理
            case <-ctx.Done():
                return  // 退出
            }
        }
    }()
}

// 2. map无限增长
var cache = make(map[string]string)
func addCache(key, value string) {
    cache[key] = value  // 永远不删除
}

// 正确做法：使用LRU缓存或定期清理
```

---

## 4. 网络问题排查SOP

### 场景1：连接数暴涨

```bash
# 查看连接数
ss -s

# 按状态统计
ss -antp | awk '{print $1}' | sort | uniq -c

# 查看ESTABLISHED连接
ss -antp | grep ESTABLISHED | wc -l

# 查看TIME_WAIT连接
ss -antp | grep TIME_WAIT | wc -l
```

**TIME_WAIT过多原因**：
- 短连接过多（改用长连接）
- 主动关闭方（客户端）积累

**解决方案**：
```bash
# 调整内核参数
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=30
sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# 永久生效
echo "net.ipv4.tcp_tw_reuse = 1" >> /etc/sysctl.conf
sysctl -p
```

### 场景2：网络延迟高

```bash
# ping测试
ping -c 10 example.com

# 路由追踪
traceroute example.com
mtr example.com  # 更强大，持续监控

# 抓包分析延迟
tcpdump -i eth0 -s 0 -w capture.pcap host example.com

# 使用Wireshark分析 capture.pcap
# 查看：Time、Delta time displayed
```

**常见延迟原因**：
1. 网络拥塞
2. 路由问题
3. DNS解析慢
4. 服务端响应慢

### 场景3：丢包

```bash
# 查看网卡统计
ethtool -S eth0 | grep -E "drop|error"
ifconfig eth0 | grep -E "drop|error"

# 查看内核丢包
netstat -s | grep -E "drop|error"

# 检查连接队列满
ss -lnt
# Recv-Q > 0 说明队列积压
# Send-Q > 0 说明发送缓冲区满

# 调整队列大小
sysctl -w net.core.somaxconn=1024
sysctl -w net.ipv4.tcp_max_syn_backlog=2048
```

---

## 5. 磁盘I/O排查SOP

### Step 1: 确认I/O状况

```bash
# I/O统计
iostat -x 1 10
# 关注：
# - %util: 磁盘繁忙程度（>80%说明瓶颈）
# - await: 平均等待时间（ms）
# - r/s, w/s: 每秒读写次数

# 实时I/O
iotop
iotop -o  # 只显示有I/O的进程
```

### Step 2: 定位进程

```bash
# 查看进程I/O
pidstat -d 1

# 查看进程打开的文件
lsof -p <pid>

# 查看进程读写的文件
lsof -p <pid> | grep -E "REG|DIR"
```

### Step 3: 优化建议

**常见I/O问题**：
1. **频繁小文件读写** → 批量操作、缓冲
2. **日志写入过多** → 异步日志、日志采样
3. **数据库查询慢** → 索引优化、查询优化
4. **磁盘满** → 清理日志、扩容

### 5.1 延迟问题的分层排查视角

很多线上问题并不是“系统挂了”，而是**P99 变高**。这类问题如果只盯着 tcpdump 或 top，往往容易误判。

建议按下面三层拆开看：

| 层级 | 典型现象 | 先看什么 |
|------|----------|----------|
| **应用层** | 某个接口慢、线程池积压、锁等待 | 应用日志、pprof/jstack、GC、线程池队列 |
| **内核/系统层** | syscall 慢、iowait 高、上下文切换高 | `pidstat`、`vmstat`、`perf`、`/proc/<pid>/stack` |
| **网络层** | RTT 抖动、连接建立慢、DNS 间歇失败 | `ss`、`sar -n DEV`、`ethtool -S`、`tcpdump` |

一个实用原则是：**先判断慢发生在“请求进入进程前”还是“进入进程后”**。
- 如果连接建立、DNS 解析、TLS 握手就慢，优先看网络和系统。
- 如果请求已经进进程，但业务处理长尾明显，优先看线程、锁、GC、磁盘和下游依赖。

### 5.2 CPU Busy / Wait / Steal 的判断方法

同样是“服务变慢”，CPU 指标代表的含义完全不同：

| 指标 | 含义 | 典型问题 |
|------|------|----------|
| **us/sys 高** | 用户态/内核态真的在忙 | 死循环、热点函数、频繁 syscall |
| **iowait 高** | CPU 没干活，在等 I/O | 磁盘慢、网络存储慢、刷盘抖动 |
| **steal 高** | 虚拟机 CPU 时间被宿主机抢走 | 宿主超卖、 noisy neighbor |

常用命令：

```bash
# 看 CPU 各状态拆分
mpstat -P ALL 1 5
sar -u 1 5

# 看进程是否在等 I/O 或频繁切换
pidstat -u -w -p <pid> 1
vmstat 1

# 虚拟机里看 steal
top
mpstat 1 5
```

经验上可以这样判断：
- `us`/`sys` 高，说明 CPU 真忙，先找热点线程和热点函数。
- `wa` 高，说明 CPU 不忙但系统在等，先查磁盘、网络存储、下游阻塞。
- `st` 高，说明业务未必有 bug，而是算力被外部抢占，要看宿主机或云资源层。

---

## 6. 面试题自查

### Q1: 如何定位CPU 100%？完整SOP是什么？

**答案**：
1. `top` 查看整体CPU，找到CPU最高的进程PID
2. `top -Hp <pid>` 找到进程内CPU最高的线程TID
3. 将TID转为16进制：`printf "%x\n" <tid>`
4. 查看线程堆栈：
   - Go：`curl http://localhost:6060/debug/pprof/goroutine?debug=2`
   - Java：`jstack <pid> | grep -A 50 <hex_tid>`
5. 分析代码定位根因：死循环/频繁GC/锁竞争

### Q2: Load Average 3个数字什么意思？如何判断系统负载是否正常？

**答案**：
- **含义**：1分钟、5分钟、15分钟的平均负载
- **计算公式**：正在运行的进程(R) + 不可中断睡眠的进程(D)
- **判断标准**：
  - 小于CPU核心数：正常
  - 等于CPU核心数：满载
  - 大于CPU核心数：过载

**示例**：4核CPU，load average: 2.5, 1.8, 1.2 → 2.5/4=62.5%负载，正常

### Q3: 为什么Load Average高但CPU使用率低？

**答案**：
Load Average包含**D状态进程**（等待I/O）。

**常见场景**：
- 磁盘I/O慢：大量进程等待磁盘读写
- NFS挂载故障：进程卡在NFS请求
- 硬件故障：设备响应超时

**排查命令**：
```bash
ps aux | awk '{if($8=="D") print $0}'  # 查看D状态进程
cat /proc/<pid>/stack                   # 查看进程卡在哪里
```

### Q4: 如何定位内存泄漏？Go程序和Java程序有什么不同？

**答案**：

**通用流程**：
1. `free -h` 确认内存持续增长
2. `ps aux --sort=-rss | head` 找到内存最高的进程
3. dump内存并分析

**Go程序**：
```bash
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof
(pprof) top  # 查看内存占用
```

**Java程序**：
```bash
jmap -histo <pid> | head -20  # 对象统计
jmap -dump:format=b,file=heap.hprof <pid>  # dump堆
# 用MAT分析hprof文件
```

### Q5: TIME_WAIT过多怎么办？为什么会出现？

**答案**：

**原因**：主动关闭连接的一方进入TIME_WAIT，等待2MSL

**解决方案**：
1. **改用长连接**：HTTP Keep-Alive，连接池
2. **调整内核参数**：
   ```bash
   sysctl -w net.ipv4.tcp_tw_reuse=1     # 允许复用TIME_WAIT
   sysctl -w net.ipv4.tcp_fin_timeout=30 # 缩短FIN超时
   sysctl -w net.ipv4.ip_local_port_range="1024 65535"  # 扩大端口范围
   ```

### Q6: 如何查看某个端口被哪个进程占用？

**答案**：
```bash
lsof -i :8080                # 推荐
ss -lntp | grep :8080        # 性能更好
netstat -lntp | grep :8080   # 老版本
```

### Q7: TCP三次握手的SYN队列和Accept队列有什么区别？队列满了会怎样？

**答案**：

**两个队列**：
- **SYN队列（半连接）**：收到SYN后，等待ACK
- **Accept队列（全连接）**：三次握手完成，等待accept()

**队列满的影响**：
- SYN队列满：新SYN被丢弃，客户端超时 "Connection timed out"
- Accept队列满：服务器不回复ACK或发RST，"Connection refused"

**调优**：
```bash
sysctl -w net.ipv4.tcp_max_syn_backlog=4096  # SYN队列
sysctl -w net.core.somaxconn=2048            # Accept队列
```

### Q8: 为什么TCP需要TIME_WAIT？为什么是2MSL？

**答案**：

**两个原因**：
1. **确保最后的ACK到达**：如果ACK丢失，服务器会重传FIN，客户端需要能响应
2. **防止旧连接数据干扰新连接**：等待旧连接的包过期

**为什么2MSL**：
- 1MSL：客户端ACK最多1MSL到达服务器
- +1MSL：服务器重传FIN最多1MSL返回
- 总共2MSL，客户端一定能收到重传的FIN

### Q9: Page Cache是什么？为什么数据库不用Page Cache？

**答案**：

**Page Cache**：Linux内核的磁盘缓存
- 读文件时，内容缓存到内存
- 下次读取直接从内存返回，无磁盘I/O

**数据库不用的原因**：
1. **双重缓存**：数据库有自己的Buffer Pool，再用Page Cache浪费内存
2. **无法精确控制**：Page Cache由内核LRU管理，数据库需要自己控制

**解决方案**：Direct I/O绕过Page Cache
```sql
-- MySQL配置
innodb_flush_method = O_DIRECT
```

### Q10: strace和perf的区别？生产环境用哪个？

**答案**：

| 工具 | 原理 | 性能影响 | 适用场景 |
|------|------|----------|----------|
| **strace** | ptrace拦截系统调用 | 高（10-100x慢） | 开发调试 |
| **perf** | 硬件PMU计数器采样 | 低（<5%） | **生产环境** |

**生产环境推荐perf**：采样方式，对性能影响小

### Q11: 如何查看进程打开了哪些文件？文件描述符泄漏怎么排查？

**答案**：

**查看文件描述符**：
```bash
lsof -p <pid>            # 进程打开的所有文件
ls -l /proc/<pid>/fd     # fd目录
cat /proc/<pid>/limits   # fd限制
```

**排查泄漏**：
1. `ls /proc/<pid>/fd | wc -l` 统计fd数量
2. 定期采样，观察是否持续增长
3. `lsof -p <pid>` 找到大量重复打开的文件
4. 检查代码：是否忘记close()

### Q12: 如何排查磁盘I/O瓶颈？%util和await分别表示什么？

**答案**：

**排查命令**：
```bash
iostat -x 1  # I/O统计
iotop        # 实时I/O进程
```

**关键指标**：
- **%util**：磁盘繁忙程度，>80%说明瓶颈
- **await**：I/O平均等待时间(ms)，>20ms说明较慢
- **r/s, w/s**：每秒读写次数

**常见问题**：
- 频繁小文件读写 → 批量操作
- 日志写入过多 → 异步日志
- 数据库查询慢 → 索引优化

### Q13: 如何排查网络丢包？

**答案**：

**排查步骤**：
```bash
# 1. 查看网卡丢包
ethtool -S eth0 | grep -E "drop|error"
ifconfig eth0 | grep dropped

# 2. 查看内核丢包
netstat -s | grep -E "drop|error"

# 3. 检查连接队列
ss -lnt  # Recv-Q > 0 说明积压
```

**常见原因**：
- 网卡队列满
- 连接队列满（somaxconn太小）
- 内核缓冲区不足

### Q14: 如何触发crash dump进行故障分析？

**答案**：

**两种方法**：
1. **SysRq魔术键**：`echo c > /proc/sysrq-trigger`
2. **NMI中断**：`ipmitool chassis power diag`（通过BMC）

**kdump配置**：
```bash
# 1. 安装
yum install kexec-tools

# 2. 配置crashkernel
GRUB_CMDLINE_LINUX="crashkernel=256M"

# 3. 启用服务
systemctl enable kdump
systemctl start kdump
```

**分析vmcore**：
```bash
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/*/vmcore
crash> bt   # 调用栈
crash> log  # 内核日志
```

### Q15: 如何定位内核内存泄漏？

**答案**：

**排查流程**：
1. **对比meminfo**：正常机 vs 异常机，找到差异指标
2. **定位泄漏类型**：
   - Slab高：`slabtop -s a` 查看slab分配
   - Vmalloc高：`cat /proc/vmallocinfo` 统计各函数
   - Page Alloc：计算公式排除法

**跟踪工具**：
```bash
# ftrace跟踪kmalloc
echo 'stacktrace if bytes_req == 256' > /sys/kernel/debug/tracing/events/kmem/kmalloc/trigger

# perf采样
perf record -e 'kmem:kmem_cache_alloc' -a -g -- sleep 30
perf report
```

### Q16: 抓包看到重传不多，但 P99 还是很高，下一步你先查应用还是内核？为什么？

**答案**：
先不急着二选一，而是先判断**长尾是在连接建立前还是进入进程后**。

1. 如果 DNS、建连、TLS 握手阶段就慢，优先查网络和内核。
2. 如果请求已进入业务进程，优先查应用线程池、锁竞争、GC、下游依赖。
3. 重传不多只能说明“明显丢包”不突出，不能排除排队、阻塞、软中断、连接池耗尽等问题。
4. 最有效的做法是把一次请求拆成 DNS、connect、TLS、应用处理、下游 RPC 五段分别打点。

面试里更好的回答不是“我先查应用”或“我先查内核”，而是说明你会先确定**延迟发生在哪一层**。

### Q17: 操作系统层面，如何判断是 CPU 忙、CPU 等、还是 CPU 被抢占导致慢？

**答案**：
1. 用 `mpstat 1` 或 `sar -u 1` 看 `us/sys/wa/st`。
2. `us` 或 `sys` 高，说明 CPU 真忙，优先查热点线程、热点函数、频繁系统调用。
3. `wa` 高，说明主要在等 I/O，优先查磁盘、网络存储、刷盘、NFS。
4. `st` 高，说明虚拟机 CPU 被宿主机抢占，优先查宿主超卖或 noisy neighbor。
5. 再结合 `pidstat -u -w`、`vmstat 1` 看上下文切换和阻塞情况，避免只看单个指标。

### Q18: 站在 Linux/节点视角，DNS 间歇失败时如何快速判断是网络策略、DNS 服务还是节点本地问题？

**答案**：
1. 先在故障节点执行 `dig` 或 `nslookup`，确认是解析超时、SERVFAIL 还是 NXDOMAIN。
2. 检查 `/etc/resolv.conf`、本地 DNS 配置、节点到 DNS 服务器的连通性。
3. 用 `tcpdump -i any port 53` 看请求是否发出、响应是否返回。
4. 如果请求都没发出去，优先查本地网络、iptables、NetworkPolicy、安全组。
5. 如果请求发出但响应慢或丢失，再看 CoreDNS/QPS、上游递归 DNS、节点 conntrack 是否打满。

核心是先分清：**请求没出去、请求出去了没回来、还是 DNS 服务本身答错了**。

### Q19: 站在 Linux/节点视角，readiness 正常但接口延迟很高时你通常先看哪三类指标？

**答案**：
1. **CPU 指标**：`us/sys/wa/st`，判断是真忙、在等 I/O，还是被抢占。
2. **调度与阻塞指标**：上下文切换、运行队列、D 状态进程、线程池堆积。
3. **网络与磁盘指标**：网卡丢包、连接队列、磁盘 `await/%util`。

readiness 只表示“探针能过”，不代表服务没有长尾。很多线上问题都是探针正常，但线程池、连接池或下游依赖已经接近饱和。

### Q20: 一次排障中，如何建立“从现象到根因”的最小闭环，避免只会堆命令？

**答案**：
建议固定成一个四步闭环：

1. **先定义现象**：谁慢、从什么时候开始、影响面多大。
2. **再做分层**：应用层、系统层、网络层，先判断问题落在哪一层。
3. **然后找证据**：每一层只抓最关键的 1-2 个证据，不要无序堆命令。
4. **最后做回证**：修复后确认指标恢复，并能解释为什么这个动作有效。

面试官真正看重的是你能不能把排障过程收敛成一条因果链，而不是背了多少命令。

---

### 开放式设计题

**D1：线上服务器突然出现大量TIME_WAIT连接（10万+），导致新连接建立失败，如何快速处理？**

**参考思路**：
- 确认现象：ss -s查看TIME_WAIT数量、确认是客户端主动关闭（出方向）还是服务端主动关闭
- 紧急处理：开启tcp_tw_reuse（安全复用TIME_WAIT）、调小tcp_fin_timeout（加速回收）
- 根因治理：改用长连接（HTTP Keep-Alive/连接池）、调整负载均衡策略减少短连接
- 架构优化：引入连接池代理（PgBouncer/ProxySQL）、客户端SDK内置连接复用
- 注意：不要轻易开tcp_tw_recycle（NAT环境会导致连接异常）

**D2：需要在不停机的情况下给一台生产服务器扩容磁盘（根分区快满了），操作步骤是什么？**

**参考思路**：
- 在线扩容（云环境）：控制台扩容云盘 → growpart扩展分区 → resize2fs/xfs_growfs扩展文件系统
- 紧急腾空间：find大文件删除、清理日志（truncate而非rm正在写的文件）、清理Docker无用镜像
- 迁移方案：新挂盘→rsync数据→软链接/mount --bind切换目录→释放旧空间
- 长期治理：磁盘使用率>70%告警>85%禁部署、日志logrotate配置、定期清理临时文件Cron

---

## 6.5 底层原理深度剖析⭐⭐⭐

### 6.5.1 Load Average底层原理⭐⭐⭐

**为什么Load Average不只是CPU繁忙度？**

```c
// Linux内核计算Load Average的逻辑（简化版）
load_average = running_processes + uninterruptible_sleep_processes

// running_processes: 正在运行的进程（R状态）
// uninterruptible_sleep_processes: 不可中断睡眠的进程（D状态）
```

**D状态进程是什么？**

```bash
# 查看D状态进程
ps aux | awk '{if($8=="D") print $0}'

# D状态（Disk sleep）典型场景：
# 1. 等待磁盘I/O（读写）
# 2. 等待NFS响应
# 3. 等待硬件设备响应
```

**为什么D状态会导致Load Average高？**

```
示例1：磁盘故障
- 10个进程等待磁盘I/O（D状态）
- CPU空闲（%idle = 95%）
- 但Load Average = 10（很高！）

解读：
- CPU虽然空闲，但系统"负载"高
- 因为有10个进程在等待（阻塞）
- 系统吞吐量下降

示例2：NFS挂载故障
- 100个进程访问NFS（D状态）
- Load Average = 100
- 但CPU使用率很低

排查：
# 查看D状态进程的堆栈
cat /proc/<pid>/stack
# 如果看到 wait_on_page_locked，说明等待磁盘I/O
```

**Load Average的指数衰减算法**：

```c
// Linux内核源码（kernel/sched/loadavg.c）
// 计算1/5/15分钟Load Average

// 指数衰减因子
EXP_1  = 1884  // 1分钟，衰减因子 e^(-5/60)
EXP_5  = 2014  // 5分钟，衰减因子 e^(-5/300)
EXP_15 = 2037  // 15分钟，衰减因子 e^(-5/900)

// 每5秒更新一次
load_1  = load_1  * EXP_1  / 2048 + active_tasks * (2048 - EXP_1) / 2048
load_5  = load_5  * EXP_5  / 2048 + active_tasks * (2048 - EXP_5) / 2048
load_15 = load_15 * EXP_15 / 2048 + active_tasks * (2048 - EXP_15) / 2048
```

**为什么1分钟Load Average变化快，15分钟慢？**

```
指数衰减因子越小，历史权重越小，新值权重越大
- 1分钟：EXP_1 = 1884，历史权重 92%，新值权重 8%
- 15分钟：EXP_15 = 2037，历史权重 99.5%，新值权重 0.5%

结果：
- 1分钟Load Average快速响应当前负载
- 15分钟Load Average反映长期趋势
```

**内核实现深入：定点数与calc_load源码**

```c
// === 定点数表示（避免浮点运算） ===
// 内核不使用浮点数，用定点数模拟小数
// FIXED_1 = 1 << 11 = 2048
// 低11位表示小数部分，高位表示整数部分
//
// 转换方法：
// 定点数 → 实际值：整数部分 = value >> 11
//                   小数部分 = (value & 0x7FF) * 100 / 2048
//
// 例：load值 = 5120 → 整数 = 5120 >> 11 = 2
//                    → 小数 = (5120 & 2047) * 100 / 2048 = 1024*100/2048 = 50
//                    → 实际Load = 2.50

// === 衰减因子（一次指数平滑法）===
// 算法：an = an-1 * e + a * (1 - e)
// 其中e为衰减系数，a为当前采样值
//
// EXP_1  = 1884  → e^(-5/60)   ≈ 0.920  → 1884/2048 ≈ 0.920
// EXP_5  = 2014  → e^(-5/300)  ≈ 0.983  → 2014/2048 ≈ 0.983
// EXP_15 = 2037  → e^(-5/900)  ≈ 0.995  → 2037/2048 ≈ 0.995

// === 采样周期 ===
// LOAD_FREQ = 5 * HZ + 1  （约5秒采样一次，+1避免与timer_tick完全对齐）

// === 核心计算函数 ===
// kernel/sched/loadavg.c

// 1. calc_global_load()：在do_timer()中被调用，每LOAD_FREQ触发一次
void calc_global_load(unsigned long ticks) {
    unsigned long active;
    // 收集所有CPU的活跃任务数
    active = atomic_long_read(&calc_load_tasks);
    active = active > 0 ? active * FIXED_1 : 0;  // 转为定点数

    // 对1/5/15分钟分别计算
    avenrun[0] = calc_load(avenrun[0], EXP_1, active);
    avenrun[1] = calc_load(avenrun[1], EXP_5, active);
    avenrun[2] = calc_load(avenrun[2], EXP_15, active);
}

// 2. calc_load()：一次指数平滑的核心实现
// load = load * exp + active * (FIXED_1 - exp)
static unsigned long calc_load(unsigned long load, unsigned long exp,
                               unsigned long active) {
    unsigned long newload;
    newload = load * exp + active * (FIXED_1 - exp);
    if (active >= load)
        newload += FIXED_1 - 1;  // 向上取整
    return newload / FIXED_1;
}

// === SMP多核处理 ===
// 每个CPU维护 calc_load_tasks（atomic变量）
// 进程状态变化时更新：
//   - 进入R/D状态：calc_load_tasks++
//   - 离开R/D状态：calc_load_tasks--
//
// nohz（tickless）处理：
//   - CPU进入nohz idle时，调用 calc_load_fold_idle()
//     将该CPU的任务数从 calc_load_tasks 中扣除
//   - CPU退出nohz idle时，调用 calc_load_account_active()
//     重新加回任务数
//
// CPU Hotplug处理：
//   - CPU下线时，调用 calc_load_migrate()
//     将该CPU的任务迁移到其他CPU并更新计数
```

---

### 6.5.2 TCP连接状态转换底层原理⭐⭐⭐

**TCP状态机完整图**：

```
客户端                                服务器
CLOSED                               CLOSED
  |                                     |
  |                                  bind()
  |                                  listen()
  |                                  LISTEN
  |                                     |
connect() (发送SYN)                     |
SYN_SENT --------------------------> SYN_RCVD
  |          (发送SYN+ACK)              |
  |<------------------------------------'
  |          (发送ACK)
ESTABLISHED --------------------------> ESTABLISHED
  |                                     |
  |          (数据传输)                 |
  |<----------------------------------->|
  |                                     |
close() (发送FIN)                       |
FIN_WAIT1 ---------------------------> CLOSE_WAIT
  |          (发送ACK)                  |
  |<------------------------------------'
FIN_WAIT2                            (等待应用close)
  |                                     |
  |          (发送FIN)                close()
  |<------------------------------------'
TIME_WAIT                            LAST_ACK
  |          (发送ACK)                  |
  '------------------------------------>'
  |                                  CLOSED
2MSL等待
  |
CLOSED
```

**为什么需要TIME_WAIT？（深度解析）**

```
原因1：确保最后的ACK到达

场景：
t=0:   客户端发送FIN
t=1:   服务器回复ACK，进入CLOSE_WAIT
t=2:   服务器发送FIN，进入LAST_ACK
t=3:   客户端收到FIN，发送ACK，进入TIME_WAIT

问题：如果客户端的ACK丢失会怎样？
- 服务器在LAST_ACK状态，会重传FIN
- 如果客户端立即CLOSED，收到FIN后无法响应
- 服务器超时后强制关闭（不优雅）

解决：TIME_WAIT等待2MSL
- MSL（Maximum Segment Lifetime）：报文最大生存时间（Linux默认60秒）
- 2MSL = 2 * 60 = 120秒
- 等待期间如果收到FIN，重传ACK
- 确保服务器收到ACK

原因2：防止旧连接数据干扰新连接

场景：
t=0:   连接1（192.168.1.1:5000 → 10.0.0.1:80）关闭
t=1:   立即创建连接2（相同四元组）
t=2:   连接1的延迟包到达（被连接2接收）

问题：旧连接的数据被新连接接收，数据混乱

解决：TIME_WAIT等待2MSL
- 确保旧连接的所有包都消失
- 2MSL内不创建相同四元组的连接
```

**为什么是2MSL而不是1MSL或3MSL？**

```
1MSL：
- 客户端发送ACK后，等待1MSL
- 如果服务器的FIN在1MSL后到达（极限情况）
- 客户端已CLOSED，无法响应
- ❌ 不够

2MSL：
- 客户端发送ACK（最多1MSL到达服务器）
- 服务器超时重传FIN（最多1MSL返回客户端）
- 总共2MSL，客户端一定能收到重传的FIN
- ✅ 刚好

3MSL：
- 过于保守，浪费资源
- ❌ 没必要
```

---

### 6.5.3 Page Cache工作原理⭐⭐⭐

**什么是Page Cache？**

```
定义：Linux内核的磁盘缓存
- 读取文件时，内容缓存在内存中（Page Cache）
- 下次读取，直接从内存返回（无需磁盘I/O）
- 写入文件时，先写Page Cache（异步刷盘）
```

**Page Cache的生命周期**：

```bash
# 查看Page Cache大小
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           15Gi       2.0Gi       10Gi       100Mi        3.0Gi        13Gi

# buff/cache = 3.0Gi 就是Page Cache

# 手动清理Page Cache（测试用）
echo 3 > /proc/sys/vm/drop_caches

# 1: 清理page cache
# 2: 清理dentries和inodes
# 3: 清理所有缓存
```

**Page Cache如何工作？**

```c
// 读取文件的流程（简化版）
read(fd, buffer, size)
  ↓
1. 检查Page Cache（radix tree查找）
  ↓
2. 如果命中（cache hit）
   - 直接从内存拷贝到buffer
   - 返回（无磁盘I/O）
  ↓
3. 如果未命中（cache miss）
   - 从磁盘读取数据
   - 放入Page Cache
   - 拷贝到buffer
   - 返回
```

**写入文件的流程**：

```c
write(fd, buffer, size)
  ↓
1. 写入Page Cache（标记为dirty）
  ↓
2. 立即返回（异步写入）
  ↓
3. 内核后台线程（pdflush/flush）定期刷盘
   - 每30秒刷一次
   - 或dirty page超过阈值
```

**为什么数据库（MySQL/Redis）不用Page Cache？**

```
原因1：双重缓存（Double Buffering）
- 数据库有自己的Buffer Pool/内存缓存
- 如果再用Page Cache，数据在内存中存2份
- 浪费内存

原因2：无法精确控制
- Page Cache由内核管理（LRU淘汰）
- 数据库需要精确控制哪些数据常驻内存
- 例如：MySQL的热数据必须在Buffer Pool中

解决方案：Direct I/O（绕过Page Cache）
# MySQL配置
innodb_flush_method = O_DIRECT

# 代码示例（Go）
fd, _ := syscall.Open("/data/file", syscall.O_RDWR|syscall.O_DIRECT, 0644)
```

**Page Cache与mmap**：

```go
// mmap：将文件映射到内存（使用Page Cache）
func mmapFile(filename string) ([]byte, error) {
    file, _ := os.Open(filename)
    defer file.Close()
    
    stat, _ := file.Stat()
    size := int(stat.Size())
    
    // mmap映射
    data, err := syscall.Mmap(
        int(file.Fd()),
        0,
        size,
        syscall.PROT_READ,
        syscall.MAP_SHARED,
    )
    
    return data, err
}

// 优势：
// 1. 访问data[i]时，自动从Page Cache读取
// 2. 如果Page Cache miss，触发page fault，内核加载数据
// 3. 减少read()系统调用开销

// 劣势：
// 1. 随机访问可能触发大量page fault
// 2. 不适合小文件（mmap开销大）
```

---

### 6.5.4 内核参数调优底层机制⭐⭐⭐

**TCP连接队列原理**：

```
服务端监听socket有两个队列：
1. SYN队列（半连接队列）
2. Accept队列（全连接队列）

三次握手流程：
客户端                     服务器
  |                          |
  |--- SYN --->              |
  |                    (放入SYN队列)
  |                          |
  |<-- SYN+ACK ---|          |
  |                          |
  |--- ACK --->              |
  |              (从SYN队列移到Accept队列)
  |                          |
accept()从Accept队列取出连接
```

**队列满了会怎样？**

```bash
# 查看队列大小
ss -lnt
# Recv-Q: Accept队列当前大小
# Send-Q: Accept队列最大大小

# 示例：
# State  Recv-Q Send-Q Local Address:Port
# LISTEN 50     128    *:80

# Recv-Q=50, Send-Q=128
# 说明：Accept队列有50个连接等待accept()，最大128个
```

**队列溢出导致的问题**：

```
SYN队列满：
- 新的SYN包被丢弃
- 客户端超时重传SYN
- 客户端报错：Connection timed out

Accept队列满：
- 服务器不回复ACK（或回复RST）
- 客户端报错：Connection refused

排查命令：
# 查看SYN队列溢出次数
netstat -s | grep "SYNs to LISTEN"

# 查看Accept队列溢出次数
netstat -s | grep "overflowed"
```

**调优参数速查**：

| 参数 | 作用 | 推荐值 |
|------|------|--------|
| `tcp_max_syn_backlog` | SYN队列大小 | 4096-8192 |
| `somaxconn` | Accept队列大小 | 2048-4096 |
| `tcp_syncookies` | 防SYN Flood攻击 | 1 |
| `tcp_rmem/tcp_wmem` | TCP收发窗口 | 最大16MB |
| `tcp_window_scaling` | 窗口扩展 | 1 |

```bash
# 关键调优命令
sysctl -w net.ipv4.tcp_max_syn_backlog=4096
sysctl -w net.core.somaxconn=2048
sysctl -w net.ipv4.tcp_syncookies=1
```

**TCP窗口调优核心概念**：

```
TCP窗口 = 发送方能发送的未确认数据量
带宽延迟积（BDP）= 带宽 × RTT

示例：100Mbps带宽、100ms RTT → BDP = 1.25MB
如果窗口 < BDP，无法打满带宽
```

**文件描述符限制**：

```bash
# 快速排查
ulimit -n                          # 当前进程限制（默认1024）
cat /proc/sys/fs/file-max          # 系统级限制

# 为什么有限制？每个fd需要约1KB内核内存，100万fd ≈ 1GB内存
```

> 📖 **完整参数配置参考**：TCP/内存/文件系统全量内核参数调优请参考 《专项知识库/06-IaaS基础设施技术》Part 5 - Linux内核参数调优（Section 1），包含BBR拥塞控制对比、文件系统挂载优化等完整配置。

---

### 6.5.5 性能工具原理⭐⭐

**strace工作原理**：

```c
// strace使用ptrace系统调用
ptrace(PTRACE_ATTACH, pid, NULL, NULL);  // 附加到进程
ptrace(PTRACE_SYSCALL, pid, NULL, NULL); // 拦截系统调用

// 每个系统调用都会被拦截：
// 1. 进程调用系统调用（如read）
// 2. 内核通知strace
// 3. strace打印调用信息
// 4. strace通知内核继续执行

// 代价：
// - 每个系统调用都停顿2次（进入/退出）
// - 性能开销巨大（10-100倍慢）
// - 生产环境慎用
```

**perf工作原理**：

```bash
# perf使用硬件性能计数器（PMU）
perf top
# 原理：
# 1. CPU每执行N条指令（如10000条），触发中断
# 2. 中断处理函数记录当前指令地址
# 3. 统计哪些函数执行次数最多

# perf record采样
perf record -g -p <pid>
# -g: 记录调用栈
# 原理：每隔1ms采样一次，记录当前调用栈

# 代价：
# - 采样开销小（<5%）
# - 适合生产环境
```

**tcpdump工作原理**：

```c
// tcpdump使用libpcap（Packet Capture）
// 底层：BPF（Berkeley Packet Filter）

// 1. 设置BPF过滤器（内核态）
struct bpf_program filter;
pcap_compile(&filter, "tcp port 80");

// 2. 网卡收到包后：
//    - 复制到内核缓冲区
//    - BPF过滤（内核态，高效）
//    - 匹配的包复制到用户态
//    - tcpdump读取并记录

// 为什么BPF快？
// - 在内核态过滤（不需要拷贝所有包到用户态）
// - 类似JIT编译（BPF指令被编译成机器码）
```

> 📖 **完整工具用法参考**：perf火焰图生成、sar历史数据分析、iostat指标阈值判断、ss连接详情、tcpdump过滤语法等完整工具手册请参考 《专项知识库/06-IaaS基础设施技术》Part 5 - 性能分析与网络排障（Section 3-5）。

---

### 6.5.6 OS无响应诊断与Crash Dump⭐⭐

**场景**：系统完全hang住（键盘无反应、SSH无法连接），需要收集crash dump分析根因。

**两种触发Crash Dump的方法**：

| 特性 | SysRq魔术键 | NMI不可屏蔽中断 |
|------|------------|----------------|
| **触发方式** | 键盘 `Alt+SysRq+c` 或 `echo c > /proc/sysrq-trigger` | `ipmitool chassis power diag`（BMC带外触发） |
| **前提条件** | `/proc/sys/kernel/sysrq` 设为1 | 内核参数 `unknown_nmi_panic=1` |
| **自动化能力** | ❌ 需要本地键盘或系统仍可响应 | ✅ 可通过BMC带外远程触发，支持agent自动化 |
| **功能范围** | 丰富（sync磁盘、remount只读、打印进程等多种操作） | 单一（仅触发panic → crash dump） |
| **适用场景** | 系统半响应、手动调试 | 系统完全无响应、自动化运维 |

**推荐方案：NMI自动化触发流程**：

```
1. 默认 unknown_nmi_panic=0（避免误触发导致宕机）
2. 监控agent上报心跳失败
3. 运维平台判定OS无响应
4. 通过BMC设置 unknown_nmi_panic=1
   echo 1 > /proc/sys/kernel/unknown_nmi_panic   # 带内设置
   # 或通过BMC SOL执行
5. BMC触发NMI中断
   ipmitool chassis power diag
6. 内核收到NMI → panic → kdump生成vmcore
7. 分析vmcore定位根因
```

**kdump配置（4步）**：

```bash
# Step 1: 安装kexec-tools
yum install kexec-tools   # CentOS/RHEL
apt install kdump-tools   # Ubuntu

# Step 2: 配置crashkernel内核参数（预留内存给crash kernel）
# /etc/default/grub 添加：
GRUB_CMDLINE_LINUX="crashkernel=256M"
# 内存>4GB建议256M，<4GB用128M

# Step 3: 重建grub并重启
grub2-mkconfig -o /boot/grub2/grub.cfg   # CentOS
update-grub                               # Ubuntu
reboot

# Step 4: 验证kdump服务
systemctl enable kdump
systemctl start kdump
systemctl status kdump   # 应为active

# 验证crashkernel预留
cat /proc/cmdline | grep crashkernel
dmesg | grep "Reserving .* for crashkernel"
```

**vmcore分析要点**：

```bash
# crash dump默认保存路径
ls /var/crash/

# 使用vmcore-dmesg.txt快速查看崩溃时的内核日志
vmcore-dmesg.txt /var/crash/<timestamp>/vmcore

# 使用crash工具深入分析
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/<timestamp>/vmcore

# crash交互命令
crash> bt          # 查看崩溃时的调用栈
crash> log         # 查看内核日志
crash> ps          # 查看进程列表
crash> files       # 查看打开的文件
crash> vm          # 查看虚拟内存信息
```

**SysRq魔术键速查**（系统半响应时可用）：

```bash
# 启用SysRq
echo 1 > /proc/sys/kernel/sysrq

# 常用SysRq命令（Alt+SysRq+<key>或echo到/proc/sysrq-trigger）
echo b > /proc/sysrq-trigger   # 立即重启（不sync）
echo c > /proc/sysrq-trigger   # 触发crash dump
echo e > /proc/sysrq-trigger   # 向所有进程发SIGTERM
echo f > /proc/sysrq-trigger   # 调用OOM killer
echo i > /proc/sysrq-trigger   # 向所有进程发SIGKILL
echo s > /proc/sysrq-trigger   # sync所有文件系统
echo u > /proc/sysrq-trigger   # remount所有文件系统为只读
# 安全重启顺序（REISUB）：s → u → b
```

---

### 6.5.7 内核内存泄露定位⭐⭐⭐

**背景**：进程级内存泄露可用pprof/jmap等用户态工具定位，但**内核态内存泄露**（slab/vmalloc/page alloc）需要使用不同的方法。

**Step 1: /proc/meminfo 关键指标速查**

```bash
cat /proc/meminfo

# 关键指标说明
MemTotal       # 总内存
MemFree        # 空闲内存（未被使用）
MemAvailable   # 可用内存（包含可回收的cache）
Slab           # 内核slab缓存总量（= SReclaimable + SUnreclaim）
SReclaimable   # 可回收的slab（如dentry/inode缓存）
SUnreclaim     # 不可回收的slab（真正的内核内存使用）
VmallocUsed    # vmalloc分配的内存
HugePages_Total # 大页总数
HugePages_Free  # 空闲大页
Buffers        # 块设备缓冲区
Cached         # 页缓存
AnonPages      # 匿名页（进程堆栈）
```

**Step 2: 正常机 vs 异常机对比定位**

```bash
# 方法：同时查看正常机和异常机的meminfo，逐项对比
# 找到差异显著的指标，即为泄露方向

# 示例：
#           正常机        异常机
# Slab:     200MB        8000MB    ← slab泄露！
# 或
# VmallocUsed: 50MB      5000MB    ← vmalloc泄露！
# 或 两者都正常但MemFree很低      ← page alloc泄露
```

**Step 3: 泄露类型识别与定位**

**类型一：Slab泄露（最常见）**

```bash
# 查看slab分配详情
cat /proc/slabinfo
# 各列含义：
# name          — slab缓存名称
# active_objs   — 正在使用的对象数
# num_objs      — 总对象数（含空闲）
# objsize       — 单个对象大小（字节）
# objperslab    — 每个slab页包含的对象数
# pagesperslab  — 每个slab占用的页数
# active_slabs  — 正在使用的slab数
# num_slabs     — 总slab数

# 按活动对象排序查看（快速定位大户）
slabtop -s a
# 或按内存大小排序
slabtop -s c

# 对比正常机和异常机的slabinfo
# 找到异常机中 active_objs 异常偏高的slab缓存名称
# 该名称通常对应内核模块或子系统（如kmalloc-256, dentry, inode_cache等）
```

**类型二：Vmalloc泄露**

```bash
# 查看vmalloc分配详情
cat /proc/vmallocinfo
# 各列含义：
# 虚拟地址范围    — vmalloc分配的虚拟地址空间
# 内存大小        — 该次分配的大小
# 申请函数        — 调用vmalloc的内核函数名
# option信息      — 附加标志（pages/vmalloc/ioremap等）

# 统计各函数的vmalloc分配总量
cat /proc/vmallocinfo | awk '{print $3}' | sort | uniq -c | sort -rn | head -20
# 找到分配量异常大的函数，即为泄露来源
```

**类型三：Page Alloc泄露**

```bash
# 如果slab和vmalloc都正常，计算page alloc占用量
# 公式：
# page_alloc_size = MemTotal - HugePages内存 - MemFree - Slab - VmallocUsed - AnonPages - Cached - Buffers

# 如果page_alloc_size异常偏大，说明存在直接page alloc泄露
# 需要使用ftrace或perf跟踪具体的分配调用栈
```

**Step 4: 使用ftrace跟踪内核内存分配调用栈**

```bash
# 方法1: ftrace跟踪kmalloc（适用于slab泄露）
# 开启kmalloc事件的调用栈跟踪
echo 'stacktrace' > /sys/kernel/debug/tracing/events/kmem/kmalloc/trigger

# 查看跟踪结果
cat /sys/kernel/debug/tracing/trace

# 按分配大小过滤（例如只跟踪256字节的分配）
echo 'stacktrace if bytes_req == 256' > /sys/kernel/debug/tracing/events/kmem/kmalloc/trigger

# 关闭跟踪
echo '!stacktrace' > /sys/kernel/debug/tracing/events/kmem/kmalloc/trigger

# 方法2: perf record跟踪（更适合生产环境采样）
perf record -e 'kmem:kmem_cache_alloc' -a -g --filter 'bytes_req == 256' -- sleep 30
perf report
# 输出调用栈，定位哪个内核路径在频繁分配256字节的内存

# 方法3: slub debug trace（跟踪特定slab对象的申请/释放）
# 开启trace
echo 1 > /sys/kernel/slab/<slab-name>/trace
# 例如：
echo 1 > /sys/kernel/slab/kmalloc-256/trace

# 在dmesg中查看每次申请/释放的完整调用栈
dmesg -w | grep TRACE
# 对比申请和释放的数量，找到只申请不释放的调用路径

# 关闭trace
echo 0 > /sys/kernel/slab/kmalloc-256/trace
```

**排查流程总结**：

```
1. cat /proc/meminfo → 对比正常机，确定泄露方向
   ├── Slab异常高 → cat /proc/slabinfo + slabtop -s a
   │                → ftrace/perf跟踪对应大小的kmalloc
   ├── VmallocUsed异常高 → cat /proc/vmallocinfo
   │                      → 统计各函数分配总量
   └── 两者正常但内存低 → 计算page_alloc_size
                         → ftrace跟踪page_alloc事件
2. 定位到泄露的内核函数/模块
3. 分析代码：找到只alloc不free的路径
4. 修复：内核补丁或卸载问题模块
```

> 📖 **关联知识**：用户态内存泄露排查（Go pprof / Java jmap / Python memory_profiler）请参考本文 [Section 3 - 内存问题排查SOP](#3-内存问题排查sop)。

---

## 7. 实战案例

### 案例1：CPU 100%定位

**现象**：服务CPU突然飙到100%

**排查过程**：
```bash
# 1. top查看
top
# PID 12345, CPU 98.5%

# 2. 查看线程
top -Hp 12345
# TID 12350, CPU 95%

# 3. 转16进制
printf "%x\n" 12350
# 输出：303e

# 4. 查看堆栈
jstack 12345 | grep -A 50 303e
# 发现：死循环在 processData() 方法
```

**根因**：代码bug，while循环没有退出条件

**解决**：修复代码，增加退出条件

---

### 案例2：内存泄漏

**现象**：Go服务内存持续增长，3天后OOM

**排查过程**：
```bash
# 1. pprof dump
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof

# 2. 查看内存占用
(pprof) top
# httpClient.Transport 占用80%

# 3. 查看代码
(pprof) list httpClient
# 发现：每次请求创建新client，没有复用
```

**根因**：HTTP客户端没有复用连接

**解决**：
```go
// 错误代码
func request() {
    client := &http.Client{Timeout: 5 * time.Second}
    client.Get("http://example.com")
}

// 正确代码：复用client
var httpClient = &http.Client{
    Timeout: 5 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
    },
}
```

---

### 案例3：连接数爆满

**现象**：服务连接数暴涨，新请求无法建立连接

**排查过程**：
```bash
# 1. 查看连接数
ss -s
# TCP: 10000+ connections

# 2. 查看状态分布
ss -antp | awk '{print $1}' | sort | uniq -c
# 9000+ TIME_WAIT

# 3. 检查代码
# 发现：使用短连接，每次请求都close
```

**根因**：短连接 + 客户端主动close → TIME_WAIT积累

**解决**：
```go
// 启用HTTP Keep-Alive
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
}
```

**内核参数优化**：
```bash
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=30
```

---

## 总结

Linux排障核心流程：
1. **观察现象**：top/free/ss 确认问题
2. **定位进程**：ps/top 找到问题进程
3. **深入分析**：pprof/jstack/tcpdump
4. **解决问题**：修改代码/调整参数

**关键能力**：
- 熟练使用常用命令
- 掌握SOP化排查流程
- 能够快速定位问题根因


---
