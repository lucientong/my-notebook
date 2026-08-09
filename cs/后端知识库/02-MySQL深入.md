# MySQL 深入

> **版本基线**：本文默认以 **MySQL 8.0** 为准。涉及 5.7 与 8.0 行为差异处会显式标注（如 redo log 配置、复制术语、降序索引等）。

---

## 📑 目录

### 存储与索引
- [存储引擎](#存储引擎)
- [日志机制](#日志机制)
- [InnoDB 内存体系](#innodb-内存体系)
- [索引原理](#索引原理)
  - [B+Tree结构](#btree结构)
  - [索引类型](#索引类型)
  - [联合索引](#联合索引)
  - [覆盖索引与回表](#覆盖索引与回表)
  - [索引失效场景](#索引失效场景)
  - [索引下推](#索引下推)

### 执行与事务
- [SQL执行机制](#sql执行机制)
- [事务与锁](#事务与锁)
  - [事务隔离级别](#事务隔离级别)
  - [MVCC机制](#mvcc机制)
  - [锁类型](#锁类型)
  - [死锁定位](#死锁定位)

### 复制、DDL 与运维
- [复制与一致性](#复制与一致性)
- [Online DDL 与变更](#online-ddl-与变更)
- [备份与恢复](#备份与恢复)
- [MySQL 8.0 特性速览](#mysql-80-特性速览)
- [性能优化](#性能优化)
- [数据建模](#数据建模)
- [实战案例](#实战案例)
- [面试题自查](#面试题自查)

---

## 存储引擎

### InnoDB vs MyISAM⭐⭐⭐

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 完整 ACID 事务 | ❌ 不支持事务 |
| **锁粒度** | 行级锁（高并发） | 表级锁（低并发） |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ redo log 保证 | ❌ 需手动修复 |
| **MVCC** | ✅ 支持（无锁读） | ❌ 不支持 |
| **索引结构** | 聚簇索引（数据与主键索引存储在一起） | 非聚簇索引（数据和索引分开存储） |
| **全文索引** | ✅ MySQL 5.6+ 支持 | ✅ 原生支持 |
| **COUNT(*)** | 需要遍历（无精确计数） | 有行数缓存（O(1)） |
| **适用场景** | 绝大多数 OLTP 场景 | 只读/读多写少的分析场景 |

**不同引擎都用 B+ 树吗？**

**不都是。** 常见引擎对比：

| 引擎 | 普通索引结构 | 数据放哪 |
|------|--------------|----------|
| **InnoDB** | 默认 **B+Tree** | 在**主键** B+Tree 叶子里（聚簇） |
| **MyISAM** | 默认 **B+Tree** | 在单独的 **`.MYD` 数据文件**；索引叶子只存行指针 |
| **MEMORY** | 默认 **HASH**（也可 BTree） | 内存表，适合等值查找 |
| **NDB** 等 | 各自实现 | 视引擎而定 |

日常说「MySQL 索引是 B+ 树」，多半特指 **InnoDB 的默认索引**。换引擎或换索引类型（如 `USING HASH`）就不一定是。

**为什么 InnoDB 是默认引擎？**
- MySQL 5.5 之后默认使用 InnoDB
- 事务 + 行级锁 + 崩溃恢复是生产环境的基本需求
- MyISAM 的表锁在高并发下性能极差

**InnoDB 到底长什么样？（一张表 = 多棵 B+ 树）**

假设表：

```sql
CREATE TABLE users (
  id    BIGINT PRIMARY KEY,   -- 主键
  name  VARCHAR(50),
  email VARCHAR(100),
  INDEX idx_name(name)        -- 二级索引
);
```

InnoDB **不是**「整张表共用一棵树」。而是：

```text
【树 1：主键索引 = 聚簇索引 = 真正的「表数据」】
  按 id 排序的 B+Tree
  叶子节点存「完整行」：

  ... → | id=1 | name=Alice | email=a@x.com | → | id=2 | name=Bob | ... | → ...

【树 2：二级索引 idx_name】
  按 name 排序的另一棵 B+Tree
  叶子节点只存「索引列 + 主键」，没有整行：

  ... → | name=Alice | id=1 | → | name=Bob | id=2 | → ...
```

| 说法 | 对不对 |
|------|--------|
| 二级索引叶子**不存整行**，只存索引列（+ 主键） | ✅ |
| **完整行数据只出现在主键 B+ 树的叶子上** | ✅（InnoDB） |
| 「整个表对应的 B+ 树只有一棵」 | ❌：一张表通常有 **1 棵主键树 + N 棵二级索引树** |
| 二级索引叶子「只有索引值」 | 不完全：InnoDB 二级叶子是 **`索引列值 + 主键值`**，好拿着主键去主键树回表 |

```text
SELECT * FROM users WHERE name = 'Alice';

1) 在 idx_name 树找到：name=Alice → id=1
2) 拿 id=1 再走主键树，叶子上取出整行（回表）
```

**和 MyISAM 对比（帮助建立直觉）**：

```text
MyISAM：数据与索引完全分开

  users.MYD  —— 堆式/按插入顺序的行数据文件（真正的「表数据」）
  users.MYI  —— 所有索引（主键、普通）都是 B+Tree
                 叶子统一存「行在 .MYD 里的地址」，没有「聚簇」概念

InnoDB：没有单独的「纯数据文件」这一层（逻辑上）
  .ibd 里装的是这些 B+Tree 页；
  「表数据」= 主键那棵树的叶子。
```

```
InnoDB（聚簇索引）：
主键索引叶子节点 → 存储完整行数据
二级索引叶子节点 → 存储「索引列 + 主键」（需要回表）

MyISAM（非聚簇索引）：
主键索引叶子节点 → 存储数据文件的行指针
普通索引叶子节点 → 存储数据文件的行指针
（两种索引结构相同，都通过指针访问 .MYD 数据文件）
```

---

## 日志机制

### 三大日志的区别与协作⭐⭐⭐

#### redo log（重做日志）—— InnoDB 引擎层

**作用**：保证事务的**持久性**（Durability），崩溃恢复。

```
核心思想：WAL（Write-Ahead Logging）
先写日志，再写磁盘数据页。

事务提交时（以 innodb_flush_log_at_trx_commit=1 为例）：
1. 修改先写到 Buffer Pool 的数据页（内存）→ 这些页变成「脏页」
2. 同时把对应变更写入 redo log buffer，再 fsync 到 redo log 文件（磁盘）
   ← 这一步完成，COMMIT 才能对客户端返回成功（持久性靠 redo，不靠数据文件）
3. 脏页之后由后台异步刷到 .ibd 数据文件（可以远晚于 COMMIT）

崩溃恢复时：
- 扫描 redo log，将已提交但未写入数据文件的修改重新应用
```

**怎么理解脏页（dirty page）？**

| 概念 | 含义 |
|------|------|
| 干净页 | Buffer Pool 里的页与磁盘 `.ibd` 上一致，或未被修改 |
| **脏页** | Buffer Pool 里**已经改过**，但**还没刷回**磁盘数据文件的页 |

```text
UPDATE accounts SET balance = 900 WHERE id = 1;

内存 Buffer Pool：页 P 里 balance 已是 900   ← 脏页（内存比磁盘新）
磁盘 .ibd：     页 P 里 balance 可能仍是 1000 ← 暂时落后没关系

只要 redo 已落盘：宕机后可用 redo 把 P 重放到 900
脏页刷盘只是「把内存里的最新页抄到数据文件」，不是事务成功的标志
```

**「事务提交成功的标志」是什么意思？——不是「事务做完之后才 fsync」**

容易把顺序理解反：

| 误解 | 实际 |
|------|------|
| 事务先完成，然后才做第 2 步 fsync | **fsync redo 是 COMMIT 过程中的关键一步**；做完才对客户端说「提交成功」 |
| 必须等脏页刷到 `.ibd` 事务才算成功 | **不必**。WAL 正是为了让提交只等 redo，不等数据页刷盘 |

```text
时间线（简化）：

  BEGIN
  UPDATE ...          → 改 Buffer Pool（产生脏页）+ 写 redo buffer
  COMMIT
    ├─ prepare / 写 redo、写 binlog（两阶段提交细节略）
    ├─ fsync redo     → 持久化完成，可以返回 OK   ← 提交成功靠这一步
    └─ 返回客户端「Query OK」
  ……稍后……
  后台刷脏页         → 把 Buffer Pool 中的脏页写到 .ibd（异步，可批量）
```

所以：
- **第 2 步**：发生在 **COMMIT 流程内部**，是「事务对外宣告成功」的持久化门槛（默认配置下）。
- **第 3 步**：发生在 **COMMIT 成功之后**（甚至很久之后），与「这次事务是否成功」解耦。

`innodb_flush_log_at_trx_commit` 会改变第 2 步的严格程度：`=1` 每次提交都 fsync（最安全）；`=2` 只写 OS 缓存；`=0` 可能每秒才刷——都更快，但宕机可能丢最近已「提交」的事务。

**「双 1」**：生产资损场景通常要求 `innodb_flush_log_at_trx_commit=1` **且** `sync_binlog=1`（每次提交 fsync binlog）。只改其中一个仍可能在崩溃后出现 redo/binlog 不一致或主从缺口；非资损、可接受秒级丢数时再成对放宽（如 2 + 100/1000），并写进故障预案。

**redo log 是循环写的**：

```
redo log 文件（8.0.30 之前默认 2 个 ib_logfile，各 48MB；
8.0.30+ 改为 innodb_redo_log_capacity 统一控制总容量，默认 100MB）
┌──────────────┐ ┌──────────────┐
│ ib_logfile0  │ │ ib_logfile1  │
│              │ │              │
│  write pos→  │ │              │
│              │ │   ←checkpoint│
└──────────────┘ └──────────────┘

write pos：当前写入位置（前进）
checkpoint：当前擦除位置（前进，擦除前先刷脏页）
write pos 追上 checkpoint → 必须先刷脏页，腾出空间
```

---

#### binlog（归档日志）—— Server 层

**作用**：主从复制、数据恢复（Point-in-Time Recovery）。

```
binlog 格式：
1. STATEMENT：记录 SQL 语句（体积小，但可能不确定性）
2. ROW：记录行变更的前后镜像（体积大，但精确）
3. MIXED：以 STATEMENT 为主，遇到不确定函数自动切换为 ROW

注意：MySQL 5.7.7 起服务器默认 binlog_format=ROW；
「默认 STATEMENT」只是 MIXED 模式内部的回退策略，不是服务器默认值。

生产环境推荐：ROW 格式（精确，不会因函数、触发器导致主从不一致）
```

---

#### undo log（回滚日志）—— InnoDB 引擎层

**作用**：
1. 事务**回滚**（Atomicity）
2. **MVCC** 的版本链（存储历史版本数据）

```
事务修改一行数据时：
1. 将旧值写入 undo log
2. 修改数据页中的新值
3. 新值的 DB_ROLL_PTR 指向 undo log 中的旧值

回滚时：沿 undo log 恢复旧值
MVCC 读时：沿 undo log 版本链查找可见版本
```

---

#### 三者协作：一条 UPDATE 语句的执行过程⭐⭐⭐

```
UPDATE accounts SET balance = 900 WHERE id = 1;
（原始 balance = 1000）

1. 从 Buffer Pool 读取 id=1 的数据页（若不在内存则从磁盘加载）
2. 将旧值(balance=1000)写入 undo log
3. 在内存中修改数据页(balance=900)，标记为脏页
4. 将修改记录写入 redo log buffer
5. 写入 binlog cache

事务提交时（两阶段提交）：
6. redo log 写入磁盘（prepare 状态）
7. binlog 写入磁盘
8. redo log 标记为 commit 状态

后续异步：
9. Buffer Pool 的脏页刷新到数据文件
```

**两阶段提交（2PC）的意义**：

```
保证 redo log 和 binlog 的一致性：

场景1：redo log prepare → binlog 写入成功 → redo log commit
✅ 正常流程

场景2：redo log prepare → binlog 写入失败 → 崩溃
恢复时：redo log 是 prepare 状态，binlog 中没有对应记录 → 回滚

场景3：redo log prepare → binlog 写入成功 → redo log commit 前崩溃
恢复时：redo log 是 prepare 状态，但 binlog 中有对应记录 → 提交

没有两阶段提交 → redo log 和 binlog 可能不一致 → 主从数据不一致
```

---

## InnoDB 内存体系

> 面试高频：Buffer Pool 的 LRU 为什么不是标准 LRU？Change Buffer 和 redo log 有什么区别？doublewrite 解决什么问题？

### Buffer Pool（缓冲池）⭐⭐⭐

**定位**：InnoDB 最大的内存区域（生产常配到物理内存的 50%~70%），缓存**数据页和索引页**（16KB/页），所有读写都先经过它。

```
读路径：SQL → Buffer Pool 命中？→ 命中直接返回
                              → 未命中则从磁盘读入 Buffer Pool 再返回
写路径：SQL → 直接改 Buffer Pool 中的页（变脏页）→ 写 redo log → 返回
              脏页由后台线程异步刷盘
```

**为什么需要它**：磁盘随机 I/O（机械盘毫秒级）与内存访问（纳秒级）差 5~6 个数量级。Buffer Pool 把「每次读写都落盘」变成「内存操作 + 异步刷盘」，是 InnoDB 性能的地基。

#### LRU 链表：InnoDB 的改良版

标准 LRU 的问题：**全表扫描或预读会把真正的热数据挤出去**（扫描的页只读一次，却占据链表头部）。

InnoDB 的解法：**LRU 分为新生代（new，后 5/8）和老年代（old，前 3/8）**：

```
新读入的页 → 插入 old 区头部（不是整个链表头部！）
old 区的页被再次访问 → 且距首次读入超过 innodb_old_blocks_time（默认1s）
                     → 才晋升到 new 区头部
```

- 全表扫描的页只进 old 区，很快从 old 区尾部被淘汰，**不会污染 new 区的热数据**。
- `innodb_old_blocks_time` 防止「扫描时同一页被连续访问多次」误判为热数据。

**相关链表**：
| 链表 | 作用 |
|------|------|
| LRU 链表 | 管理所有缓存页（含空闲页、干净页、脏页）的淘汰顺序 |
| Free 链表 | 空闲页，新读入的页从这里取 |
| Flush 链表 | 只含**脏页**，按最早修改的 LSN 排序，刷脏页时按序写盘 |

#### 脏页刷新（Checkpoint 的触发）

redo log 是循环写的，空间用完前必须把对应的脏页刷盘（否则崩溃时无法重放）。刷脏的触发时机：

1. **redo log 快满**（sharp checkpoint，会阻塞写入，要尽量避免）
2. **Buffer Pool 空间不足**，需要淘汰脏页时先刷盘
3. **后台线程定期刷**（`innodb_io_capacity` 控制每秒刷盘页数，应按磁盘 IOPS 设置）
4. **MySQL 正常关闭**时全量刷盘

**Checkpoint 的本质**：记录「LSN ≤ checkpoint 的 redo 都已落盘」，崩溃恢复只需从 checkpoint 之后开始重放，缩短恢复时间。

### Change Buffer（写缓冲）⭐⭐

**问题**：更新**非唯一二级索引**时，如果目标索引页不在 Buffer Pool，难道要先从磁盘读进来再改？（一次随机读 I/O）

**解法**：不在内存的二级索引页，修改先记入 Change Buffer（它在 Buffer Pool 中，也会持久化到系统表空间），**直接返回**；等以后该页被读入 Buffer Pool 时再 merge（合并）真实修改。

```
UPDATE t SET name='x' WHERE id=1;   -- name 上有普通二级索引
索引页在内存？ → 是：直接改
              → 否：修改记入 Change Buffer，返回（省一次随机读）
后续该页被读入 → merge Change Buffer 中的修改
```

**限制**：只适用于**非唯一**二级索引——唯一索引必须读页检查唯一性，无法缓冲。

**Change Buffer vs redo log**（经典面试题）：
- Change Buffer 省的是「读索引页进内存」的**随机读 I/O**；
- redo log 保证的是「修改不丢」的**持久性**（Change Buffer 本身的修改也写 redo log）；
- 两者解决不同问题，不是替代关系。

**适用场景**：写多读少、且写入后不会立刻读该页的业务（如日志、流水）。读多写少的场景 Change Buffer 收益低还占内存，可调小 `innodb_change_buffer_max_size`。

### Doublewrite Buffer（双写缓冲）⭐⭐

**问题——页断裂（partial page write）**：InnoDB 页 16KB，而文件系统/磁盘原子写单位通常只有 4KB（或 512B）。刷脏页时宕机，可能只写了一半——**页损坏**。此时 redo log 也救不了：redo 是「在完好的页上重放修改」，页本身坏了无法重放。

**解法**：刷脏页时，先把页**顺序写**到共享表空间的 doublewrite 区域（2MB，顺序写很快），再写真正的数据页。崩溃恢复时若发现页损坏，从 doublewrite 副本恢复该页，再重放 redo。

```
脏页刷盘：页 → doublewrite 区（顺序写，fsync）
             → 各数据文件（随机写）
宕机恢复：页校验失败？→ 从 doublewrite 拷贝完整页 → 重放 redo
```

代价是写放大（同一页写两次），但 doublewrite 区是顺序写，实际损耗很小。若文件系统/存储本身保证 16KB 原子写（如 ZFS、部分支持原子写的 SSD），可以关闭。

### Undo 表空间与 Purge

- **undo log 的归宿**：8.0 默认 2 个 undo 表空间（`innodb_undo_tablespaces`），可随时在线增删（5.7 时代 undo 在系统表空间里，膨胀后无法回收——这是 5.7 的经典坑）。
- **Purge 线程**：事务提交后，undo 版本不能立刻删（可能有其他事务的 Read View 还要读它）。Purge 线程负责清理「不再被任何活跃 Read View 引用」的旧版本，并真正删除带 delete-mark 的记录。
- **长事务的危害在这里落地**：一个老事务的 Read View 不释放 → purge 无法清理它之后的所有版本 → undo 表空间膨胀、版本链变长、查询变慢。这就是「线上要避免长事务」的底层原因。

### 一张图串起来

```
UPDATE accounts SET balance=900 WHERE id=1;

内存中：
  ① 改 Buffer Pool 中的数据页（脏页）     ← LRU/Flush 链表管理
  ② 二级索引页不在内存 → 记 Change Buffer
  ③ 写 undo（旧值 1000，供回滚/MVCC）     ← Purge 线程事后清理
  ④ 写 redo log buffer → 提交时 fsync      ← 崩溃恢复的保障

异步落盘：
  ⑤ 脏页 → doublewrite 区 → 数据文件      ← 防页断裂
  ⑥ checkpoint 推进，redo 空间循环复用
```

---

## 索引原理

### B+Tree结构

#### 为什么用B+Tree？

**对比B-Tree的优势**：
1. **所有数据存储在叶子节点**：范围查询高效
2. **叶子节点形成链表**：支持顺序访问
3. **非叶子节点只存储键值**：单个节点可以存储更多索引项，树的高度更低

```
               [10, 20]
              /    |    \
         [5,8]  [15,18] [25,30]
        /  |  \   |  \    |  \
      [数据][数据][数据][数据][数据][数据]
               ↓
          叶子节点双向链表（支持范围查询）
```

**核心特性**：
- InnoDB默认页大小：**16KB**
- 一个3层B+Tree可以存储：**约2000万条记录**
  - 假设索引键8字节 + 指针6字节 = 14字节
  - 非叶子节点：16KB / 14B ≈ 1170个键值
  - 三层：1170 × 1170 × 16 ≈ 2000万

---

#### B+树的分裂与合并（底层原理）⭐⭐⭐

**为什么需要分裂/合并？**
- B+树必须保持**平衡**（所有叶子节点在同一层）
- 每个节点的key数量有上下限：**[m/2, m]**（m为阶数）
- 插入/删除时可能违反这个限制

---

##### 1. 插入导致的分裂

**规则**：节点 key 数超过上限 → 从中间一分为二 → 中间 key 提升到父节点 → 父节点若也满则**递归分裂**，根节点分裂时树高 +1。

**一个完整例子（3 阶 B+ 树，节点最多 2 个 key，父节点也满 → 递归分裂）**：
```
【初始】
           [20, 40]
          /   |    \
      [10] [20,30] [40,50]

【插入 45】
① 叶子 [40,50] 满 → 分裂为 [40] 和 [45,50]，提升 45
② 父节点 [20,40] 插入 45 后也满 → 分裂为 [20] 和 [45]，提升 40
③ 根节点不存在 → 创建新根 [40]，树高 +1

【结果】
              [40]
            /      \
         [20]      [45]
        /    \    /    \
    [10]  [20,30] [40] [45,50]
```

要点：B+ 树**通过根节点分裂长高**，所以所有叶子始终在同一层（保持平衡）；中间 key 提升后在父节点中充当「分隔牌」，右子树的 key 都 ≥ 它。

---

##### 2. 删除导致的合并/重分配

**规则**：删除后节点 key 数低于下限（< m/2）→ 先尝试**向兄弟节点借 key**（重分配，同时更新父节点分隔 key）→ 兄弟也不够借时**与兄弟合并**（删除父节点中的分隔 key，可能递归向上，根节点被删空时树高 -1）。

```
【借 key】删除 30 后右节点欠员，左兄弟 [10,15] 有富余：
         [20]                    [15]
        /    \      → 借出15 →   /    \
    [10,15]  [20]            [10]    [15,20]

【合并】删除 30 后左右都只剩 1 个 key，无法借：
         [20]
        /    \      → 合并 →   [10,20]   （树高 -1）
    [10]    [20]
```

要点：优先借 key 而非合并（合并会改变父节点，代价更大）；InnoDB 实际实现还有**页填充因子**（默认 15/16 满才分裂）等工程优化，减少分裂频率。

---

##### 3. 为什么用B+树而不是B树？

**B树 vs B+树对比**：

| 特性 | B树 | B+树 |
|------|-----|------|
| **数据存储** | 所有节点都存数据 | 只有叶子节点存数据 |
| **叶子节点** | 无链表 | 有双向链表 |
| **非叶子节点** | 存key+data | 只存key+指针 |
| **范围查询** | 需要中序遍历（O(n)） | 叶子链表扫描（O(logn + k)） |
| **单点查询** | 可能在非叶子节点命中（快） | 一定要到叶子节点（稳定） |

**选型一句话**：磁盘页为 IO 单位时，B+ 树「矮树 + 叶子链表」同时吃到点查稳定与范围顺序扫；上表对比已覆盖细节，不必再背四段展开。

---

##### 4. 为什么InnoDB一定要有主键？⭐⭐⭐

**问题**：如果不显式创建主键会怎样？

**InnoDB的处理逻辑**：
```
1. 优先使用显式定义的PRIMARY KEY
2. 如果没有主键，选择第一个NOT NULL的UNIQUE索引
3. 如果都没有，InnoDB自动创建一个隐藏的6字节ROW_ID列作为主键
```

**为什么必须有主键？（聚簇索引原理）**

InnoDB是**聚簇索引**存储引擎：
- 数据按主键顺序存储在B+树的叶子节点
- 没有主键，就无法组织数据

```
【主键索引（聚簇索引）】
           [50]
          /    \
       [20]    [80]
      /   \    /   \
  [id=1]  [id=20] [id=50] [id=80]
   |data|  |data|  |data|  |data|

数据和索引存储在一起！

【二级索引（非聚簇索引）】
           [Alice]
          /       \
     [Bob]        [Charlie]
      |             |
   (id=20)       (id=50)
      ↓             ↓
  回表到主键索引查data
```

**自动创建ROW_ID的问题**：

1. **性能问题**：
   ```
   ROW_ID是全局递增的，多表共享
   → 高并发插入时，ROW_ID分配成为瓶颈（全局锁）
   ```

2. **溢出问题**：
   ```
   ROW_ID只有6字节（48位）
   最大值：2^48 ≈ 280万亿
   达到最大值后会溢出，重新从0开始
   → 可能覆盖旧数据！
   ```

3. **无法利用主键的业务语义**：
   ```
   -- 有显式主键：可以直接用主键查询（一次I/O）
   SELECT * FROM users WHERE id = 123;

   -- 无显式主键：需要全表扫描或二级索引（多次I/O）
   SELECT * FROM users WHERE email = 'alice@example.com';
   ```

**最佳实践**：
- ✅ 使用自增ID作为主键（`BIGINT AUTO_INCREMENT`）
- ✅ 或使用业务上唯一的字段（如用户ID、订单号）
- ❌ 不要依赖隐藏的ROW_ID

---

### 索引类型

#### 1. 主键索引（聚簇索引）

**InnoDB中，主键索引的叶子节点存储完整行数据**。

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,  -- 主键索引
  name VARCHAR(50),
  age INT,
  created_at DATETIME
);
```

**主键选择建议**：
- ✅ 使用自增ID（`AUTO_INCREMENT`）
  - 顺序插入，避免页分裂
  - 索引结构紧凑
- ❌ 避免UUID作为主键
  - 无序插入，频繁页分裂
  - 索引碎片化

**页分裂示例**：
```
插入顺序ID：
[1,2,3] -> [4,5,6] -> [7,8,9]  ✅ 顺序插入，无页分裂

插入UUID：
[UUID1] -> [UUID2插入中间] -> 页分裂  ❌
```

---

#### 2. 普通索引（二级索引/辅助索引）

**二级索引的叶子节点存储：索引列值 + 主键值**。

```sql
CREATE INDEX idx_name ON users(name);
```

**查询过程**（回表）：
1. 在`idx_name`索引树中找到`name='Alice'`的记录，得到`id=10`
2. 拿着`id=10`去主键索引树查找完整行数据

```
SELECT * FROM users WHERE name = 'Alice';

二级索引查找：idx_name -> 找到 (Alice, id=10)
       ↓
回表查询：主键索引 -> 根据 id=10 找到完整行
```

---

#### 3. 唯一索引

```sql
CREATE UNIQUE INDEX idx_email ON users(email);
```

**与普通索引的区别**：
- 插入时检查唯一性（有额外开销）
- 等值查询优化器常走 `const`（最多一行），选择性更好

**唯一索引会不会回表？**

会——**和「唯一」无关，和「二级索引 + 是否覆盖」有关**。

在 InnoDB 里，非主键的唯一索引仍是 **二级索引**：叶子节点存的是 `索引列值 + 主键值`，**不是整行**。因此：

| 查询 | 是否回表 | 原因 |
|------|----------|------|
| `SELECT * FROM users WHERE email = 'a@x.com'` | **要回表** | 需要 `name/age/...`，二级索引里没有 |
| `SELECT id, email FROM users WHERE email = 'a@x.com'` | **不回表** | `id`（主键）和 `email` 都在索引里，覆盖索引 |
| `SELECT * FROM users WHERE id = 10`（主键） | **不回表** | 主键叶子就是整行 |

```text
唯一索引 idx_email 叶子： (email='a@x.com', id=10)   ← 没有 name、age
SELECT * WHERE email=...
  → 先在 idx_email 找到 id=10
  → 再拿 id=10 回主键索引取完整行   ← 这就是回表
```

和普通二级索引的差别主要在 **命中行数**，不在「要不要回表」这条规则：

| | 普通索引 `idx_name` | 唯一索引 `idx_email` |
|--|---------------------|----------------------|
| 叶子存什么 | 索引列 + 主键 | 一样 |
| `SELECT *` 等值 | 可能命中多行 → **多次回表** | 最多一行 → **最多一次回表** |
| 是否可覆盖避免回表 | 可以 | 可以 |

---

#### 4. 全文索引（Full-Text）

```sql
CREATE FULLTEXT INDEX idx_content ON articles(content);

SELECT * FROM articles WHERE MATCH(content) AGAINST('MySQL索引');
```

**适用场景**：
- 文章搜索、日志检索
- **不适合**：精确匹配、短文本（可用ES代替）

---

### 联合索引

#### 最左前缀原则

**索引定义**：
```sql
CREATE INDEX idx_abc ON users(a, b, c);
```

**可以使用该索引的查询**：
```sql
-- ✅ 使用索引：a
SELECT * FROM users WHERE a = 1;

-- ✅ 使用索引：a, b
SELECT * FROM users WHERE a = 1 AND b = 2;

-- ✅ 使用索引：a, b, c（全部）
SELECT * FROM users WHERE a = 1 AND b = 2 AND c = 3;

-- ✅ 使用索引：a, b（范围查询后续列失效）
SELECT * FROM users WHERE a = 1 AND b > 2 AND c = 3;
```

**不能使用该索引的查询**：
```sql
-- ❌ 不使用索引：缺少a
SELECT * FROM users WHERE b = 2;

-- ❌ 不使用索引：缺少a
SELECT * FROM users WHERE b = 2 AND c = 3;

-- ❌ 不使用索引：缺少a
SELECT * FROM users WHERE c = 3;
```

**MySQL 8.0+ 索引跳跃扫描（Index Skip Scan）**：
```sql
-- MySQL 8.0+ 可能使用索引跳跃扫描
SELECT * FROM users WHERE b = 2;
-- 但效率低于完整匹配
```

---

#### 索引顺序设计

**原则**：优先看查询形态，而不是只看区分度。

```text
经验顺序：等值列在前 → 范围列在后 → 排序/分组列按序
反例：常查 WHERE gender=? AND created_at BETWEEN ...，
     把低区分度的 gender 放前面反而合理（等值命中后再走范围）
```

```sql
-- 若主要查询是 WHERE gender=? AND city=?，低区分度列放前也没问题
CREATE INDEX idx_gc ON users(gender, city);

-- 若主要查询是 WHERE city=? AND age>?，则 city 在前
CREATE INDEX idx_ca ON users(city, age);
```

区分度用于辅助判断（区分度太低的列单独建索引意义不大），但不是索引列顺序的唯一依据。

**计算区分度**：
```sql
SELECT
  COUNT(DISTINCT city) / COUNT(*) AS city_selectivity,
  COUNT(DISTINCT age) / COUNT(*) AS age_selectivity,
  COUNT(DISTINCT gender) / COUNT(*) AS gender_selectivity
FROM users;
```

---

### 覆盖索引与回表

#### 覆盖索引（Covering Index）

**定义**：查询的列全部包含在索引中，无需回表。

```sql
-- 索引定义
CREATE INDEX idx_name_age ON users(name, age);

-- ✅ 覆盖索引：只需要 name 和 age，索引中都有
SELECT name, age FROM users WHERE name = 'Alice';

-- ❌ 需要回表：需要 email，索引中没有
SELECT name, age, email FROM users WHERE name = 'Alice';
```

**EXPLAIN中的标识**：
```
Extra: Using index  -- 覆盖索引
```

---

#### 回表（Table Lookup）

**回表是什么**：通过二级索引拿到主键后，再到主键（聚簇）索引上取完整行——多一次 B+Tree 查找。

**开销大不大？取决于「回表次数」和「Buffer Pool 命中」**：

| 场景 | 大致代价 | 说明 |
|------|----------|------|
| 唯一/主键等值，回表 **1 次** | 通常可接受 | 二级索引 1 次 + 主键 1 次；热数据多在内存 |
| 范围查询命中 **成百上千行**，每行都回表 | **很贵** | 主键页往往不连续 → 大量随机 I/O |
| 覆盖索引，回表 **0 次** | 最好 | 只扫二级索引叶子 |

```text
单次回表 ≈ 再走一遍主键 B+Tree（深度约 3～4 层）
  · Buffer Pool 命中：主要是 CPU/指针开销，毫秒内
  · 未命中：可能一次磁盘随机读（机械盘更痛，SSD 仍比顺序扫贵）

N 次回表 ≈ N 次「随机找主键页」
  · N=1（唯一索引等值）：一般不是瓶颈
  · N=10000（普通索引 LIKE 'A%' + SELECT *）：容易成为慢查询主因
```

因此：**「唯一索引会回表吗」→ 可能；「回表开销大吗」→ 一次不大，一万次就大。** 优化重点是减少回表次数（覆盖索引、延迟关联、少 `SELECT *`），而不是纠结「唯一会不会回表」。

**优化策略**：
```sql
-- 场景：查询用户信息，条件是 name
-- 表结构：id(主键), name, age, email, address

-- 延迟关联的真正收益场景：深分页（大 OFFSET）
-- ❌ 慢：LIMIT 100000, 10 需要先扫 100010 行并逐行回表再丢弃
SELECT * FROM users WHERE name LIKE 'A%' ORDER BY id LIMIT 100000, 10;

-- ✅ 快：子查询只走覆盖索引定位 10 个 id，再回表 10 次
CREATE INDEX idx_name ON users(name);

SELECT u.*
FROM users u
INNER JOIN (
  SELECT id FROM users WHERE name LIKE 'A%' ORDER BY id LIMIT 100000, 10
) t ON u.id = t.id;
-- 回表次数从 100010 降到 10
```

---

### 索引失效场景

#### 1. 函数操作

```sql
-- ❌ 索引失效：对索引列使用函数
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- ✅ 使用索引
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

---

#### 2. 隐式类型转换

```sql
-- 表结构：phone VARCHAR(20), INDEX(phone)

-- ❌ 索引失效：phone是字符串，传了数字
SELECT * FROM users WHERE phone = 13800138000;
-- MySQL会转换为：WHERE CAST(phone AS SIGNED) = 13800138000

-- ✅ 使用索引
SELECT * FROM users WHERE phone = '13800138000';
```

---

#### 3. 前导模糊查询

```sql
-- ❌ 索引失效
SELECT * FROM users WHERE name LIKE '%Alice';

-- ✅ 使用索引（前缀匹配）
SELECT * FROM users WHERE name LIKE 'Alice%';
```

---

#### 4. OR条件

```sql
-- 索引：idx_name, idx_age

-- ❌ 可能不使用索引（取决于优化器）
SELECT * FROM users WHERE name = 'Alice' OR age = 30;

-- ✅ 改写为 UNION（各自使用索引）
SELECT * FROM users WHERE name = 'Alice'
UNION
SELECT * FROM users WHERE age = 30;
```

---

#### 5. 不等于条件

```sql
-- ❌ 可能不使用索引
SELECT * FROM users WHERE status != 1;

-- ✅ 改写为等值查询
SELECT * FROM users WHERE status IN (0, 2, 3, 4);
```

---

### 索引下推

#### ICP（Index Condition Pushdown）⭐⭐

MySQL 5.6 引入的优化，将 WHERE 条件的一部分下推到存储引擎层在索引遍历时直接过滤，减少回表次数。

```sql
-- 联合索引：idx_name_age(name, age)
SELECT * FROM users WHERE name LIKE 'A%' AND age = 25;
```

```
无 ICP（MySQL 5.5 及之前）：
1. 索引扫描：找到所有 name LIKE 'A%' 的记录（可能 1000 条）
2. 回表 1000 次，取完整行
3. Server 层过滤 age = 25（最终可能只有 10 条）
→ 回表 1000 次，浪费 990 次

有 ICP（MySQL 5.6+）：
1. 索引扫描：找到 name LIKE 'A%' 的记录
2. 在索引中直接检查 age = 25（age 在联合索引中有值）
3. 只有满足 age = 25 的记录才回表（10 次）
→ 回表 10 次，大幅减少 I/O
```

**EXPLAIN 中的标识**：
```
Extra: Using index condition  -- 使用了索引下推
```

---

## SQL执行机制

### 一条SQL的生命周期

MySQL 的执行过程可以先分成几层看：

```
客户端 / 应用程序
   ↓  TCP 连接 / Unix Socket
MySQL Server 层
   ↓  解析器 → 预处理器 → 优化器 → 执行器
存储引擎层（InnoDB）
   ↓  Buffer Pool / B+Tree / 锁 / MVCC / redo & undo
磁盘文件
```

**普通 `SELECT` 的大致流程**：

```sql
SELECT name FROM users WHERE id = 1;
```

```
1. 连接器：建立连接，认证用户，读取权限和会话变量
2. 解析器：词法/语法分析，生成语法树
3. 预处理器：检查表、列是否存在，展开 `*`，做名字解析
4. 优化器：选择访问路径（走哪个索引、JOIN 顺序、是否 filesort）
5. 执行器：按照执行计划调用存储引擎接口
6. InnoDB：通过 B+Tree 定位数据页，结合 MVCC/锁返回记录
7. Server 层：过滤、排序、聚合、返回结果集
```

> 预处理语句（Prepared Statement）属于这条链路中的「解析/预处理」优化：把 SQL 模板提前解析好，后续执行只绑定参数。它不是事务机制，也不是索引原理的一部分。

---

### 连接与会话状态

连接建立后，MySQL 会为当前连接维护一份会话上下文：

| 会话状态 | 说明 |
|----------|------|
| 用户与权限 | 认证后确定当前用户能访问哪些库表 |
| 字符集与排序规则 | 影响字符串比较、排序和索引使用 |
| 隔离级别 | 影响 Read View、锁和一致性读 |
| autocommit | 决定每条语句是否自动提交 |
| 临时表 / 用户变量 | 只在当前连接内可见 |
| Prepared Statement handle | 服务端预处理语句的 stmt_id 绑定在连接上 |

**关键点**：
- MySQL 连接是有状态的，不只是一次请求/响应。
- 服务端 Prepared Statement 不能跨连接复用；连接池里如果连接被关闭，对应的 stmt_id 也会失效。
- 长连接如果大量创建 Prepared Statement 但不释放，会占用服务端内存，受 `max_prepared_stmt_count` 限制。

---

### 解析器与预处理器

这两个阶段容易被混在一起：

| 阶段 | 做什么 | 常见错误 |
|------|--------|----------|
| 解析器（Parser） | 词法分析、语法分析，判断 SQL 语法是否合法 | `You have an error in your SQL syntax` |
| 预处理器（Preprocessor） | 检查表/列是否存在，解析库名、表名、列名，展开 `*` | `Unknown column` / `Table doesn't exist` |

```sql
-- 语法错误：解析器阶段失败
SELECT FROM users;

-- 语法正确，但字段不存在：预处理器阶段失败
SELECT not_exist_column FROM users;
```

**为什么这里重要？**
- Prepared Statement 的 `PREPARE` 阶段会先经过解析和预处理，所以 SQL 模板的结构必须提前合法。
- 这也解释了为什么占位符 `?` 只能替代值，不能替代表名/列名：表名和列名必须在预处理阶段就解析完成。

---

### 优化器与执行器

优化器负责「选计划」，执行器负责「按计划执行」。

**优化器会考虑**：
1. 能用哪些索引（`possible_keys`）
2. 选择哪个索引（`key`）
3. 预计扫描多少行（`rows`）
4. 是否需要排序、临时表、回表
5. JOIN 时先访问哪张表

```sql
SELECT *
FROM orders
WHERE user_id = 123
  AND status = 1
ORDER BY created_at DESC
LIMIT 10;
```

这个查询可能有多种计划：
- 走 `idx_user`：先找某个用户的订单，再过滤状态和排序。
- 走 `idx_status`：先找某种状态的订单，再过滤用户。
- 走 `idx_created`：按时间顺序扫描，避免额外排序，但可能过滤很多行。
- 走联合索引 `(user_id, status, created_at)`：如果设计得好，过滤和排序都能利用索引。

**执行器**拿到计划后，会不断调用 InnoDB 的接口读取下一行，再在 Server 层做必要的过滤、排序、聚合和返回。

---

### EXPLAIN详解

```sql
EXPLAIN SELECT * FROM users WHERE name = 'Alice';
```

**输出列说明**：

| 列名 | 说明 | 重点关注 |
|------|------|----------|
| **id** | 查询序号 | 越大越先执行 |
| **select_type** | 查询类型 | SIMPLE, SUBQUERY, UNION |
| **table** | 表名 | - |
| **type** ⭐ | 访问类型 | **性能从好到差**：system > const > eq_ref > ref > range > index > ALL |
| **possible_keys** | 可能用到的索引 | - |
| **key** ⭐ | 实际使用的索引 | NULL表示未使用索引 |
| **key_len** | 实际使用的索引字节数 | 联合索引中越大通常表示用到了更多列；结合 key/rows 判断，不是越短越好 |
| **ref** | 索引引用 | const, func |
| **rows** ⭐ | 扫描行数 | 越少越好 |
| **filtered** | 过滤比例 | 百分比，越大越好 |
| **Extra** ⭐ | 额外信息 | **Using index**(覆盖索引), **Using filesort**(需排序), **Using temporary**(需临时表) |

---

#### type类型详解

**从好到差的顺序**：

1. **system**：表只有一行（系统表）
2. **const**：主键或唯一索引的等值查询
   ```sql
   SELECT * FROM users WHERE id = 1;  -- type: const
   ```

3. **eq_ref**：唯一索引扫描（JOIN时使用）
   ```sql
   SELECT * FROM orders o
   JOIN users u ON o.user_id = u.id;  -- u表 type: eq_ref
   ```

4. **ref**：非唯一索引的等值查询
   ```sql
   SELECT * FROM users WHERE name = 'Alice';  -- type: ref
   ```

5. **range**：范围查询
   ```sql
   SELECT * FROM users WHERE age > 18;  -- type: range
   ```

6. **index**：全索引扫描（比ALL好，但也不理想）
   ```sql
   SELECT id FROM users;  -- type: index（扫描主键索引）
   ```

7. **ALL**：全表扫描（最差）⚠️
   ```sql
   SELECT * FROM users WHERE email LIKE '%@gmail.com';  -- type: ALL
   ```

---

#### Extra字段解读

**好的标识**：
- ✅ **Using index**：覆盖索引，无需回表
- ✅ **Using index condition**：索引下推（ICP）

**需要优化的标识**：
- ⚠️ **Using where**：WHERE条件在存储引擎层无法过滤，需在Server层过滤
- ⚠️ **Using filesort**：需要额外排序（无法使用索引排序）
- ⚠️ **Using temporary**：需要临时表（GROUP BY或DISTINCT时）

---

### 索引选择

#### 优化器的索引选择

**MySQL优化器选择索引的依据**：
1. **扫描行数**（最重要）
2. **是否需要排序**
3. **是否需要回表**

**示例**：
```sql
-- 表结构
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  user_id BIGINT,
  status TINYINT,
  created_at DATETIME,
  INDEX idx_user(user_id),
  INDEX idx_status(status),
  INDEX idx_created(created_at)
);

-- 查询
SELECT * FROM orders
WHERE user_id = 123 AND status = 1
ORDER BY created_at DESC
LIMIT 10;
```

**优化器的选择逻辑**：
- 如果 `user_id=123` 的记录很少（如10条）：选择 `idx_user`
- 如果 `status=1` 的记录很少（如100条）：选择 `idx_status`
- 如果都不少，但需要按 `created_at` 排序：可能选择 `idx_created`（避免filesort）

**强制指定索引**：
```sql
SELECT * FROM orders FORCE INDEX(idx_user)
WHERE user_id = 123 AND status = 1;
```

---

### 预处理语句

#### 什么是预处理语句

**Prepared Statement（预处理语句 / 预编译语句）**：把 SQL 的**模板（结构）**和**参数（数据）**分开，先把带占位符的 SQL 模板发给服务端编译好，之后只传参数即可反复执行。

```sql
-- 普通拼接 SQL（每次都是一条完整的新语句）
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;

-- 预处理语句（模板只编译一次，参数用 ? 占位）
PREPARE stmt FROM 'SELECT * FROM users WHERE id = ?';

SET @id = 1;
EXECUTE stmt USING @id;   -- 第一次执行

SET @id = 2;
EXECUTE stmt USING @id;   -- 复用模板，只换参数

DEALLOCATE PREPARE stmt;  -- 释放
```

**三个阶段**：

| 阶段 | 命令 | 作用 |
|------|------|------|
| **PREPARE** | `COM_STMT_PREPARE` | 服务端解析 SQL 模板、做语法/权限检查、生成执行结构，返回一个 statement handle（stmt_id） |
| **EXECUTE** | `COM_STMT_EXECUTE` | 绑定参数并执行，可重复多次 |
| **DEALLOCATE** | `COM_STMT_CLOSE` | 释放服务端为该语句分配的资源 |

> 占位符是 `?`（位置占位符），**不能**用来替代表名、列名、关键字，只能替代「值」。

---

#### 执行流程与二进制协议

MySQL 客户端与服务端有两套协议：

```
普通查询（Text Protocol）：
  COM_QUERY → 发送完整 SQL 字符串 → 服务端每次都要 解析 + 优化 + 执行
  结果集以「文本」形式返回（数字也按字符串传）

预处理（Binary Protocol）：
  ① COM_STMT_PREPARE  →  发送一次 SQL 模板，服务端编译，返回 stmt_id + 参数/列元数据
  ② COM_STMT_EXECUTE  →  只发 stmt_id + 参数（二进制编码），可执行 N 次
  结果集以「二进制」形式返回（int/datetime 等按原始类型传，更紧凑）
```

**带来的收益**：
1. **省去重复解析**：同一模板执行 1000 次，只解析 1 次（SQL 解析 = 词法分析 + 语法树构建，有成本）。
2. **网络传输更省**：参数二进制编码，不用每次重传完整 SQL 文本；大整数、时间类型比文本更短。
3. **类型更精确**：二进制协议明确携带参数类型，避免文本协议的来回转换。

---

#### 防SQL注入原理

这是预处理**最重要**的价值。参数和 SQL 结构是**分开发送**的，参数永远被当作「值」，绝不会被解析成 SQL 语法。

```sql
-- ❌ 字符串拼接：存在注入风险
-- 用户输入 name = "' OR '1'='1"
query = "SELECT * FROM users WHERE name = '" + name + "'";
-- 实际执行：SELECT * FROM users WHERE name = '' OR '1'='1'  → 拖库

-- ✅ 预处理：参数永远是值
PREPARE stmt FROM 'SELECT * FROM users WHERE name = ?';
SET @name = "' OR '1'='1";
EXECUTE stmt USING @name;
-- 实际语义：name 字段 == 字符串 "' OR '1'='1"（查不到任何行，无法改变 SQL 结构）
```

**关键点**：注入之所以发生，是因为「数据」被当成了「代码」。预处理在编译阶段就已经确定了 SQL 结构，`EXECUTE` 阶段传入的参数只能填进占位符，无法新增语法。

> ⚠️ 注意：占位符不能用于表名/列名/`ORDER BY` 字段。如果这些必须动态拼接，要用**白名单**校验，不能直接拼用户输入。

```go
// Go database/sql：底层走的就是预处理（占位符 ?）
db.Query("SELECT * FROM users WHERE name = ? AND age > ?", name, age)

// Java JDBC
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM users WHERE name = ? AND age > ?");
ps.setString(1, name);
ps.setInt(2, age);
```

---

#### 服务端与客户端预处理

预处理分为两种实现，面试常被追问区别：

| 类型 | 实现位置 | 说明 |
|------|----------|------|
| **服务端预处理** | MySQL Server | 真正走 `COM_STMT_PREPARE`，服务端持有 stmt_id 和元数据 |
| **客户端预处理（模拟）** | 数据库驱动 | 驱动在本地把参数转义后拼进 SQL，仍以 `COM_QUERY` 文本协议发送 |

**为什么有客户端模拟？**
- 服务端预处理每条语句要在服务端维护状态（占内存），且 PREPARE/EXECUTE 是两次网络往返；对「只执行一次」的语句反而更慢。
- 客户端模拟由驱动做安全转义，同样能防注入，且只需一次网络往返。

**常见驱动配置**：
```
JDBC (MySQL Connector/J)：
  useServerPrepStmts=true   → 启用服务端预处理
  cachePrepStmts=true       → 缓存预处理语句（配合连接池复用 stmt）
  默认 useServerPrepStmts=false（默认是客户端模拟）

Go (go-sql-driver/mysql)：
  interpolateParams=true    → 客户端插值（单次往返，不在服务端 prepare）
  默认 false → 走服务端预处理
```

> 实践建议：高频、重复执行的 SQL（如循环里同一条语句）用服务端预处理 + 语句缓存收益最大；一次性语句可用客户端插值减少往返。

---

#### 预处理与执行计划缓存

一个常见误区：**「预处理会缓存执行计划，所以一定更快」**——这在 MySQL 上并不完全成立。

```
预处理一定省的：SQL 解析（parse）开销
预处理不一定省的：查询优化（optimize）开销

MySQL 的行为（与 Oracle / SQL Server 不同）：
- 早期版本：每次 EXECUTE 仍会重新优化，生成执行计划
- MySQL 8.0.x 中后期引入「prepared statement 执行计划缓存」等优化，
  但仍受参数变化、统计信息影响，不能简单认为计划被永久复用
```

**为什么不总缓存计划？**
- 参数不同，最优计划可能不同（典型如范围条件 `WHERE age > ?`，age>1 和 age>99 的最优索引/扫描方式可能不一样）。
- 缓存了「针对某个参数最优」的计划，换个参数可能反而更慢（即 bind peeking 带来的计划倾斜问题）。

**结论**：
- 预处理的核心收益是**防注入** + **省解析** + **二进制协议省传输**，而不是「缓存执行计划让查询变快」。
- 不要为了「让计划缓存」而过度依赖预处理，正确的性能优化仍是索引设计 + EXPLAIN 分析。

---

## 事务与锁

### 事务隔离级别

#### ACID特性

- **Atomicity（原子性）**：事务要么全部成功，要么全部失败
- **Consistency（一致性）**：事务前后数据一致
- **Isolation（隔离性）**：事务之间相互隔离
- **Durability（持久性）**：事务提交后永久保存

---

#### 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 |
|---------|------|-----------|------|------|
| **READ UNCOMMITTED(读未提交)** | ✅ 可能 | ✅ 可能 | ✅ 可能 | 高 |
| **READ COMMITTED(读已提交)** | ❌ 不可能 | ✅ 可能 | ✅ 可能 | 中高 |
| **REPEATABLE READ(可重复读)** (默认) | ❌ 不可能 | ❌ 不可能 | ⚠️ 可能 | 中 |
| **SERIALIZABLE(串行化)** | ❌ 不可能 | ❌ 不可能 | ❌ 不可能 | 低 |

**InnoDB默认：REPEATABLE READ（RR）**

---

#### 问题定义

**1. 脏读（Dirty Read）**：读到其他事务未提交的数据

```sql
-- 时间线
T1: BEGIN;
T1: UPDATE accounts SET balance = balance - 100 WHERE id = 1;
T2:                                                          BEGIN;
T2:                                                          SELECT balance FROM accounts WHERE id = 1;  -- 读到-100（未提交）
T1: ROLLBACK;  -- T2读到的数据无效了
```

---

**2. 不可重复读（Non-Repeatable Read）**：同一事务中，多次读取同一行数据不一致

```sql
-- 时间线
T1: BEGIN;
T1: SELECT balance FROM accounts WHERE id = 1;  -- 读到 1000
T2:                                                BEGIN;
T2:                                                UPDATE accounts SET balance = 900 WHERE id = 1;
T2:                                                COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  -- 读到 900（不一致）
```

---

**3. 幻读（Phantom Read）**：同一事务中，多次范围查询返回的行数不一致

```sql
-- 时间线
T1: BEGIN;
T1: SELECT COUNT(*) FROM orders WHERE user_id = 123;  -- 结果：10条
T2:                                                     BEGIN;
T2:                                                     INSERT INTO orders (user_id, ...) VALUES (123, ...);
T2:                                                     COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE user_id = 123;  -- 结果：11条（幻读）
```

---

### MVCC机制

#### 什么是MVCC？

**Multi-Version Concurrency Control（多版本并发控制）**：
- InnoDB通过**保存数据的多个版本**，实现无锁读（快照读）
- 只在 RR 和 RC 隔离级别下生效（RU 可脏读，Serializable 偏锁）

**本质**：

1. 每行有一条 **undo 版本链**（通过 `DB_ROLL_PTR` 串起来）
2. 普通 `SELECT`（快照读）带着一份 **Read View**，从**当前版本开始**沿链往回找
3. 找到**第一个对本事务可见**的版本就返回——可见性由 Read View 规则判定，不是「永远取事务开始前的最后一版」

```text
读某一行时：
  当前版本 → 不可见？沿 undo 往下 → 再判 → … → 找到第一个可见版本
```

| 错误说法 | 更准确的说法 |
|----------|--------------|
| 只能读「事务开始以前」的历史 | **取决于隔离级别**：RR 大致是第一次快照读时的「当时已提交世界」；RC 是**每次 SELECT** 重新定快照 |
| 读「最后一个」历史版本 | 读的是版本链上 **第一个满足可见性** 的版本（可能是当前行，也可能是更早的 undo） |
| 以此解决脏读和不可重复读 | **脏读**：RC/RR 的快照读都不读别人未提交的版本 ✅；**不可重复读**：主要靠 **RR 复用同一 Read View** ✅，**RC 每次新建 Read View，仍可能不可重复读** ❌ |

**和三种异常的关系**：

| 问题 | MVCC（快照读）能否解决 | 说明 |
|------|------------------------|------|
| 脏读 | ✅ RC / RR 都能 | 未提交事务的 `trx_id` 在 `m_ids` 里 → 不可见，继续沿 undo 找旧版 |
| 不可重复读 | ✅ 主要靠 RR；❌ RC 不能 | RR：同一事务共用一个 Read View；RC：每句 SELECT 新 Read View，能看到别人中途提交的修改 |
| 幻读 | ⚠️ 快照读大部分能挡；当前读要靠锁 | 普通 SELECT 在 RR 下对新插入且当时未「可见」的行会过滤；`SELECT FOR UPDATE` / `UPDATE` 等**当前读**看最新数据，靠 Next-Key Lock |

**补充两点**（面试常踩坑）：

1. **自己的未提交修改对自己可见**（`trx_id == creator_trx_id`），不是「只能看别人提交前的历史」。
2. InnoDB 的 RR 下，Read View 一般在事务内**第一次一致性读（第一次普通 SELECT）** 时创建，不是严格等于 `BEGIN` 那一刻（`BEGIN` 之后若很晚才第一次 SELECT，中间别人已提交的修改，第一次读就能看到）。

---

#### 实现原理

**每行记录隐藏列**：
- `DB_TRX_ID`：最后修改该行的事务ID
- `DB_ROLL_PTR`：指向undo log中的历史版本
- `DB_ROW_ID`：隐藏主键（无主键时）

**undo log版本链**：
```
当前版本: (id=1, name='Alice', balance=1000, trx_id=100)
           ↓ DB_ROLL_PTR
历史版本: (id=1, name='Alice', balance=900, trx_id=99)
           ↓
历史版本: (id=1, name='Alice', balance=800, trx_id=98)
```

---

#### Read View（读视图）

**Read View记录**：
- `m_ids`：当前活跃事务ID列表
- `min_trx_id`：最小活跃事务ID
- `max_trx_id`：下一个要分配的事务ID
- `creator_trx_id`：当前事务ID

**可见性判断规则**：
```python
def is_visible(trx_id, read_view):
    # 1. 如果记录的事务ID等于当前事务ID，可见（自己的修改）
    if trx_id == read_view.creator_trx_id:
        return True

    # 2. 如果记录的事务ID小于最小活跃事务ID，可见（已提交）
    if trx_id < read_view.min_trx_id:
        return True

    # 3. 如果记录的事务ID大于等于下一个要分配的事务ID，不可见（未来事务）
    if trx_id >= read_view.max_trx_id:
        return False

    # 4. 如果记录的事务ID在活跃事务列表中，不可见（未提交）
    if trx_id in read_view.m_ids:
        return False

    # 5. 否则可见（已提交但不在活跃列表）
    return True
```

---

#### RC vs RR的区别（底层原理）⭐⭐⭐

**READ COMMITTED（RC）**：
- 每次读取时创建新的Read View
- 可以读到其他事务已提交的数据（不可重复读）

**REPEATABLE READ（RR）**：
- 事务内**第一次一致性读（第一次普通 SELECT）** 时创建 Read View，之后整个事务复用同一个
- 读到的数据始终一致（可重复读）
- 注意：`BEGIN` 后若很久才第一次 SELECT，中间别人已提交的修改在第一次读时可见

```sql
-- RR隔离级别
T1: BEGIN;
T1: SELECT balance FROM accounts WHERE id = 1;  -- 创建Read View，读到1000
T2:                                                BEGIN;
T2:                                                UPDATE accounts SET balance = 900 WHERE id = 1;
T2:                                                COMMIT;  -- T2提交
T1: SELECT balance FROM accounts WHERE id = 1;  -- 仍然读到1000（使用旧的Read View）
```

---

#### 完整的MVCC查询过程（深度剖析）⭐⭐⭐

**场景**：在RR隔离级别下，事务T1查询 `id=1` 的记录

**Step 1：创建/复用Read View**
```
T1 第一次执行普通 SELECT 时创建 Read View（不是 BEGIN 那一刻）：
- m_ids: [100, 102, 105]  （活跃事务列表）
- min_trx_id: 100
- max_trx_id: 106  （下一个要分配的事务ID）
- creator_trx_id: 102  （T1的事务ID）
```

**Step 2：读取当前版本的记录**
```
当前版本: (id=1, balance=1000, trx_id=105, roll_ptr → undo_log)
```

**Step 3：判断可见性**
```python
trx_id = 105
read_view = {m_ids: [100,102,105], min_trx_id: 100, max_trx_id: 106, creator_trx_id: 102}

# 判断逻辑
trx_id == creator_trx_id?  # 105 == 102? No
trx_id < min_trx_id?       # 105 < 100? No
trx_id >= max_trx_id?      # 105 >= 106? No
trx_id in m_ids?           # 105 in [100,102,105]? Yes → 不可见！
```

**Step 4：沿undo log版本链查找可见版本**
```
当前版本: (balance=1000, trx_id=105, roll_ptr) → 不可见
    ↓ 沿roll_ptr找历史版本
历史版本1: (balance=900, trx_id=99, roll_ptr) → 判断可见性
    trx_id=99 < min_trx_id=100 → 可见！
    返回 balance=900
```

**完整查询流程图**：
```
1. 创建/复用Read View
   ↓
2. 定位到记录的当前版本
   ↓
3. 判断当前版本是否可见
   ├→ 可见：返回
   └→ 不可见：继续
       ↓
4. 沿roll_ptr找undo log中的历史版本
   ↓
5. 判断历史版本是否可见
   ├→ 可见：返回
   └→ 不可见：继续找下一个历史版本
       ↓
6. 直到找到可见版本 or 到达版本链末尾（返回NULL）
```

---

#### 为什么MVCC能防止不可重复读？

**不可重复读场景**：
```sql
T1: BEGIN;
T1: SELECT * FROM accounts WHERE id = 1;  -- balance=1000
T2:                                        BEGIN;
T2:                                        UPDATE accounts SET balance = 900 WHERE id = 1;
T2:                                        COMMIT;
T1: SELECT * FROM accounts WHERE id = 1;  -- 期望还是1000
```

**MVCC的解决方案**：
```
T1第一次查询：
- 创建Read View: m_ids=[102,103], min=102, max=104, creator=102
- 读取记录：(balance=1000, trx_id=101)
- 判断：trx_id=101 < min_trx_id=102 → 可见
- 返回：balance=1000

T2更新并提交：
- 更新记录：(balance=900, trx_id=103)
- 生成undo log：(balance=1000, trx_id=101) ← roll_ptr指向这里

T1第二次查询（使用同一个Read View）：
- 读取记录：(balance=900, trx_id=103)
- 判断：trx_id=103 in m_ids → 不可见！
- 沿roll_ptr找历史版本：(balance=1000, trx_id=101)
- 判断：trx_id=101 < min_trx_id=102 → 可见
- 返回：balance=1000（和第一次一致！）
```

**关键点**：
- T1的Read View在第一次一致性读时创建，之后**不会更新**
- T2的修改对T1不可见（trx_id=103在T1的活跃列表中）
- T1读取的是undo log中的历史版本

---

#### RR如何防止部分幻读？（结合Next-Key Lock）

**幻读场景**：
```sql
T1: BEGIN;
T1: SELECT COUNT(*) FROM users WHERE age BETWEEN 20 AND 30;  -- 结果：10
T2:                                                           BEGIN;
T2:                                                           INSERT INTO users (age) VALUES (25);
T2:                                                           COMMIT;
T1: SELECT COUNT(*) FROM users WHERE age BETWEEN 20 AND 30;  -- 期望还是10
```

**先厘清概念**：
- 幻读的核心是**同一事务内两次范围查询的结果集不同**（多出行或少出行），主要由 INSERT/DELETE 引起；UPDATE 引起的同行值变化属于不可重复读范畴。
- 新插入的行也有 `trx_id`，可见性判断不依赖它有历史版本——所以 MVCC 对 INSERT 同样有效。

**InnoDB 的解决方案：快照读靠 MVCC，当前读靠 Next-Key Lock**

1. **快照读（Snapshot Read）**：
   ```sql
   -- 普通SELECT：使用MVCC，读历史版本
   SELECT COUNT(*) FROM users WHERE age BETWEEN 20 AND 30;
   -- T2的INSERT对T1不可见（trx_id > T1的Read View范围）
   ```

2. **当前读（Current Read）**：
   ```sql
   -- 加锁SELECT：使用Next-Key Lock，锁定范围
   SELECT COUNT(*) FROM users WHERE age BETWEEN 20 AND 30 FOR UPDATE;
   -- 锁定 (19, 20], [20, 30], (30, 31] 范围
   -- T2的INSERT会被阻塞
   ```

**为什么普通SELECT能防止部分幻读？**
```
T1的Read View: m_ids=[100,102], min=100, max=103, creator=100

T2 INSERT新记录：
- 新记录: (age=25, trx_id=102)
- T1查询时判断可见性：
  trx_id=102 in m_ids → 不可见！
- 虽然记录存在，但T1看不见 → 没有幻读

关键：T2的trx_id在T1的活跃列表中
```

**MVCC 防不住的场景：当前读**
```sql
T1: BEGIN;
T1: SELECT COUNT(*) FROM users WHERE age BETWEEN 20 AND 30;  -- 快照读：10
T2: INSERT INTO users (age) VALUES (25); COMMIT;
T1: UPDATE users SET level=1 WHERE age BETWEEN 20 AND 30;    -- 当前读：影响到 11 行！
T1: SELECT COUNT(*) FROM users WHERE age BETWEEN 20 AND 30;  -- 11（自己改的可见）
```
当前读（UPDATE/FOR UPDATE）读的是最新数据而非快照，T2 新插入的行会被 UPDATE 命中，随后的快照读又能看到自己改过的这行——这就是 RR 下仍可能出现的幻读。要彻底防止，范围当前读必须配合 Next-Key Lock 阻止间隙插入。

**总结**：
- **快照读（普通 SELECT）**：靠 MVCC 的 Read View 过滤不可见版本，RR 下可防幻读
- **当前读（FOR UPDATE/UPDATE/DELETE）**：靠 Next-Key Lock 锁定记录+间隙，阻止其他事务插入
- **RR 隔离级别** = 快照读走 MVCC + 当前读走 Next-Key Lock（两者配合）

---

### 锁类型

#### 当前读 vs 快照读

**快照读（Snapshot Read）**：
- 普通SELECT语句
- 读取MVCC版本链中的历史数据
- **不加锁**

```sql
SELECT * FROM users WHERE id = 1;  -- 快照读
```

**当前读（Current Read）**：
- `SELECT ... FOR UPDATE`（排他锁）
- `SELECT ... FOR SHARE`（共享锁；8.0 推荐写法，旧写法 `LOCK IN SHARE MODE` 仍兼容）
- `UPDATE`, `DELETE`, `INSERT`
- 读取最新版本的数据
- **加锁**

```sql
SELECT * FROM users WHERE id = 1 FOR UPDATE;  -- 当前读，加排他锁
SELECT * FROM users WHERE id = 1 FOR SHARE;   -- 当前读，加共享锁
-- 不等待 / 跳过已锁行（8.0+，适合秒杀占库存、队列抢任务）：
SELECT * FROM jobs WHERE status='ready' LIMIT 1 FOR UPDATE NOWAIT;
SELECT * FROM jobs WHERE status='ready' LIMIT 1 FOR UPDATE SKIP LOCKED;
```

---

#### 锁粒度

**1. 表锁**：
```sql
LOCK TABLES users WRITE;  -- 锁整张表（很少用）
```

**2. 行锁（Row Lock）**：
- InnoDB默认使用行锁
- 只锁定需要的行，并发性能高

**3. 间隙锁（Gap Lock）**：
- RR隔离级别下，锁定索引记录之间的间隙
- 防止幻读

**4. Next-Key Lock**：
- 行锁 + 间隙锁
- 锁定记录本身 + 记录前的间隙

---

#### 意向锁（Intention Lock）⭐⭐

**问题**：事务 A 锁了某一行（行锁），事务 B 想锁整张表（如 `ALTER TABLE`）——B 怎么快速知道表里有没有行被锁？逐行检查太慢。

**解法**：事务加行锁**之前**，先在表上加一把**意向锁**：
- 加行级 X 锁前 → 先加表级 **IX（意向排他锁）**
- 加行级 S 锁前 → 先加表级 **IS（意向共享锁）**

意向锁是**表级**的，只和「表级锁」互斥，**行锁之间不会因为意向锁互相阻塞**：

|  | IS | IX | S（表锁） | X（表锁） |
|--|----|----|----|----|
| **IS** | ✅ | ✅ | ✅ | ❌ |
| **IX** | ✅ | ✅ | ❌ | ❌ |
| **S** | ✅ | ❌ | ✅ | ❌ |
| **X** | ❌ | ❌ | ❌ | ❌ |

关键结论：**IX 与 IX 兼容**——两个事务可以同时持有同一张表的 IX（各自锁不同的行），这正是行锁高并发的基础；而 `ALTER TABLE`（需要表级 X）会被任何 IX 挡住，这就是「有未提交事务时 DDL 会等待」的原因。

#### 插入意向锁（Insert Intention Lock）⭐

INSERT 前要在间隙上设置的一种特殊 Gap Lock：**多个事务在同一间隙插入不同的值，互不阻塞**（只要没有别的 Gap Lock 占着该间隙）。

```sql
-- id 已有 5, 10。T1 和 T2 并发插入，互不阻塞：
T1: INSERT INTO t VALUES (7);   -- 在 (5,10) 间隙上加插入意向锁
T2: INSERT INTO t VALUES (8);   -- 同间隙不同值，不冲突
```

但如果已有事务对 `(5,10)` 持 Gap Lock（如范围 FOR UPDATE），INSERT 就必须等待——这是「范围更新阻塞插入」的底层机制。

#### AUTO-INC 锁（自增锁）⭐

自增主键的值从哪来？三种模式（`innodb_autoinc_lock_mode`）：

| 模式 | 名称 | 行为 |
|------|------|------|
| 0 | traditional | 语句级表锁：INSERT 执行期间持有 AUTO-INC 表锁，并发差 |
| 1（8.0 前默认） | consecutive | 简单 INSERT 用轻量互斥量（分配即释放）；`INSERT ... SELECT` 等批量语句仍用表锁 |
| 2（8.0 默认） | interleaved | 全部用轻量互斥量，并发最好；但同一语句内多行可能拿到不连续的自增值 |

注意：模式 2 下自增值**不保证连续**（事务回滚、批量插入交错都会产生空洞），业务不要把自增 ID 当「连续序号」用。

#### 加锁规则速查（RR 隔离级别）

| 场景 | 加锁 |
|------|------|
| 唯一索引等值，**命中** | Record Lock |
| 唯一索引等值，**未命中** | Gap Lock（该值所在间隙） |
| 非唯一索引等值 | Next-Key Lock + 后一个间隙的 Gap Lock |
| 范围查询（`>`、`BETWEEN`） | Next-Key Lock 覆盖扫描范围 |
| 无索引列的 WHERE | **锁全表所有记录+间隙**（相当于锁表，必须避免） |
| INSERT | 行锁 + 插入意向锁（遇唯一键冲突时加 S 锁） |
| RC 隔离级别 | 一律无 Gap Lock（只有 Record Lock） |

> 面试追问「为什么没索引的 UPDATE 会锁全表」：行锁依附于索引——InnoDB 通过索引定位记录才能给记录加锁；没有可用索引时只能全表扫描，扫到的每一行都要加锁。

#### MDL（Metadata Lock）⭐⭐

行锁管**数据**，MDL 管**元数据**（表结构）。`SELECT/DML` 会加 **MDL 读锁**，`ALTER/DROP` 需要 **MDL 写锁**——二者互斥。

典型线上事故：一个慢查询（或忘记 COMMIT 的事务）一直持有表的 MDL 读锁 → `ALTER TABLE` 排队 → 后续所有新查询也在等 MDL → **整表不可访问**。排查：`performance_schema.metadata_locks` / `sys.schema_table_lock_waits`；治理：禁长事务、DDL 用 Online/gh-ost、高峰避开。

#### 锁观测（8.0）

```sql
-- 谁锁了谁（替代信息不全的 INFORMATION_SCHEMA.INNODB_LOCKS）
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
```

---

#### 锁示例

```sql
-- 表数据：id = 1, 5, 10, 15, 20
-- 索引：PRIMARY KEY(id)

-- T1执行
BEGIN;
SELECT * FROM users WHERE id = 10 FOR UPDATE;
```

**RR 隔离级别下，唯一索引等值命中已存在记录**：
- 只加 **Record Lock**（`locks rec but not gap`），锁定 `id=10` 这一行
- **不加间隙锁**——这是唯一索引等值命中的特殊优化

**防止的操作**（其他事务）：
```sql
-- ❌ 阻塞：更新 id=10
UPDATE users SET name = 'xxx' WHERE id = 10;

-- ✅ 不阻塞：插入 id=7（没有间隙锁）
INSERT INTO users (id, name) VALUES (7, 'Bob');
```

**什么时候才会加 Next-Key Lock / Gap Lock？**

| 场景 | 加锁 |
|------|------|
| 唯一索引等值，**命中** | 仅 Record Lock |
| 唯一索引等值，**未命中** | Gap Lock（锁定该值所在的间隙） |
| 非唯一索引等值 | Next-Key Lock + 后一个间隙的 Gap Lock |
| 范围查询（`>`、`BETWEEN`） | Next-Key Lock 覆盖整个扫描范围 |
| RC 隔离级别 | 一律无 Gap Lock（只有 Record Lock） |

```sql
-- 范围查询才会锁间隙：
SELECT * FROM users WHERE id BETWEEN 8 AND 12 FOR UPDATE;
-- 锁定 (5,10] 和 (10,15] 的 Next-Key Lock，插入 id=7/11 都会阻塞
```

---

### 死锁定位

#### 死锁示例

```sql
-- T1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 锁住id=1

-- T2
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;  -- 锁住id=2

-- T1继续
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 等待T2释放id=2

-- T2继续
UPDATE accounts SET balance = balance + 100 WHERE id = 1;  -- 等待T1释放id=1

-- 💥 死锁！
```

---

#### 查看死锁日志

```sql
SHOW ENGINE INNODB STATUS\G

-- 输出中查找 LATEST DETECTED DEADLOCK 部分
```

**死锁日志示例**：
```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-01-15 10:30:00
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 5 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 10, OS thread handle 0x7f8c4c0a9700, query id 1000 localhost root updating
UPDATE accounts SET balance = balance + 100 WHERE id = 2

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 100 page no 3 n bits 72 index `PRIMARY` of table `db`.`accounts` trx id 12345 lock_mode X locks rec but not gap waiting

*** (2) TRANSACTION:
TRANSACTION 12346, ACTIVE 3 sec starting index read
mysql tables in use 1, locked 1
2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 11, OS thread handle 0x7f8c4c0a8700, query id 1001 localhost root updating
UPDATE accounts SET balance = balance + 100 WHERE id = 1

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 100 page no 3 n bits 72 index `PRIMARY` of table `db`.`accounts` trx id 12346 lock_mode X locks rec but not gap

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 100 page no 3 n bits 72 index `PRIMARY` of table `db`.`accounts` trx id 12346 lock_mode X locks rec but not gap waiting

*** WE ROLL BACK TRANSACTION (1)
```

---

#### 避免死锁

**1. 固定加锁顺序**：
```sql
-- ❌ 不同顺序可能死锁
T1: UPDATE ... WHERE id = 1; UPDATE ... WHERE id = 2;
T2: UPDATE ... WHERE id = 2; UPDATE ... WHERE id = 1;

-- ✅ 固定顺序
T1: UPDATE ... WHERE id IN (1, 2) ORDER BY id;  -- 按id升序加锁
T2: UPDATE ... WHERE id IN (1, 2) ORDER BY id;
```

**2. 减少事务持有锁的时间**：
```sql
-- ❌ 慢：事务中执行复杂计算
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- 执行复杂业务逻辑（耗时）
UPDATE accounts SET balance = ... WHERE id = 1;
COMMIT;

-- ✅ 快：先计算，再开事务
-- 先查询数据（不加锁）
SELECT * FROM accounts WHERE id = 1;
-- 执行业务逻辑
-- 开启事务，快速更新
BEGIN;
UPDATE accounts SET balance = ... WHERE id = 1;
COMMIT;
```

**3. 使用乐观锁**：
```sql
-- 使用版本号
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE id = 1 AND version = @old_version;

-- 检查影响行数
IF ROW_COUNT() = 0 THEN
  -- 版本冲突，重试
END IF;
```

---

## 复制与一致性

### 主从复制

#### 复制原理

```
主库(Master)
    ↓ binlog
从库(Slave)
    ↓ IO Thread（拉取binlog）
    ↓ Relay Log
    ↓ SQL Thread（重放日志）
从库数据
```

**复制步骤**（异步复制的默认路径，也是半同步的「搬运」部分）：
1. 主库写入 binlog
2. 从库 IO 线程拉取 binlog 到 Relay Log
3. 从库 SQL 线程读取 Relay Log 并重放（5.6 前常是单线程；后可用并行复制）

#### 异步 vs 半同步：是两种「提交等待策略」，不是两套并行特性

| 模式 | 主库 COMMIT 何时对客户端返回成功 | 数据丢失风险 |
|------|----------------------------------|--------------|
| **异步复制**（默认） | 写完自己的 binlog（以及引擎提交）就返回，**不等**从库 | 主库宕机时，可能有已提交事务从未到过任何从库 |
| **半同步复制**（插件） | 至少等 **一个**（可配置）从库 **ACK：已收到 binlog 并写入 relay log** 才返回 | 降低「主挂、从没收到」的丢数窗口；**不保证**从库已重放完 |

```text
【异步】
Master: 写 binlog → 立刻返回 OK
Slave:  稍后自己拉 binlog → relay → 重放   （与客户端成功解耦）

【半同步】
Master: 写 binlog → 发给 Slave → 等 ACK（收到并写入 relay）→ 再返回 OK
Slave:  SQL 线程仍可能稍后才重放   （半同步保证的是「日志已到从库」，不是「从库已执行完」）
```

**关系怎么理解？**

| 说法 | 对不对 |
|------|--------|
| 异步和半同步是「同时打开的两个独立特性」 | ❌ 对同一套主从，提交路径选的是 **其中一种等待策略** |
| 半同步 = 在「异步那条 IO/SQL 流水线」上，多加「主库等 ACK」 | ✅ 底层仍是 binlog → IO → relay → SQL |
| 半同步超时后可能退回异步 | ✅ `rpl_semi_sync_master_timeout` 超时未收到 ACK，会降级为异步并继续提交 |

#### binlog 格式

| 格式 | 记什么 | 特点 |
|------|--------|------|
| **STATEMENT** | SQL 文本 | 体积小；`NOW()`、`UUID()` 等可能导致主从不一致 |
| **ROW** | 行前后镜像 | 最安全、可复现；DTS / 增量订阅常用；体积更大 |
| **MIXED** | 能 STATEMENT 则 STATEMENT，否则 ROW | 折中 |

生产与订阅场景通常优先 **ROW**。

#### 哪些变更可能不走 binlog / 导致主从不一致？

分两类：**主库根本没记进 binlog**，以及 **记了但两边执行结果不同**。

**一、主库不记 binlog（从库永远收不到）**

| 情况 | 说明 |
|------|------|
| 主库未开 binlog | `log_bin` 关闭则无复制基础 |
| 会话 `SET sql_log_bin = 0` | 运维/刷数时常用；**只改主库**，从库无对应事件 |
| 主库过滤：`binlog_do_db` / `binlog_ignore_db` | 指定库的变更可能不写 binlog（规则易踩坑，生产慎用） |
| 只在从库上直接写 | 绕过复制；主从必然分叉（除非故意做只读从库却误写） |
| 直接改数据文件 / 拷贝 `.ibd` | 不经 SQL 层，无 binlog；极易损坏或不一致 |
| 临时表相关 | `TEMPORARY TABLE` 的生命周期绑定会话；跨会话/跨机器行为与普通表不同，STATEMENT 下更易踩坑 |

```sql
-- 危险示例：主库刷数关掉 binlog
SET sql_log_bin = 0;
UPDATE users SET score = score + 100;  -- 从库永远没有这条
SET sql_log_bin = 1;
```

**二、走了 binlog，仍可能主从不一致**

| 情况 | 说明 |
|------|------|
| **STATEMENT** + 非确定性 | `NOW()`、`UUID()`、`RAND()`、`USER()`、无序 `LIMIT` 无 `ORDER BY`：主从算出的行不同 |
| 从库过滤：`replicate_do_*` / `replicate_ignore_*` | 主库有日志，从库故意不重放 → 「主有从无」 |
| 表结构/引擎不一致 | 从库缺列、缺索引、引擎不同 → 重放失败或结果不同 |
| 从库曾出错后 `sql_slave_skip_counter` / 跳 GTID | 人为跳过事件 → 永久少一段变更 |
| 触发器 / 存储过程 | ROW 一般按行镜像较稳；STATEMENT 下若主从定义不一致会发散 |
| 大事务中途主挂、半同步未 ACK | 更偏「丢已提交」或「主从位点错乱」，需结合备份/GTID 处理 |

```text
不走 binlog          → 从库「不知道有这事」
走了但 STATEMENT 坑  → 从库「按另一套结果执行」
走了但从库过滤/跳过  → 从库「故意或人为没执行」
```

**实践建议**：生产开 binlog + 优先 **ROW**；禁止业务路径关 `sql_log_bin`；从库设 `read_only`/`super_read_only`；避免主从过滤库表；定期用校验工具（如 pt-table-checksum）查不一致。

#### 主从延迟

**常见原因**：大事务、从库重放跟不上（历史单线程 SQL 线程）、从库硬件弱、从库上长查询占锁、网络抖动。

**常见手段**：
- 并行复制：`replica_parallel_workers`（8.0.26+；旧名 `slave_parallel_workers`，配合 `replica_parallel_type` / `binlog_transaction_dependency_tracking` 等）
- 拆小事务、避免超长事务
- 读写分离时用「写后读主 / 等位点」处理一致性（见下节）

**监控**：
```sql
SHOW REPLICA STATUS\G   -- 8.0.22+；旧写法 SHOW SLAVE STATUS 仍兼容但已弃用
-- Seconds_Behind_Source: 延迟秒数（粗指标；GTID 拓扑还可看 Retrieved/Executed GTID）
```

---

#### GTID 复制⭐⭐

**位点复制的痛点**：`CHANGE MASTER TO MASTER_LOG_FILE='binlog.000123', MASTER_LOG_POS=4567`——位点是**物理位置**，主从切换、级联复制、日志清理后位点全乱，运维极易出错。

**GTID（Global Transaction ID）**：给每个事务分配全局唯一 ID，格式 `server_uuid:事务序号`（如 `3E11FA47-71CA-11E1-9E33-C80AA9429562:23`）。

```
位点复制：从库记录「我重放到 binlog.000123 的 4567 字节」
GTID 复制：从库记录「我已执行的事务集合 gtid_executed = uuid:1-100」
          主库自动比对差集，把从库缺的事务发过去
```

**优势**：
- 主从切换时无需算位点：`CHANGE MASTER TO MASTER_AUTO_POSITION=1` 自动对齐
- 幂等：已执行的 GTID 自动跳过，不会重复执行
- 拓扑灵活：级联、环形、一主多从随意组合

**两个关键集合**：
- `gtid_executed`：本实例已执行的事务集合
- `gtid_purged`：已被清理（binlog 删除）的事务集合——**恢复/搭从库时的坑**：导入的备份若缺 `gtid_purged`，从库会重复执行已清理的事务

**跳事务的正确姿势**（从库某事务报错时）：不能简单 skip，要注入空事务占位：
```sql
STOP REPLICA;
SET GTID_NEXT = 'uuid:105';  -- 报错那个事务的 GTID
BEGIN; COMMIT;               -- 注入空事务
SET GTID_NEXT = 'AUTOMATIC';
START REPLICA;
```

#### MGR（MySQL Group Replication）⭐⭐

**定位**：MySQL 官方的**组复制**（5.7.17+ 引入，8.0 成熟），基于 Paxos 变体（XCom）实现的多节点共识复制，是 InnoDB Cluster 的核心。

**与主从复制的本质区别**：

| | 异步/半同步主从 | MGR |
|--|----------------|-----|
| 一致性 | 最终一致（可能丢数据） | 多数派确认后才提交（类强一致） |
| 角色 | 主写从读，人工/中间件切换 | 单主模式自动选主 failover |
| 冲突 | 无检测 | 写冲突检测（认证阶段，后提交者回滚） |
| 架构 | 一主多从 | 组内对等（单主/多主模式） |

**提交流程（单主模式）**：
```
事务本地执行 → COMMIT 时把写集广播给组内成员
→ 各成员「认证」（certification）：检查写集是否与并发事务冲突
→ 多数派（N/2+1）接受 → 提交；否则回滚
```

**实践要点**：
- 组大小建议 3 或 5（容忍 (N-1)/2 个节点故障）
- 要求：表必须有主键、InnoDB、GTID 开启、binlog ROW 格式
- 多主模式理论可行，但写冲突回滚成本高，**生产几乎都用单主模式**
- 网络分区时少数派自动只读（防脑裂），这是相对「双主+VIP」方案的核心优势

**选型对比**：简单读写分离 → 异步/半同步主从；要求不丢数据 + 自动 failover → MGR/InnoDB Cluster；超大规模分片 → Vitess/中间件分库分表。

---

### 读写一致性

#### 问题场景

```sql
-- 用户A发布文章
INSERT INTO articles (title, content) VALUES ('标题', '内容');
-- 主库写入成功

-- 用户A立即刷新页面（读从库）
SELECT * FROM articles WHERE user_id = A;
-- ❌ 从库还没同步，查不到刚发布的文章
```

---

#### 解决方案

**方案1：强制读主库**（推荐：金融/支付场景）
```sql
-- 写入后，短时间内读主库
-- 配置：写入后30秒内读主库
if (Time.now() - last_write_time < 30s) {
  read_from_master();
} else {
  read_from_slave();
}
```

**方案2：等待主从同步**
```sql
-- 位点方式（5.7.2+）
SELECT MASTER_POS_WAIT('binlog_file', binlog_pos, timeout);
-- GTID 方式（推荐，8.0 常用）
SELECT WAIT_FOR_EXECUTED_GTID_SET('gtid_set', timeout);
-- 等待从库同步到指定位置/GTID 集合
```

**方案3：会话级别绑定**
```sql
-- 用户A在会话期间，所有读操作都读主库
session.bind_to_master = true;
```

**方案4：缓存 + 异步写**（推荐：游戏场景）
```sql
-- 写入主库 + 写入缓存
INSERT INTO articles (...) VALUES (...);
cache.set('article:123', data, ttl=60);

-- 读取时先查缓存
data = cache.get('article:123');
if (!data) {
  data = db.query(...);  -- 查从库
}
```

---

## Online DDL 与变更

线上改表不能「停服 COPY」一把梭。MySQL 8.0 的算法优先级大致是：

| 算法 | 典型操作 | 特点 |
|------|----------|------|
| **INSTANT** | 加列（末尾）、部分改默认值、改部分元数据 | 只改数据字典，秒级；有版本限制，不是所有 ALTER 都能 Instant |
| **INPLACE** | 加/删二级索引、部分改列 | 重建索引或原地改，通常允许 DML，但可能短暂拿 MDL |
| **COPY** | 大改列类型、改主键等 | 建临时表拷数据，锁表窗口大，尽量避免 |

```sql
-- 先看优化器会选哪种（8.0）
ALTER TABLE t ADD COLUMN c INT, ALGORITHM=INSTANT;
-- 不支持会报错，再评估 INPLACE / 外部工具
```

**大表变更工具**：
- **gh-ost**：通过 binlog + 幽灵表切流，对主库压力可控，GitHub 系常用
- **pt-online-schema-change**：触发器维护增量，成熟但触发器有额外写放大
- 原则：先评估是否 INSTANT/INPLACE → 不行再用工具；变更窗口盯 MDL 等待与复制延迟

---

## 备份与恢复

| 方式 | 类型 | 用途 |
|------|------|------|
| `mysqldump` / `mysqlpump` | 逻辑备份 | 小库、迁移、结构+数据导出；大库慢且一致性窗口难控 |
| **XtraBackup / mysqlbackup** | 物理热备 | 大库在线备份，拷贝数据文件 + redo；恢复快 |
| binlog | 增量/PITR | 物理全备 + binlog 重放到指定时间/GTID |

**PITR 闭环**：定期物理全备 → 保留全备之后的全部 binlog → 恢复时「还原全备 + `mysqlbinlog`/`START REPLICA UNTIL` 到故障前一刻」。GTID 环境下务必同步设置 `gtid_purged`，否则从库会重放已清理事务。

生产检查清单：备份可恢复性演练（不只「备份成功」）、binlog 保留天数 vs 全备周期、跨机房拷贝带宽、加密与权限。

---

## MySQL 8.0 特性速览

- **原子 DDL**：DDL 与数据字典事务化，中途崩溃不再留下「半成品表」
- **降序索引**：`INDEX (a DESC, b ASC)` 真正按降序存储，匹配 `ORDER BY a DESC, b ASC` 可避免 filesort
- **直方图**：`ANALYZE TABLE t UPDATE HISTOGRAM ON col`，给优化器更准的非索引列分布
- **EXPLAIN ANALYZE**：真正执行并输出实际耗时/行数，对比估算偏差（比 EXPLAIN 只看估计更有用）
- **窗口函数 / CTE / 角色权限 / 不可见索引**：复杂分析与灰度加索引（`INVISIBLE` 先观察优化器是否选用再删除）
- **复制术语**：`SHOW REPLICA STATUS`、`SOURCE_*` 参数名取代 slave/master（旧名仍兼容）

---

## 性能优化

### 慢查询优化

#### 治理闭环（推荐流程）

```
采集慢日志 / performance_schema
  → 聚合（mysqldumpslow / pt-query-digest）
  → EXPLAIN / EXPLAIN ANALYZE 定位
  → 改 SQL / 加索引 / 改模型
  → 回归对比（耗时、扫描行、锁等待）
  → 归档样例到知识库，防复发
```

#### 1. 开启慢查询日志

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;          -- 生产可先 1s，再按 P99 收紧
SET GLOBAL log_queries_not_using_indexes = ON;  -- 慎用，量大时采样
```

#### 2. 分析工具

```bash
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
# 更推荐（按指纹聚合、看 95/99 分位）：
pt-query-digest /var/log/mysql/slow.log
```

也可查 `performance_schema.events_statements_summary_by_digest` 看线上真实耗时 Top SQL（不一定先开慢日志）。

#### 3. 优化步骤

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 1;
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123 AND status = 1;  -- 8.0.18+
-- 关注：type、key、rows vs 实际 rows、Extra(Using filesort/temporary)
CREATE INDEX idx_user_status ON orders(user_id, status);
```

---

### 批量操作

#### 批量插入

```sql
-- ❌ 慢：逐条插入
FOR each row:
  INSERT INTO users (name, age) VALUES ('Alice', 30);
-- 1000条需要1000次网络往返

-- ✅ 快：批量插入
INSERT INTO users (name, age) VALUES
  ('Alice', 30),
  ('Bob', 25),
  ('Charlie', 35),
  ...  -- 一次插入1000条
-- 只需1次网络往返
```

**建议**：
- 每批次1000-5000条
- 过大可能导致锁等待时间过长

---

#### 批量更新

```sql
-- ❌ 慢：逐条更新
UPDATE users SET status = 1 WHERE id = 1;
UPDATE users SET status = 1 WHERE id = 2;
...

-- ✅ 快：批量更新
UPDATE users SET status = 1 WHERE id IN (1, 2, 3, ...);
-- 或
UPDATE users SET status = 1 WHERE id BETWEEN 1 AND 1000;
```

---

### 深分页优化

#### 问题

```sql
-- 深分页：LIMIT offset 很大时性能极差
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
-- MySQL 需要扫描 1000010 行，丢弃前 1000000 行，只返回 10 行
```

#### 优化方案

**方案1：游标分页（Keyset Pagination / Cursor-based）⭐推荐**

```sql
-- 记住上一页最后一条记录的 id
SELECT * FROM orders WHERE id > @last_id ORDER BY id LIMIT 10;
-- 利用主键索引直接定位，不需要扫描前面的记录
-- 性能恒定，与页码无关
```

**方案2：延迟关联（Deferred Join）**

```sql
-- 先用覆盖索引查出 id，再回表
SELECT o.* FROM orders o
INNER JOIN (
  SELECT id FROM orders ORDER BY id LIMIT 1000000, 10
) t ON o.id = t.id;
-- 子查询只扫描索引（覆盖索引），不回表
-- 外层只对 10 条记录回表
```

**方案3：业务限制**
- 限制用户最大可访问页码（如最多翻到第100页）
- 提供搜索/筛选代替深度翻页

---

## 数据建模

### 范式与反范式

#### 三大范式

**第一范式（1NF）**：字段不可再分
```sql
-- ❌ 违反1NF
CREATE TABLE users (
  id INT,
  name VARCHAR(50),
  address VARCHAR(200)  -- "北京市朝阳区xxx路xxx号"（可再分）
);

-- ✅ 符合1NF
CREATE TABLE users (
  id INT,
  name VARCHAR(50),
  province VARCHAR(20),
  city VARCHAR(20),
  district VARCHAR(20),
  street VARCHAR(100)
);
```

**第二范式（2NF）**：非主键字段完全依赖于主键
```sql
-- ❌ 违反2NF
CREATE TABLE order_items (
  order_id INT,
  product_id INT,
  product_name VARCHAR(50),  -- 只依赖product_id，不依赖order_id
  quantity INT,
  PRIMARY KEY (order_id, product_id)
);

-- ✅ 符合2NF：拆分表
CREATE TABLE order_items (
  order_id INT,
  product_id INT,
  quantity INT,
  PRIMARY KEY (order_id, product_id)
);

CREATE TABLE products (
  product_id INT PRIMARY KEY,
  product_name VARCHAR(50)
);
```

**第三范式（3NF）**：非主键字段不依赖于其他非主键字段
```sql
-- ❌ 违反3NF
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  user_id INT,
  user_name VARCHAR(50),  -- 依赖于user_id（非主键）
  total_amount DECIMAL(10,2)
);

-- ✅ 符合3NF：拆分表
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  user_id INT,
  total_amount DECIMAL(10,2)
);

CREATE TABLE users (
  user_id INT PRIMARY KEY,
  user_name VARCHAR(50)
);
```

---

#### 反范式设计

**适用场景**：
- 读多写少
- 查询性能要求高
- 允许数据冗余

**示例**：
```sql
-- 范式设计：需要JOIN
SELECT o.order_id, o.total_amount, u.user_name
FROM orders o
JOIN users u ON o.user_id = u.user_id;

-- 反范式设计：冗余user_name，无需JOIN
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  user_id INT,
  user_name VARCHAR(50),  -- 冗余字段
  total_amount DECIMAL(10,2)
);

SELECT order_id, total_amount, user_name FROM orders;
```

**注意**：需要保证冗余数据的一致性（通过触发器或应用层逻辑）。

---

### 分库分表

#### 垂直拆分

**垂直分库**：按业务拆分
```
原数据库：
  - users表
  - orders表
  - products表
  - payments表

拆分后：
  - user_db: users表
  - order_db: orders表
  - product_db: products表
  - payment_db: payments表
```

**垂直分表**：按字段拆分
```sql
-- 原表：字段很多
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(50),
  email VARCHAR(100),
  avatar BLOB,  -- 大字段
  profile TEXT  -- 大字段
);

-- 拆分：冷热数据分离
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(50),
  email VARCHAR(100)
);

CREATE TABLE user_profiles (
  user_id INT PRIMARY KEY,
  avatar BLOB,
  profile TEXT
);
```

---

#### 水平拆分

**按范围分片**：
```sql
-- 按用户ID范围
users_0: id 1-100万
users_1: id 100万-200万
users_2: id 200万-300万
```

**按哈希分片**：
```sql
-- 按用户ID取模
shard_id = user_id % 4

users_0: user_id % 4 = 0
users_1: user_id % 4 = 1
users_2: user_id % 4 = 2
users_3: user_id % 4 = 3
```

**优缺点对比**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| **范围分片** | 扩容方便、范围查询快 | 数据分布可能不均匀 |
| **哈希分片** | 数据分布均匀 | 扩容困难、范围查询慢 |

---


## 实战案例

### 金融交易表设计

#### 需求
- 支持高并发扣款
- 保证账户余额不能为负
- 记录所有交易流水
- 支持对账

---

#### 表结构设计

```sql
-- 账户表
CREATE TABLE accounts (
  account_id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  balance DECIMAL(20,2) NOT NULL DEFAULT 0,
  version INT NOT NULL DEFAULT 0,  -- 乐观锁版本号
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  INDEX idx_user(user_id)
) ENGINE=InnoDB;

-- 交易流水表
CREATE TABLE transactions (
  txn_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  account_id BIGINT NOT NULL,
  txn_type TINYINT NOT NULL,  -- 1:充值 2:扣款 3:退款
  amount DECIMAL(20,2) NOT NULL,
  balance_before DECIMAL(20,2) NOT NULL,  -- 交易前余额
  balance_after DECIMAL(20,2) NOT NULL,   -- 交易后余额
  request_id VARCHAR(64) NOT NULL UNIQUE, -- 幂等键
  status TINYINT NOT NULL DEFAULT 0,      -- 0:处理中 1:成功 2:失败
  created_at DATETIME NOT NULL,
  INDEX idx_account(account_id, created_at),
  INDEX idx_request(request_id)
) ENGINE=InnoDB;
```

---

#### 扣款逻辑（悲观锁）

```sql
BEGIN;

-- 1. 锁定账户（FOR UPDATE）
SELECT account_id, balance, version
FROM accounts
WHERE account_id = @account_id
FOR UPDATE;

-- 2. 检查余额
IF balance < @amount THEN
  ROLLBACK;
  RETURN 'insufficient_balance';
END IF;

-- 3. 扣款
UPDATE accounts
SET balance = balance - @amount,
    version = version + 1,
    updated_at = NOW()
WHERE account_id = @account_id;

-- 4. 记录流水
INSERT INTO transactions (
  account_id, txn_type, amount,
  balance_before, balance_after,
  request_id, status
) VALUES (
  @account_id, 2, @amount,
  @old_balance, @old_balance - @amount,
  @request_id, 1
);

COMMIT;
```

---

#### 扣款逻辑（乐观锁）

```sql
-- 1. 查询账户（不加锁）
SELECT account_id, balance, version
FROM accounts
WHERE account_id = @account_id;

-- 2. 检查余额
IF balance < @amount THEN
  RETURN 'insufficient_balance';
END IF;

-- 3. 尝试更新（使用version）
UPDATE accounts
SET balance = balance - @amount,
    version = version + 1,
    updated_at = NOW()
WHERE account_id = @account_id
  AND version = @old_version
  AND balance >= @amount;  -- 双重检查

-- 4. 检查更新结果
IF ROW_COUNT() = 0 THEN
  -- 版本冲突或余额不足，重试
  RETURN 'retry';
END IF;

-- 5. 记录流水
INSERT INTO transactions (...) VALUES (...);
```

---

#### 幂等处理

```sql
-- 使用 request_id 保证幂等
INSERT INTO transactions (
  account_id, txn_type, amount,
  request_id, status
) VALUES (
  @account_id, 2, @amount,
  @request_id, 0
)
ON DUPLICATE KEY UPDATE
  txn_id = LAST_INSERT_ID(txn_id);
-- 如果 request_id 已存在，返回已有记录的 txn_id

-- 查询是否已处理
SELECT status FROM transactions WHERE txn_id = LAST_INSERT_ID();
```

---

### 游戏排行榜设计

#### 需求
- 支持实时更新
- 查询TOP 100
- 查询某用户的排名

---

#### 方案1：MySQL（简单场景）

```sql
CREATE TABLE leaderboard (
  user_id BIGINT PRIMARY KEY,
  score INT NOT NULL,
  nickname VARCHAR(50) NOT NULL,
  updated_at DATETIME NOT NULL,
  INDEX idx_score(score DESC, user_id)  -- 降序索引
) ENGINE=InnoDB;

-- 查询TOP 100
SELECT user_id, nickname, score
FROM leaderboard
ORDER BY score DESC, user_id ASC
LIMIT 100;

-- 查询某用户排名（复杂）
SELECT COUNT(*) + 1 AS rank
FROM leaderboard
WHERE score > (SELECT score FROM leaderboard WHERE user_id = @user_id)
   OR (score = (SELECT score FROM leaderboard WHERE user_id = @user_id)
       AND user_id < @user_id);
```

**问题**：
- 查询排名需要全表扫描（数据量大时很慢）
- 高并发更新有锁竞争

---

#### 方案2：Redis ZSet（推荐）

```python
import redis

r = redis.Redis()

# 更新分数
r.zadd('leaderboard', {user_id: score})

# 查询TOP 100
top_100 = r.zrevrange('leaderboard', 0, 99, withscores=True)

# 查询某用户排名
rank = r.zrevrank('leaderboard', user_id)  # O(log N)
if rank is not None:
    rank = rank + 1  # 排名从1开始
```

**优势**：
- 读写速度快（内存操作）
- 排名查询O(log N)
- 天然支持分页

**持久化**：
- 定期同步到MySQL（如每小时）
- 用于对账和历史查询

---

**总结**

MySQL是金融面试的重中之重，重点掌握：
- **索引原理**：B+Tree、联合索引、覆盖索引
- **SQL执行机制**：连接、解析、优化、执行、预处理语句、EXPLAIN
- **事务与锁**：隔离级别、MVCC、死锁
- **性能优化**：EXPLAIN、慢查询优化
- **一致性**：主从复制、读写一致性
- **数据建模**：范式、分库分表

面试重点考察方向：
1. 原理理解（为什么这么设计）
2. 问题排查（慢查询、死锁如何定位）
3. 实战经验（金融场景的幂等、一致性保证）

---

## 面试题自查

> 统一编号 **Q1–Qn**。每题格式：要点 → 易错点 → 可选追问。细节回正文章节。

#### Q1: `CHAR` / `VARCHAR`、`COUNT(*)` / `COUNT(列)`、主键/唯一/普通索引怎么区分？

`CHAR` 定长、`VARCHAR` 变长按业务选；`COUNT(*)`/`COUNT(1)` 计行，`COUNT(列)` 忽略 NULL；主键决定 InnoDB 聚簇组织且非空唯一，唯一索引可多条、普通索引只加速。
**追问**：为何建议短而稳定的主键？→ 二级索引叶子存主键，主键过长会放大所有二级索引体积。

#### Q2: `WHERE` vs `HAVING`？内/左/右连接返回什么？

`WHERE` 聚合前过滤，`HAVING` 聚合后过滤；内连接两边匹配，左连接保左表、右补 NULL。
**追问**：能下推到 `WHERE` 的条件为何不要写 `HAVING`？→ 少聚合无效行。

#### Q3: 长事务、大事务、`SELECT *` 为什么都要警惕？

长事务拖住 Read View → purge 停、undo 膨胀、锁持有久、复制延迟；大事务放大锁/回滚/binlog/从库回放；`SELECT *` 增 IO/网络并易失去覆盖索引。
**追问**：如何发现长事务？→ `information_schema.innodb_trx` / `performance_schema`。

#### Q4: 为什么 InnoDB 选 B+Tree 而不是 B-Tree、Hash、红黑树？

叶子存数据+链表适合范围与顺序 IO；非叶只存 key 树更矮；Hash 无范围；红黑树太高。
**追问**：三层大约能存多少行？估算假设是什么？→ 见「B+Tree结构」。

#### Q5: InnoDB vs MyISAM 核心区别？现代为何几乎都用 InnoDB？

事务/行锁/MVCC/崩溃恢复/聚簇索引 vs 表锁无事务。现代需要并发与安全。
**追问**：还有什么场景会碰到 MyISAM？→ 极少，历史系统或临时特殊表。

#### Q6: redo / binlog / undo 各自作用？两阶段提交解决什么？「双 1」是什么？

redo 持久化与崩溃恢复；binlog 复制与 PITR；undo 回滚与 MVCC。2PC 保证 redo 与 binlog 一致。双 1=`innodb_flush_log_at_trx_commit=1` + `sync_binlog=1`。
**追问**：只改其中一个会怎样？→ 仍可能丢提交或主从缺口。

#### Q7: Buffer Pool LRU 为何不是标准 LRU？Change Buffer 与 redo 区别？doublewrite 解决什么？

midpoint insertion：新页进 old 区，防全表扫描污染热数据。Change Buffer 缓存**非唯一二级索引**的页修改以省随机读；redo 记物理页变更保崩溃恢复。doublewrite 防部分写页断裂。
**追问**：唯一二级索引为什么通常不进 Change Buffer？→ 要立即判断唯一性冲突。

#### Q8: 聚簇索引是什么？为何必须有主键？UUID 主键有何问题？

数据按主键聚簇；无显式主键会用隐藏 ROW_ID（全局分配瓶颈）。UUID 无序 → 页分裂与碎片。
**追问**：用 UUID v7 / 雪花 ID 呢？→ 时间有序可缓解，仍比 BIGINT 自增占空间。

#### Q9: 覆盖索引与回表？如何优化？

索引含所需列则不回表（`Using index`）。优化：联合覆盖、延迟关联、少 `SELECT *`。
**追问**：唯一索引等值是否一定不回表？→ 查非索引列仍要回表。

#### Q10: 最左前缀与「范围后列失效」？

`(a,b,c)` 必须从左用起；范围条件后的列顺序不确定，一般用不上索引排序/过滤。列顺序看**查询形态**，不是「区分度高一定放前」。
**追问**：8.0 Index Skip Scan 是什么？→ 缺最左列时可跳跃，通常仍不如完整前缀。

#### Q11: MVCC / Read View 规则？RC 与 RR 区别？

隐藏列+undo 版本链；可见性看 m_ids/min/max。RC 每句新 Read View；RR **第一次一致性读**创建并复用（≠ BEGIN 瞬间）。
**追问**：BEGIN 后过很久才第一次 SELECT，能看到中间已提交的修改吗？→ 能。

#### Q12: RR 如何解决幻读？MVCC 能完全解决吗？

快照读靠 MVCC 过滤新插入；当前读靠 Next-Key。先快照再当前读（UPDATE 命中新行）仍可能幻读。
**追问**：RC 为何几乎无间隙锁？→ 不承诺可重复读，间隙锁收益低、易死锁。

#### Q13: Next-Key Lock 组成与加锁范围？

Record + Gap，左开右闭。**唯一索引等值命中只加 Record Lock**；未命中加 Gap；非唯一/范围加 Next-Key。
**追问**：`FOR UPDATE NOWAIT` / `SKIP LOCKED` 何时用？→ 不等待失败 / 跳过已锁行抢任务。

#### Q14: 意向锁、插入意向锁、MDL 各解决什么？

IS/IX 让表锁快速判断是否有行锁；插入意向锁让同间隙不同值 INSERT 互不等待；MDL 保护元数据，慢查询可堵住 DDL 进而堵住全表。
**追问**：如何看锁等待？→ `performance_schema.data_locks` / `data_lock_waits`。

#### Q15: ICP、索引失效常见场景、EXPLAIN type 好坏顺序？

ICP 在引擎层用索引过滤减回表。失效：函数、隐式转换、`%前模糊`、OR 部分无索引等。type：const > eq_ref > ref > range > index > ALL。
**追问**：`EXPLAIN ANALYZE` 比 EXPLAIN 多什么？→ 真实执行耗时与行数。

#### Q16: 死锁如何排查与预防？

`SHOW ENGINE INNODB STATUS` 看 LATEST DETECTED DEADLOCK；固定加锁顺序、缩短事务、RC、乐观锁。
**追问**：InnoDB 死锁时回滚谁？→ 代价较小的一方。

#### Q17: 主从复制原理？GTID 相对位点复制的优势？读写不一致怎么解？

binlog → relay → 重放。GTID 全局事务号，自动定位、易切换。方案：读主、`WAIT_FOR_EXECUTED_GTID_SET`、会话粘主、缓存。
**追问**：从库报错能否 `sql_replica_skip_counter`？→ GTID 下应注入空事务占位。

#### Q18: Online DDL 算法怎么选？大表变更用什么工具？

优先 INSTANT → INPLACE → COPY；大表用 gh-ost / pt-osc，并盯 MDL 与复制延迟。
**追问**：加一列什么时候能 Instant？→ 末尾加列等受限场景，先 `ALGORITHM=INSTANT` 试探。

#### Q19: 备份与 PITR 怎么做？

物理热备（XtraBackup）+ binlog 重放到点；逻辑备份适合小库。演练可恢复性；注意 `gtid_purged`。
**追问**：只做 mysqldump 不做 binlog 行不行？→ 只能回到备份点，不能 PITR。

#### Q20: 深分页为何慢？怎么优化？

大 OFFSET 白扫。Keyset 分页、延迟关联、限翻页深度。
**追问**：延迟关联收益主要在哪？→ 减少深翻页的回表次数。

#### Q21: 分库分表策略？范围 vs 哈希？

垂直按业务/字段；水平范围易扩容易热点，哈希均匀难扩容。
**追问**：跨片查询与分布式事务怎么收敛？→ 尽量避免，细节见分库分表章与系统设计篇。

#### Q22: 乐观锁 vs 悲观锁？

悲观：`FOR UPDATE` 堵当前读/写；快照读不堵。乐观：version 条件更新。高冲突金融偏悲观，低冲突偏乐观。
**追问**：普通 SELECT 会被 FOR UPDATE 堵住吗？→ RR/RC 下快照读一般不会。

#### Q23: binlog 格式与生产推荐？

STATEMENT / ROW / MIXED；5.7.7+ 服务器默认 ROW。生产推荐 ROW。
**追问**：ROW 体积大如何缓解？→ 磁盘与压缩、避免无谓更新大列。

#### Q24: 金融扣款流程要考虑什么？

事务内扣款+流水、幂等键、`balance >=`、加锁顺序、对账字段。
**追问**：如何防重复扣款与超扣？→ 唯一 request_id + SQL 条件更新。

#### Q25: 一条 SQL 的执行流程？Prepared Statement 在哪一环？

连接→解析→预处理→优化→执行→引擎。PS 在解析/预处理：结构固定防注入，不自动等于计划缓存。
**追问**：优化器选错索引怎么办？→ 统计信息、直方图、`FORCE INDEX` 慎用、改写 SQL。

### 开放式设计题

**D1：设计一个日均10亿条写入的订单表存储方案，你会怎么做？**

**参考思路**：
- 分库分表策略：按用户ID哈希分库（16库×64表）、按时间范围分区（归档查询）
- 写入优化：批量Insert、异步写入（MQ削峰）、非资损场景可放宽持久化：innodb_flush_log_at_trx_commit=2 且 sync_binlog=1000（"双1"指这两个参数都为1，放宽需成对评估）
- 查询兼容：订单号内嵌分片信息（Snowflake编码用户ID后缀）、全局唯一索引方案
- 数据归档：冷热分离（3个月以上迁移到TiDB/ClickHouse）、在线DDL工具（gh-ost）
- 关键取舍：强一致性（同步复制+半同步）vs 性能，跨分片查询的代价

**D2：如果线上MySQL主从延迟持续增大到30秒以上，你的排查和解决思路是什么？**

**参考思路**：
- 确认延迟：SHOW REPLICA STATUS → Seconds_Behind_Source、GTID 差距（Retrieved vs Executed）
- IO线程瓶颈：网络带宽/Binlog传输速度 → 并行复制配置
- SQL线程瓶颈：大事务回放慢 → binlog_group_commit、replica_parallel_workers 调大、LOGICAL_CLOCK 模式
- 业务层应对：写后读走主库（设置Hint或中间件路由）、降级方案
- 根因治理：拆分大事务、避免DDL长时间锁表、升级为MGR/组复制
