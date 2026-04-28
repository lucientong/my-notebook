# SRE实践全景

---

## 📑 目录

### 第一部分：可观测性基础
1. [可观测性三大支柱](#1-可观测性三大支柱)
2. [监控指标体系（Prometheus）](#2-监控指标体系prometheus)
3. [日志管理（ELK）](#3-日志管理elk)
4. [分布式链路追踪（Jaeger）](#4-分布式链路追踪jaeger)
5. [告警体系（Alertmanager）](#5-告警体系alertmanager)
6. [故障排查SOP](#6-故障排查sop)

### 第二部分：SRE高级实践
7. [容灾架构设计](#7-容灾架构设计)
8. [故障管理与自愈](#8-故障管理与自愈)
9. [监控体系建设](#9-监控体系建设)
10. [性能优化实战](#10-性能优化实战)
11. [成本优化](#11-成本优化)
12. [稳定性治理](#12-稳定性治理)
13. [Chaos Engineering](#13-混沌工程)
14. [容量规划与成本优化](#14-容量规划与成本优化)
15. [面试题自查](#15-面试题自查)

---

# 第一部分：可观测性基础

## 1. 可观测性三大支柱

### 1.1 概述

| 支柱 | 作用 | 工具 |
|------|------|------|
| **Metrics（指标）** | 系统健康状况（QPS、延迟、错误率） | Prometheus、InfluxDB |
| **Logs（日志）** | 详细事件记录（错误日志、业务日志） | ELK、Loki |
| **Traces（链路）** | 请求调用链（分布式系统调试） | Jaeger、Zipkin、SkyWalking |

**为什么需要可观测性？**
- **发现问题**：监控发现异常（QPS下降、错误率上升）
- **定位问题**：日志+链路定位根因
- **解决问题**：快速止血，恢复服务
- **避免问题**：复盘优化，预防再次发生

### 1.2 监控vs可观测性

| 维度 | 监控（Monitoring） | 可观测性（Observability） |
|------|-------------------|--------------------------|
| **目标** | 已知问题检测 | 未知问题探索 |
| **数据** | 预定义指标 | 日志+指标+链路 |
| **方式** | 仪表盘+告警 | 查询+分析+关联 |
| **场景** | "系统是否正常？" | "为什么不正常？" |

---

## 2. 监控指标体系（Prometheus）

### 2.1 四大黄金指标

**Google SRE推荐的核心指标**：

| 指标 | 说明 | 正常值 | 告警阈值 |
|------|------|--------|----------|
| **Latency（延迟）** | 请求响应时间 | P99 < 100ms | P99 > 500ms |
| **Traffic（流量）** | 请求QPS | 1000 QPS | 偏离均值30% |
| **Errors（错误率）** | 5xx错误占比 | < 0.1% | > 1% |
| **Saturation（饱和度）** | 资源使用率（CPU/内存/磁盘） | < 70% | > 90% |

### 2.2 Prometheus指标类型

**Counter（计数器）**：累计值（请求数、错误数）
```go
import "github.com/prometheus/client_golang/prometheus"

var (
    requestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )
)

func init() {
    prometheus.MustRegister(requestsTotal)
}

// 使用
func HandleRequest(method, path string, statusCode int) {
    requestsTotal.WithLabelValues(method, path, fmt.Sprint(statusCode)).Inc()
}
```

**Gauge（仪表盘）**：瞬时值（CPU使用率、内存使用量）
```go
var (
    goroutineCount = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "goroutine_count",
            Help: "Number of goroutines",
        },
    )
)

func RecordGoroutineCount() {
    goroutineCount.Set(float64(runtime.NumGoroutine()))
}
```

**Histogram（直方图）**：分布统计（延迟分布）
```go
var (
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1, 2, 5},
        },
        []string{"method", "path"},
    )
)

func HandleRequest(method, path string) {
    start := time.Now()
    defer func() {
        duration := time.Since(start).Seconds()
        requestDuration.WithLabelValues(method, path).Observe(duration)
    }()
    // 处理请求
}
```

**Summary（摘要）**：分位数（P50、P90、P99）
```go
var (
    requestDurationSummary = prometheus.NewSummaryVec(
        prometheus.SummaryOpts{
            Name:       "http_request_duration_summary",
            Help:       "HTTP request duration summary",
            Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
        },
        []string{"method", "path"},
    )
)
```

### 2.3 PromQL查询

**常用查询**：
```promql
# QPS（每秒请求数）
rate(http_requests_total[1m])

# 错误率
sum(rate(http_requests_total{status=~"5.."}[1m])) /
sum(rate(http_requests_total[1m]))

# P99延迟
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# CPU使用率
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

### 2.4 指标方法论：RED 与 USE

**RED 方法**：更适合面向请求的服务

| 指标 | 含义 | 常见实现 |
|------|------|----------|
| **Rate** | 请求速率 | `rate(http_requests_total[1m])` |
| **Errors** | 错误比例 | `5xx / 总请求` |
| **Duration** | 请求耗时 | P50/P95/P99 延迟 |

**USE 方法**：更适合基础设施和资源层

| 指标 | 含义 | 典型对象 |
|------|------|----------|
| **Utilization** | 资源利用率 | CPU 使用率、磁盘带宽利用率 |
| **Saturation** | 饱和度/排队情况 | Run Queue、磁盘队列长度、连接池等待 |
| **Errors** | 资源级错误 | 丢包、磁盘错误、OOM、连接失败 |

**怎么用**：
- 用户入口、网关、API 服务优先看 RED
- MySQL、Redis、Kafka、K8s 节点优先看 USE
- 真正排障时要把两者串起来：先看 RED 判断用户是否受损，再用 USE 找到瓶颈资源

**一个常见误区**：只盯 CPU/内存不看排队长度。很多服务 CPU 只有 40%，但线程池、连接池、磁盘队列已经饱和，用户照样超时。

---

## 3. 日志管理（ELK）

### 3.1 结构化日志

**JSON格式（推荐）**：
```go
import "github.com/sirupsen/logrus"

log := logrus.New()
log.SetFormatter(&logrus.JSONFormatter{})

log.WithFields(logrus.Fields{
    "user_id":    1001,
    "request_id": "abc-123",
    "method":     "GET",
    "path":       "/api/users",
    "status":     200,
    "duration_ms": 50,
}).Info("request completed")

// 输出：
// {"duration_ms":50,"level":"info","method":"GET","msg":"request completed",
//  "path":"/api/users","request_id":"abc-123","status":200,...}
```

### 3.2 ELK架构

```
[应用] → [Filebeat] → [Kafka] → [Logstash] → [Elasticsearch] → [Kibana]
```

**Filebeat配置**：
```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.log
    fields:
      service: myapp
      env: production

output.kafka:
  hosts: ["localhost:9092"]
  topic: "logs"
```

### 3.3 日志查询（Kibana）

```
# 查询错误日志
level:ERROR

# 查询慢请求
duration_ms:>1000 AND @timestamp:[now-1h TO now]

# 查询请求链路
request_id:"abc-123"
```

---

## 4. 分布式链路追踪（Jaeger）

### 4.1 核心概念

| 概念 | 说明 |
|------|------|
| **Trace** | 一次完整的请求链路 |
| **Span** | 请求中的一个操作（RPC调用、DB查询） |
| **TraceID** | 全局唯一追踪ID |
| **SpanID** | Span的唯一ID |

**示例**：
```
Trace: abc-123
├─ Span1: API Gateway (100ms)
   ├─ Span2: User Service (20ms)
   └─ Span3: Order Service (70ms)
      ├─ Span4: DB Query (40ms)
      └─ Span5: Payment Service (20ms)
```

### 4.2 OpenTelemetry实现

```go
import (
    "go.opentelemetry.io/otel"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

func HandleRequest(ctx context.Context, userID int64) {
    tracer := otel.Tracer("my-service")
    
    // 创建Span
    ctx, span := tracer.Start(ctx, "HandleRequest")
    defer span.End()
    
    // 添加属性
    span.SetAttributes(
        attribute.Int64("user_id", userID),
    )
    
    // 调用其他服务（传递ctx）
    user, err := getUserService(ctx, userID)
    if err != nil {
        span.RecordError(err)
        return
    }
}
```

---

## 5. 告警体系（Alertmanager）

### 5.1 Prometheus告警规则

```yaml
groups:
  - name: my-service
    rules:
      # 错误率告警
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) /
          sum(rate(http_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "错误率过高"
          description: "服务{{ $labels.service }}错误率超过1%"
      
      # P99延迟告警
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
      
      # 服务不可用告警
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
```

### 5.2 Alertmanager配置

```yaml
route:
  group_by: ['alertname', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'default'
  
  routes:
    - match:
        severity: critical
      receiver: 'oncall'  # 电话通知
    - match:
        severity: warning
      receiver: 'wechat'  # 企业微信
```

---

## 6. 故障排查SOP

### 6.1 接口慢排查流程

1. **查看监控**：Grafana确认P99延迟
2. **查看日志**：Kibana查询慢请求
3. **链路追踪**：Jaeger定位慢的Span
4. **系统资源**：top/free/ss检查CPU/内存/网络
5. **定位根因**：数据库慢查询？下游服务慢？代码问题？

### 6.2 服务宕机排查流程

1. **查看监控**：up{job="my-service"}确认服务状态
2. **查看日志**：tail -f error.log查看最后错误
3. **查看进程**：ps aux | grep myapp
4. **重启服务**：systemctl restart myapp
5. **根因分析**：OOM？Panic？依赖故障？

---

# 第二部分：SRE高级实践

## 7. 容灾架构设计

### 7.1 容灾等级（RPO & RTO）

| 容灾等级 | RPO（数据丢失） | RTO（恢复时间） | 成本 | 适用场景 |
|----------|----------------|----------------|------|----------|
| **备份容灾** | 24小时 | 数小时-数天 | 💰 低 | 非核心业务 |
| **双活容灾** | 秒级 | 分钟级 | 💰💰💰 中 | 一般业务 |
| **异地多活** | 0（实时同步） | 秒级 | 💰💰💰💰 高 | 核心业务（金融、支付） |

**RTO vs RPO**：
- **RTO（Recovery Time Objective）**：故障后多久恢复服务
- **RPO（Recovery Point Objective）**：最多丢失多长时间的数据

### 7.2 同城双活架构

**架构图**：
```
                      [用户]
                        ↓
                   [DNS/LB]
                  /         \
         [IDC-A]              [IDC-B]
       (主机房 50%)          (备机房 50%)
            ↓                    ↓
      [应用集群]            [应用集群]
            ↓                    ↓
        [MySQL主]  ←→同步→  [MySQL从]
            ↓                    ↓
     [Redis Cluster]     [Redis Cluster]
        (分片1,3,5)         (分片2,4,6)
```

**核心技术点**：

#### 7.2.1 数据库双主同步（MySQL）
```sql
-- 主库A配置（server-id=1）
[mysqld]
server-id=1
log-bin=mysql-bin
auto_increment_increment=2  # 自增步长=2
auto_increment_offset=1     # 起始值=1 (1,3,5,7...)

-- 主库B配置（server-id=2）
[mysqld]
server-id=2
log-bin=mysql-bin
auto_increment_increment=2
auto_increment_offset=2     # 起始值=2 (2,4,6,8...)
```

**优点**：避免主键冲突（A生成奇数ID，B生成偶数ID）

**缺点**：数据一致性依赖binlog同步，网络延迟会导致延迟

#### 7.2.2 缓存跨机房同步（Redis）
```bash
# Redis Cluster分片策略
IDC-A: slot 0-5461, 10923-16383  (主)
IDC-B: slot 5462-10922           (主)

# 每个主节点在对方机房有从节点
redis-cli --cluster create \
  10.0.1.1:6379 10.0.1.2:6379 10.0.1.3:6379 \  # IDC-A主节点
  10.0.2.1:6379 10.0.2.2:6379 10.0.2.3:6379 \  # IDC-B主节点
  --cluster-replicas 1
```

#### 7.2.3 流量调度策略
```go
// 基于地理位置的就近路由
type LoadBalancer struct {
    idcANodes []string  // IDC-A节点列表
    idcBNodes []string  // IDC-B节点列表
}

func (lb *LoadBalancer) Route(clientIP string) string {
    // 1. 根据客户端IP判断地理位置
    region := geoip.GetRegion(clientIP)
    
    // 2. 就近路由（延迟最低）
    if region == "south" {
        return lb.pickHealthy(lb.idcANodes)
    } else {
        return lb.pickHealthy(lb.idcBNodes)
    }
}

func (lb *LoadBalancer) pickHealthy(nodes []string) string {
    // 健康检查：只返回健康节点
    for _, node := range nodes {
        if healthCheck(node) {
            return node
        }
    }
    // 如果本机房全挂，切到对方机房
    return lb.pickHealthy(lb.otherIDC())
}
```

### 7.3 异地多活架构

**适用场景**：金融、支付、电商核心业务

**架构图**：
```
         [用户-华南]        [用户-华北]        [用户-华东]
              ↓                 ↓                 ↓
         [深圳IDC]          [北京IDC]          [上海IDC]
              ↓                 ↓                 ↓
        [应用+DB]          [应用+DB]          [应用+DB]
              ↓────────────→   ↓   ←────────────↓
                   实时双向数据同步
```

**核心挑战**：分布式数据一致性

#### 7.3.1 分库分表策略
```go
// 按用户ID分片，同一用户数据永远在同一机房
func GetShardIDC(userID int64) string {
    // hash取模，保证同一用户始终路由到同一机房
    idc := userID % 3
    switch idc {
    case 0:
        return "shenzhen"
    case 1:
        return "beijing"
    case 2:
        return "shanghai"
    }
}

// 优点：避免跨机房事务，延迟低
// 缺点：机房故障时，该机房用户无法访问
```

#### 7.3.2 跨机房数据同步（Canal）
```yaml
# Canal配置：订阅MySQL binlog，同步到其他机房
canal.instance.master.address=10.0.1.1:3306
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.filter.regex=.*\\..*  # 同步所有库表

# MQ消息同步到其他机房
canal.destinations=shenzhen,beijing,shanghai
```

**数据同步延迟处理**：
```go
// 写后读问题：刚写入深圳机房，立即从北京机房读，可能读不到
func GetUser(userID int64) (*User, error) {
    // 1. 优先从本机房读
    user, err := localDB.GetUser(userID)
    if err == nil {
        return user, nil
    }
    
    // 2. 本机房没有，尝试从主机房读（带版本号）
    masterIDC := GetShardIDC(userID)
    user, err = remoteDB[masterIDC].GetUser(userID)
    
    // 3. 缓存到本地，下次直接读
    localDB.SetUser(userID, user)
    return user, err
}
```

### 7.4 容灾切换演练

**目标**：定期演练，确保故障时能快速切换

**演练步骤**：
```bash
# 1. 模拟机房A故障（禁止流量进入）
iptables -A INPUT -s 10.0.1.0/24 -j DROP

# 2. DNS切换：将A的流量切到B
# 修改DNS记录，TTL=60s
api.example.com  60  IN  A  10.0.2.1  # 从10.0.1.1改为10.0.2.1

# 3. 观察监控
# - QPS是否正常
# - 错误率是否上升
# - 数据库主从延迟

# 4. 回滚
# 恢复DNS记录
iptables -D INPUT -s 10.0.1.0/24 -j DROP
```

**演练频率**：每季度一次，核心业务每月一次

---

## 8. 故障管理与自愈

### 8.1 故障等级定义

| 等级 | 影响范围 | 响应时间 | 示例 |
|------|----------|----------|------|
| **P0** | 全站不可用/数据丢失 | 5分钟 | 数据库宕机、机房断网 |
| **P1** | 核心功能不可用 | 15分钟 | 支付失败、登录失败 |
| **P2** | 部分功能异常 | 1小时 | 搜索慢、图片加载失败 |
| **P3** | 非核心功能异常 | 1天 | 统计报表错误 |

### 8.2 故障响应流程（Incident Response）

**流程图**：
```
[告警触发] → [值班人员响应] → [问题确认] → [建立作战室]
                                              ↓
                        [止血] ← [定位根因] ← [拉人协助]
                          ↓
                    [恢复服务] → [复盘] → [改进措施]
```

**作战室工具**：
```go
// 1. 钉钉/企业微信群自动建群
func CreateIncidentRoom(incident *Incident) {
    room := dingtalk.CreateGroup(fmt.Sprintf("P%d故障-%s", incident.Level, incident.Title))
    
    // 2. 拉相关人员
    room.AddMembers(incident.Owners)
    room.AddMembers(getOncallEngineers())
    
    // 3. 自动推送监控数据
    room.SendCard(getMonitoringCard(incident))
    
    // 4. 启动录屏（用于复盘）
    startRecording(room.ID)
}
```

### 8.3 自动止血策略

#### 8.3.1 自动熔断（Circuit Breaker）
```go
import "github.com/sony/gobreaker"

// 熔断器配置
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "payment-service",
    MaxRequests: 3,              // 半开状态允许的最大请求数
    Interval:    60 * time.Second, // 统计周期
    Timeout:     30 * time.Second, // 熔断持续时间
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        // 触发条件：错误率 > 50% 且请求数 > 10
        failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 10 && failureRatio >= 0.5
    },
    OnStateChange: func(name string, from, to gobreaker.State) {
        log.Printf("熔断器状态变化: %s -> %s", from, to)
        // 发送告警
        alertmanager.Send(fmt.Sprintf("服务%s熔断", name))
    },
})

// 使用
func CallPaymentService(ctx context.Context, req *PaymentRequest) (*PaymentResponse, error) {
    result, err := cb.Execute(func() (interface{}, error) {
        return httpClient.Post("/payment", req)
    })
    
    if err != nil {
        // 熔断时，执行降级逻辑
        return fallback(req), nil
    }
    return result.(*PaymentResponse), nil
}
```

**熔断器三状态**：
- **Closed（关闭）**：正常请求，记录错误率
- **Open（打开）**：错误率超阈值，拒绝所有请求，返回降级结果
- **Half-Open（半开）**：等待一段时间后，放行少量请求测试是否恢复

#### 8.3.2 自动限流（Rate Limiting）
```go
import "golang.org/x/time/rate"

// 令牌桶限流器
limiter := rate.NewLimiter(1000, 2000)  // 1000 QPS，桶容量2000

func HandleRequest(w http.ResponseWriter, r *http.Request) {
    if !limiter.Allow() {
        // 触发限流
        http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
        
        // 记录指标
        rateLimitCounter.Inc()
        return
    }
    
    // 正常处理
    handle(w, r)
}
```

**动态限流**（基于CPU/内存）：
```go
type AdaptiveLimiter struct {
    baseLimiter *rate.Limiter
    cpuThreshold float64
}

func (al *AdaptiveLimiter) Allow() bool {
    // 1. 获取当前CPU使用率
    cpuUsage := getCPUUsage()
    
    // 2. CPU超过80%，限流更严格
    if cpuUsage > 0.8 {
        ratio := (cpuUsage - 0.8) / 0.2  // 0.8-1.0映射到0-1
        currentLimit := al.baseLimiter.Limit() * (1 - ratio*0.5)  // 最多降低50%
        al.baseLimiter.SetLimit(rate.Limit(currentLimit))
    }
    
    return al.baseLimiter.Allow()
}
```

#### 8.3.3 自动扩容（Auto Scaling）
```yaml
# Kubernetes HPA（Horizontal Pod Autoscaler）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 10
  maxReplicas: 100
  metrics:
    # 基于CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    # 基于内存
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # 基于自定义指标（QPS）
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # 扩容冷却时间
      policies:
        - type: Percent
          value: 50                    # 每次扩容50%
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容冷却时间（更长）
      policies:
        - type: Pods
          value: 1                     # 每次缩容1个
          periodSeconds: 60
```

### 8.4 故障自愈实战

**场景**：数据库主从延迟过大

**监控告警**：
```promql
# Prometheus规则
- alert: MySQLReplicationLag
  expr: mysql_slave_lag_seconds > 60
  for: 5m
  annotations:
    summary: "MySQL主从延迟 {{ $value }}秒"
```

**自动处理脚本**：
```bash
#!/bin/bash
# 自愈脚本：主从延迟时自动切换到主库

SLAVE_LAG=$(mysql -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master | awk '{print $2}')

if [ "$SLAVE_LAG" -gt 60 ]; then
    echo "主从延迟${SLAVE_LAG}秒，切换到主库"
    
    # 1. 修改应用配置，读请求切到主库
    kubectl set env deployment/myapp DB_READ_HOST=mysql-master
    
    # 2. 发送告警
    curl -X POST http://alertmanager/api/v1/alerts -d '{
        "labels": {"alertname": "MySQLReplicationLag", "severity": "warning"},
        "annotations": {"summary": "主从延迟，已切换到主库"}
    }'
    
    # 3. 等待从库追上
    while [ "$SLAVE_LAG" -gt 10 ]; do
        sleep 10
        SLAVE_LAG=$(mysql -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master | awk '{print $2}')
    done
    
    # 4. 切回从库
    kubectl set env deployment/myapp DB_READ_HOST=mysql-slave
    echo "从库已追上，切回从库"
fi
```

---

## 9. 监控体系建设

### 9.1 监控全景图

```
┌─────────────────────────────────────────────────┐
│                   用户侧监控                      │
│  拨测（Synthetic Monitoring）、真实用户监控（RUM） │
└────────────────┬────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────┐
│                   应用层监控                      │
│  QPS、延迟、错误率、链路追踪、业务指标            │
└────────────────┬────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────┐
│                 中间件层监控                      │
│  数据库、缓存、消息队列、RPC                       │
└────────────────┬────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────┐
│                 基础设施监控                      │
│  CPU、内存、磁盘、网络、容器、K8s                  │
└─────────────────────────────────────────────────┘
```

### 9.2 用户侧监控

#### 9.2.1 拨测（黑盒监控）
```yaml
# Prometheus Blackbox Exporter配置
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      fail_if_not_matches_regexp:
        - "success"
      preferred_ip_protocol: "ip4"

# Prometheus配置
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://api.example.com/health
          - https://www.example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

**告警规则**：
```yaml
- alert: ServiceDown
  expr: probe_success == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "服务{{ $labels.instance }}不可用"
```

#### 9.2.2 真实用户监控（RUM）
```javascript
// 前端埋点：上报页面加载时间
window.addEventListener('load', function() {
    const perfData = window.performance.timing;
    const pageLoadTime = perfData.loadEventEnd - perfData.navigationStart;
    const dnsTime = perfData.domainLookupEnd - perfData.domainLookupStart;
    const tcpTime = perfData.connectEnd - perfData.connectStart;
    const ttfb = perfData.responseStart - perfData.navigationStart;
    
    // 上报到监控系统
    fetch('/api/metrics', {
        method: 'POST',
        body: JSON.stringify({
            page_load_time: pageLoadTime,
            dns_time: dnsTime,
            tcp_time: tcpTime,
            ttfb: ttfb,
            url: window.location.href,
            user_agent: navigator.userAgent
        })
    });
});
```

**分析维度**：
- 地域：北京用户加载慢？
- 浏览器：Chrome快，IE慢？
- 页面：首页慢，还是详情页慢？

### 9.3 业务监控

**核心业务指标**：
```go
// 订单相关
var (
    orderCreatedTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "order_created_total"},
        []string{"status"},  // success, failed
    )
    
    orderAmountTotal = prometheus.NewCounter(
        prometheus.CounterOpts{Name: "order_amount_total"},
    )
    
    orderConversionRate = prometheus.NewGauge(
        prometheus.GaugeOpts{Name: "order_conversion_rate"},
    )
)

func CreateOrder(userID int64, amount float64) error {
    err := orderService.Create(userID, amount)
    if err != nil {
        orderCreatedTotal.WithLabelValues("failed").Inc()
        return err
    }
    
    orderCreatedTotal.WithLabelValues("success").Inc()
    orderAmountTotal.Add(amount)
    return nil
}

// 定时计算转化率
func UpdateConversionRate() {
    orders := getOrderCount()
    visits := getVisitCount()
    rate := float64(orders) / float64(visits)
    orderConversionRate.Set(rate)
}
```

**业务大盘**（Grafana）：
```promql
# 实时订单量
sum(rate(order_created_total{status="success"}[1m])) * 60

# 订单转化率
order_conversion_rate * 100

# 今日总成交额
sum(increase(order_amount_total[24h]))
```

### 9.4 中间件监控

#### 9.4.1 MySQL监控
```yaml
# mysqld_exporter
docker run -d \
  -p 9104:9104 \
  -e DATA_SOURCE_NAME="exporter:password@(mysql:3306)/" \
  prom/mysqld-exporter
```

**关键指标**：
```promql
# QPS
rate(mysql_global_status_queries[1m])

# 慢查询
rate(mysql_global_status_slow_queries[1m])

# 连接数
mysql_global_status_threads_connected

# 主从延迟
mysql_slave_status_seconds_behind_master
```

#### 9.4.2 Redis监控
```yaml
# redis_exporter
docker run -d \
  -p 9121:9121 \
  oliver006/redis_exporter \
  --redis.addr=redis://redis:6379
```

**关键指标**：
```promql
# 命令QPS
rate(redis_commands_processed_total[1m])

# 命中率
redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total)

# 内存使用
redis_memory_used_bytes / redis_memory_max_bytes

# 慢查询
redis_slowlog_length
```

### 9.5 监控数据治理

**问题**：监控指标爆炸（10万+时间序列），Prometheus内存不足

**解决**：

#### 9.5.1 降低基数（Cardinality）
```go
// ❌ 错误：userID作为label（100万用户=100万时间序列）
requestCounter.WithLabelValues(userID, path).Inc()

// ✅ 正确：只保留必要的label
requestCounter.WithLabelValues(method, path, status).Inc()

// 用户维度的统计放到日志或独立存储
log.WithFields(logrus.Fields{
    "user_id": userID,
    "path": path,
}).Info("request")
```

#### 9.5.2 数据分级存储
```yaml
# Prometheus配置：不同数据保留不同时长
global:
  scrape_interval: 15s

# 高精度数据：保留7天
- job_name: 'critical-services'
  scrape_interval: 10s
  sample_limit: 100000

# 普通数据：保留30天，降采样
- job_name: 'normal-services'
  scrape_interval: 30s

# 长期存储：写入Thanos/VictoriaMetrics
remote_write:
  - url: http://thanos-receiver:19291/api/v1/receive
```

#### 9.5.3 指标聚合
```yaml
# Prometheus Recording Rules：预聚合，减少查询压力
groups:
  - name: api_metrics
    interval: 60s
    rules:
      # 预聚合：每分钟计算一次，避免实时计算
      - record: api:request_rate:1m
        expr: sum(rate(http_requests_total[1m])) by (service, method, path)
      
      - record: api:error_rate:1m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[1m])) by (service)
          /
          sum(rate(http_requests_total[1m])) by (service)
```

---

## 10. 性能优化实战

### 10.1 性能优化方法论

```
┌─────────────────────────────────────────┐
│ 1. 定义目标（P99延迟<100ms）             │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 2. 建立基线（当前P99=500ms）             │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 3. 定位瓶颈（火焰图、pprof、慢查询）     │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 4. 优化实施（加索引、缓存、并发）        │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 5. 验证效果（P99降到100ms）              │
└─────────────────────────────────────────┘
```

### 10.2 应用层优化

#### 10.2.1 CPU优化（Go）
```bash
# 1. 启用pprof
import _ "net/http/pprof"
go func() {
    http.ListenAndServe("localhost:6060", nil)
}()

# 2. 采集30秒CPU profile
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof

# 3. 分析
go tool pprof -http=:8080 cpu.prof
```

**常见优化**：
```go
// ❌ 频繁字符串拼接（每次分配新内存）
func buildSQL(ids []int) string {
    sql := "SELECT * FROM users WHERE id IN ("
    for i, id := range ids {
        sql += fmt.Sprintf("%d", id)
        if i < len(ids)-1 {
            sql += ","
        }
    }
    sql += ")"
    return sql
}

// ✅ 使用strings.Builder（减少内存分配）
func buildSQL(ids []int) string {
    var sb strings.Builder
    sb.WriteString("SELECT * FROM users WHERE id IN (")
    for i, id := range ids {
        sb.WriteString(strconv.Itoa(id))
        if i < len(ids)-1 {
            sb.WriteByte(',')
        }
    }
    sb.WriteByte(')')
    return sb.String()
}

// 性能提升：10倍+（减少GC压力）
```

#### 10.2.2 内存优化
```bash
# 1. 采集heap profile
curl http://localhost:6060/debug/pprof/heap > heap.prof

# 2. 分析
go tool pprof -http=:8080 heap.prof
```

**常见优化**：
```go
// ❌ 无限增长的slice
func processData() {
    var results []Result
    for {
        data := fetchData()
        results = append(results, process(data))  // 内存泄漏
    }
}

// ✅ 定期清理
func processData() {
    results := make([]Result, 0, 1000)
    for {
        data := fetchData()
        results = append(results, process(data))
        
        if len(results) > 1000 {
            flush(results)
            results = results[:0]  // 复用底层数组
        }
    }
}
```

### 10.3 数据库优化

#### 10.3.1 慢查询优化
```sql
-- 1. 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.5;  -- 超过0.5秒记录

-- 2. 分析慢查询
SELECT * FROM orders WHERE user_id = 1001 ORDER BY created_at DESC LIMIT 10;

-- 3. EXPLAIN分析
EXPLAIN SELECT * FROM orders WHERE user_id = 1001 ORDER BY created_at DESC LIMIT 10;
-- type: ALL (全表扫描)
-- rows: 1000000

-- 4. 添加索引
CREATE INDEX idx_user_created ON orders(user_id, created_at);

-- 5. 再次EXPLAIN
EXPLAIN SELECT * FROM orders WHERE user_id = 1001 ORDER BY created_at DESC LIMIT 10;
-- type: ref (索引查询)
-- rows: 100
-- Extra: Using index

-- 性能提升：100倍+（1000ms → 10ms）
```

#### 10.3.2 批量操作优化
```go
// ❌ 逐条插入（10000次网络往返）
for _, user := range users {
    db.Exec("INSERT INTO users (name, email) VALUES (?, ?)", user.Name, user.Email)
}

// ✅ 批量插入（1次网络往返）
func BulkInsert(users []User) error {
    query := "INSERT INTO users (name, email) VALUES "
    values := []interface{}{}
    
    for i, user := range users {
        if i > 0 {
            query += ","
        }
        query += "(?, ?)"
        values = append(values, user.Name, user.Email)
    }
    
    _, err := db.Exec(query, values...)
    return err
}

// 性能提升：100倍+（10s → 0.1s）
```

### 10.4 缓存优化

#### 10.4.1 缓存穿透（查询不存在的key）
```go
// ❌ 没有缓存空值，每次都查DB
func GetUser(userID int64) (*User, error) {
    // 1. 查缓存
    if user, ok := cache.Get(userID); ok {
        return user, nil
    }
    
    // 2. 查数据库
    user, err := db.GetUser(userID)
    if err != nil {
        return nil, err  // 没有缓存，每次都查DB
    }
    
    cache.Set(userID, user, 5*time.Minute)
    return user, nil
}

// ✅ 缓存空值（Null Object）
func GetUser(userID int64) (*User, error) {
    if user, ok := cache.Get(userID); ok {
        if user == nil {
            return nil, ErrNotFound  // 缓存的空值
        }
        return user, nil
    }
    
    user, err := db.GetUser(userID)
    if err == ErrNotFound {
        cache.Set(userID, nil, 1*time.Minute)  // 缓存空值，TTL较短
        return nil, err
    }
    
    cache.Set(userID, user, 5*time.Minute)
    return user, nil
}
```

#### 10.4.2 缓存击穿（热key过期）
```go
// 使用singleflight防止缓存击穿
import "golang.org/x/sync/singleflight"

var sg singleflight.Group

func GetUser(userID int64) (*User, error) {
    // 1. 查缓存
    if user, ok := cache.Get(userID); ok {
        return user, nil
    }
    
    // 2. singleflight：同一时刻只有一个请求查DB
    key := fmt.Sprintf("user:%d", userID)
    v, err, _ := sg.Do(key, func() (interface{}, error) {
        // 再次检查缓存（可能已被其他goroutine设置）
        if user, ok := cache.Get(userID); ok {
            return user, nil
        }
        
        // 查数据库
        user, err := db.GetUser(userID)
        if err != nil {
            return nil, err
        }
        
        cache.Set(userID, user, 5*time.Minute)
        return user, nil
    })
    
    if err != nil {
        return nil, err
    }
    return v.(*User), nil
}

// 效果：1000个并发请求，只有1个查DB，其他999个等待
```

### 10.5 网络优化

#### 10.5.1 连接池优化
```go
// ❌ 每次创建新连接
func callAPI(url string) (*Response, error) {
    client := &http.Client{Timeout: 5 * time.Second}
    return client.Get(url)
}

// ✅ 复用连接（连接池）
var httpClient = &http.Client{
    Timeout: 5 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,              // 最大空闲连接数
        MaxIdleConnsPerHost: 10,               // 每个host的最大空闲连接
        IdleConnTimeout:     90 * time.Second, // 空闲连接超时
        DisableKeepAlives:   false,            // 开启keep-alive
    },
}

func callAPI(url string) (*Response, error) {
    return httpClient.Get(url)
}

// 性能提升：3-5倍（减少TCP握手）
```

#### 10.5.2 并发请求
```go
// ❌ 串行调用（总耗时=3秒）
func GetUserInfo(userID int64) (*UserInfo, error) {
    user := getUserService(userID)        // 1s
    orders := getOrderService(userID)     // 1s
    payments := getPaymentService(userID) // 1s
    
    return &UserInfo{User: user, Orders: orders, Payments: payments}, nil
}

// ✅ 并发调用（总耗时=1秒）
func GetUserInfo(userID int64) (*UserInfo, error) {
    var wg sync.WaitGroup
    var user *User
    var orders []*Order
    var payments []*Payment
    
    wg.Add(3)
    go func() {
        defer wg.Done()
        user = getUserService(userID)
    }()
    go func() {
        defer wg.Done()
        orders = getOrderService(userID)
    }()
    go func() {
        defer wg.Done()
        payments = getPaymentService(userID)
    }()
    
    wg.Wait()
    return &UserInfo{User: user, Orders: orders, Payments: payments}, nil
}
```

#### 10.5.3 HTTP/2 与 HTTP/3(QUIC) 的取舍

很多团队把“升级 HTTP/3”当成网络优化万能药，这是不准确的。更合理的判断方式是先看链路特征：

| 场景 | HTTP/3 优势是否明显 | 原因 |
|------|--------------------|------|
| **移动网络抖动大、弱网切换频繁** | 明显 | QUIC 基于 UDP，连接迁移更友好，丢单个包不容易形成整连接级队头阻塞 |
| **跨公网高 RTT 场景** | 较明显 | 0-RTT/握手优化更容易体现收益 |
| **机房内 RPC、内网稳定链路** | 不明显 | 网络本身已稳定，收益通常被业务处理耗时淹没 |
| **大量中间盒或安全设备敏感环境** | 需谨慎 | UDP 路径、观测和治理链路可能更复杂 |

SRE 视角下，协议升级前至少要回答三件事：
1. 现在的长尾主要卡在握手、丢包、迁移，还是卡在应用处理和下游依赖？
2. 现有 WAF、网关、观测、限流体系是否已经支持 QUIC。
3. 升级后问题出了，抓包、回滚、灰度和兼容策略是否准备好了。

协议优化只有在“网络真是瓶颈”时才值回复杂度。

---

## 11. 成本优化

### 11.1 资源利用率优化

#### 11.1.1 Pod资源配置优化
```yaml
# ❌ 资源配置不合理
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: myapp
          resources:
            requests:
              cpu: "1"       # 实际只用0.2
              memory: "2Gi"  # 实际只用500Mi
            limits:
              cpu: "2"
              memory: "4Gi"
# 浪费：80%的CPU、75%的内存

# ✅ 根据监控数据调整
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: myapp
          resources:
            requests:
              cpu: "200m"      # P99 CPU使用量
              memory: "512Mi"  # P99内存使用量
            limits:
              cpu: "500m"      # 留20%余量
              memory: "768Mi"
# 节省：60%成本
```

**优化方法**：
```bash
# 1. 查看资源使用情况
kubectl top pods

# 2. 分析历史数据（Prometheus）
avg_over_time(container_cpu_usage_seconds_total[7d])
avg_over_time(container_memory_usage_bytes[7d])

# 3. 调整配置
# requests = P90使用量
# limits = P99使用量 * 1.2
```

#### 11.1.2 节点利用率优化
```bash
# 查看节点利用率
kubectl describe nodes | grep -A 5 "Allocated resources"

# 输出：
# CPU Requests: 2 (50% of 4 cores)
# Memory Requests: 4Gi (50% of 8Gi)

# 目标：60-70%
```

**Bin Packing优化**：
```yaml
# 使用Pod反亲和性，避免单个节点过载
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - myapp
                topologyKey: kubernetes.io/hostname
```

### 11.2 存储成本优化

#### 11.2.1 日志分级存储
```yaml
# Elasticsearch ILM（Index Lifecycle Management）
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1  # 合并分片
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}  # 冻结索引（只读，减少内存）
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}  # 删除
        }
      }
    }
  }
}

# 成本节省：70%（通过分级存储）
```

#### 11.2.2 指标数据降采样
```yaml
# Thanos Compactor：自动降采样
thanos compact \
  --data-dir /data \
  --retention.resolution-raw=7d \      # 原始数据保留7天
  --retention.resolution-5m=30d \      # 5分钟聚合保留30天
  --retention.resolution-1h=180d       # 1小时聚合保留180天

# 存储节省：90%（原始数据→聚合数据）
```

### 11.3 流量成本优化

#### 11.3.1 CDN回源优化
```nginx
# Nginx反向代理：减少回源请求
location /static/ {
    proxy_cache my_cache;
    proxy_cache_valid 200 1d;        # 200响应缓存1天
    proxy_cache_valid 404 10m;       # 404响应缓存10分钟
    proxy_cache_use_stale error timeout;  # 源站故障时使用过期缓存
    
    add_header X-Cache-Status $upstream_cache_status;
    proxy_pass http://backend;
}

# CDN回源减少50% → 成本降低50%
```

#### 11.3.2 图片压缩
```go
// WebP格式转换（减少60%大小）
import "github.com/chai2010/webp"

func convertToWebP(inputPath, outputPath string) error {
    img, err := loadImage(inputPath)
    if err != nil {
        return err
    }
    
    // 转换为WebP，质量75
    return webp.Encode(outputPath, img, &webp.Options{Quality: 75})
}
```

---

## 12. 稳定性治理

### 12.1 SLO（Service Level Objective）体系

#### 12.1.1 SLI、SLO、SLA 的区别

| 概念 | 面向对象 | 含义 | 示例 |
|------|----------|------|------|
| **SLI** | 工程指标 | 可观测的服务质量指标 | 30天内成功率 99.97% |
| **SLO** | 内部目标 | 团队承诺要达到的目标 | API 可用性 >= 99.95% |
| **SLA** | 外部合同 | 对客户的合同承诺，违约有赔偿 | 月可用性 < 99.9% 赔偿 10% |

**经验原则**：
- SLI 必须可精确测量，且和用户体验直接相关
- SLO 通常严于 SLA，给自己留缓冲
- 不要为每个内部组件都定义 SLA，先为关键用户路径定义 SLI/SLO

**定义SLO**：
```yaml
# 示例：API服务SLO
service: payment-api
slo:
  availability: 99.95%     # 可用性：每月最多停机21.6分钟
  latency_p99: 200ms       # P99延迟 < 200ms
  error_rate: 0.1%         # 错误率 < 0.1%
  
measurement:
  window: 30d              # 滚动窗口30天
  
error_budget:
  total_minutes: 21.6      # 每月错误预算
  consumed: 5.2            # 已消耗
  remaining: 16.4          # 剩余

alerts:
  - name: SLO_ErrorBudget_Low
    condition: remaining < 25%  # 剩余预算<25%时告警
    action: 冻结非紧急变更
```

**Error Budget计算**：
```go
func CalculateErrorBudget(slo float64, window time.Duration) time.Duration {
    // SLO=99.95%，则允许0.05%的错误
    allowedDowntime := window * time.Duration((1-slo)*100) / 100
    return allowedDowntime
}

// 示例：
// SLO=99.95%, window=30天
// 允许停机时间 = 30天 * 0.05% = 21.6分钟
```

#### 12.1.2 Burn Rate 告警

**为什么要看 Burn Rate**：错误预算不是只看"还剩多少"，更关键是看"消耗速度"。如果服务今天 1 小时就烧掉了本月 30% 的预算，必须立刻干预。

**计算公式**：
```text
Burn Rate = 当前错误率 / 允许错误率
```

示例：SLO=99.95%，允许错误率=0.05%。如果最近 5 分钟错误率是 1%，则：
```text
Burn Rate = 1% / 0.05% = 20
```
含义：预算消耗速度是正常情况的 20 倍。

**推荐做法：多窗口 + 多 Burn Rate 告警**

| 窗口 | 目标 | 典型阈值 | 作用 |
|------|------|----------|------|
| 5m + 1h | 快速发现大故障 | Burn Rate > 14.4 | 抓突发事故 |
| 30m + 6h | 发现持续性劣化 | Burn Rate > 6 | 抓慢性故障 |
| 6h + 3d | 发现长期退化 | Burn Rate > 1 | 抓长期质量下降 |

**PromQL 示例**：
```promql
# 假设 SLO 可用性 99.95%，允许错误率 0.0005
(
  sum(rate(http_requests_total{job="payment-api",status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total{job="payment-api"}[5m]))
) / 0.0005
```

**经验原则**：
- 告警要同时满足短窗口和长窗口，避免毛刺误报
- Burn Rate 告警应该直连值班，而不是只挂在周报里
- Burn Rate 触发后，默认动作不是继续观察，而是先冻结变更、确认止血手段

#### 12.1.3 Error Budget Policy

**Error Budget Policy** 不是一句“预算耗尽就少发版”，而是一套提前约定好的动作规则。

| 预算状态 | 典型阈值 | 发布策略 | 团队动作 |
|----------|----------|----------|----------|
| 充足 | 剩余 > 50% | 正常发布 | 维持常规变更节奏 |
| 紧张 | 剩余 25%~50% | 提高门禁 | 高风险变更需审批，灰度更保守 |
| 危险 | 剩余 < 25% | 限制发布 | 只允许修复性变更和必要配置调整 |
| 耗尽 | <= 0 | 冻结变更 | 进入稳定性专项，优先做止血和治理 |

**关键原则**：
- Policy 必须在事故前写好，不能在预算烧穿后临时拍脑袋
- Policy 必须能映射到发布系统、审批流和灰度门禁，而不是停留在文档里
- 预算恢复之前，默认优先级应从“交付速度”切到“稳定性修复”

### 12.2 变更管理

#### 12.2.1 灰度发布
```yaml
# Argo Rollouts：金丝雀发布
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 100
  strategy:
    canary:
      steps:
        - setWeight: 5        # 5%流量到新版本
        - pause: {duration: 5m}
        - setWeight: 20       # 20%流量
        - pause: {duration: 10m}
        - setWeight: 50       # 50%流量
        - pause: {duration: 10m}
        - setWeight: 100      # 全量
      analysis:
        templates:
          - templateName: success-rate
        args:
          - name: service-name
            value: myapp
      trafficRouting:
        istio:
          virtualService:
            name: myapp-vsvc

---
# AnalysisTemplate：自动检查指标
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result >= 0.99  # 成功率 >= 99%
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[1m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[1m]))
```

**灰度决策树**：
```
发布新版本
    ↓
5%流量，观察5分钟
    ↓
成功率 >= 99%? → No → 自动回滚
    ↓ Yes
20%流量，观察10分钟
    ↓
成功率 >= 99% && P99延迟正常? → No → 自动回滚
    ↓ Yes
全量发布
```

#### 12.2.2 自动回滚
```go
// 回滚策略
type RollbackPolicy struct {
    ErrorRateThreshold float64  // 错误率阈值
    LatencyP99Threshold time.Duration
    CheckInterval time.Duration
    ConsecutiveFails int  // 连续失败次数
}

func (p *RollbackPolicy) ShouldRollback() bool {
    fails := 0
    ticker := time.NewTicker(p.CheckInterval)
    
    for range ticker.C {
        errorRate := getErrorRate()
        latencyP99 := getLatencyP99()
        
        if errorRate > p.ErrorRateThreshold || latencyP99 > p.LatencyP99Threshold {
            fails++
            if fails >= p.ConsecutiveFails {
                log.Warn("指标异常，触发自动回滚")
                return true
            }
        } else {
            fails = 0  // 重置
        }
    }
    
    return false
}
```

#### 12.2.3 变更风险分级与发布前检查

| 变更类型 | 风险级别 | 示例 | 建议策略 |
|----------|----------|------|----------|
| 配置变更 | 中 | 限流阈值、连接池参数 | 小流量灰度 + 自动回滚 |
| 应用发布 | 中高 | 新版本上线 | 金丝雀 + 指标门禁 |
| 数据库变更 | 高 | 加列、改索引、DDL | 影子验证 + 低峰执行 |
| 基础设施变更 | 高 | K8s 升级、网络策略调整 | 预演 + 分批次升级 |

**发布前检查清单**：
- 是否有明确回滚路径，且回滚时间在 RTO 内
- 是否完成容量评估和压测验证
- 是否设置灰度指标门禁：成功率、P99、核心业务转化率
- 是否通知值班与相关依赖方，避免多团队同时高风险变更
- 是否避开错误预算紧张时段和业务高峰期

#### 12.2.4 DORA 指标与稳定性治理边界

**DORA** 更关注交付效率与交付质量，典型包括发布频率、变更前置时间、变更失败率、恢复时间；**SRE 指标** 更关注服务质量与可靠性，典型包括 SLI/SLO、Error Budget、可用性、延迟、告警质量。

它们的关系不是替代，而是互补：
- DORA 告诉你团队交付得快不快、变更是不是经常出事故
- SRE 指标告诉你用户到底有没有受损、可靠性目标是否被破坏
- 真正成熟的团队会把 `Change Failure Rate`、`MTTR` 和 `Error Budget` 联动，而不是只追求发布更快

### 12.3 Toil 与 Runbook

#### 12.3.1 什么是 Toil

**Toil**：重复、手工、可自动化、与长期工程收益不成正比的运维劳动。

典型例子：
- 每天手动扩容/缩容
- 每次磁盘满都靠人清理日志
- 同类故障反复人工执行相同步骤恢复

**SRE 的目标不是把 Toil 做得更熟练，而是持续消灭 Toil**。

**衡量方式**：
```text
Toil Ratio = 运维重复劳动时间 / 团队总工程时间
```

经验上，如果团队超过 50% 时间都在处理 Toil，说明系统已经进入失控状态，必须优先做自动化和平台化。

#### 12.3.2 Runbook 与 Playbook

| 名称 | 用途 | 特点 |
|------|------|------|
| **Runbook** | 具体故障处理步骤 | 面向执行，步骤明确 |
| **Playbook** | 复杂场景的协同策略 | 面向指挥，强调分工与决策 |

**一个合格 Runbook 至少包含**：
- 告警含义和影响范围
- 先确认什么，再执行什么
- 明确的止血步骤、回滚步骤、验证步骤
- 升级条件：什么情况下必须拉高优先级或通知谁
- 关联仪表盘、日志查询、链路追踪链接

#### 12.3.3 值班制度设计

**关键原则**：
- 告警必须可行动，不可行动的告警要删除或降级
- 一线值班负责止血和升级，不要求一线独自完成深度修复
- 必须控制告警噪音，否则值班制度会失效

**建议指标**：
- 单人单周夜间高优告警次数可控
- 告警确认时间（MTTA）和恢复时间（MTTR）持续下降
- 噪音告警占比持续下降

#### 12.3.4 升级策略与复盘行动项

**Escalation Policy（升级策略）** 的核心不是“谁官大找谁”，而是明确什么级别的故障在什么时间点必须拉谁进来。

| 场景 | 典型触发条件 | 必须升级对象 |
|------|--------------|--------------|
| 一线无法止血 | 10~15 分钟内没有形成有效止血动作 | 二线/服务负责人 |
| 核心链路持续恶化 | SLI 持续掉线、Burn Rate 过高 | 值班经理/相关依赖方 |
| 涉及跨团队依赖 | 网关、数据库、网络、支付通道同时异常 | 跨团队作战室 |
| 资损/安全/合规风险 | 出现错误扣款、数据泄漏、权限异常 | 业务负责人 + 安全/法务 |

**Postmortem Action Item 管理原则**：
- 每个行动项必须有 owner、截止时间、验收标准，不能只写“后续优化”
- 行动项要区分止血型、预防型、平台型，避免全变成低优先级 backlog
- 复盘完成不等于工作完成，必须追踪行动项关闭率和重复事故率

### 12.4 故障演练

**Netflix Chaos Monkey思想**：定期制造故障，验证系统韧性

#### 12.4.1 Pod随机删除
```bash
# 使用Chaos Mesh
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - default
    labelSelectors:
      app: myapp
  scheduler:
    cron: "@every 1h"  # 每小时随机杀一个Pod
EOF
```

#### 12.4.2 网络延迟注入
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - default
    labelSelectors:
      app: myapp
  delay:
    latency: "100ms"
    correlation: "25"
    jitter: "10ms"
  duration: "10m"
```

---

## 13. 混沌工程

### 13.1 混沌工程原则

1. **建立稳态假设**：系统正常时，QPS=1000，错误率<0.1%
2. **设计实验**：模拟故障（杀Pod、网络延迟、磁盘满）
3. **在生产环境验证**：灰度注入故障（5%流量）
4. **自动化**：持续演练

### 13.2 实验示例

**实验1：数据库主库宕机**
```yaml
目标：验证主从切换是否正常

步骤：
1. 停止MySQL主库
   docker stop mysql-master

2. 观察应用是否自动切换到从库
   - 错误率是否上升
   - 延迟是否增加
   - 是否有数据丢失

3. 恢复主库
   docker start mysql-master

4. 验证主从同步是否正常

期望结果：
- 切换时间 < 30秒
- 错误率 < 1%
- 无数据丢失
```

**实验2：依赖服务延迟**
```yaml
目标：验证熔断器是否生效

步骤：
1. 注入网络延迟（下游服务响应慢）
   tc qdisc add dev eth0 root netem delay 5000ms

2. 观察
   - 熔断器是否打开
   - 是否返回降级结果
   - 上游服务是否正常

3. 恢复
   tc qdisc del dev eth0 root

期望结果：
- 熔断器5秒内打开
- 降级结果正常返回
- 上游服务不受影响
```

---

## 14. 容量规划与成本优化

### 14.1 容量预估方法

容量规划是 SRE 最核心的前置能力之一。低估导致故障，高估导致浪费。成熟的容量规划需要多种方法交叉验证。

#### 14.1.1 基于QPS的容量预估

```
核心公式：
所需实例数 = (峰值QPS × 安全系数) / 单实例承载QPS

示例：
- 业务峰值QPS：50,000
- 单实例压测结果：2,000 QPS（P99 < 200ms）
- 安全系数：1.5（预留50%余量应对突发）
- 所需实例数 = (50,000 × 1.5) / 2,000 = 37.5 → 38台

注意事项：
- 单实例QPS必须通过压测获得，不能拍脑袋
- 安全系数通常取1.3~2.0，取决于业务波动性
- 峰值QPS要考虑业务增长（如大促、节假日）
```

```go
package capacity

type QPSBasedPlanner struct {
    SingleInstanceQPS float64 // 单实例承载QPS（压测值）
    SafetyFactor      float64 // 安全系数
    MinInstances      int     // 最小实例数（高可用）
}

// Plan 根据目标QPS计算所需实例数
func (p *QPSBasedPlanner) Plan(targetQPS float64) *CapacityPlan {
    required := targetQPS * p.SafetyFactor / p.SingleInstanceQPS
    instances := int(math.Ceil(required))

    // 保证最小实例数（跨AZ部署至少3个）
    if instances < p.MinInstances {
        instances = p.MinInstances
    }

    return &CapacityPlan{
        TargetQPS:     targetQPS,
        Instances:     instances,
        CPUPerInstance: p.estimateCPU(targetQPS / float64(instances)),
        MemPerInstance: p.estimateMemory(targetQPS / float64(instances)),
        Headroom:      float64(instances)*p.SingleInstanceQPS - targetQPS,
    }
}

// BurstPlan 突发流量容量规划（如秒杀、开服）
func (p *QPSBasedPlanner) BurstPlan(baseQPS, burstMultiplier float64) *CapacityPlan {
    burstQPS := baseQPS * burstMultiplier
    plan := p.Plan(burstQPS)
    plan.BurstDuration = 30 * time.Minute // 预计突发持续时间
    plan.ScaleUpLeadTime = 5 * time.Minute // 扩容提前量
    return plan
}
```

#### 14.1.2 基于资源的容量预估

```go
type ResourceBasedPlanner struct{}

type ResourceProfile struct {
    CPUCoresPerReq    float64       // 每请求CPU消耗（核·秒）
    MemoryPerConn     int64         // 每连接内存消耗（MB）
    DiskIOPerReq      float64       // 每请求磁盘IO（IOPS）
    NetworkPerReq     int64         // 每请求网络带宽（KB）
}

// PlanByResource 根据资源消耗模型规划
func (p *ResourceBasedPlanner) PlanByResource(
    targetQPS float64,
    profile *ResourceProfile,
    instanceSpec *InstanceSpec,
) *CapacityPlan {

    // CPU维度
    cpuInstances := math.Ceil(
        targetQPS * profile.CPUCoresPerReq / (float64(instanceSpec.CPUCores) * 0.7), // CPU不超70%
    )

    // 内存维度（长连接场景）
    memInstances := math.Ceil(
        float64(profile.MemoryPerConn) * targetQPS / (float64(instanceSpec.MemoryMB) * 0.8 * 1024),
    )

    // 磁盘IO维度
    diskInstances := math.Ceil(
        targetQPS * profile.DiskIOPerReq / float64(instanceSpec.MaxIOPS),
    )

    // 取最大值（木桶原理：瓶颈资源决定容量）
    instances := int(math.Max(cpuInstances, math.Max(memInstances, diskInstances)))

    return &CapacityPlan{
        Instances:  instances,
        Bottleneck: p.identifyBottleneck(cpuInstances, memInstances, diskInstances),
    }
}

func (p *ResourceBasedPlanner) identifyBottleneck(cpu, mem, disk float64) string {
    max := math.Max(cpu, math.Max(mem, disk))
    switch max {
    case cpu:
        return "CPU"
    case mem:
        return "Memory"
    default:
        return "DiskIO"
    }
}
```

#### 14.1.3 基于历史增长的容量预估

```go
type GrowthBasedPlanner struct {
    historyData []DataPoint // 历史数据（QPS/DAU/存储量）
}

type DataPoint struct {
    Timestamp time.Time
    Value     float64
}

// LinearForecast 线性回归预测
func (p *GrowthBasedPlanner) LinearForecast(targetDate time.Time) float64 {
    n := float64(len(p.historyData))
    var sumX, sumY, sumXY, sumX2 float64

    baseTime := p.historyData[0].Timestamp
    for _, dp := range p.historyData {
        x := dp.Timestamp.Sub(baseTime).Hours() / 24 // 天数
        y := dp.Value
        sumX += x
        sumY += y
        sumXY += x * y
        sumX2 += x * x
    }

    // y = a + b*x
    b := (n*sumXY - sumX*sumY) / (n*sumX2 - sumX*sumX)
    a := (sumY - b*sumX) / n

    targetX := targetDate.Sub(baseTime).Hours() / 24
    return a + b*targetX
}

// SeasonalForecast 考虑季节性的预测（如节假日高峰）
func (p *GrowthBasedPlanner) SeasonalForecast(targetDate time.Time, seasonalFactor float64) float64 {
    baseForecast := p.LinearForecast(targetDate)
    return baseForecast * seasonalFactor
}

/*
使用示例：
- 过去90天DAU数据：线性增长
- 预测30天后DAU → 基于线性回归
- 春节期间 seasonalFactor = 2.5（历史数据表明春节流量是平时2.5倍）
- 最终容量 = 预测DAU × 2.5 × 单用户资源消耗
*/
```

---

### 14.2 弹性伸缩策略

#### 14.2.1 HPA / VPA / KEDA

```yaml
# HPA（Horizontal Pod Autoscaler）- 水平扩缩
# 适用：无状态服务（API网关、Web服务）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: game-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: game-api
  minReplicas: 3
  maxReplicas: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # 扩容冷却30秒
      policies:
        - type: Percent
          value: 100                    # 每次最多翻倍
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容冷却5分钟（防抖动）
      policies:
        - type: Percent
          value: 10                     # 每次最多缩10%
          periodSeconds: 60
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60       # CPU > 60% 触发扩容
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"         # 每Pod QPS > 1000 触发扩容
```

```yaml
# VPA（Vertical Pod Autoscaler）- 垂直扩缩
# 适用：有状态服务（数据库代理、缓存节点）
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: redis-proxy-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: redis-proxy
  updatePolicy:
    updateMode: "Auto"        # 自动调整（会重启Pod）
  resourcePolicy:
    containerPolicies:
      - containerName: redis-proxy
        minAllowed:
          cpu: "500m"
          memory: "512Mi"
        maxAllowed:
          cpu: "4"
          memory: "8Gi"
        controlledResources: ["cpu", "memory"]
```

```yaml
# KEDA（Kubernetes Event-Driven Autoscaling）- 事件驱动扩缩
# 适用：消息消费者、异步任务处理器
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: reward-consumer
spec:
  scaleTargetRef:
    name: reward-consumer
  minReplicaCount: 1
  maxReplicaCount: 30
  pollingInterval: 10
  triggers:
    # 基于Kafka消费积压扩缩
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: reward-group
        topic: reward-events
        lagThreshold: "100"          # 积压超过100条就扩容
    # 基于Prometheus自定义指标扩缩
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: pending_tasks
        threshold: "50"
        query: |
          sum(pending_tasks{service="reward"})
```

#### 14.2.2 预测式扩容

```go
package autoscale

import (
    "math"
    "time"
)

type PredictiveScaler struct {
    historyWindow time.Duration // 历史窗口（如7天）
    forecastAhead time.Duration // 提前预测时间（如10分钟）
    metricsStore  MetricsStore
}

// PredictAndScale 基于历史模式预测并提前扩容
func (s *PredictiveScaler) PredictAndScale(serviceName string) *ScaleDecision {
    now := time.Now()
    targetTime := now.Add(s.forecastAhead)

    // 1. 获取上周同时段的历史数据
    lastWeekSameTime := targetTime.Add(-7 * 24 * time.Hour)
    historicalQPS := s.metricsStore.GetQPS(serviceName, lastWeekSameTime, 30*time.Minute)

    // 2. 获取本周同时段趋势
    recentQPS := s.metricsStore.GetQPS(serviceName, now.Add(-1*time.Hour), 1*time.Hour)

    // 3. 加权预测：历史模式60% + 近期趋势40%
    predictedQPS := historicalQPS*0.6 + recentQPS*1.1*0.4

    // 4. 计算所需实例数
    currentInstances := s.metricsStore.GetCurrentInstances(serviceName)
    currentQPS := s.metricsStore.GetCurrentQPS(serviceName)
    qpsPerInstance := currentQPS / float64(currentInstances)

    neededInstances := int(math.Ceil(predictedQPS / qpsPerInstance * 1.3)) // 30%余量

    if neededInstances > currentInstances {
        return &ScaleDecision{
            Action:    "scale_up",
            Current:   currentInstances,
            Target:    neededInstances,
            Reason:    fmt.Sprintf("predicted QPS %.0f in %v", predictedQPS, s.forecastAhead),
            Confidence: s.calculateConfidence(historicalQPS, recentQPS),
        }
    }

    return &ScaleDecision{Action: "no_change"}
}

// calculateConfidence 计算预测置信度
func (s *PredictiveScaler) calculateConfidence(historical, recent float64) float64 {
    // 历史与近期偏差越小，置信度越高
    deviation := math.Abs(historical-recent) / math.Max(historical, 1)
    return math.Max(0, 1-deviation)
}

/*
预测式扩容 vs 被动式扩容对比：

被动式：
  14:00 流量开始上涨
  14:02 CPU超过60%阈值，触发HPA
  14:03 新Pod调度+启动
  14:05 新Pod就绪，开始接流量
  → 中间5分钟处于过载状态

预测式：
  13:50 预测14:00会有流量高峰（基于上周同时段数据）
  13:50 提前扩容，新Pod开始启动
  13:55 新Pod就绪
  14:00 流量到来时已有足够容量
  → 零过载时间
*/
```

---

### 14.3 FinOps实践

#### 14.3.1 资源利用率优化

```go
package finops

type ResourceOptimizer struct {
    metricsStore MetricsStore
}

type OptimizationReport struct {
    Service          string
    CurrentCost      float64
    OptimizedCost    float64
    SavingPercentage float64
    Recommendations  []Recommendation
}

type Recommendation struct {
    Type        string // "rightsize", "spot", "reserved", "idle"
    Description string
    Saving      float64
}

// Analyze 分析服务资源利用率并给出优化建议
func (o *ResourceOptimizer) Analyze(serviceName string, window time.Duration) *OptimizationReport {
    report := &OptimizationReport{Service: serviceName}

    // 1. CPU利用率分析
    avgCPU := o.metricsStore.GetAvgCPUUtilization(serviceName, window)
    peakCPU := o.metricsStore.GetP99CPUUtilization(serviceName, window)

    if avgCPU < 20 && peakCPU < 50 {
        report.Recommendations = append(report.Recommendations, Recommendation{
            Type:        "rightsize",
            Description: fmt.Sprintf("CPU avg %.1f%%, peak %.1f%% → 建议降配50%%", avgCPU, peakCPU),
            Saving:      o.calculateRightsizeSaving(serviceName, 0.5),
        })
    }

    // 2. 内存利用率分析
    avgMem := o.metricsStore.GetAvgMemUtilization(serviceName, window)
    if avgMem < 30 {
        report.Recommendations = append(report.Recommendations, Recommendation{
            Type:        "rightsize",
            Description: fmt.Sprintf("Memory avg %.1f%% → 建议降配至当前的60%%", avgMem),
            Saving:      o.calculateRightsizeSaving(serviceName, 0.4),
        })
    }

    // 3. 空闲实例检测
    idleInstances := o.metricsStore.GetIdleInstances(serviceName, window)
    if len(idleInstances) > 0 {
        report.Recommendations = append(report.Recommendations, Recommendation{
            Type:        "idle",
            Description: fmt.Sprintf("发现%d个空闲实例，建议缩容", len(idleInstances)),
            Saving:      o.calculateIdleSaving(idleInstances),
        })
    }

    return report
}
```

#### 14.3.2 Spot实例策略

```go
type SpotStrategy struct {
    OnDemandRatio float64 // 按需实例比例（保底）
    SpotPools     []SpotPool
}

type SpotPool struct {
    InstanceType string
    AZ           string
    MaxPrice     float64 // 最高竞价
    Weight       int     // 权重（用于多池分散风险）
}

/*
Spot实例最佳实践：

1. 混合部署策略：
   ┌─────────────────────────────┐
   │  30% On-Demand（保底容量）    │  ← 保证最低服务可用
   │  70% Spot（弹性容量）         │  ← 成本节省60-80%
   └─────────────────────────────┘

2. 多实例类型+多AZ分散风险：
   Pool 1: c5.xlarge  / us-east-1a
   Pool 2: c5.2xlarge / us-east-1b
   Pool 3: m5.xlarge  / us-east-1c
   Pool 4: c6i.xlarge / us-east-1a
   → 任一池被回收，其他池仍可用

3. 中断处理：
*/

// HandleSpotInterruption Spot实例中断处理
func HandleSpotInterruption(instanceID string) {
    // AWS会提前2分钟发出中断通知
    
    // 1. 标记节点为不可调度
    kubectl.CordonNode(instanceID)
    
    // 2. 优雅驱逐Pod（触发PreStop Hook）
    kubectl.DrainNode(instanceID, 90*time.Second)
    
    // 3. 通知HPA立即扩容补充容量
    notifyHPA("scale_up", 1)
    
    log.Infof("spot instance %s interrupted, drain completed", instanceID)
}

/*
适用场景：
✅ 无状态服务（API、Web、消息消费者）
✅ 批处理任务（日志分析、数据ETL）
✅ CI/CD构建节点
✅ 开发/测试环境

❌ 数据库（有状态、不可中断）
❌ 核心支付链路（可用性要求极高）
❌ 长时间运行的单体任务
*/
```

#### 14.3.3 Reserved Instance / Savings Plans策略

```
Reserved Instance（RI）选购决策框架：

Step 1: 分析基线负载
  - 统计过去90天的最低资源使用量
  - 这部分是"确定会用"的容量 → 适合RI

Step 2: 选择承诺类型
  ┌────────────────┬────────────┬────────────┬────────────┐
  │    类型         │  折扣力度   │  灵活性     │  适用场景   │
  ├────────────────┼────────────┼────────────┼────────────┤
  │ 1年 标准RI      │  ~40%      │  低         │  稳定负载   │
  │ 3年 标准RI      │  ~60%      │  很低       │  长期项目   │
  │ 1年 可转换RI    │  ~30%      │  中         │  不确定规格  │
  │ Savings Plans  │  ~35-45%   │  高         │  多服务混合  │
  └────────────────┴────────────┴────────────┴────────────┘

Step 3: 覆盖率目标
  - 基线负载的80%用RI覆盖
  - 剩余20%用On-Demand兜底
  - 弹性部分用Spot

最终成本结构示例：
  基线容量（80台）：RI覆盖 → 月费$8,000（原价$20,000，节省60%）
  弹性容量（0-40台）：Spot → 平均月费$2,000（原价$10,000，节省80%）
  兜底容量（10台）：On-Demand → 月费$2,500
  总计：$12,500/月（原价$32,500，整体节省62%）
```

```go
// RI覆盖率监控
type RICoverageMonitor struct {
    cloudProvider CloudAPI
}

type CoverageReport struct {
    TotalInstances    int
    RICovered         int
    SpotInstances     int
    OnDemandInstances int
    CoverageRate      float64 // RI覆盖率
    WastedRI          int     // 未使用的RI数量
    MonthlyCost       float64
    PotentialSaving   float64
}

func (m *RICoverageMonitor) GenerateReport() *CoverageReport {
    instances := m.cloudProvider.ListInstances()
    ris := m.cloudProvider.ListReservedInstances()

    report := &CoverageReport{}
    for _, inst := range instances {
        report.TotalInstances++
        switch inst.PricingModel {
        case "reserved":
            report.RICovered++
        case "spot":
            report.SpotInstances++
        default:
            report.OnDemandInstances++
        }
    }

    // 检查是否有未使用的RI
    for _, ri := range ris {
        if !ri.IsUtilized {
            report.WastedRI++
        }
    }

    report.CoverageRate = float64(report.RICovered) / float64(report.TotalInstances) * 100

    // 如果覆盖率低于70%，计算潜在节省
    if report.CoverageRate < 70 {
        unconverted := report.OnDemandInstances - report.TotalInstances*20/100 // 保留20% OD
        if unconverted > 0 {
            report.PotentialSaving = float64(unconverted) * 150 // 每台每月可节省$150
        }
    }

    return report
}

/*
FinOps月度Review Checklist：
□ RI利用率 > 95%（未使用的RI = 纯浪费）
□ RI覆盖率 > 70%（基线负载应被RI覆盖）
□ Spot中断率 < 5%（多池策略是否有效）
□ 资源利用率 > 40%（低于40%需rightsize）
□ 闲置资源清理（未使用的EBS、EIP、快照等）
□ 数据传输成本（跨AZ/跨Region流量）
*/
```

---

## 15. 面试题自查

以下问题按“从基础到治理、从技术到协同”的顺序组织，适合作为 SRE 面试系统自查清单。

### Q1: Google SRE的四大黄金指标是什么？分别监控什么？

**答案**：

| 指标 | 监控内容 | 示例阈值 |
|------|----------|----------|
| **Latency** | 请求响应时间 | P99 < 200ms |
| **Traffic** | 请求吞吐量 | QPS波动 < 30% |
| **Errors** | 错误率 | 5xx < 0.1% |
| **Saturation** | 资源饱和度 | CPU/内存 < 80% |

### Q2: 如何设计一个异地多活架构？核心挑战是什么？

**答案**：

**设计要点**：
1. **数据分片**：按用户ID hash分片，同一用户数据在同一机房，避免跨机房事务
2. **数据同步**：Canal订阅binlog，通过MQ异步同步到其他机房
3. **流量调度**：GeoDNS就近路由，机房故障时自动切换
4. **读写分离**：写入主机房，读取本地机房（可接受延迟）

**核心挑战**：
- 分布式数据一致性
- 跨机房网络延迟
- 机房故障的快速检测和切换

### Q3: SLO和Error Budget是什么？如何应用？

**答案**：

**SLO（Service Level Objective）**：服务级别目标
- 可用性：99.95%（每月最多停机21.6分钟）
- P99延迟：< 200ms
- 错误率：< 0.1%

**Error Budget**：错误预算 = 1 - SLO
- 99.95% SLO → 0.05%错误预算 → 21.6分钟/月

**应用方式**：
- 预算充足：可以发布新功能、做变更
- 预算耗尽：冻结变更，专注稳定性提升

### Q4: 熔断器的三种状态是什么？如何配置阈值？

**答案**：

**三种状态**：
- **Closed（关闭）**：正常工作，记录错误率
- **Open（打开）**：错误率超阈值，拒绝所有请求，返回降级结果
- **Half-Open（半开）**：等待超时后，放行少量请求测试

**阈值配置示例**：
```go
ReadyToTrip: func(counts gobreaker.Counts) bool {
    return counts.Requests >= 10 && 
           float64(counts.TotalFailures)/float64(counts.Requests) >= 0.5
}
```
- 10次请求以上
- 错误率 ≥ 50%

### Q5: 缓存穿透、缓存击穿、缓存雪崩的区别和解决方案？

**答案**：

| 问题 | 定义 | 解决方案 |
|------|------|----------|
| **缓存穿透** | 查询不存在的key | 缓存空值、布隆过滤器 |
| **缓存击穿** | 热key过期瞬间大量请求 | singleflight、永不过期+后台刷新 |
| **缓存雪崩** | 大量key同时过期 | 过期时间加随机值、多级缓存 |

### Q6: 如何优化P99延迟从500ms降到100ms？

**答案**：

**排查流程**：
1. **火焰图分析**：定位CPU热点函数
2. **链路追踪**：找到耗时最长的Span（DB/RPC）
3. **慢查询分析**：EXPLAIN优化SQL，添加索引

**优化手段**：
1. **数据库**：索引优化、读写分离、批量查询
2. **缓存**：热数据缓存、预热
3. **并发**：串行改并行、连接池复用
4. **降级**：超时快速失败、返回降级结果

### Q7: 如何降低监控成本？Prometheus指标爆炸怎么办？

**答案**：

**降低基数**：
```go
// ❌ 高基数label
requestCounter.WithLabelValues(userID, path).Inc()

// ✅ 只保留必要label
requestCounter.WithLabelValues(method, path, status).Inc()
```

**数据分级存储**：
- 核心指标：15s采样，保留30天
- 普通指标：30s采样，保留7天

**预聚合**：Recording Rules提前计算

### Q8: 灰度发布如何设计？什么时候需要回滚？

**答案**：

**灰度策略**：
1. 5%流量 → 观察5分钟 → 检查成功率/延迟
2. 20%流量 → 观察10分钟
3. 50%流量 → 观察10分钟
4. 100%全量

**自动回滚条件**：
- 成功率 < 99%
- P99延迟超过基线30%
- 连续3次检查失败

### Q9: 故障响应流程是什么？P0故障如何处理？

**答案**：

**响应流程**：
1. **5分钟内响应**：值班人员确认告警
2. **建立作战室**：自动拉群，通知相关人员
3. **止血优先**：回滚、降级、扩容
4. **定位根因**：日志、监控、链路追踪
5. **恢复服务**：修复问题、验证
6. **事后复盘**：48小时内完成复盘报告

**P0故障标准**：全站不可用/数据丢失，响应时间5分钟

### Q10: Chaos Engineering的核心原则是什么？如何实施？

**答案**：

**核心原则**：
1. **建立稳态假设**：定义正常状态指标
2. **设计实验**：模拟真实故障场景
3. **在生产环境验证**：灰度注入故障
4. **自动化**：持续演练

**实施步骤**：
1. 选择故障类型：Pod删除、网络延迟、磁盘满
2. 定义预期结果：切换时间<30s，错误率<1%
3. 执行实验：Chaos Mesh/Litmus
4. 观察验证：监控指标变化
5. 问题修复：发现问题则改进系统

### Q11: RPO和RTO是什么？不同容灾等级如何选择？

**答案**：

| 指标 | 含义 | 示例 |
|------|------|------|
| **RPO** | 最多丢失多长时间数据 | RPO=0表示不能丢数据 |
| **RTO** | 故障后多久恢复服务 | RTO=5min表示5分钟内恢复 |

**容灾等级选择**：
- **备份容灾**：RPO=24h，RTO=数小时，非核心业务
- **同城双活**：RPO=秒级，RTO=分钟级，一般业务
- **异地多活**：RPO=0，RTO=秒级，核心金融业务

### Q12: 如何设计自动限流？有哪些限流算法？

**答案**：

**限流算法**：
1. **固定窗口**：每秒限制N次，简单但有边界问题
2. **滑动窗口**：解决边界问题，更平滑
3. **令牌桶**：允许突发流量，桶满后平滑限流
4. **漏桶**：恒定速率处理，平滑流量

**动态限流**：
```go
// 根据CPU使用率动态调整
if cpuUsage > 0.8 {
    currentLimit = baseLimit * (1 - (cpuUsage-0.8)/0.2*0.5)
}
```

### Q13: 如何做好容量规划？

**答案**：

**数据收集**：
- 历史流量峰值（大促、活动）
- 资源使用率P99
- 业务增长预期

**计算公式**：
```
所需容量 = 峰值流量 × 冗余系数(1.5) / 单实例QPS
```

**验证方法**：
- 压测验证单实例QPS
- 预扩容测试
- 定期复审调整

### Q14: 日志和链路追踪如何关联？如何快速定位问题？

**答案**：

**关联方式**：
- 在日志和链路中使用相同的`trace_id`
- 请求入口生成trace_id，全链路传递

**定位流程**：
1. **告警触发**：错误率上升
2. **Kibana查日志**：`level:ERROR AND @timestamp:[now-5m TO now]`
3. **获取trace_id**：从日志中提取
4. **Jaeger查链路**：定位具体哪个服务/操作慢

### Q15: 监控全景图包含哪些层次？每层监控什么？

**答案**：

| 层次 | 监控内容 | 工具 |
|------|----------|------|
| **用户侧** | 拨测、真实用户体验(RUM) | Blackbox、前端埋点 |
| **应用层** | QPS/延迟/错误率/链路 | Prometheus、Jaeger |
| **中间件层** | DB/Cache/MQ性能 | mysqld_exporter、redis_exporter |
| **基础设施** | CPU/内存/网络/磁盘 | node_exporter、cadvisor |

**核心原则**：从用户视角出发，自顶向下监控

### Q16: 如何定义一个靠谱的 SLI/SLO？为什么很多团队 SLO 做不起来？

**答案**：

- 靠谱的 SLI 必须和用户体验直接相关，比如成功率、可用性、P99 延迟，而不是单纯 CPU、内存利用率。
- SLO 是围绕这些 SLI 设定的团队目标，比如支付接口月可用性 99.95%。
- 设计时要兼顾可测量性、业务重要性和历史现实水平，不能拍脑袋定目标。

很多团队做不起来的根因有三类：
1. 指标选错了，选成 CPU/内存这类资源指标，而不是用户可感知指标。
2. 目标和现实脱节，历史上只能做到 99.5%，却直接定 99.99%。
3. SLO 没有进入变更流程，预算耗尽后团队还在照常发版。

### Q17: 什么是 Burn Rate？为什么线上告警不能只看错误率阈值？

**答案**：

Burn Rate 是错误预算消耗速度，公式是 `当前错误率 / 允许错误率`。如果 SLO 是 99.95%，允许错误率是 0.05%，当前错误率 1% 时 Burn Rate=20，意味着预算正在以 20 倍速度消耗。

只看错误率阈值的问题在于：
- 不同服务的 SLO 不同，绝对错误率阈值没有统一意义。
- 只看 5 分钟错误率容易被毛刺误导，也抓不到慢性退化。
- Burn Rate 直接反映对 SLO 的破坏程度，更适合和 Error Budget 联动。

### Q18: RED 和 USE 方法分别适合什么层次？面试里怎么把它们串起来？

**答案**：

RED 适合请求型服务，关注 Rate、Errors、Duration；USE 适合资源型对象，关注 Utilization、Saturation、Errors。面试回答不要把它们当成二选一，而要讲清楚排障链路：先用 RED 发现用户请求是否受损，再用 USE 找资源瓶颈。例如 API 的 P99 上升，进一步看数据库连接池等待、磁盘队列、CPU run queue 是否饱和。

### Q19: 什么是 Toil？如果团队 70% 时间都在救火，你会怎么治理？

**答案**：

Toil 是重复、手工、可自动化、低杠杆的运维劳动。团队 70% 时间都在救火时，不能继续只谈规范，要先做三件事：
1. 按频次和时长统计 Toil Top 10，先打最耗时的几个点。
2. 把高频恢复动作做成自动化，比如自动扩容、自动回滚、自愈脚本。
3. 清理无效告警，建立 runbook，让一线能快速止血，减少反复人工排查。

治理目标不是把故障处理更快，而是让这类故障以后不再需要人工处理。

### Q20: Runbook 和 Playbook 有什么区别？一个高质量 Runbook 应该长什么样？

**答案**：

Runbook 面向具体操作步骤，强调可执行；Playbook 面向复杂事件协同，强调角色分工和决策路径。高质量 Runbook 应至少包含：故障现象、影响判断、排查顺序、止血动作、回滚方式、验证方法、升级条件，以及相关 dashboard/log/trace 链接。写成大段原理说明没有用，值班时需要的是 5 分钟内能落地执行的步骤。

### Q21: 值班体系应该怎么设计，才能避免把团队拖入告警疲劳？

**答案**：

核心是三点：
1. 告警必须可行动，没有动作方案的告警不应该直接打人。
2. 值班职责要分层，一线止血、二线定位、负责人决策，避免所有问题都压给一个人。
3. 用数据治理告警噪音，比如统计每周高优告警量、误报率、重复告警数、夜间告警数。

如果一个团队的告警系统让人习惯性忽略，那么它已经失效了。

### Q22: 变更失败率（Change Failure Rate）是什么？你会怎么把它接入发布治理？

**答案**：

变更失败率是发布后引发故障、回滚、热修复的变更占比，也是 DORA 指标之一。接入治理时要做三件事：
1. 明确定义失败标准，比如发布后 24 小时内触发 Sev1/Sev2、自动回滚、人工回滚都算失败。
2. 在发布平台自动记录变更单、版本号、灰度阶段、回滚记录。
3. 把 Change Failure Rate 与 Error Budget 联动，预算紧张时自动提高发布门槛。

### Q23: 什么是 Cell-Based Architecture（单元化架构）？它为什么适合高可用场景？

**答案**：

单元化架构是把系统拆成多个相对独立的小单元，每个单元拥有自己的应用、副本和部分数据，故障被限制在单元内部，不会全站扩散。它的核心价值是故障隔离和可控扩容，典型场景是电商、支付、出行这类超大规模系统。

代价也很明确：路由、数据分片、跨单元查询、容量均衡都会更复杂。如果业务规模还没到这个级别，贸然做单元化只会增加复杂度。

### Q24: 级联故障一般是怎么发生的？SRE 会优先在哪些点做隔离？

**答案**：

典型路径是：下游变慢 -> 上游线程池/连接池堆积 -> 重试放大流量 -> 更多依赖被拖死，最后全链路雪崩。优先隔离点通常有：
1. 超时和重试预算，避免无限等待和重试风暴。
2. 线程池/连接池舱壁隔离，避免一个依赖拖垮整个进程。
3. 熔断和降级，确保非核心链路先失败，核心链路保活。
4. 流量分级和限流，把故障流量挡在入口。

### Q25: 如果支付成功率从 99.99% 掉到 99.9%，但 CPU、内存都正常，你会怎么排查？

**答案**：

这是典型的“资源指标正常但用户指标受损”场景，排查顺序应该是：
1. 先看 RED：错误类型、失败接口、地域、机房、版本维度是否集中。
2. 再看链路：是网关、风控、支付路由、银行通道还是数据库哪个环节放大了失败。
3. 再看饱和度：连接池等待、线程池队列、下游超时、外部依赖错误码，而不是只盯 CPU。
4. 同时检查最近变更、灰度发布、配置变更和第三方通道波动。

资深面试官通常要的不是“查日志”，而是你有没有稳定的排障优先级和止血策略。

### Q26: 一个可执行的 Error Budget Policy 应该长什么样？预算耗尽后哪些动作必须自动发生？

**答案**：

一份可执行的 Error Budget Policy 至少要回答四件事：谁来判断预算状态、预算分几个档位、每个档位对应什么发布权限、预算恢复前允许做哪些例外变更。它不该只是“预算低了注意一点”，而应该能直接映射到发布平台和审批流。

预算耗尽后，通常至少要自动触发这些动作：
1. 冻结非修复性发布，只保留回滚、热修和必要配置调整。
2. 提高灰度门禁，核心 SLI 一旦继续恶化立即停止推进。
3. 自动通知值班经理、服务负责人和变更平台，避免团队各自继续发版。
4. 进入稳定性专项，优先处理告警治理、容量短板、重复故障和高频 Toil。

如果预算耗尽后系统行为和预算充足时完全一样，那这套 SLO 体系本质上还没有落到工程治理里。

### Q27: 升级策略（Escalation Policy）如何设计才算成熟？什么情况下必须升级到更高层？

**答案**：

成熟的升级策略要同时覆盖时间维度、影响维度和职责维度。时间维度上，要写清楚“多久没有止血就必须升级”；影响维度上，要写清楚哪些故障级别、哪些业务链路、哪些资损风险必须立即升级；职责维度上，要明确一线值班、二线研发、负责人、跨团队支持各自负责什么。

必须升级的典型场景包括：
1. 一线在预定时间窗口内没有形成有效止血动作。
2. 核心业务 SLI 持续劣化，Burn Rate 很高，说明影响在快速扩大。
3. 故障跨多个依赖域，已经不是单服务可以独立处理的问题。
4. 出现资损、安全、合规或舆情风险，需要更高层参与决策。

面试里如果只回答“出问题就拉群”，通常不够，关键在于你有没有把升级条件制度化、阈值化。

### Q28: Postmortem 的 Action Item 应该怎么管理，才能避免复盘写完就结束？

**答案**：

很多团队复盘失败，不是不会写报告，而是行动项没有进入真实的工程跟踪系统。正确做法是把每个 Action Item 写成可验收的工程任务，至少包含 owner、截止时间、优先级、验收标准和关联事故编号。

管理上通常要抓三件事：
1. **分类型**：区分立即止血、短期修复、长期治理，避免所有事情都堆在一个桶里。
2. **可验收**：例如“接入双窗口 Burn Rate 告警并完成演练”，而不是“加强监控”。
3. **可追踪**：每周复盘行动项关闭率、逾期项和重复事故率，直到验证系统性问题真的被修掉。

无责复盘不等于没有责任落点，它只是把责任从“找人背锅”转成“把机制修掉”。

### Q29: DORA 指标和 SRE 指标是什么关系？为什么不能只看发布频率和 MTTR？

**答案**：

DORA 主要衡量交付体系，比如发版频率、交付前置时间、变更失败率、恢复时间；SRE 指标主要衡量服务可靠性，比如 SLI/SLO、可用性、延迟、Error Budget。两者交集在于都关心故障和恢复，但观察角度不同。

不能只看发布频率和 MTTR 的原因是：
1. 发布很快，不代表用户体验稳定，可能只是更快地把问题发上去。
2. MTTR 很低，也不代表问题少，可能只是团队长期在“快速救火”。
3. 没有 SLI/SLO，你无法判断这些交付行为到底有没有透支可靠性。

更成熟的表达是：DORA 负责回答“交付系统运转得怎么样”，SRE 指标负责回答“用户服务质量守住了没有”。

### Q30: SLO 已经定义好了，为什么团队还是经常在事故后“各说各话”？

**答案**：

常见原因不是没有指标，而是没有统一口径和动作映射。比如 A 团队看成功率，B 团队看错误码，C 团队看 CPU，事故时很难形成一致决策。要避免“各说各话”，需要把 SLI 定义、查询语句、阈值解释、值班动作写成统一规范，并在发布门禁和告警策略中强制执行。

### Q31: 如何设计一个“可演练”的应急响应体系，而不是只写流程图？

**答案**：

可演练体系至少包含四部分：明确角色（值班、一线、二线、负责人）、标准场景（依赖超时、发布回滚、容量耗尽）、可执行 runbook、以及演练后的量化复盘。演练不应只验证“流程有没有跑完”，还要看 MTTA/MTTR 是否下降、升级路径是否顺畅、工具链是否可用。

### Q32: 为什么很多团队做了 Chaos Engineering 仍然在真实故障中表现很差？

**答案**：

因为实验只做了“注入动作”，没有把结果接入治理闭环。常见问题是：实验覆盖面窄、只在低风险时段做、没有明确验收指标、演练后没有行动项落地。Chaos 的价值不在于“制造过故障”，而在于持续暴露系统脆弱点并推动修复，否则只是一次技术秀。

### Q33: 作为面试官，如何判断候选人是否真的做过 SRE 治理而不是背模板？

**答案**：

我通常看三点：
1. 能否说出一个具体事故里“先做了什么止血动作、为什么这么选”。
2. 能否讲清楚一条治理改进从指标发现到流程落地的完整链路。
3. 能否给出带约束的取舍，而不是“全都要”。

真正做过治理的人，回答里会自然带出指标口径、决策阈值、协作角色和复盘闭环，而不是只列一串名词。

### Q34: 给一个 P1 事故设计升级（Escalation）模板，你会怎么写到“值班可直接执行”？

**答案**：

模板的关键不是格式，而是让一线同学在高压场景下不用临场拼凑决策。一个可执行模板至少包含：
1. **触发条件**：例如核心接口成功率连续 5 分钟低于 99.5%，或 Burn Rate > 10。
2. **时间阈值**：例如 10 分钟内未止血必须升级到二线，20 分钟升级到负责人。
3. **角色分工**：一线止血、二线定位、负责人做回滚/限流/跨团队决策。
4. **升级路径**：拉群模板、电话顺序、必须通知的业务方和管理方。
5. **退出条件**：核心 SLI 恢复且稳定 30 分钟后降级响应。

面试里若只回答“出问题就拉群”，通常不够，重点是阈值化和动作化。

### Q35: Postmortem 的 Action Item 怎么验收才算“真的完成”？

**答案**：

Action Item 关闭不等于问题关闭，验收要看“措施是否生效”。建议按四步验收：
1. **工程完成**：代码、配置、规则或流程改动已经上线。
2. **能力验证**：通过演练或真实流量验证措施可触发、可执行。
3. **指标改善**：对应指标有变化，如 MTTA 降低、误报率下降、同类告警减少。
4. **复发检查**：观察一个统计周期（如 2-4 周）确认同类事故未重复发生。

一个高质量验收标准应写成“可测语句”，例如“新增双窗口 Burn Rate 告警并在演练中 5 分钟内触发升级”。

### Q36: 你会如何设计一个“发布前稳定性门禁”清单，避免靠个人经验拍板？

**答案**：

发布前门禁建议分三层：
1. **技术层**：核心链路自动化测试通过、错误率/延迟基线未劣化、容量余量满足预估峰值。
2. **治理层**：最近周期 Error Budget 状态可接受，重大未关闭风险项有明确豁免审批。
3. **应急层**：回滚路径已验证、值班和升级责任人在线、Runbook 可用。

门禁要尽量系统化，避免口头判断。最实用的做法是把清单项接入发布平台，失败项直接阻断推进或强制走高等级审批。

### Q37: SLO 达标但用户投诉变多，可能是什么问题？你怎么修正 SLI 定义？

**答案**：

这通常说明“测到了系统稳定性，但没测到用户真实体验”。典型问题有：
1. SLI 口径过粗，只看总体成功率，掩盖了核心路径或关键人群的劣化。
2. 聚合方式不合理，只看均值或全量，忽略 P95/P99 和地域、运营商、端类型分层。
3. 技术指标达标但业务指标受损，比如下单成功率正常但支付回调延迟导致用户感知失败。

修正方式：
1. 把 SLI 从“系统视角”改成“用户旅程视角”，拆分关键链路。
2. 增加分层 SLI（地域、机房、版本、客户端）和尾延迟指标。
3. 将投诉、工单、业务失败码纳入辅助观测，形成口径校准闭环。

### Q38: 当 Error Budget 临近耗尽，但业务要求大促前必须发版，你如何做“可解释的例外策略”？

**答案**：

例外策略不是“口头特批”，而是有边界的风险交易。建议做四件事：
1. 明确例外范围：只允许与大促强相关的最小变更集，禁止顺带需求。
2. 提高门槛：缩小灰度批次、加严回滚阈值、缩短观察窗口。
3. 强化兜底：值班升级到位、回滚脚本预演、关键依赖预扩容。
4. 可追责记录：记录审批人、风险说明、触发条件、复盘验收项。

“可解释”意味着任何人都能复述这次例外为什么发生、承担了什么风险、如何避免常态化。

### Q39: 一次事故里技术已经恢复了，但舆情还在扩大，SRE 在沟通链路里应该承担什么职责？

**答案**：

SRE 不只负责技术恢复，还要负责“状态可信传达”。核心职责：
1. 提供统一事实口径：影响范围、恢复时间线、当前风险、下一次更新点。
2. 维持状态更新节奏：固定频率同步，不让业务和客服在信息真空中各自猜测。
3. 给出可执行建议：哪些入口可恢复、哪些操作暂缓、客服话术边界。
4. 事故后补齐对外沟通改进项：把信息流断点纳入 postmortem 行动项。

技术恢复是“服务可用”，沟通恢复是“组织可控”，两者都属于事故治理完成标准。

### Q40: K8s 里 readiness 正常但延迟很高，SRE 通常先查哪三类指标？

**答案**：

先看三类最容易暴露长尾的信号：
1. **CPU 与调度**：`us/sys/wa/st`、run queue、上下文切换，判断是真忙、在等 I/O，还是被抢占。
2. **应用容量**：线程池、连接池、下游 RPC 超时、GC、工作队列堆积。
3. **网络与存储**：网卡丢包、连接队列、磁盘 `await/%util`、DNS 解析耗时。

readiness 只能说明“探针请求还能过”，不能说明主业务路径没有长尾。很多故障都是探针正常，但主链路已经接近饱和。

### Q41: 为什么 PDB 配了也可能出现业务不可用？请给一个真实故障路径。

**答案**：

PDB 只限制“自愿驱逐”数量，不保证业务一定可用。常见故障路径：
1. 服务副本数本来就偏少，比如只有 2 个副本。
2. 节点升级时 PDB 允许驱逐 1 个，看上去合理。
3. 剩下那个 Pod 虽然还在，但刚好本身有 GC 长尾、连接池耗尽或下游异常。
4. 结果从 Kubernetes 视角看“满足 PDB”，从用户视角看服务已经不可用。

真实治理上不能只看 PDB，还要联动最小副本数、拓扑分散、探针质量和容量余量。

### Q42: NetworkPolicy 全开后 DNS 间歇失败，怎么快速判断是策略问题还是 CoreDNS 容量问题？

**答案**：

排查顺序建议固定下来：
1. 先在故障 Pod 内 `dig`，判断是超时、SERVFAIL 还是 NXDOMAIN。
2. 抓 `53` 端口包，看请求有没有发出去、响应有没有回来。
3. 如果请求没出去，优先查 NetworkPolicy、CNI、节点本地规则。
4. 如果请求发出去了但响应慢，再看 CoreDNS QPS、CPU、内存、上游递归 DNS 和 conntrack。

关键不是“先怀疑谁”，而是先分清楚问题发生在**策略阻断**还是**DNS 服务过载**。

### Q43: HTTP/3(QUIC) 在什么场景下明显优于 HTTP/2，什么场景收益不明显？

**答案**：

明显优于 HTTP/2 的场景：
1. 移动弱网，网络抖动和切网频繁。
2. 跨公网高 RTT，握手成本显著。
3. 丢包率不高但一旦丢包就很影响长尾体验的场景。

收益不明显的场景：
1. 机房内稳定 RPC。
2. 主要瓶颈在应用处理、数据库、缓存，而不是网络。
3. 现网安全和观测体系对 UDP/QUIC 支持不足，运维成本可能高于收益。

面试里更好的回答不是“HTTP/3 更先进”，而是说明你会先判断**网络是不是当前瓶颈**。

### Q44: 你主导过的最失败一次架构决策是什么？如果重来一次你会先验证什么？

**答案**：

这类题没有标准事故，但高质量回答通常要包含四个要素：
1. **当时为什么做这个决策**：背景、目标、约束是什么。
2. **后来为什么失败**：是复杂度被低估、观测缺失、容量评估错误，还是组织协同跟不上。
3. **造成了什么影响**：性能、稳定性、成本、研发效率哪个被拖垮了。
4. **如果重来会先验证什么**：例如先做流量压测、灰度实验、故障演练、可回滚设计，而不是直接全量落地。

SRE 面试里，这题不是看你是否“没犯过错”，而是看你是否能把失败沉淀成可复用的验证方法和治理原则。

---

## 总结

**高级SRE的核心能力**：

1. **容灾架构**：同城双活、异地多活，RPO/RTO设计
2. **故障自愈**：熔断、限流、自动扩容、自动回滚
3. **监控体系**：全链路监控（用户侧→应用→中间件→基础设施）
4. **性能优化**：CPU、内存、数据库、缓存、网络优化
5. **成本优化**：资源利用率、存储分级、流量优化
6. **稳定性治理**：SLO、Error Budget、变更管理、故障演练

**可进一步拓展的能力点**：
- ✅ 设计过异地多活架构
- ✅ 实施过自动化故障自愈
- ✅ 建设过完整的监控体系
- ✅ 优化过P99延迟和成本
- ✅ 进行过混沌工程实验

**从救火到架构师的进阶路径**：
```
初级SRE：监控告警、日志分析、故障排查
         ↓
中级SRE：自动化运维、性能优化、容量规划
         ↓
高级SRE：容灾架构、稳定性治理、成本优化
         ↓
SRE架构师：技术体系建设、规范制定、团队赋能
```


---
