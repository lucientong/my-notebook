# MySQL 深入

---

## 📑 目录

- [存储引擎](#存储引擎)
- [日志机制](#日志机制)
- [索引原理](#索引原理)
  - [B+Tree结构](#btree结构)
  - [索引类型](#索引类型)
  - [联合索引](#联合索引)
  - [覆盖索引与回表](#覆盖索引与回表)
  - [索引失效场景](#索引失效场景)
  - [索引下推](#索引下推)
- [执行计划](#执行计划)
  - [EXPLAIN详解](#explain详解)
  - [索引选择](#索引选择)
- [事务与锁](#事务与锁)
  - [事务隔离级别](#事务隔离级别)
  - [MVCC机制](#mvcc机制)
  - [锁类型](#锁类型)
  - [死锁定位](#死锁定位)
- [复制与一致性](#复制与一致性)
  - [主从复制](#主从复制)
  - [读写一致性](#读写一致性)
- [性能优化](#性能优化)
  - [慢查询优化](#慢查询优化)
  - [批量操作](#批量操作)
  - [深分页优化](#深分页优化)
- [数据建模](#数据建模)
  - [范式与反范式](#范式与反范式)
  - [分库分表](#分库分表)
- [常见面试题](#常见面试题)
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

**为什么 InnoDB 是默认引擎？**
- MySQL 5.5 之后默认使用 InnoDB
- 事务 + 行级锁 + 崩溃恢复是生产环境的基本需求
- MyISAM 的表锁在高并发下性能极差

**InnoDB 聚簇索引 vs MyISAM 非聚簇索引**：

```
InnoDB（聚簇索引）：
主键索引叶子节点 → 存储完整行数据
二级索引叶子节点 → 存储主键值（需要回表）

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

事务提交时：
1. 将修改写入 redo log buffer
2. fsync 到 redo log 文件（磁盘）← 事务提交成功的标志
3. 后续异步将脏页刷新到数据文件

崩溃恢复时：
- 扫描 redo log，将已提交但未写入数据文件的修改重新应用
```

**redo log 是循环写的**：

```
redo log 文件（默认 2 个，每个 48MB）
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
3. MIXED：默认 STATEMENT，遇到不确定函数自动切换为 ROW

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

**场景**：向已满的叶子节点插入key

**分裂步骤**：
```
【初始状态】叶子节点已满（假设最多3个key）
[10, 20, 30]  ← 已满

【插入 25】
Step 1: 发现节点已满，触发分裂
Step 2: 将节点分成两个
  左节点: [10, 20]
  右节点: [25, 30]

Step 3: 中间key（20）提升到父节点
  父节点: [..., 20, ...]
         /            \
  [10, 20]          [25, 30]
```

**完整分裂过程（含父节点分裂）**：
```
【初始】3阶B+树（每个节点最多2个key）
         [20]
        /    \
    [10,15]  [20,30]

【插入 25】
Step 1: 找到插入位置（右叶子节点）
    [20,30] → 插入25 → [20,25,30]（超过2个key，分裂）

Step 2: 叶子节点分裂
    左: [20]
    右: [25,30]
    提升中间key: 25

Step 3: 25插入父节点
    [20] → 插入25 → [20,25]（未满，完成）

【最终结果】
         [20, 25]
        /    |    \
    [10,15] [20] [25,30]
```

**父节点也满的情况（递归分裂）**：
```
【初始】父节点已满
           [20, 40]
          /   |    \
      [10] [20,30] [40,50]

【插入 45 到 [40,50]】
Step 1: 叶子节点分裂
    [40,50] → 插入45 → [40,45,50] → 分裂
    左: [40]
    右: [45,50]
    提升: 45

Step 2: 45插入父节点 [20,40]
    [20,40] → 插入45 → [20,40,45]（超过2个key，分裂）

Step 3: 父节点分裂
    左: [20]
    右: [45]
    提升: 40

Step 4: 创建新根节点
    [40]
   /    \
 [20]   [45]

【最终结果】树高度+1
              [40]
            /      \
         [20]      [45]
        /    \    /    \
    [10]  [20,30] [40] [45,50]
```

---

##### 2. 删除导致的合并/重分配

**场景**：删除key后，节点key数量< m/2

**策略1：从兄弟节点借key（重分配）**
```
【初始】
         [20]
        /    \
    [10,15]  [20,30]

【删除 30】
Step 1: 删除后，右节点只有1个key
    [20] → 不满足最少2个key的要求

Step 2: 尝试从左兄弟借（左兄弟有富余）
    左兄弟: [10,15] → 借出15
    右节点: [20] → 接收15 → [15,20]

Step 3: 更新父节点的分隔key
    [20] → [15]

【最终结果】
         [15]
        /    \
    [10]    [15,20]
```

**策略2：无法借key，合并节点**
```
【初始】
         [20]
        /    \
    [10]    [20,30]

【删除 30】
Step 1: 删除后，右节点只有1个key [20]
Step 2: 左兄弟也只有1个key [10]，无法借
Step 3: 合并两个节点
    [10] + [20] → [10,20]

Step 4: 删除父节点中的分隔key
    [20] → 删除

【最终结果】树高度-1
    [10,20]
```

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

**为什么MySQL选择B+树？**

1. **范围查询高效**：
   ```sql
   SELECT * FROM users WHERE age BETWEEN 20 AND 30;
   
   B树：需要中序遍历整棵树（随机I/O）
   B+树：找到 age=20 的叶子节点，沿链表扫描（顺序I/O）
   ```

2. **非叶子节点容量更大**：
   ```
   B树节点：[key1, data1, ptr1, key2, data2, ptr2]
   - data可能很大（几KB），节点容纳key少
   
   B+树节点：[key1, ptr1, key2, ptr2, key3, ptr3]
   - 只有key和指针，节点容纳key多
   - 树高度更低，I/O更少
   
   举例（16KB页）：
   B树：假设data 1KB，一个节点最多16个key → 4层树存 16^4 = 65万条
   B+树：假设key+ptr 14B，一个节点最多1170个key → 3层树存 2000万条
   ```

3. **磁盘预读优化**：
   ```
   磁盘读取单位：页（16KB）
   B+树叶子节点：相邻key的data也相邻 → 一次I/O读多条记录
   B树：data分散在各层 → 需要多次I/O
   ```

4. **查询性能稳定**：
   ```
   B树：查找路径不固定（可能在非叶子节点命中）
   - 最好：O(1)（根节点命中）
   - 最坏：O(logn)（叶子节点）
   
   B+树：查找路径固定（一定到叶子节点）
   - 稳定：O(logn)
   - 有利于查询性能预测
   ```

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
- 查询优化器认为唯一索引选择性更好

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

**原则**：区分度高的列放在前面。

```sql
-- ❌ 不好：性别只有2个值，区分度低
CREATE INDEX idx_bad ON users(gender, age, city);

-- ✅ 好：城市区分度高
CREATE INDEX idx_good ON users(city, age, gender);
```

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

**回表的性能开销**：
1. 二级索引查找：随机I/O
2. 主键索引查找：随机I/O
3. **两次随机I/O**，性能差

**优化策略**：
```sql
-- 场景：查询用户信息，条件是 name
-- 表结构：id(主键), name, age, email, address

-- ❌ 慢：大量回表
SELECT * FROM users WHERE name LIKE 'A%';

-- ✅ 快：覆盖索引 + 延迟关联
CREATE INDEX idx_name ON users(name);

SELECT u.* 
FROM users u
INNER JOIN (
  SELECT id FROM users WHERE name LIKE 'A%' LIMIT 1000
) t ON u.id = t.id;
-- 先用覆盖索引查出 id，再回表（减少回表次数）
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

## 执行计划

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
| **key_len** | 索引长度 | 越短越好 |
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
| **READ UNCOMMITTED** | ✅ 可能 | ✅ 可能 | ✅ 可能 | 高 |
| **READ COMMITTED** | ❌ 不可能 | ✅ 可能 | ✅ 可能 | 中高 |
| **REPEATABLE READ** (默认) | ❌ 不可能 | ❌ 不可能 | ⚠️ 可能 | 中 |
| **SERIALIZABLE** | ❌ 不可能 | ❌ 不可能 | ❌ 不可能 | 低 |

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
- InnoDB通过**保存数据的多个版本**，实现无锁读
- 只在RR和RC隔离级别下生效

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
- 事务开始时创建Read View，整个事务期间使用同一个
- 读到的数据始终一致（可重复读）

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
T1事务开始时创建Read View：
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
- T1的Read View在事务开始时创建，**不会更新**
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

**MVCC的局限**：
- MVCC只能防止**UPDATE**导致的幻读
- 对于**INSERT**的新记录，MVCC无能为力（新记录在undo log中没有历史版本）

**InnoDB的解决方案：MVCC + Next-Key Lock**

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

**什么情况下MVCC无法防止幻读？**
```sql
-- 场景：T2在T1的Read View创建之前就已提交

T2: BEGIN;
T2: INSERT INTO users (age) VALUES (25);  -- trx_id=99
T2: COMMIT;

T1: BEGIN;  -- 此时创建Read View: min_trx_id=100
T1: SELECT COUNT(*) FROM users WHERE age BETWEEN 20 AND 30;
    -- T2的记录 trx_id=99 < min_trx_id=100 → 可见！
    -- 如果T1之前在其他事务中查过，现在看到了新记录 → 幻读
```

**总结**：
- **MVCC**：防止UPDATE导致的不可重复读
- **Next-Key Lock**：防止INSERT/DELETE导致的幻读
- **RR隔离级别** = MVCC + Next-Key Lock（两者配合）

---

#### RC vs RR的区别（底层原理）

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
- `SELECT ... LOCK IN SHARE MODE`（共享锁）
- `UPDATE`, `DELETE`, `INSERT`
- 读取最新版本的数据
- **加锁**

```sql
SELECT * FROM users WHERE id = 1 FOR UPDATE;  -- 当前读，加排他锁
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

#### 锁示例

```sql
-- 表数据：id = 1, 5, 10, 15, 20
-- 索引：PRIMARY KEY(id)

-- T1执行
BEGIN;
SELECT * FROM users WHERE id = 10 FOR UPDATE;
```

**RR隔离级别下的加锁范围**：
- **行锁**：锁定 `id=10` 的记录
- **间隙锁**：锁定 `(5, 10)` 的间隙
- **Next-Key Lock**：锁定 `(5, 10]`

**防止的操作**（其他事务）：
```sql
-- ❌ 阻塞：更新 id=10
UPDATE users SET name = 'xxx' WHERE id = 10;

-- ❌ 阻塞：插入 id=6, 7, 8, 9（间隙内）
INSERT INTO users (id, name) VALUES (7, 'Bob');

-- ✅ 不阻塞：插入 id=11（不在间隙内）
INSERT INTO users (id, name) VALUES (11, 'Charlie');
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

**复制步骤**：
1. 主库写入binlog
2. 从库IO线程拉取binlog到Relay Log
3. 从库SQL线程读取Relay Log，执行SQL

---

#### 复制延迟

**延迟原因**：
1. **主库写入过快**：从库SQL线程来不及执行
2. **从库硬件差**：IO/CPU性能不足
3. **大事务**：一个大事务在从库执行很久
4. **锁冲突**：从库有长时间的查询阻塞

**监控延迟**：
```sql
-- 在从库执行
SHOW SLAVE STATUS\G

-- 关注字段
Seconds_Behind_Master: 5  -- 延迟秒数
```

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
-- MySQL 5.7.2+
SELECT MASTER_POS_WAIT('binlog_file', binlog_pos, timeout);
-- 等待从库同步到指定位置
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

## 性能优化

### 慢查询优化

#### 1. 开启慢查询日志

```sql
-- 配置
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 2;  -- 超过2秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

---

#### 2. 分析慢查询

**使用 `mysqldumpslow`**：
```bash
# 按查询时间排序，显示前10条
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 按查询次数排序
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log
```

---

#### 3. 优化步骤

**Step 1: 查看执行计划**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 1;
```

**Step 2: 检查索引**
```sql
SHOW INDEX FROM orders;
```

**Step 3: 创建合适的索引**
```sql
CREATE INDEX idx_user_status ON orders(user_id, status);
```

**Step 4: 验证效果**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 1;
-- 检查 type: ref, key: idx_user_status
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

## 常见面试题

### 1. 为什么索引能提升查询速度？

**答**：
- 索引使用B+Tree数据结构，查询复杂度O(log n)
- 减少磁盘I/O次数（3层B+Tree可存储2000万记录）
- 避免全表扫描

---

### 2. 什么时候不应该使用索引？

**答**：
- 表数据量很小（<1000行）
- 字段区分度很低（如性别、状态）
- 频繁更新的列（索引维护开销大）
- 长字符串字段（考虑前缀索引）

---

### 3. 事务隔离级别如何选择？

**答**：
- **金融/支付**：REPEATABLE READ 或 SERIALIZABLE（强一致性）
- **一般业务**：REPEATABLE READ（MySQL默认）
- **高并发场景**：READ COMMITTED（减少锁竞争）

---

### 4. 如何保证主从数据一致性？

**答**：
- **强一致**：写入后读主库（金融场景）
- **最终一致**：缓存 + 异步同步（游戏场景）
- **会话一致**：同一会话绑定主库

---

### 5. 死锁如何排查和解决？

**答**：
1. 查看死锁日志：`SHOW ENGINE INNODB STATUS`
2. 分析锁等待链路
3. 解决方案：
   - 固定加锁顺序
   - 缩短事务时间
   - 降低隔离级别
   - 使用乐观锁

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
- **事务与锁**：隔离级别、MVCC、死锁
- **性能优化**：EXPLAIN、慢查询优化
- **一致性**：主从复制、读写一致性
- **数据建模**：范式、分库分表

面试官会重点考察：
1. 原理理解（为什么这么设计）
2. 问题排查（慢查询、死锁如何定位）
3. 实战经验（金融场景的幂等、一致性保证）

**记住**：不要只背概念，要理解原理并能结合实际业务场景讲解！💪

---

## 📝 面试题自查

> 以下问题可作为学习本文后的自查清单，以资深面试官视角设计。每题附有简要回答，详细原理请回到文章对应章节深入学习。

---

**Q1: 为什么 MySQL InnoDB 选择 B+Tree 而不是 B-Tree、Hash 索引或红黑树？**

B+Tree 相比 B-Tree 的优势：① 所有数据都在叶子节点，非叶子节点只存 key+指针，单个节点能容纳更多索引项，树更矮，磁盘 I/O 更少（3层可存约2000万记录）；② 叶子节点形成双向链表，范围查询只需沿链表扫描（顺序 I/O），而 B-Tree 需要中序遍历（随机 I/O）；③ 查询路径长度固定（都到叶子节点），性能稳定可预测。Hash 索引不支持范围查询和排序。红黑树是二叉树，高度远大于 B+Tree，磁盘 I/O 次数多。

---

**Q2: InnoDB 和 MyISAM 的核心区别是什么？为什么现代项目几乎都用 InnoDB？**

核心区别：① InnoDB 支持事务（ACID），MyISAM 不支持；② InnoDB 使用行级锁，MyISAM 使用表级锁；③ InnoDB 支持外键和 MVCC，MyISAM 不支持；④ InnoDB 有 redo log 保证崩溃恢复，MyISAM 崩溃后需要手动修复；⑤ InnoDB 使用聚簇索引（数据和主键索引存储在一起），MyISAM 使用非聚簇索引（数据和索引分开存储）。现代项目需要事务支持、高并发能力和数据安全性，这些都是 InnoDB 的强项。

---

**Q3: redo log、binlog、undo log 各自的作用是什么？两阶段提交是怎么回事？**

redo log（InnoDB 层）：保证持久性，WAL 机制，先写日志后刷脏页，崩溃恢复时重放已提交的修改。binlog（Server 层）：用于主从复制和数据恢复（PITR），记录所有 DDL/DML 操作。undo log（InnoDB 层）：保证原子性（回滚）和 MVCC 版本链。两阶段提交：事务提交时 ① redo log 写入 prepare 状态 → ② binlog 写入磁盘 → ③ redo log 标记为 commit。目的是保证 redo log 和 binlog 的一致性，防止崩溃后主从数据不一致。

---

**Q4: 什么是聚簇索引？为什么 InnoDB 表必须有主键？使用 UUID 作为主键有什么问题？**

聚簇索引指数据按主键顺序物理存储在 B+Tree 的叶子节点中，InnoDB 的数据文件本身就是按主键组织的 B+Tree。因此必须有主键来组织数据（没有显式主键时 InnoDB 自动创建隐藏的 ROW_ID，但有全局锁瓶颈和溢出覆盖风险）。UUID 作为主键的问题：无序插入导致频繁页分裂，索引碎片化严重，占用空间大（36字节 vs BIGINT 8字节），性能远差于自增 ID。

---

**Q5: 什么是覆盖索引？什么是回表？如何通过覆盖索引优化查询？**

覆盖索引指查询所需的列全部包含在索引中，不需要再回到主键索引查完整行（EXPLAIN 中 Extra 显示 `Using index`）。回表指通过二级索引找到主键值后，再去主键索引查完整行数据，涉及两次 B+Tree 查找和两次随机 I/O。优化方式：① 创建包含查询列的联合索引；② 使用延迟关联：先用覆盖索引子查询获取 id 列表，再与原表 JOIN 回表（减少回表次数）。

---

**Q6: 联合索引的最左前缀原则是什么？为什么范围查询后面的列无法使用索引？**

联合索引 `(a, b, c)` 的 B+Tree 按 a→b→c 的顺序排列：先按 a 排序，a 相同时按 b 排序，b 相同时按 c 排序。因此查询必须从最左列 a 开始才能利用索引的有序性。当 a 是等值查询时，同一 a 值内 b 有序，可以继续用索引；但当 a 或 b 是范围查询时（如 `b > 5`），匹配到的范围内 c 的顺序不确定，无法利用索引对 c 进行过滤。MySQL 8.0+ 的 Index Skip Scan 可以在缺少最左列时跳跃扫描，但效率低于完整匹配。

---

**Q7: MVCC 是如何实现的？Read View 的可见性判断规则是什么？RC 和 RR 隔离级别下 Read View 有什么区别？**

MVCC 通过每行记录的隐藏列（DB_TRX_ID、DB_ROLL_PTR）和 undo log 版本链实现。读取时创建 Read View，包含当前活跃事务 ID 列表（m_ids）、最小活跃事务 ID（min_trx_id）、下一个分配的事务 ID（max_trx_id）。可见性规则：trx_id == 当前事务 → 可见；trx_id < min_trx_id → 已提交可见；trx_id ≥ max_trx_id → 未来事务不可见；trx_id 在 m_ids 中 → 未提交不可见；否则可见。RC 每次 SELECT 创建新 Read View（能看到其他事务已提交的最新数据），RR 整个事务共用同一个 Read View（保证可重复读）。

---

**Q8: InnoDB 在 RR 隔离级别下是如何解决幻读问题的？MVCC 能完全解决幻读吗？**

RR 下通过 MVCC + Next-Key Lock 配合解决幻读。快照读（普通 SELECT）通过 MVCC 的 Read View 过滤掉新插入的行（新行的 trx_id 在活跃列表中或大于 max_trx_id），但 MVCC 不能完全防止幻读（如果新插入的事务在 Read View 创建前已提交，则对当前事务可见）。当前读（SELECT FOR UPDATE / UPDATE / DELETE）通过 Next-Key Lock 锁定索引记录及间隙，阻止其他事务在锁定范围内插入新行，从根本上防止幻读。

---

**Q9: Next-Key Lock 是什么？它由哪些锁组成？请举例说明加锁范围。**

Next-Key Lock = 行锁（Record Lock）+ 间隙锁（Gap Lock），锁定一条记录本身以及该记录之前的间隙，是一个左开右闭的区间。例如表中有 id=1,5,10,15，执行 `SELECT * FROM t WHERE id = 10 FOR UPDATE` 时，在 RR 隔离级别下加锁范围是 `(5, 10]` 的 Next-Key Lock。其他事务无法更新 id=10 的记录，也无法在 (5,10) 的间隙中插入新记录，但可以在 id>10 的位置插入。间隙锁仅在 RR 隔离级别下生效，RC 隔离级别不使用间隙锁。

---

**Q10: 索引下推（ICP）是什么？它在什么场景下能提升性能？**

Index Condition Pushdown（MySQL 5.6+）将部分 WHERE 条件下推到存储引擎层，在索引遍历时直接过滤不满足条件的记录，减少回表次数。典型场景：联合索引 `(name, age)` 查询 `WHERE name LIKE 'A%' AND age = 25`，无 ICP 时索引只能用 name 条件扫描出所有 `A%` 的记录再回表，然后 Server 层过滤 age；有 ICP 时在索引中就检查 age=25，只对满足条件的记录回表。EXPLAIN 中 Extra 显示 `Using index condition`。

---

**Q11: 有哪些常见的索引失效场景？如何避免？**

常见场景：① 对索引列使用函数或表达式（如 `YEAR(created_at) = 2024`，改用范围查询）；② 隐式类型转换（VARCHAR 列传数字，MySQL 对索引列做 CAST）；③ 前导模糊查询（`LIKE '%xxx'`）；④ OR 条件中部分列无索引；⑤ 不等于条件（`!=` / `NOT IN`）可能导致优化器选择全表扫描；⑥ 联合索引不满足最左前缀。避免方法：保持查询条件与索引列类型一致、避免对索引列做运算、用 UNION 替代 OR、合理设计联合索引顺序。

---

**Q12: EXPLAIN 执行计划中的 type 字段有哪些值？从好到差的排列是什么？**

从好到差：system（表只一行）> const（主键/唯一索引等值）> eq_ref（JOIN时唯一索引匹配）> ref（非唯一索引等值）> range（范围扫描）> index（全索引扫描）> ALL（全表扫描）。生产环境至少要达到 range 级别，出现 ALL 必须优化。另外关注 Extra 字段：`Using index` 好（覆盖索引），`Using filesort` 和 `Using temporary` 差（需额外排序或临时表）。

---

**Q13: 死锁是如何产生的？如何排查和预防？**

死锁产生条件：两个或多个事务互相等待对方持有的锁。典型场景：T1 锁 id=1 等 id=2，T2 锁 id=2 等 id=1。排查方法：`SHOW ENGINE INNODB STATUS\G` 查看 LATEST DETECTED DEADLOCK 部分，分析两个事务各自持有和等待的锁。预防措施：① 固定加锁顺序（所有事务按相同顺序访问资源）；② 缩短事务持有锁的时间（先计算后开事务）；③ 降低隔离级别到 RC（减少间隙锁）；④ 使用乐观锁（版本号机制）替代悲观锁。InnoDB 的死锁检测会自动回滚代价小的事务。

---

**Q14: 主从复制的原理是什么？有哪些方式解决主从延迟导致的读写不一致？**

原理：① 主库将变更写入 binlog；② 从库 IO Thread 拉取 binlog 到 Relay Log；③ 从库 SQL Thread 重放 Relay Log。延迟原因：主库写入过快、从库硬件差、大事务、锁冲突。解决读写不一致的方案：① 强制读主库（写入后短时间内读主库，适合金融场景）；② 等待同步完成（`MASTER_POS_WAIT`）；③ 会话级别绑定主库；④ 缓存+异步写（写入主库同时写缓存，读时先查缓存，适合游戏场景）。

---

**Q15: 深分页（如 LIMIT 1000000, 10）为什么慢？如何优化？**

慢的原因：MySQL 需要扫描 offset+limit 行（1000010行），丢弃前 offset 行，只返回 limit 行，大量无效扫描和回表。优化方案：① 游标分页（Keyset Pagination）：记住上一页最后一条记录的 ID，用 `WHERE id > @last_id ORDER BY id LIMIT 10`，利用主键索引直接定位，性能恒定；② 延迟关联：子查询用覆盖索引先查出 ID（`SELECT id FROM orders ORDER BY id LIMIT 1000000, 10`），再与原表 JOIN 回表，减少回表开销；③ 业务层面限制最大翻页深度。

---

**Q16: 分库分表有哪些策略？水平拆分的范围分片和哈希分片各有什么优缺点？**

分库分表分为垂直拆分（按业务拆库/按冷热字段拆表）和水平拆分（按行拆分到多个表/库）。范围分片（如 id 1-100万到表1，100万-200万到表2）：优点是扩容方便（新增范围即可）、范围查询高效；缺点是数据分布可能不均（新数据集中在最新分片，热点问题）。哈希分片（如 id % 4）：优点是数据分布均匀；缺点是扩容困难（改变模数需要数据迁移）、范围查询需要扫描所有分片。实际中常用一致性哈希来缓解扩容问题。

---

**Q17: 乐观锁和悲观锁在 MySQL 中如何实现？各自适用什么场景？**

悲观锁：通过 `SELECT ... FOR UPDATE` 在事务中对行加排他锁，其他事务读写都被阻塞。适用于写冲突概率高的场景（如金融扣款），确保数据一致性，但并发性能较差。乐观锁：通过版本号（version）或时间戳实现，UPDATE 时 `WHERE version = @old_version`，如果 ROW_COUNT()=0 说明被其他事务修改过，需要重试。适用于读多写少、冲突概率低的场景，不加锁，并发性能好。实际中金融核心扣款用悲观锁保证强一致，普通业务更新用乐观锁提高吞吐。

---

**Q18: binlog 有几种格式？生产环境推荐用哪种？为什么？**

三种格式：① STATEMENT 记录原始 SQL 语句，体积小但存在不确定性（如 `NOW()`、`UUID()` 在主从可能不一致）；② ROW 记录每行变更的前后镜像，体积大但精确无歧义；③ MIXED 默认用 STATEMENT，遇到不确定函数自动切 ROW。生产环境推荐 ROW 格式：主从数据完全一致，不受函数/触发器/存储过程影响，方便用 binlog 做数据恢复和审计。缺点是日志量大，但磁盘成本远低于数据不一致的风险。

---

**Q19: B+树的分裂和合并是什么？为什么自增主键能减少页分裂？**

分裂：当一个节点的 key 数量超过上限时，将节点拆分为两个，中间 key 提升到父节点；如果父节点也满了则递归分裂，极端情况下根节点分裂导致树增高。合并：当删除 key 后节点的 key 数量低于下限时，先尝试从兄弟节点借 key（重分配），借不到则与兄弟合并。自增主键保证新记录总是追加在 B+Tree 最右侧的叶子节点，顺序插入不会触发中间节点的分裂；而 UUID 等无序主键会随机插入到已满的中间节点，频繁触发分裂，产生大量碎片。

---

**Q20: 如何设计一个金融交易系统的扣款流程？需要考虑哪些问题？**

核心考虑：① 原子性：在事务中完成余额查询、检查、扣减、记录流水，使用 `FOR UPDATE` 悲观锁或 version 乐观锁保证一致；② 幂等性：通过 request_id 唯一键保证同一请求不重复扣款（`INSERT ... ON DUPLICATE KEY`）；③ 余额校验：SQL 层面 `WHERE balance >= @amount` 双重检查（应用层+SQL层）防止并发超扣；④ 流水记录：记录交易前后余额（balance_before/balance_after）用于对账；⑤ 防止死锁：多账户操作时按 account_id 升序加锁。
