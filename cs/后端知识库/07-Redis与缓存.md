# Redis 与缓存 - 深度解析

---

## 📑 目录

### 数据结构与缓存一致性
1. [Redis数据结构](#1-redis数据结构)
2. [缓存三大问题](#2-缓存三大问题)
3. [分布式锁](#3-分布式锁)

### 持久化与高可用
4. [持久化机制](#4-持久化机制)
5. [高可用架构](#5-高可用架构)

### 实战与自查
6. [实战案例](#6-实战案例)
7. [Redis 7.0+ 新特性](#7-redis-70-新特性)
8. [面试题自查](#8-面试题自查)

---

## 1. Redis数据结构

### 1.1 五大基础数据类型

| 类型 | 底层实现 | 应用场景 | 常用命令 |
|------|----------|----------|----------|
| **String** | SDS（简单动态字符串） | 缓存、计数器、分布式锁 | SET、GET、INCR、SETNX |
| **List** | quicklist（双向链表+ziplist） | 消息队列、时间线 | LPUSH、RPOP、LRANGE |
| **Hash** | ziplist/hashtable | 对象缓存、购物车 | HSET、HGET、HGETALL |
| **Set** | intset/hashtable | 去重、共同好友 | SADD、SMEMBERS、SINTER |
| **ZSet** | ziplist/skiplist+hashtable | 排行榜、延时队列 | ZADD、ZRANGE、ZRANK |

### 1.2 三大高级数据类型

| 类型 | 说明 | 应用场景 |
|------|------|----------|
| **Bitmap** | 位图 | 签到、用户在线状态 |
| **HyperLogLog** | 基数统计 | UV统计（允许误差） |
| **Geo** | 地理位置 | 附近的人、打车 |

### 1.3 String应用：缓存对象

```bash
# 缓存用户信息（JSON）
SET user:1001 '{"id":1001,"name":"张三","age":25}'
GET user:1001

# 缓存用户信息（Hash，更灵活）
HSET user:1001 id 1001 name "张三" age 25
HGET user:1001 name
HGETALL user:1001

# 设置过期时间（1小时）
SETEX user:1001 3600 '{"id":1001,"name":"张三"}'
```

### 1.4 ZSet应用：排行榜

```bash
# 更新分数
ZADD rank:score 1000 user:1001
ZADD rank:score 1500 user:1002
ZADD rank:score 1200 user:1003

# 查询前10名
ZREVRANGE rank:score 0 9 WITHSCORES

# 查询用户排名（0-based，需要+1）
ZREVRANK rank:score user:1001

# 查询用户分数
ZSCORE rank:score user:1001

# 增加分数
ZINCRBY rank:score 100 user:1001

# 查询某个分数范围的用户
ZRANGEBYSCORE rank:score 1000 2000
```

### 1.5 Bitmap应用：签到

```bash
# 用户1001在2024年1月每天的签到情况（1bit/天，31天只需31bit）
SETBIT sign:1001:202401 0 1   # 1月1日签到
SETBIT sign:1001:202401 1 1   # 1月2日签到
SETBIT sign:1001:202401 2 0   # 1月3日未签到

# 查询某天是否签到
GETBIT sign:1001:202401 0

# 统计本月签到天数
BITCOUNT sign:1001:202401

# 查询首次未签到的日期
BITPOS sign:1001:202401 0
```

---

### 1.6 底层数据结构编码（深度剖析）⭐⭐⭐

**为什么要了解底层编码？**
- Redis会根据数据大小/数量自动选择最优编码
- 不同编码内存占用和性能差异巨大
- 面试高频：为什么Redis这么快？底层如何优化内存？

---

#### 1.6.1 String的底层：SDS（Simple Dynamic String）

**为什么不用C字符串？**

| 特性 | C字符串 | SDS |
|------|---------|-----|
| **获取长度** | O(n)（遍历到\0） | O(1)（存储len字段） |
| **二进制安全** | ❌（\0结尾） | ✅（存储len，可包含\0） |
| **缓冲区溢出** | ❌（容易溢出） | ✅（自动扩容） |
| **内存分配** | 每次修改都realloc | 预分配+惰性释放 |

**SDS结构**：
```c
struct sdshdr {
    int len;      // 已使用长度
    int free;     // 未使用长度
    char buf[];   // 字节数组
};

// 示例：存储 "hello"
{
    len: 5,
    free: 0,
    buf: ['h','e','l','l','o','\0']  // C兼容，仍以\0结尾
}
```

**SDS的优化策略**：

1. **预分配（空间换时间）**：
   ```
   场景：字符串从 "hello" (5字节) 扩展到 "hello world" (11字节)
   
   C字符串：realloc(11字节)，每次扩展都重新分配
   SDS：
   - 如果len < 1MB：分配 2 * len
   - 如果len >= 1MB：额外分配 1MB
   
   示例：扩展到11字节
   - 实际分配：22字节
   - len=11, free=11
   - 下次追加6字节，无需realloc（使用free空间）
   ```

2. **惰性释放**：
   ```
   场景：字符串从 "hello world" (11字节) 截断到 "hello" (5字节)
   
   C字符串：realloc(5字节)，立即释放6字节
   SDS：
   - len=5, free=6（不释放内存）
   - 下次追加6字节，无需realloc
   ```

**为什么INCR这么快？**
```c
// INCR key 实现（伪代码）
if (isEncodedAsInt(key)) {
    // 如果value可以用long表示，直接操作整数（O(1)）
    long val = getLong(key);
    val++;
    setLong(key, val);
} else {
    // 否则转换为SDS，字符串操作
    char *str = getSDS(key);
    long val = atol(str);
    val++;
    setSDS(key, itoa(val));
}
```

---

#### 1.6.2 List的底层：quicklist（ziplist + linkedlist）

**为什么不直接用双向链表？**
- 链表每个节点需要**prev/next指针**（16字节）+ 数据
- 如果数据只有8字节，指针占用比数据还多（内存浪费）

**Redis的优化：quicklist = 压缩列表 + 双向链表**

```
quicklist节点（双向链表）：
[prev] ← → [ziplist] ← → [ziplist] ← → [ziplist] ← → [next]
           ↓               ↓               ↓
       连续内存块      连续内存块      连续内存块
       [1][2][3]      [4][5][6]      [7][8][9]
```

**ziplist（压缩列表）原理**：

```
ziplist结构：一块连续内存，紧凑存储
[zlbytes][zltail][zllen][entry1][entry2]...[zlend]

zlbytes: 4字节，ziplist总字节数
zltail:  4字节，尾节点偏移量（O(1)访问尾部）
zllen:   2字节，节点数量
zlend:   1字节，0xFF，标记结尾

每个entry：
[prevlen][encoding][data]

prevlen：前一个entry长度（1或5字节）
- 如果 < 254字节：1字节存储
- 如果 >= 254字节：5字节存储（1字节0xFE + 4字节长度）

encoding：编码类型（1/2/5字节）
- 整数：00|01|10 开头
- 字符串：11 开头

data：实际数据
```

**示例：存储 [1, 2, 3]**
```
ziplist（假设整数编码）：
[zlbytes=20][zltail=15][zllen=3]
[prevlen=0][encoding=INT][1]     // entry1
[prevlen=3][encoding=INT][2]     // entry2
[prevlen=3][encoding=INT][3]     // entry3
[zlend=0xFF]

总共20字节，平均每个元素6.7字节
对比链表：每个节点至少 16(指针) + 8(数据) = 24字节
节省空间：(24*3 - 20) / (24*3) = 72%
```

**ziplist的问题：连锁更新**

```
场景：在头部插入一个长度254的元素

原ziplist：
[entry1:253字节] [entry2:253字节] [entry3:253字节]
- entry2的prevlen=253（1字节存储）
- entry3的prevlen=253（1字节存储）

插入254字节元素：
[新entry:254字节] [entry1:253字节] [entry2:253字节] [entry3:253字节]
- entry1的prevlen=254（需要5字节！）→ entry1扩展
- entry2的prevlen=entry1新长度（可能>=254）→ entry2扩展
- entry3的prevlen=entry2新长度 → entry3扩展
→ 连锁更新，最坏O(n²)

Redis的缓解方案：
1. 限制ziplist大小（默认512字节或64个元素）
2. 超过阈值，转为quicklist
```

**quicklist的配置**：
```bash
# 每个ziplist节点的字节数限制
list-max-ziplist-size -2
# -1: 4KB
# -2: 8KB（默认）
# -3: 16KB

# 两端不压缩的节点数（LRU优化）
list-compress-depth 0
# 0: 不压缩（默认）
# 1: 头尾各1个节点不压缩
# 2: 头尾各2个节点不压缩
```

---

#### 1.6.3 Hash的底层：ziplist / hashtable

**编码切换条件**：
```bash
# Hash使用ziplist的条件（同时满足）
hash-max-ziplist-entries 512  # 元素个数 <= 512
hash-max-ziplist-value 64     # 单个value <= 64字节

# 超过任一阈值，转为hashtable
```

**ziplist编码（小Hash优化）**：
```
存储：{name: "Alice", age: "25", city: "Beijing"}

ziplist结构：
[zlbytes][zltail][zllen=6]
[prevlen][encoding]["name"]    // key1
[prevlen][encoding]["Alice"]   // value1
[prevlen][encoding]["age"]     // key2
[prevlen][encoding]["25"]      // value2
[prevlen][encoding]["city"]    // key3
[prevlen][encoding]["Beijing"] // value3
[zlend]

key-value对依次存储，查找O(n)，但n很小（<=512）时可接受
```

**hashtable编码（大Hash）**：
```c
// Redis的hashtable结构（渐进式rehash）
typedef struct dict {
    dictht ht[2];        // 两个哈希表（用于rehash）
    long rehashidx;      // rehash进度（-1表示未进行）
} dict;

typedef struct dictht {
    dictEntry **table;   // 哈希表数组
    unsigned long size;  // 桶数量
    unsigned long used;  // 已有元素数
} dictht;
```

**渐进式rehash原理**：

```
为什么需要渐进式？
- 一次性rehash可能阻塞Redis（大Hash有百万key）
- 渐进式：每次操作顺便迁移1个桶

rehash过程：
1. 为ht[1]分配更大空间（ht[0].size * 2）
2. rehashidx=0，开始渐进式迁移
3. 每次增删改查时：
   - 顺便迁移ht[0].table[rehashidx]的所有key到ht[1]
   - rehashidx++
4. 迁移完成：交换ht[0]和ht[1]，rehashidx=-1

查询过程：
- 如果正在rehash（rehashidx >= 0）：
  - 先查ht[0]，未命中再查ht[1]
- 否则只查ht[0]
```

---

#### 1.6.4 Set的底层：intset / hashtable

**编码切换条件**：
```bash
# Set使用intset的条件（同时满足）
set-max-intset-entries 512  # 元素个数 <= 512
# 所有元素都是整数

# 超过阈值或出现非整数，转为hashtable
```

**intset编码（整数集合优化）**：
```c
typedef struct intset {
    uint32_t encoding;  // 编码类型：INT16/INT32/INT64
    uint32_t length;    // 元素个数
    int8_t contents[];  // 柔性数组，存储整数
} intset;

// 示例：存储 [1, 2, 3]（都可用INT16表示）
intset {
    encoding: INT16,
    length: 3,
    contents: [1, 2, 3]  // 每个占2字节，共6字节
}

// 如果插入一个大整数 65536（需要INT32）
intset {
    encoding: INT32,  // 升级为INT32
    length: 4,
    contents: [1, 2, 3, 65536]  // 每个占4字节，共16字节
}
```

**intset的升级机制**：
```
插入一个超出当前encoding范围的整数时触发升级：

步骤：
1. 计算新encoding（INT16 → INT32 → INT64）
2. 重新分配内存（length * new_size）
3. 倒序迁移元素（从后往前，避免覆盖）
4. 插入新元素

为什么倒序迁移？
[1][2][3] (INT16, 每个2字节) → 升级为INT32（每个4字节）
如果正序：
- 移动1到新位置，可能覆盖2的数据
倒序：
- 从3开始移动，不会覆盖未迁移的数据

注意：只升级不降级！
- 删除65536后，仍然是INT32编码
```

---

#### 1.6.5 ZSet的底层：ziplist / skiplist + hashtable

**编码切换条件**：
```bash
# ZSet使用ziplist的条件（同时满足）
zset-max-ziplist-entries 128  # 元素个数 <= 128
zset-max-ziplist-value 64     # 单个member <= 64字节

# 超过任一阈值，转为skiplist + hashtable
```

**ziplist编码**：
```
存储：{Alice: 100, Bob: 90, Charlie: 95}（按score排序）

ziplist结构：
[zlbytes][zltail][zllen=6]
[prevlen][encoding]["Bob"]     // member1
[prevlen][encoding][90]        // score1
[prevlen][encoding]["Charlie"] // member2
[prevlen][encoding][95]        // score2
[prevlen][encoding]["Alice"]   // member3
[prevlen][encoding][100]       // score3
[zlend]

按score升序存储：Bob(90) → Charlie(95) → Alice(100)
```

**skiplist + hashtable编码**：

```c
// ZSet的复合结构
typedef struct zset {
    dict *dict;        // hashtable：member → score（O(1)查分数）
    zskiplist *zsl;    // skiplist：按score排序（O(logn)范围查询）
} zset;

// skiplist结构
typedef struct zskiplist {
    zskiplistNode *header, *tail;
    unsigned long length;
    int level;  // 当前最大层数
} zskiplist;

typedef struct zskiplistNode {
    sds member;               // 成员
    double score;             // 分数
    zskiplistNode *backward;  // 后向指针（层0）
    struct zskiplistLevel {
        zskiplistNode *forward;  // 前向指针
        unsigned int span;       // 跨度（用于计算排名）
    } level[];  // 柔性数组，每个节点层数随机
} zskiplistNode;
```

**skiplist原理**（深度讲解）：

```
为什么用skiplist而不是红黑树？
1. 范围查询更友好：找到起点后，沿level[0]顺序遍历
2. 实现更简单：不需要复杂的旋转操作
3. 内存局部性更好：level[0]是双向链表，缓存友好

skiplist结构（4层示例）：
Level 3: head --------------------------> Alice(100) ------> NULL
Level 2: head -------> Bob(90) ---------> Alice(100) ------> NULL
Level 1: head -> Charlie(95) -> Bob(90) -> Alice(100) ------> NULL
Level 0: head -> Charlie(95) -> Bob(90) -> Alice(100) -> David(110) -> NULL
               ↑          ↓        ↑         ↓         ↑          ↓
               |__________|________|_________|_________|__________|
                        backward指针（双向链表）

查找Alice(100)的过程：
1. 从head的最高层（level 3）开始
2. level 3: head → Alice(100)（找到！）
3. 时间复杂度：O(logn)

范围查询 ZRANGE 0 1（前2个）：
1. 找到起点（head.level[0].forward = Charlie）
2. 沿level[0]顺序遍历：Charlie → Bob
3. 返回2个元素
```

**skiplist的随机层数算法**：
```c
// Redis的层数生成（幂次定律）
int randomLevel() {
    int level = 1;
    while (rand() < SKIPLIST_P && level < MAX_LEVEL) {
        level++;
    }
    return level;
}

// SKIPLIST_P = 0.25
// MAX_LEVEL = 32

概率分布：
- level 1: 75%
- level 2: 18.75%
- level 3: 4.69%
- level 4: 1.17%
...

为什么P=0.25？
- 平衡空间和时间：每4个节点有1个提升到上层
- 期望查找次数：O(logn)，常数因子较小
```

**为什么ZSet需要两个数据结构？**

| 操作 | skiplist | hashtable | 复合结构 |
|------|----------|-----------|----------|
| **ZADD** | O(logn) | O(1) | O(logn) |
| **ZSCORE** | O(logn) | O(1) | **O(1)** ✅ |
| **ZRANGE** | **O(logn + k)** ✅ | ❌ 无序 | **O(logn + k)** ✅ |
| **ZRANK** | **O(logn)** ✅ | ❌ 无排名 | **O(logn)** ✅ |

---

### 1.7 为什么Redis单线程还这么快？⭐⭐⭐

**误区**：Redis是单线程的
**事实**：
- **6.0之前**：核心网络I/O是单线程
- **6.0之后**：网络I/O多线程（可选），命令执行仍是单线程

**单线程快的原因**：

#### 原因1：纯内存操作
```
内存速度：100ns（纳秒）
SSD速度：100μs（微秒） = 100,000ns
HDD速度：10ms（毫秒） = 10,000,000ns

Redis全内存操作，比磁盘快 10万 ~ 1000万倍！
```

#### 原因2：IO多路复用（epoll）

**什么是IO多路复用？**

```
传统多线程模型（每个连接一个线程）：
Thread 1 → read(socket1) → 阻塞等待数据
Thread 2 → read(socket2) → 阻塞等待数据
Thread 3 → read(socket3) → 阻塞等待数据
→ 10000个连接需要10000个线程（内存爆炸、上下文切换开销大）

IO多路复用（一个线程监听多个socket）：
Thread 1 → epoll_wait([socket1, socket2, ..., socket10000])
           → 返回就绪的socket列表：[socket2, socket5]
           → 处理socket2和socket5
           → 继续epoll_wait
→ 1个线程处理10000个连接！
```

**epoll原理（深度剖析）**：

```c
// 1. 创建epoll实例
int epfd = epoll_create(1024);

// 2. 添加socket到epoll
struct epoll_event ev;
ev.events = EPOLLIN;  // 监听可读事件
ev.data.fd = socket_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, socket_fd, &ev);

// 3. 等待事件（阻塞）
struct epoll_event events[MAX_EVENTS];
int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout);

// 4. 处理就绪的socket
for (int i = 0; i < nfds; i++) {
    if (events[i].events & EPOLLIN) {
        // 有数据可读
        read(events[i].data.fd, buf, sizeof(buf));
        processCommand(buf);
    }
}
```

**epoll的优势（vs select/poll）**：

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| **文件描述符数量** | 1024（FD_SETSIZE） | 无限制 | 无限制 |
| **性能** | O(n) | O(n) | **O(1)** ✅ |
| **数据拷贝** | 每次拷贝fd_set | 每次拷贝pollfd | **内核直接操作** ✅ |
| **触发方式** | 水平触发（LT） | 水平触发（LT） | **边缘触发（ET）** ✅ |

**为什么epoll是O(1)？**
```
select/poll：
- 每次调用都需要把所有fd从用户空间拷贝到内核空间
- 内核遍历所有fd，检查是否就绪（O(n)）

epoll：
- fd注册到内核的红黑树（一次性）
- 内核维护一个就绪列表（链表）
- 当fd就绪时，内核回调函数将fd加入就绪列表
- epoll_wait只需返回就绪列表（O(1)）
```

#### 原因3：高效的数据结构
- SDS、ziplist、intset等内存优化
- skiplist平衡了范围查询和插入性能

#### 原因4：避免上下文切换
- 单线程无锁，无上下文切换开销
- 多线程的互斥锁、条件变量开销很大

**Redis 6.0的多线程优化**：
```
[客户端] → [网络I/O线程1] \
[客户端] → [网络I/O线程2]  → [主线程（命令执行）] → [网络I/O线程1] → [客户端]
[客户端] → [网络I/O线程3] /                        \[网络I/O线程2] → [客户端]

优化点：
- 网络I/O（读取socket、发送响应）用多线程
- 命令执行仍是单线程（保持简单、无锁）
- 适用场景：网络I/O成为瓶颈（大value场景）

配置：
io-threads 4             # 启用4个I/O线程
io-threads-do-reads yes  # I/O线程处理读操作（默认只处理写）
```

---

## 2. 缓存三大问题

### 2.1 缓存穿透

**定义**：查询**不存在的数据**，缓存和DB都没有，每次请求都打到DB

**场景**：恶意攻击，查询 id=-1 的数据

**危害**：DB压力大，可能被打垮

**解决方案**：

**方案1：缓存空值**
```go
func GetUser(id int64) (*User, error) {
    // 1. 查缓存
    user, err := redis.Get(fmt.Sprintf("user:%d", id))
    if err == nil {
        if user == "null" {
            return nil, ErrUserNotFound  // 缓存的空值
        }
        return parseUser(user), nil
    }
    
    // 2. 查数据库
    user, err = db.GetUser(id)
    if err != nil {
        // 缓存空值，5分钟过期
        redis.Set(fmt.Sprintf("user:%d", id), "null", 5*time.Minute)
        return nil, err
    }
    
    // 3. 写缓存
    redis.Set(fmt.Sprintf("user:%d", id), user.ToJSON(), 1*time.Hour)
    return user, nil
}
```

**方案2：布隆过滤器**（推荐）
```go
import "github.com/bits-and-blooms/bloom/v3"

var bf = bloom.NewWithEstimates(10000000, 0.01)  // 1000万元素，1%误判率

// 初始化：将所有用户ID加入布隆过滤器
func InitBloomFilter() {
    users := db.GetAllUserIDs()
    for _, uid := range users {
        bf.AddString(fmt.Sprintf("%d", uid))
    }
}

// 查询时先判断
func GetUser(id int64) (*User, error) {
    // 1. 布隆过滤器判断（一定不存在 or 可能存在）
    if !bf.TestString(fmt.Sprintf("%d", id)) {
        return nil, ErrUserNotFound  // 一定不存在，直接返回
    }
    
    // 2. 查缓存
    // 3. 查数据库
    // ...
}
```

**对比**：
| 方案 | 优点 | 缺点 |
|------|------|------|
| 缓存空值 | 简单 | 占用缓存空间，大量无效key |
| 布隆过滤器 | 内存占用小，判断快 | 有误判率（1%），无法删除 |

---

### 2.2 缓存击穿

**定义**：**热点key过期**，瞬间大量请求打到DB

**场景**：秒杀商品缓存过期，1万并发查询DB

**危害**：DB瞬间压力暴增

**解决方案**：

**方案1：互斥锁（推荐）**
```go
import "golang.org/x/sync/singleflight"

var sf singleflight.Group

func GetProduct(id int64) (*Product, error) {
    // 1. 查缓存
    product, err := redis.Get(fmt.Sprintf("product:%d", id))
    if err == nil {
        return parseProduct(product), nil
    }
    
    // 2. 缓存未命中，使用singleflight（同一时刻只有一个请求查DB）
    v, err, _ := sf.Do(fmt.Sprintf("product:%d", id), func() (interface{}, error) {
        // 双重检查：其他请求可能已经写入缓存
        product, err := redis.Get(fmt.Sprintf("product:%d", id))
        if err == nil {
            return parseProduct(product), nil
        }
        
        // 查数据库
        product, err := db.GetProduct(id)
        if err != nil {
            return nil, err
        }
        
        // 写缓存
        redis.Set(fmt.Sprintf("product:%d", id), product.ToJSON(), 1*time.Hour)
        return product, nil
    })
    
    if err != nil {
        return nil, err
    }
    return v.(*Product), nil
}
```

**方案2：热点数据永不过期**
```go
// 逻辑过期（物理上不过期，业务层判断）
type CacheValue struct {
    Data      string
    ExpireAt  int64  // 逻辑过期时间
}

func GetProduct(id int64) (*Product, error) {
    // 1. 查缓存
    val, err := redis.Get(fmt.Sprintf("product:%d", id))
    if err != nil {
        // 缓存未命中，查DB
        return queryDB(id)
    }
    
    cacheVal := parseCacheValue(val)
    
    // 2. 判断逻辑过期
    if time.Now().Unix() > cacheVal.ExpireAt {
        // 过期了，异步更新（当前请求先返回旧数据）
        go func() {
            updateCache(id)
        }()
    }
    
    return parseProduct(cacheVal.Data), nil
}
```

**对比**：
| 方案 | 优点 | 缺点 |
|------|------|------|
| 互斥锁 | 保证一致性 | 并发请求等待 |
| 永不过期 | 高可用 | 可能返回旧数据 |

---

### 2.3 缓存雪崩

**定义**：**大量key同时过期**，请求全部打到DB

**场景**：批量导入数据，设置相同过期时间（如1小时），1小时后同时失效

**危害**：DB瞬间压力暴增，可能崩溃

**解决方案**：

**方案1：过期时间加随机值**
```go
// 基础过期时间：1小时
baseExpire := 1 * time.Hour

// 随机偏移：±10分钟
randomOffset := time.Duration(rand.Intn(600)) * time.Second

// 最终过期时间
expire := baseExpire + randomOffset

redis.Set(key, value, expire)
```

**方案2：热点数据永不过期**（同缓存击穿）

**方案3：多级缓存**
```
[本地缓存(1s)] → [Redis(1min)] → [DB]
```

**方案4：限流降级**
```go
import "golang.org/x/time/rate"

var limiter = rate.NewLimiter(1000, 2000)  // 1000 QPS, burst 2000

func GetData(key string) (string, error) {
    if !limiter.Allow() {
        return "", ErrRateLimitExceeded  // 限流
    }
    
    // 查询数据
    // ...
}
```

---

## 3. 分布式锁

### 3.1 基于SETNX实现

**版本1：最简单（❌ 有问题）**
```go
// 问题：获取锁后进程崩溃，锁永远不释放
func Lock(key string) bool {
    return redis.SetNX(key, "1", 0).Val()  // 没有过期时间
}

func Unlock(key string) {
    redis.Del(key)
}
```

**版本2：加过期时间（❌ 仍有问题）**
```go
// 问题：SetNX和Expire不是原子操作，中间崩溃锁不释放
func Lock(key string) bool {
    if redis.SetNX(key, "1", 0).Val() {
        redis.Expire(key, 10*time.Second)
        return true
    }
    return false
}
```

**版本3：原子操作（❌ 仍有问题）**
```go
// 问题：可能释放别人的锁
func Lock(key string) bool {
    return redis.SetNX(key, "1", 10*time.Second).Val()
}

func Unlock(key string) {
    redis.Del(key)  // 可能删除其他进程的锁
}
```

**版本4：加锁标识（✅ 基本可用）**
```go
func Lock(key string, value string, expire time.Duration) bool {
    return redis.SetNX(key, value, expire).Val()
}

func Unlock(key string, value string) bool {
    // Lua脚本保证原子性
    script := `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `
    result := redis.Eval(script, []string{key}, value).Val()
    return result == int64(1)
}

// 使用示例
func DoSomething() {
    lockKey := "lock:order:1001"
    lockValue := uuid.New().String()  // 唯一标识
    
    // 获取锁
    if !Lock(lockKey, lockValue, 10*time.Second) {
        return  // 获取锁失败
    }
    defer Unlock(lockKey, lockValue)
    
    // 业务逻辑
    // ...
}
```

**版本5：自动续期（✅ 生产可用）**
```go
type RedisLock struct {
    key    string
    value  string
    expire time.Duration
    stopCh chan struct{}
}

func (l *RedisLock) Lock() bool {
    if !redis.SetNX(l.key, l.value, l.expire).Val() {
        return false
    }
    
    // 启动自动续期
    l.stopCh = make(chan struct{})
    go l.renewLoop()
    
    return true
}

func (l *RedisLock) renewLoop() {
    ticker := time.NewTicker(l.expire / 3)  // 每1/3过期时间续期
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            // Lua脚本续期
            script := `
                if redis.call("GET", KEYS[1]) == ARGV[1] then
                    return redis.call("EXPIRE", KEYS[1], ARGV[2])
                else
                    return 0
                end
            `
            redis.Eval(script, []string{l.key}, l.value, int(l.expire.Seconds()))
        case <-l.stopCh:
            return
        }
    }
}

func (l *RedisLock) Unlock() {
    close(l.stopCh)  // 停止续期
    
    script := `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `
    redis.Eval(script, []string{l.key}, l.value)
}
```

### 3.2 Redlock算法（多节点）

**场景**：单Redis节点挂了，锁失效

**方案**：Redlock（向多个独立的Redis节点获取锁）

**步骤**：
1. 向N个Redis节点**依次**获取锁（N=5，奇数）
2. 如果**超过半数**（≥3）成功，且总耗时 < 锁有效期，则获取锁成功
3. 否则释放所有已获取的锁

**Go实现**：使用 `github.com/go-redsync/redsync` 库
```go
import (
    "github.com/go-redsync/redsync/v4"
    "github.com/go-redsync/redsync/v4/redis/goredis/v8"
)

func main() {
    // 创建多个Redis客户端
    client1 := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    client2 := redis.NewClient(&redis.Options{Addr: "localhost:6380"})
    client3 := redis.NewClient(&redis.Options{Addr: "localhost:6381"})
    
    pool := goredis.NewPool(client1, client2, client3)
    rs := redsync.New(pool)
    
    // 获取锁
    mutex := rs.NewMutex("lock:order:1001")
    if err := mutex.Lock(); err != nil {
        panic(err)
    }
    defer mutex.Unlock()
    
    // 业务逻辑
    // ...
}
```

---

## 4. 持久化机制

### 4.1 RDB（快照）

**原理**：定期将内存数据dump到磁盘（二进制文件 dump.rdb）

**触发方式**：
- 手动：`SAVE`（阻塞） 或 `BGSAVE`（fork子进程）
- 自动：配置 `save 900 1`（900秒内至少1次写操作）

**配置**（redis.conf）：
```
save 900 1       # 900秒内至少1次写操作，触发BGSAVE
save 300 10      # 300秒内至少10次写操作
save 60 10000    # 60秒内至少1万次写操作

dbfilename dump.rdb
dir /var/lib/redis
```

**优点**：
- 文件紧凑，恢复速度快
- 对性能影响小（fork子进程）

**缺点**：
- 数据可能丢失（两次快照之间的数据）
- fork子进程消耗内存（COW：写时复制）

---

### 4.2 AOF（追加日志）

**原理**：记录每条**写命令**（append-only file）

**配置**：
```
appendonly yes
appendfilename "appendonly.aof"

# 同步策略
appendfsync always     # 每条命令都fsync（最安全，最慢）
appendfsync everysec   # 每秒fsync（推荐）
appendfsync no         # 由OS决定（最快，可能丢失数据）
```

**AOF重写**（压缩）：
```
# 自动重写
auto-aof-rewrite-percentage 100  # AOF文件增长100%时重写
auto-aof-rewrite-min-size 64mb   # AOF文件至少64MB才重写

# 手动重写
BGREWRITEAOF
```

**优点**：
- 数据安全（最多丢失1秒数据）
- 日志可读，可手动修复

**缺点**：
- 文件大，恢复慢
- 性能略低于RDB

---

### 4.3 混合持久化（推荐）

**原理**：RDB + AOF，兼顾两者优点

**配置**：
```
aof-use-rdb-preamble yes
```

**效果**：
- AOF文件前半部分是RDB快照（恢复快）
- 后半部分是增量AOF日志（数据不丢）

---

## 5. 高可用架构

### 5.1 主从复制

**架构**：
```
[主节点 Master] → [从节点 Slave1]
                 → [从节点 Slave2]
```

**配置**：
```bash
# 从节点配置（redis.conf）
replicaof 103.230.239.204 6379
masterauth <master_password>
```

**同步过程**：
1. **全量同步**：从节点首次连接，主节点生成RDB发送
2. **增量同步**：主节点写命令同步到从节点

**读写分离**：
- 主节点：写
- 从节点：读（可能有主从延迟）

---

### 5.2 哨兵模式（Sentinel）

**作用**：监控主从节点，自动故障转移

**架构**：
```
[哨兵1]   [哨兵2]   [哨兵3]
    ↓         ↓         ↓
[Master] → [Slave1] → [Slave2]
```

**配置**（sentinel.conf）：
```
sentinel monitor mymaster 106.75.144.36 6379 2  # 2个哨兵认为down才故障转移
sentinel down-after-milliseconds mymaster 5000  # 5秒无响应判定down
sentinel failover-timeout mymaster 15000        # 故障转移超时
```

**故障转移流程**：
1. 哨兵检测到Master下线
2. 多数哨兵确认（≥2）
3. 选举一个Slave为新Master
4. 其他Slave指向新Master
5. 通知客户端新Master地址

---

### 5.3 集群模式（Cluster）

**特点**：
- 数据分片（16384个槽）
- 自动故障转移
- 横向扩展

**配置**：
```bash
# redis.conf
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
```

**创建集群**：
```bash
redis-cli --cluster create \
    172.200.156.167:7001 172.200.156.167:7002 172.200.156.167:7003 \
    172.200.156.167:7004 172.200.156.167:7005 172.200.156.167:7006 \
    --cluster-replicas 1
```

**槽分配**：
- 0-5460：节点1
- 5461-10922：节点2
- 10923-16383：节点3

**计算槽**：
```go
func GetSlot(key string) int {
    return int(crc16.Checksum([]byte(key), crc16.IBMTable)) % 16384
}
```

---

### 5.4 Redis过期键删除策略⭐⭐

> Redis如何删除已过期的键？并非到期立即删除。

**两种删除策略并行**：

```
1. 惰性删除（Lazy Expiration）
   - 不主动检查过期键
   - 每次读取key时，先检查是否过期
   - 过期则删除并返回nil
   - 优点：CPU友好，不额外消耗CPU
   - 缺点：内存不友好，过期key不被访问就永远不删

2. 定期删除（Active Expiration）
   - Redis每100ms执行一次定期删除（默认hz=10）
   - 每次从设有过期时间的key中随机取20个
   - 删除其中已过期的key
   - 如果过期key比例 > 25%，重复上述步骤（最多执行25ms）
   - 优点：内存友好，定期清理
   - 缺点：CPU消耗，有清理延迟

两者结合：
- 定期删除兜底清理大量过期key
- 惰性删除确保访问时不返回过期数据
- 仍可能有漏网之鱼 → 内存淘汰策略兜底
```

```bash
# 控制定期删除频率
# hz越大扫描越频繁，CPU消耗越高
hz 10  # 默认每秒10次
```

---

### 5.5 内存淘汰策略底层原理⭐⭐⭐

> 当Redis内存达到maxmemory上限时，如何选择淘汰哪些key？

**8种淘汰策略**：

| 策略 | 范围 | 算法 | 说明 |
|------|------|------|------|
| noeviction | — | — | 禁止写入，返回OOM错误（默认） |
| allkeys-lru | 所有key | 近似LRU | 淘汰最久未使用的key（**推荐**） |
| allkeys-lfu | 所有key | LFU | 淘汰最不常用的key（Redis 4.0+） |
| allkeys-random | 所有key | 随机 | 随机淘汰 |
| volatile-lru | 设有过期时间的key | 近似LRU | 淘汰最久未使用的过期key |
| volatile-lfu | 设有过期时间的key | LFU | 淘汰最不常用的过期key |
| volatile-random | 设有过期时间的key | 随机 | 随机淘汰过期key |
| volatile-ttl | 设有过期时间的key | TTL | 淘汰即将过期的key |

**Redis近似LRU原理（非精确LRU）**：

```
为什么不用精确LRU？
- 精确LRU需要维护一个双向链表（如Java的LinkedHashMap）
- 每次访问都要移动节点，开销大
- 额外占用24字节/key（prev/next指针）

Redis的近似LRU：
- 每个key的对象头存一个24bit的lru时钟（最近访问时间，秒级）
- 淘汰时：随机采样N个key（maxmemory-samples，默认5）
- 比较采样key的lru时钟，淘汰最旧的

采样数量对精度的影响：
- 5个采样（默认）：已经非常接近精确LRU
- 10个采样：几乎等同精确LRU
- 采样数越多越精确，但CPU开销也越大

配置：
maxmemory-samples 5  # 每次随机采样5个key
```

**LFU（Least Frequently Used）原理（Redis 4.0+）**：

```
LRU的问题：
- 新写入的key，如果刚好在淘汰前被访问一次，就不会被淘汰
- 但它可能只是偶尔访问一次（低频），真正的热key反而被淘汰

LFU优化：
- 每个key维护：8bit counter（访问频率计数）+ 16bit ldt（上次衰减时间）
- counter不是简单+1，而是对数递增（避免快速溢出）：
  1/（counter * lfu_log_factor + 1）概率增加counter
- counter会随时间衰减：lfu-decay-time控制衰减速度

配置：
lfu-log-factor 10     # 计数器对数因子（越大增长越慢）
lfu-decay-time 1      # 每分钟衰减一次

factor=10时 counter增长情况：
  100次访问 → counter≈10
  1000次访问 → counter≈18
  10万次访问 → counter≈100
  100万次访问 → counter≈255（上限）
```

---

### 5.6 Redis事务与Lua脚本⭐⭐

#### Redis事务（MULTI/EXEC）

```
MULTI      → 开启事务（命令入队，不立即执行）
命令1      → 入队
命令2      → 入队
EXEC       → 批量执行所有命令

DISCARD    → 放弃事务

注意：Redis事务 ≠ 数据库事务！
- 不支持回滚：命令执行出错，其他命令仍然执行
- 不保证原子性：只保证命令按顺序批量执行，中间不插入其他客户端命令
- 支持CAS乐观锁：WATCH机制
```

```bash
# WATCH实现乐观锁
WATCH balance:1001         # 监视key
MULTI
DECRBY balance:1001 100    # 扣款
INCRBY balance:1002 100    # 加款
EXEC
# 如果WATCH的key在MULTI和EXEC之间被其他客户端修改
# EXEC返回nil，事务失败，客户端需要重试
```

#### Lua脚本（真正的原子性）

```
为什么用Lua脚本？
- Redis事务不保证原子性（失败不回滚）
- Lua脚本在Redis中是原子执行的（执行期间不会被其他命令打断）
- 减少网络往返：多个命令合并为一次调用

示例：原子性地实现"先读后写"
```

```bash
# 限流：固定窗口限流器（Lua实现）
EVAL "
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local current = tonumber(redis.call('GET', key) or '0')
    if current < limit then
        redis.call('INCR', key)
        if current == 0 then
            redis.call('EXPIRE', key, window)
        end
        return 1  -- 允许通过
    else
        return 0  -- 限流
    end
" 1 rate:api:/user 100 60
-- 含义：/user接口每60秒最多100次请求
```

```
Lua脚本注意事项：
1. 脚本应该尽量短小（长脚本会阻塞Redis）
2. 避免在Lua中执行耗时操作（如大量循环）
3. 使用EVALSHA缓存脚本（避免重复传输脚本内容）
4. Lua中的redis.call()出错会终止脚本，redis.pcall()会捕获错误
5. 集群模式下，Lua脚本涉及的所有key必须在同一个slot（使用{hash_tag}）
```

---


## 6. 实战案例

### 案例1：缓存穿透导致DB压力大

**背景**：接口被攻击，查询不存在的用户ID（id=-1、id=999999...）

**现象**：Redis miss，每次请求都打到MySQL，QPS暴涨

**解决**：
```go
// 布隆过滤器
var bf = bloom.NewWithEstimates(10000000, 0.01)

func GetUser(id int64) (*User, error) {
    if !bf.TestString(fmt.Sprintf("%d", id)) {
        return nil, ErrUserNotFound  // 拦截
    }
    // 查缓存 → 查DB
}
```

**效果**：DB压力从5000 QPS降到50 QPS

---

### 案例2：秒杀商品缓存击穿

**背景**：秒杀开始，商品缓存刚好过期，1万并发查询DB

**现象**：DB慢查询暴增，接口超时

**解决**：
```go
var sf singleflight.Group

func GetProduct(id int64) (*Product, error) {
    v, err, _ := sf.Do(fmt.Sprintf("product:%d", id), func() (interface{}, error) {
        // 只有一个请求查DB，其他等待
        return queryDB(id)
    })
    return v.(*Product), err
}
```

**效果**：DB查询从1万次降到1次

---

## 总结

Redis核心考点：
1. **数据结构**：String、List、Hash、Set、ZSet应用
2. **缓存问题**：穿透、击穿、雪崩及解决方案
3. **分布式锁**：SETNX + Lua + 自动续期
4. **持久化**：RDB、AOF、混合持久化
5. **高可用**：主从、哨兵、集群


---

## 7. Redis 7.0+ 新特性

### 7.1 Redis Functions（替代 Lua Script 的新方案）

#### 为什么需要 Functions？

Lua Script 在生产环境中存在几个核心痛点：

| 痛点 | 说明 |
|------|------|
| **无法持久化** | 脚本存储在内存中，重启后丢失，需要客户端重新加载 |
| **无命名机制** | 通过 SHA1 调用，不直观，难以管理 |
| **无版本管理** | 无法平滑升级脚本逻辑，只能全量替换 |
| **无库/模块概念** | 脚本之间不能复用代码，每个脚本都是独立的 |

Redis Functions（7.0+）解决了这些问题：脚本持久化到 RDB/AOF、有命名和库的概念、支持集群间复制。

#### Function vs Lua Script 对比

| 特性 | Lua Script (EVAL/EVALSHA) | Functions (FUNCTION/FCALL) |
|------|--------------------------|---------------------------|
| **持久化** | ❌ 重启丢失 | ✅ 持久化到 RDB/AOF |
| **命名** | SHA1 哈希值 | 自定义函数名 |
| **复制** | ❌ 需手动同步 | ✅ 自动复制到从节点 |
| **库管理** | ❌ 无 | ✅ Library 概念，支持多函数 |
| **升级** | 全量替换 | REPLACE 标志平滑升级 |
| **调用方式** | `EVALSHA <sha1> numkeys ...` | `FCALL <name> numkeys ...` |
| **集群友好** | 需自行管理 | 自动随 slot 迁移 |

#### 使用示例

```bash
# 1. 定义一个 Library（包含多个函数）
FUNCTION LOAD "#!lua name=mylib
-- 限流函数：固定窗口
redis.register_function('rate_limit', function(keys, args)
    local key = keys[1]
    local limit = tonumber(args[1])
    local window = tonumber(args[2])
    local current = tonumber(redis.call('GET', key) or '0')
    if current < limit then
        redis.call('INCR', key)
        if current == 0 then
            redis.call('EXPIRE', key, window)
        end
        return 1
    end
    return 0
end)

-- 原子性扣库存函数
redis.register_function('deduct_stock', function(keys, args)
    local key = keys[1]
    local amount = tonumber(args[1])
    local stock = tonumber(redis.call('GET', key) or '0')
    if stock >= amount then
        redis.call('DECRBY', key, amount)
        return stock - amount
    end
    return -1  -- 库存不足
end)
"

# 2. 调用函数
FCALL rate_limit 1 rate:api:/order 100 60
# 含义：对 rate:api:/order 这个 key 做限流，60秒内最多100次

FCALL deduct_stock 1 stock:sku:1001 2
# 含义：对 sku:1001 扣减2个库存

# 3. 管理函数
FUNCTION LIST                    # 列出所有 Library
FUNCTION DUMP                    # 导出所有函数（二进制）
FUNCTION RESTORE <dump>          # 恢复函数
FUNCTION DELETE mylib            # 删除 Library
FUNCTION LOAD REPLACE "#!lua name=mylib ..."  # 升级替换
```

#### Library 代码组织建议

```
# 按业务域组织 Library
- orderlib：订单相关原子操作（扣库存、创建订单号、限流）
- cachelib：缓存治理（带版本的写入、条件刷新）
- locklib：分布式锁操作（加锁、续期、释放）

# 生产建议
1. 函数命名：{library}_{function}，如 order_deduct_stock
2. 在 CI/CD 中管理函数代码，部署时 FUNCTION LOAD REPLACE
3. 测试环境先验证，避免线上直接修改
```

---

### 7.2 Sharded Pub/Sub

#### 传统 Pub/Sub 的问题

```
传统 Pub/Sub（Redis Cluster 模式下）：

Client → PUBLISH channel:order msg
           ↓
[Node1] → 广播 → [Node2]
                → [Node3]
                → [Node4]
                → [Node5]
                → [Node6]

问题：
- 消息会广播到集群中 所有节点（不管该节点有没有订阅者）
- N个节点的集群，每条消息产生 N 次网络传输
- 集群越大，带宽浪费越严重
- 不适合大规模集群的高频消息场景
```

#### Sharded Pub/Sub 原理（Redis 7.0+）

```
Sharded Pub/Sub：消息只路由到负责该 channel slot 的节点

Client → SPUBLISH channel:order msg
           ↓
[Node1] → 只发送到负责 channel:order 所在 slot 的节点
           ↓
[Node3]（负责该 slot）→ 通知本地订阅者

优势：
- 消息只在负责该 slot 的主从节点间传播
- 带宽消耗与集群规模无关
- channel 也可使用 hash tag 控制路由：{order}:channel1
```

#### 使用示例

```bash
# Sharded Pub/Sub 命令（带 S 前缀）
SSUBSCRIBE channel:order:1001    # 订阅（只在对应 slot 节点）
SPUBLISH channel:order:1001 "paid"  # 发布（只路由到对应节点）
SUNSUBSCRIBE channel:order:1001  # 取消订阅

# 适用场景
# 1. 订单状态变更通知（按订单ID分片）
# 2. 用户在线状态变更（按用户ID分片）
# 3. 集群模式下的实时通知（替代全量广播）
```

#### Sharded Pub/Sub vs Stream

| 特性 | Sharded Pub/Sub | Stream |
|------|----------------|--------|
| **消息持久化** | ❌ 不持久化 | ✅ 持久化存储 |
| **消费组** | ❌ 不支持 | ✅ 支持消费组 |
| **消息回溯** | ❌ 不支持 | ✅ 支持历史消息 |
| **集群友好** | ✅ 分片路由 | ✅ 按 key 分片 |
| **延迟** | 极低（推模式） | 低（拉模式/阻塞读） |
| **适用场景** | 实时通知、状态广播 | 消息队列、事件溯源 |

选型建议：需要"发完即忘"的实时通知用 Sharded Pub/Sub；需要可靠消费、消息回溯用 Stream。

---

### 7.3 Redis Stack 简介

Redis Stack 是 Redis 官方推出的扩展模块集合，将多个数据能力集成到 Redis 中：

| 模块 | 功能 | 典型场景 |
|------|------|----------|
| **RediSearch** | 全文搜索 + 二级索引 + 聚合 | 商品搜索、日志检索 |
| **RedisJSON** | 原生 JSON 文档存储 | 复杂对象存储、部分更新 |
| **RedisGraph** | 图数据库（Cypher 查询） | 社交关系、推荐（已停止维护） |
| **RedisTimeSeries** | 时序数据 | 监控指标、IoT 数据 |
| **RedisBloom** | 概率数据结构（布隆/CuckooFilter/TopK） | 去重、频率统计 |

#### RediSearch：轻量级全文搜索

```bash
# 创建索引
FT.CREATE idx:product ON HASH PREFIX 1 product: SCHEMA
    name TEXT WEIGHT 5.0
    description TEXT
    price NUMERIC SORTABLE
    category TAG

# 搜索
FT.SEARCH idx:product "@name:手机 @price:[1000 5000]" SORTBY price ASC LIMIT 0 10

# 聚合（按类目统计平均价格）
FT.AGGREGATE idx:product "*"
    GROUPBY 1 @category
    REDUCE AVG 1 @price AS avg_price
    SORTBY 2 @avg_price DESC
```

**RediSearch vs Elasticsearch**：
- RediSearch 适合：数据量 < 千万级、对延迟敏感（亚毫秒）、已有 Redis 基础设施
- ES 适合：数据量大（亿级）、需要复杂分词/相关性算法、全文搜索是核心能力

#### RedisJSON：原生 JSON 支持

```bash
# 存储 JSON 文档
JSON.SET user:1001 $ '{"name":"张三","age":25,"address":{"city":"北京","district":"朝阳"},"tags":["vip","active"]}'

# JSONPath 查询
JSON.GET user:1001 $.name                    # "张三"
JSON.GET user:1001 $.address.city            # "北京"
JSON.GET user:1001 $.tags[0]                 # "vip"

# 部分更新（无需读取整个文档）
JSON.SET user:1001 $.age 26                  # 只更新 age
JSON.ARRAPPEND user:1001 $.tags '"new"'      # 追加数组元素
JSON.NUMINCRBY user:1001 $.age 1             # 数字自增

# 对比 String 存 JSON：
# String：必须整体读取 → 修改 → 整体写回（非原子、带宽浪费）
# RedisJSON：支持路径级原子读写，节省带宽
```

#### 选型建议

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 简单 KV 缓存 | 原生 Redis | 无需额外模块 |
| 复杂对象频繁局部更新 | RedisJSON | 避免整体序列化开销 |
| 轻量搜索（< 千万数据） | RediSearch | 延迟低、运维简单 |
| 重度搜索（亿级 + 复杂分析） | Elasticsearch | 生态成熟、扩展能力强 |
| 时序监控数据 | RedisTimeSeries | 内置降采样、聚合 |
| 大规模时序（长期存储） | InfluxDB/Prometheus | 压缩率高、长期存储成本低 |
| 概率去重/频率统计 | RedisBloom | 原生集成、API 简洁 |

---

### 7.4 其他 7.0 重要改进

#### Multi-part AOF（AOF 文件管理革新）

```
7.0 之前：AOF 是单文件，重写时产生临时文件，完成后原子替换
问题：大 AOF 重写期间磁盘空间翻倍、重写失败可能丢数据

7.0+：AOF 变为多文件管理（manifest 清单 + 多段文件）
appendonlydir/
├── appendonly.aof.1.base.rdb    # 基础 RDB 快照部分
├── appendonly.aof.1.incr.aof    # 增量 AOF（当前写入）
├── appendonly.aof.2.incr.aof    # 增量 AOF（历史段）
└── appendonly.aof.manifest      # 清单文件

优势：
- 重写不再需要临时文件，原地管理
- 历史段可渐进清理，磁盘空间更可控
- 重写失败不影响已有数据
```

#### Client Eviction（按客户端内存驱逐）

```bash
# 问题：某个客户端大量 pipeline/subscribe 占用巨大输出缓冲区，撑爆内存
# 7.0+ 方案：按客户端维度统计内存，超限驱逐

maxmemory-clients 1gb           # 所有客户端总共最多占 1GB
# 或按百分比：
maxmemory-clients 5%            # 客户端内存最多占 maxmemory 的 5%

# 超限时，Redis 驱逐占用内存最多的客户端连接
# 某些客户端可豁免：CLIENT NO-EVICT ON
```

#### ACL 改进

```bash
# 7.0+ 支持基于 Selector 的细粒度权限
ACL SETUSER app1 ON >password
    (+GET +SET ~cache:*)           # selector 1：允许读写 cache:* 前缀
    (+SUBSCRIBE ~channel:order:*)  # selector 2：允许订阅 order 频道

# 多个 selector 之间是 OR 关系，实现更灵活的权限组合
# 6.x 只支持单一权限规则，7.0 可按不同 key 模式给不同权限
```

#### listpack 完全替代 ziplist

```
7.0 版本中 listpack 完全替代了 ziplist 作为底层紧凑编码：

ziplist 的核心问题：连锁更新（prevlen 变化引发级联扩展）
listpack 解决方案：每个 entry 只记录自身长度，不记录前一个 entry 长度

listpack entry 结构：
[encoding][data][backlen]
- encoding：当前元素的编码类型和长度
- data：实际数据
- backlen：当前 entry 的总长度（用于反向遍历）

优势：
- 彻底消除连锁更新问题
- 反向遍历通过 backlen 实现（类似 ziplist 的 prevlen 功能）
- 对外 API 兼容，用户无感知升级
- Hash/ZSet/List 的紧凑编码全部切换到 listpack

配置项也随之更名：
- hash-max-ziplist-entries → hash-max-listpack-entries
- zset-max-ziplist-entries → zset-max-listpack-entries
```

---

## 8. 面试题自查

### 缓存治理补充

- 缓存策略不是“上 Redis 就结束”，而是要同时定义写路径一致性、失效策略、热点治理和降级预案。
- 热 key 和大 key 要长期治理，不能等到延迟抖动或实例迁移失败时才排查。
- 当业务一致性要求很高时，缓存层要允许“主动降级只读或旁路”，优先保证正确性。

### 初级高频速刷

**1. 为什么 Redis 常被放在 MySQL 前面做缓存，而不是直接替代 MySQL？**

因为 Redis 擅长低延迟读写和热点数据承接，但不擅长复杂事务、持久化一致性和关系查询。它通常是数据库前面的加速层，而不是业务主存储的完全替代品。

**2. String 和 Hash 都能缓存对象，什么时候更适合用 Hash？**

如果对象字段较多、可能只更新局部字段，Hash 更合适；如果只是整体读写一个 JSON 串，String 更简单。面试里这个问题常用来判断你是否真的做过缓存建模，而不是只会背数据结构名称。

**3. 为什么缓存除了设置过期时间，还要设置容量上限和淘汰策略？**

因为只设过期时间不能防止瞬时内存打满，热点和脏流量同样可能把实例撑爆。容量上限和淘汰策略是 Redis 作为缓存层能够长期稳定运行的基本前提。

**4. 缓存命中率为什么重要？只看 Redis QPS 为什么不够？**

高 QPS 不代表缓存设计成功，真正关键的是多少请求被缓存挡住了。如果命中率低，说明大量流量还是穿到了数据库，Redis 只是看起来很忙，实际上没有真正减压。

**5. 为什么删除缓存通常比更新缓存更常见？**

因为更新缓存需要保证新值构造逻辑和数据库完全一致，复杂度更高；删缓存则简单且更不容易写错。Cache Aside 模式里常见的做法就是写库后删缓存，而不是同步改两份数据。

### 中级高频进阶

**6. Redis 的 Pipeline 和事务有什么区别？**

Pipeline 主要解决性能问题，通过批量发送减少 RTT；事务主要解决命令打包执行的问题。Pipeline 不保证原子性，事务也不等于关系型数据库事务，这个区分在面试里非常高频。

**7. 热 key 和大 key 分别是什么？为什么都危险？**

热 key 是访问量极端集中，容易形成单点热点；大 key 是单条数据过大，读取、删除、迁移都可能拖垮 Redis。前者主要是流量分布问题，后者主要是数据形态问题，但都会在高峰时把缓存层打出故障。

**8. 为什么删除大 key 可能导致阻塞？怎么处理更稳妥？**

因为单线程执行模型下，删除超大对象可能在主线程里做大量释放工作，直接拉高延迟。更稳妥的做法是拆分大 key、使用 `UNLINK` 异步回收、在低峰期分批清理，并从源头避免生成大 key。

**9. 什么时候不应该继续把 Redis 当消息队列硬扛，而应该换成专业 MQ？**

当你需要可回溯消费、复杂重试、死信治理、消费组再均衡、长时间堆积承载时，Redis 的简易队列模型就会越来越吃力。此时更合理的方案是让 Redis 回到缓存/高频 KV 场景，把异步消息语义交给 Kafka 或 RabbitMQ。

**10. `appendfsync always/everysec/no` 怎么选？为什么不能只谈性能？**

`always` 数据最稳但写放大大，`everysec` 是最常见折中，`no` 依赖系统刷盘、性能高但数据风险大。面试里更成熟的回答是结合业务容忍度说“最多可接受丢多少数据”，而不是只说哪个更快。

### 深入问答

#### Q1: Redis有哪些数据结构？各自的底层实现和典型应用场景是什么？

**答**：5种基础类型：String（SDS，缓存/计数器/分布式锁）、List（quicklist=ziplist+双向链表，消息队列/时间线）、Hash（ziplist/hashtable，对象缓存）、Set（intset/hashtable，去重/共同好友）、ZSet（ziplist/skiplist+hashtable，排行榜/延时队列）。3种高级类型：Bitmap（签到/在线状态）、HyperLogLog（UV统计，有误差）、Geo（地理位置/附近的人）。

#### Q2: SDS相比C字符串有哪些优势？预分配策略是什么？

**答**：SDS优势：①O(1)获取长度（存len字段）②二进制安全（不依赖\0判断结束）③自动扩容防溢出④预分配+惰性释放减少内存分配次数。预分配策略：扩容时如果len<1MB则分配2*len空间，如果len≥1MB则额外分配1MB。惰性释放：截断时不立即释放多余空间，保留在free字段供后续使用。

#### Q3: ZSet为什么用skiplist而不是红黑树？skiplist的层数如何决定？

**答**：skiplist优于红黑树：①范围查询O(logn+k)，找到起点后沿底层链表顺序遍历②实现简单，无需复杂旋转③内存局部性好，缓存友好。层数由随机算法决定：每个节点以P=0.25概率提升到上一层，最大32层。结果分布：75%为1层，18.75%为2层，4.69%为3层。平均查找复杂度O(logn)。

#### Q4: ziplist的连锁更新是什么？Redis如何缓解这个问题？

**答**：ziplist中每个entry存储前一个entry的长度（prevlen），用1或5字节表示（分界线254字节）。当插入/修改一个元素使长度从<254变为≥254时，后续entry的prevlen从1字节扩展到5字节，可能引发级联扩展，最坏O(n²)。缓解方案：限制ziplist大小（默认8KB或64个元素），超过阈值转为quicklist/hashtable/skiplist。

#### Q5: 缓存穿透、击穿、雪崩分别是什么？各自的最佳解决方案？

**答**：**穿透**：查询不存在的数据，缓存和DB都miss，最佳方案是布隆过滤器拦截。**击穿**：单个热点key过期，大量并发打DB，最佳方案是singleflight互斥锁保证只有一个请求查DB。**雪崩**：大量key同时过期，最佳方案是过期时间加随机偏移 + 多级缓存 + 限流降级。

#### Q6: 布隆过滤器的原理是什么？有什么优缺点？

**答**：布隆过滤器由一个bit数组+多个hash函数组成。添加元素时，用K个hash函数计算K个位置置为1。查询时，K个位置都为1则"可能存在"，有任一个为0则"一定不存在"。优点：内存极小（1000万元素+1%误判率约12MB），查询O(K)常数时间。缺点：有假阳性（可能误判存在），不支持删除（可用Counting Bloom Filter）。

#### Q7: Redis分布式锁从简单到生产级，有哪些演进步骤？各解决什么问题？

**答**：V1 SETNX无过期时间→进程崩溃死锁。V2 SETNX+Expire分开→非原子，中间崩溃死锁。V3 SETNX带过期时间→可能误删别人的锁。V4 SETNX+唯一value+Lua原子删除→基本可用，但业务超时锁过期。V5 自动续期（看门狗）→生产级，每1/3过期时间续期一次。多节点用Redlock：向5个独立节点获取锁，半数以上成功才算获取成功。

#### Q8: RDB和AOF持久化各自的原理、优缺点是什么？生产环境推荐哪种？

**答**：RDB：定期fork子进程生成内存快照（二进制dump.rdb），恢复快但可能丢失两次快照间数据，fork时COW占内存。AOF：追加写命令日志，everysec模式最多丢1秒数据，文件大恢复慢，需定期BGREWRITEAOF重写压缩。生产推荐**混合持久化**（Redis 4.0+）：AOF文件前半部分是RDB快照（恢复快），后半部分是增量AOF（数据不丢）。

#### Q9: Redis主从复制的同步过程是怎样的？全量同步和增量同步的区别？

**答**：从节点首次连接主节点触发**全量同步**：主节点BGSAVE生成RDB发送给从节点，期间新命令写入复制缓冲区，RDB加载完后发送缓冲区数据。后续正常通信为**增量同步**：主节点每执行写命令都同步给从节点。断线重连时通过repl_backlog_buffer（复制积压缓冲区）和offset做增量同步，如果offset已不在缓冲区则重新全量同步。

#### Q10: 哨兵模式如何实现自动故障转移？选举新主节点的依据是什么？

**答**：哨兵定期Ping主节点，超时（down-after-milliseconds）判定主观下线；超过quorum个哨兵确认则客观下线。选举新Master依据：①排除下线/断连的从节点②优先级高的（slave-priority小的）③复制偏移量大的（数据最新）④Run ID最小的。选举后通知其他从节点replicaof新主节点，通知客户端新主节点地址。

#### Q11: Redis Cluster的数据分片原理是什么？如何处理key的路由？

**答**：Cluster将keyspace划分为16384个slot（哈希槽），每个节点负责部分slot。key路由：对key做CRC16哈希后对16384取模得到slot号，根据slot→节点映射表路由到对应节点。客户端缓存映射表，如果访问了错误节点，节点返回MOVED/ASK重定向。支持hash tag：{user}:1001和{user}:1002会路由到同一个slot。

#### Q12: Redis为什么单线程还这么快？6.0多线程优化了什么？

**答**：快的原因：①纯内存操作（比磁盘快10万倍）②IO多路复用epoll（单线程管理数万连接，O(1)就绪事件通知）③高效数据结构（SDS/ziplist/skiplist等内存优化）④无锁无上下文切换开销。6.0多线程：只用于网络I/O的读写（socket read/write），命令执行仍是单线程保证无锁安全，适用于大value场景网络I/O成为瓶颈时。

#### Q13: epoll为什么比select/poll快？边缘触发（ET）和水平触发（LT）的区别？

**答**：select/poll每次调用都要拷贝所有fd到内核并O(n)遍历检查就绪状态。epoll：fd一次性注册到内核红黑树，就绪时通过回调加入就绪链表，epoll_wait只返回就绪列表，O(1)。LT模式：只要fd可读/可写就会通知（没处理完下次还通知）。ET模式：只在状态变化时通知一次（必须一次读完所有数据，否则不再通知），效率更高但编程复杂。

#### Q14: 如何保证缓存和数据库的一致性？延迟双删的原理是什么？

**答**：推荐策略：**先更新DB，再删缓存**（Cache Aside Pattern）。延迟双删：更新DB后删缓存，再延迟500ms后再删一次。原因：并发场景下，线程A删缓存→线程B读到旧DB写入缓存→线程A更新DB，此时缓存有旧数据。延迟双删的第二次删除清理这种竞态写入的旧数据。更可靠的方案：订阅MySQL binlog（Canal）异步删除缓存。

#### Q15: Redis过期键是如何删除的？为什么不在过期时立即删除？

**答**：立即删除需要为每个key设置定时器，大量定时器CPU开销大。Redis采用惰性删除+定期删除：惰性删除在每次访问key时检查是否过期，过期则删除；定期删除每100ms随机采样20个有过期时间的key，删除其中过期的，如果过期比例>25%则继续。两者结合平衡CPU和内存开销，漏网之鱼由内存淘汰策略兜底。

#### Q16: Redis的LRU淘汰是精确LRU吗？近似LRU如何实现的？

**答**：不是精确LRU。精确LRU需维护双向链表，每次访问移动节点，额外24字节/key开销大。Redis近似LRU：每个key对象头存24bit的lru时钟（秒级），淘汰时随机采样N个key（maxmemory-samples默认5），淘汰lru时钟最旧的。5个采样已非常接近精确LRU效果，10个几乎等同。Redis 4.0+引入LFU，用8bit对数计数器+时间衰减追踪访问频率，避免LRU的偶尔访问问题。

#### Q17: Redis事务和Lua脚本有什么区别？各自适用什么场景？

**答**：Redis事务（MULTI/EXEC）：命令批量入队后一次执行，保证不被其他客户端命令插入，但**不支持回滚**（某条命令出错其他命令仍执行），不是真正的原子性。Lua脚本：在Redis中原子执行，执行期间不会被其他命令打断，支持条件判断和复杂逻辑。事务适合简单的批量操作+WATCH乐观锁；Lua适合需要"先读后写"原子性的场景（如分布式锁释放、限流器）。

#### Q18: Redis渐进式rehash是什么？为什么需要渐进式？

**答**：Hash底层hashtable扩容时需要rehash（重新映射所有key到新桶）。一次性rehash大hash（百万key）会阻塞Redis数秒。渐进式rehash：维护ht[0]和ht[1]两个哈希表，开始rehash后每次增删改查操作顺便迁移ht[0]中一个桶到ht[1]，查询时先查ht[0]再查ht[1]，迁移完后交换两个表。将O(n)的大操作分摊到每次命令中。

#### Q19: Redis Cluster模式下有哪些限制？如何解决跨slot操作？

**答**：限制：①不支持跨slot的多key操作（MGET/MSET/事务/Lua脚本涉及的key必须在同一slot）②不支持SELECT切换数据库（只有db0）③批量操作需要客户端自行分组。解决跨slot：使用hash tag `{tag}:key1` 和 `{tag}:key2`，Redis只对{}内部分做hash，保证相同tag的key在同一slot。

#### Q20: 线上Redis出现大key该如何排查和处理？大key有什么危害？

**答**：危害：①单次读写大key阻塞Redis（单线程）②大key过期或删除时阻塞（DEL同步删除）③主从复制/迁移时数据倾斜④内存碎片化。排查：`redis-cli --bigkeys`扫描、`MEMORY USAGE key`查看单个key大小、`redis-rdb-tools`离线分析RDB。处理：①拆分大key（大hash按前缀拆分为多个小hash）②使用UNLINK异步删除（替代DEL）③设置`lazyfree-lazy-expire/eviction/server-del yes`开启懒删除。

#### Q21: 热 key 治理为什么不能只靠“加机器”？给一个可执行的治理方案。

**答**：热 key 的核心问题是流量分布极不均匀，单 key 成为系统瓶颈，简单扩容只能缓解总体容量，无法消除单点热点。可执行方案一般是四步：①识别热点（按 key 维度统计 QPS、带宽、RT）②分层缓存（本地缓存 + Redis）③读扩散（热点 key 多副本或逻辑分片）④降级限流（热点高峰时返回兜底或旧值）。治理目标不是“热点永远不存在”，而是热点出现时系统仍可控。

#### Q22: Redis 分布式锁在哪些场景不建议使用？

**答**：当业务要求强事务一致、锁持有时间长、跨多资源原子提交、或锁失败会直接导致资损时，不建议只靠 Redis 锁。原因是 Redis 锁更偏“高性能互斥协调”，不是完整事务系统，网络分区、时钟漂移、进程暂停都可能带来边界风险。更稳妥做法是把关键一致性放在数据库约束、事务和状态机里，Redis 锁只做并发削峰和协作优化。

#### Q23: 缓存一致性要求突然提高时，如何把系统从“激进缓存”切到“稳态正确”？

**答**：常见策略是按风险分级降级：先缩短 TTL、收紧热点键更新，再对关键读链路切旁路读库或强制读主；写路径保持“先写库后删缓存”，并用 binlog 订阅做二次校正。必要时临时关闭部分缓存命中，牺牲延迟换正确性。核心原则是把一致性策略做成可开关的运行时能力，而不是写死在代码里。

#### Q24: 哨兵模式和 Cluster 模式怎么选？

**答**：哨兵模式适合容量可控、以高可用为主、单主多从即可满足的场景；Cluster 适合需要数据分片和水平扩展的场景。前者运维和客户端复杂度更低，后者吞吐和容量上限更高但跨 slot、多 key 操作和迁移治理更复杂。面试里不要只说“Cluster 更高级”，关键是按数据规模、操作模式和运维能力做取舍。

#### Q25: 缓存双删在高并发下为什么仍可能不一致？你会怎么补偿？

**答**：双删（写库前删缓存 + 延迟再删）只能降低不一致概率，不能消除并发窗口。典型风险是：读请求在两次删之间回源到旧数据并回填缓存，导致短期脏值驻留。

补偿思路：
1. 对关键键引入 binlog/CDC 二次校正，发现更新事件后再次失效缓存。
2. 缩短关键键 TTL，并给热点键增加版本号校验。
3. 写路径增加互斥或请求合并，减少并发回填窗口。

真正高一致场景要结合“失效 + 校正 + 版本控制”，而不是只押注双删。

#### Q26: Redis 热 key 治理里，“本地缓存 + 分片 + 限流”怎么组合才不会引入新雪崩？

**答**：组合要按优先级分层：
1. **本地缓存**抗抖动：短 TTL + 抖动过期，避免同一时刻集体失效。
2. **逻辑分片**分散热点：将单 key 热点拆到多 key，但对外保持统一聚合语义。
3. **入口限流**兜底：当缓存失效或回源异常时，快速保护后端。

关键是三者联动：本地缓存过期策略若不加随机抖动，会把流量同时打到 Redis/DB，形成新雪崩。

### 开放式设计题

**D1：设计一个支持每秒100万次读取的分布式缓存系统，你会怎么架构？**

**参考思路**：
- 多级缓存：本地缓存（Ristretto/BigCache）→ Redis Cluster → DB，99%命中在本地层
- Redis Cluster：至少3主3从、读写分离（读从节点）、Pipeline批量操作
- 热点Key治理：本地缓存兜底、Key加随机后缀分散到多Slot、Proxy层做请求合并
- 缓存一致性：写DB后删缓存+延迟双删、Binlog异步更新缓存、最终一致性SLA
- 容灾：Redis全挂时降级到DB限流查询、本地缓存自动延长TTL、熔断保护DB
- 监控：命中率/延迟/内存/连接数Dashboard、命中率<95%自动告警

**D2：Redis Cluster中一个节点内存突然暴涨导致OOM，如何快速定位和恢复？**

**参考思路**：
- 紧急恢复：该节点自动failover到从节点（如果有Replica）、手动CLUSTER FAILOVER
- 定位大Key：redis-cli --bigkeys扫描、MEMORY USAGE采样、RDB离线分析
- 常见原因：业务误写超大Hash/Set、Stream消费滞后堆积、未设置maxmemory-policy
- 治理：大Key拆分（Hash分桶）、TTL治理（所有Key必须设过期）、内存告警（>70%告警>85%禁写）
