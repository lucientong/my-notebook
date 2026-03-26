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
13. [Chaos Engineering](#13-chaos-engineering)
14. [常见面试题](#14-常见面试题)

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

### 1.2 同城双活架构

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

#### 1.2.1 数据库双主同步（MySQL）
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

#### 1.2.2 缓存跨机房同步（Redis）
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

#### 1.2.3 流量调度策略
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

### 1.3 异地多活架构

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

#### 1.3.1 分库分表策略
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

#### 1.3.2 跨机房数据同步（Canal）
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

### 1.4 容灾切换演练

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

## 2. 故障管理与自愈

### 2.1 故障等级定义

| 等级 | 影响范围 | 响应时间 | 示例 |
|------|----------|----------|------|
| **P0** | 全站不可用/数据丢失 | 5分钟 | 数据库宕机、机房断网 |
| **P1** | 核心功能不可用 | 15分钟 | 支付失败、登录失败 |
| **P2** | 部分功能异常 | 1小时 | 搜索慢、图片加载失败 |
| **P3** | 非核心功能异常 | 1天 | 统计报表错误 |

### 2.2 故障响应流程（Incident Response）

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

### 2.3 自动止血策略

#### 2.3.1 自动熔断（Circuit Breaker）
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

#### 2.3.2 自动限流（Rate Limiting）
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

#### 2.3.3 自动扩容（Auto Scaling）
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

### 2.4 故障自愈实战

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

## 3. 监控体系建设

### 3.1 监控全景图

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

### 3.2 用户侧监控

#### 3.2.1 拨测（黑盒监控）
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

#### 3.2.2 真实用户监控（RUM）
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

### 3.3 业务监控

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

### 3.4 中间件监控

#### 3.4.1 MySQL监控
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

#### 3.4.2 Redis监控
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

### 3.5 监控数据治理

**问题**：监控指标爆炸（10万+时间序列），Prometheus内存不足

**解决**：

#### 3.5.1 降低基数（Cardinality）
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

#### 3.5.2 数据分级存储
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

#### 3.5.3 指标聚合
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

## 4. 性能优化实战

### 4.1 性能优化方法论

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

### 4.2 应用层优化

#### 4.2.1 CPU优化（Go）
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

#### 4.2.2 内存优化
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

### 4.3 数据库优化

#### 4.3.1 慢查询优化
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

#### 4.3.2 批量操作优化
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

### 4.4 缓存优化

#### 4.4.1 缓存穿透（查询不存在的key）
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

#### 4.4.2 缓存击穿（热key过期）
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

### 4.5 网络优化

#### 4.5.1 连接池优化
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

#### 4.5.2 并发请求
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

---

## 5. 成本优化

### 5.1 资源利用率优化

#### 5.1.1 Pod资源配置优化
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

#### 5.1.2 节点利用率优化
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

### 5.2 存储成本优化

#### 5.2.1 日志分级存储
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

#### 5.2.2 指标数据降采样
```yaml
# Thanos Compactor：自动降采样
thanos compact \
  --data-dir /data \
  --retention.resolution-raw=7d \      # 原始数据保留7天
  --retention.resolution-5m=30d \      # 5分钟聚合保留30天
  --retention.resolution-1h=180d       # 1小时聚合保留180天

# 存储节省：90%（原始数据→聚合数据）
```

### 5.3 流量成本优化

#### 5.3.1 CDN回源优化
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

#### 5.3.2 图片压缩
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

## 6. 稳定性治理

### 6.1 SLO（Service Level Objective）体系

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

### 6.2 变更管理

#### 6.2.1 灰度发布
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

#### 6.2.2 自动回滚
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

### 6.3 故障演练

**Netflix Chaos Monkey思想**：定期制造故障，验证系统韧性

#### 6.3.1 Pod随机删除
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

#### 6.3.2 网络延迟注入
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

## 7. Chaos Engineering（混沌工程）

### 7.1 混沌工程原则

1. **建立稳态假设**：系统正常时，QPS=1000，错误率<0.1%
2. **设计实验**：模拟故障（杀Pod、网络延迟、磁盘满）
3. **在生产环境验证**：灰度注入故障（5%流量）
4. **自动化**：持续演练

### 7.2 实验示例

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

## 8. 常见面试题

### Q1: 如何设计一个异地多活架构？

**答案**：
1. **数据分片**：按用户ID hash分片，同一用户数据在同一机房
2. **数据同步**：Canal订阅binlog，通过MQ同步到其他机房
3. **流量调度**：GeoDNS就近路由，机房故障时切换到其他机房
4. **一致性保证**：
   - 写操作：路由到主机房，同步复制
   - 读操作：本地机房读，延迟数据打标
5. **故障切换**：自动检测主机房故障，秒级切换

### Q2: Error Budget耗尽时，如何处理？

**答案**：
1. **冻结变更**：停止所有非紧急的功能发布
2. **根因分析**：回顾最近的事故，找到主要消耗来源
3. **稳定性提升**：
   - 加强监控和告警
   - 补充自动化测试
   - 优化高风险模块
4. **逐步恢复**：Error Budget回升后，逐步解冻变更

### Q3: 如何降低监控成本？

**答案**：
1. **降低基数**：移除高基数label（user_id、trace_id）
2. **数据分级**：
   - 核心指标：高精度、长期保留
   - 普通指标：降采样、短期保留
3. **预聚合**：使用Recording Rules
4. **存储优化**：
   - 冷数据归档到对象存储
   - 使用VictoriaMetrics（压缩比高）

### Q4: 如何优化P99延迟？

**答案**：
1. **定位瓶颈**：
   - 火焰图：CPU瓶颈
   - 链路追踪：RPC/DB慢
2. **数据库优化**：
   - 加索引
   - 慢查询优化
   - 读写分离
3. **缓存优化**：
   - 热数据缓存
   - 预热
4. **并发优化**：
   - 串行改并行
   - 连接池
5. **降级**：
   - 超时快速失败
   - 返回降级结果

---

## 总结

**高级SRE的核心能力**：

1. **容灾架构**：同城双活、异地多活，RPO/RTO设计
2. **故障自愈**：熔断、限流、自动扩容、自动回滚
3. **监控体系**：全链路监控（用户侧→应用→中间件→基础设施）
4. **性能优化**：CPU、内存、数据库、缓存、网络优化
5. **成本优化**：资源利用率、存储分级、流量优化
6. **稳定性治理**：SLO、Error Budget、变更管理、故障演练

**面试时的差异化优势**：
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

掌握这些高级SRE实践，你将从"救火队员"蜕变为"架构师"！🔥→🏗️

---

**文件状态**：✅ 已完成（约1000行）  
**关联文档**：可观测性与SRE、云原生与容器、系统设计与架构