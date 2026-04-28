# API 设计与治理

---

## 📑 目录

### 一、API 设计范式
1. [RESTful API 设计规范](#1-restful-api-设计规范) ⭐⭐⭐
2. [gRPC 深入](#2-grpc-深入) ⭐⭐⭐
3. [GraphQL](#3-graphql) ⭐⭐

### 二、API 质量保障
4. [API 幂等性设计](#4-api-幂等性设计) ⭐⭐⭐
5. [API 版本管理](#5-api-版本管理) ⭐⭐
6. [API 文档与测试](#7-api-文档与测试) ⭐⭐

### 三、API 基础设施
7. [API 网关](#6-api-网关) ⭐⭐
8. [接口性能优化](#8-接口性能优化) ⭐⭐
9. [API 安全](#9-api-安全) ⭐⭐

### 四、综合实战
10. [实战案例：开放平台 API 体系设计](#10-实战案例设计一套完整的开放平台-api-体系) ⭐⭐⭐

### 五、面试题自查
11. [面试题自查](#面试题自查)

---

## 1. RESTful API 设计规范

> ⭐⭐⭐ 核心考点：REST 是面试中出镜率最高的 API 设计风格，需从 URL 设计、HTTP 方法语义、状态码、HATEOAS、版本管理五个维度掌握。

### 1.1 REST 核心约束

REST（Representational State Transfer）由 Roy Fielding 在 2000 年博士论文中提出，定义了六大架构约束：

| 约束 | 含义 | 作用 |
|------|------|------|
| Client-Server | 客户端与服务器分离 | 关注点分离，独立演进 |
| Stateless | 无状态 | 每次请求包含全部信息，服务端不保存会话 |
| Cacheable | 可缓存 | 响应声明可缓存性，减少交互 |
| Uniform Interface | 统一接口 | 资源标识、资源操作、自描述消息、HATEOAS |
| Layered System | 分层系统 | 客户端不知道直连还是经过中间层 |
| Code on Demand (可选) | 按需代码 | 服务端可返回可执行代码 |

### 1.2 URL 设计规范

**核心原则：URL 标识资源（名词），HTTP 方法标识操作（动词）。**

```
# ✅ 正确：用名词复数表示资源集合
GET    /api/v1/users
GET    /api/v1/users/123
POST   /api/v1/users
PUT    /api/v1/users/123
PATCH  /api/v1/users/123
DELETE /api/v1/users/123

# ✅ 嵌套资源（表达从属关系，最多两层）
GET    /api/v1/users/123/orders
GET    /api/v1/users/123/orders/456

# ❌ 错误：URL 中出现动词
GET    /api/v1/getUser?id=123
POST   /api/v1/createUser
POST   /api/v1/deleteUser/123
```

**URL 设计检查清单：**

| 规则 | 正确示例 | 错误示例 |
|------|----------|----------|
| 使用名词复数 | `/users` | `/user`, `/getUsers` |
| 使用小写字母 | `/user-profiles` | `/userProfiles`, `/User_Profiles` |
| 用连字符分隔 | `/user-profiles` | `/user_profiles` |
| 层级不超过 3 层 | `/users/123/orders` | `/users/123/orders/456/items/789/comments` |
| 不包含文件扩展名 | `/users/123` | `/users/123.json` |
| 过滤排序用查询参数 | `/users?status=active&sort=name` | `/users/active/sort-by-name` |

**Go — Gin 框架路由定义示例：**

```go
package main

import (
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
)

// User 资源模型
type User struct {
    ID       int64  `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email"`
    Status   string `json:"status"`
    CreateAt string `json:"created_at"`
}

// 统一响应体
type Response struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
    Meta    *Meta       `json:"meta,omitempty"`
}

type Meta struct {
    Total   int `json:"total"`
    Page    int `json:"page"`
    PerPage int `json:"per_page"`
}

func main() {
    r := gin.Default()

    v1 := r.Group("/api/v1")
    {
        users := v1.Group("/users")
        {
            users.GET("", listUsers)       // GET /api/v1/users?page=1&per_page=20
            users.GET("/:id", getUser)     // GET /api/v1/users/123
            users.POST("", createUser)     // POST /api/v1/users
            users.PUT("/:id", updateUser)  // PUT /api/v1/users/123
            users.PATCH("/:id", patchUser) // PATCH /api/v1/users/123
            users.DELETE("/:id", deleteUser) // DELETE /api/v1/users/123

            // 嵌套资源
            users.GET("/:id/orders", listUserOrders)
        }
    }

    r.Run(":8080")
}

func listUsers(c *gin.Context) {
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    perPage, _ := strconv.Atoi(c.DefaultQuery("per_page", "20"))
    status := c.Query("status")
    sort := c.DefaultQuery("sort", "created_at")

    // 模拟查询
    _ = status
    _ = sort

    c.JSON(http.StatusOK, Response{
        Code:    0,
        Message: "success",
        Data:    []User{},
        Meta:    &Meta{Total: 100, Page: page, PerPage: perPage},
    })
}

func getUser(c *gin.Context) {
    id := c.Param("id")
    _ = id
    c.JSON(http.StatusOK, Response{
        Code:    0,
        Message: "success",
        Data:    User{ID: 123, Name: "Alice", Email: "alice@example.com"},
    })
}

func createUser(c *gin.Context) {
    var user User
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, Response{Code: 400, Message: err.Error()})
        return
    }
    // 创建成功，返回 201
    c.JSON(http.StatusCreated, Response{
        Code:    0,
        Message: "created",
        Data:    user,
    })
}

func updateUser(c *gin.Context)      { c.JSON(http.StatusOK, Response{Code: 0, Message: "updated"}) }
func patchUser(c *gin.Context)       { c.JSON(http.StatusOK, Response{Code: 0, Message: "patched"}) }
func deleteUser(c *gin.Context)      { c.JSON(http.StatusNoContent, nil) }
func listUserOrders(c *gin.Context)  { c.JSON(http.StatusOK, Response{Code: 0, Message: "success"}) }
```

### 1.3 HTTP 方法语义

| 方法 | 语义 | 幂等 | 安全 | 请求体 | 典型状态码 |
|------|------|------|------|--------|-----------|
| GET | 读取资源 | ✅ | ✅ | 无 | 200 |
| POST | 创建资源 | ❌ | ❌ | 有 | 201 |
| PUT | 全量替换 | ✅ | ❌ | 有 | 200 |
| PATCH | 部分更新 | ❌* | ❌ | 有 | 200 |
| DELETE | 删除资源 | ✅ | ❌ | 通常无 | 204 |
| HEAD | 获取头信息 | ✅ | ✅ | 无 | 200 |
| OPTIONS | 获取支持方法 | ✅ | ✅ | 无 | 200 |

> *注：PATCH 的幂等性取决于实现。`{"op": "add", "path": "/tags/-", "value": "new"}` 这种 JSON Patch 操作不幂等；但 `{"name": "Alice"}` 这种 Merge Patch 是幂等的。

**PUT vs PATCH 关键区别：**

```json
// 原始数据
{"id": 1, "name": "Alice", "email": "alice@a.com", "age": 25}

// PUT /users/1 — 全量替换，未提供的字段会被清空
{"name": "Alice", "email": "alice@b.com"}
// 结果：{"id": 1, "name": "Alice", "email": "alice@b.com", "age": null}

// PATCH /users/1 — 部分更新，只修改提供的字段
{"email": "alice@b.com"}
// 结果：{"id": 1, "name": "Alice", "email": "alice@b.com", "age": 25}
```

### 1.4 HTTP 状态码速查

| 状态码 | 含义 | 使用场景 |
|--------|------|----------|
| **2xx 成功** | | |
| 200 OK | 请求成功 | GET/PUT/PATCH 成功 |
| 201 Created | 资源已创建 | POST 创建成功 |
| 204 No Content | 无内容 | DELETE 成功 |
| **3xx 重定向** | | |
| 301 Moved Permanently | 永久重定向 | 资源 URL 已变更 |
| 304 Not Modified | 未修改 | 缓存命中 |
| **4xx 客户端错误** | | |
| 400 Bad Request | 请求格式错误 | 参数校验失败 |
| 401 Unauthorized | 未认证 | 未携带/无效 Token |
| 403 Forbidden | 无权限 | 已认证但无权访问 |
| 404 Not Found | 资源不存在 | ID 不存在 |
| 405 Method Not Allowed | 方法不允许 | 对只读资源发 DELETE |
| 409 Conflict | 冲突 | 乐观锁版本冲突 |
| 422 Unprocessable Entity | 语义错误 | 参数合法但业务不允许 |
| 429 Too Many Requests | 请求过多 | 限流触发 |
| **5xx 服务端错误** | | |
| 500 Internal Server Error | 内部错误 | 未预期异常 |
| 502 Bad Gateway | 网关错误 | 上游服务不可用 |
| 503 Service Unavailable | 服务不可用 | 维护/过载 |
| 504 Gateway Timeout | 网关超时 | 上游响应超时 |

**Go — 统一错误处理中间件：**

```go
// ErrorCode 业务错误码
type ErrorCode struct {
    HTTPStatus int    `json:"-"`
    Code       int    `json:"code"`
    Message    string `json:"message"`
}

var (
    ErrNotFound     = ErrorCode{HTTPStatus: 404, Code: 40401, Message: "resource not found"}
    ErrUnauthorized = ErrorCode{HTTPStatus: 401, Code: 40101, Message: "authentication required"}
    ErrForbidden    = ErrorCode{HTTPStatus: 403, Code: 40301, Message: "permission denied"}
    ErrBadRequest   = ErrorCode{HTTPStatus: 400, Code: 40001, Message: "invalid request parameters"}
    ErrConflict     = ErrorCode{HTTPStatus: 409, Code: 40901, Message: "resource conflict"}
    ErrInternal     = ErrorCode{HTTPStatus: 500, Code: 50001, Message: "internal server error"}
    ErrRateLimit    = ErrorCode{HTTPStatus: 429, Code: 42901, Message: "rate limit exceeded"}
)

// APIError 包装业务错误
type APIError struct {
    ErrorCode
    Detail string `json:"detail,omitempty"`
}

func (e *APIError) Error() string {
    return e.Message
}

// 错误处理中间件
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        if len(c.Errors) > 0 {
            err := c.Errors.Last().Err
            switch e := err.(type) {
            case *APIError:
                c.JSON(e.HTTPStatus, gin.H{
                    "code":    e.Code,
                    "message": e.Message,
                    "detail":  e.Detail,
                })
            default:
                c.JSON(500, gin.H{
                    "code":    50001,
                    "message": "internal server error",
                })
            }
        }
    }
}
```

### 1.5 HATEOAS

**HATEOAS（Hypermedia As The Engine Of Application State）是 REST 成熟度模型（Richardson Maturity Model）最高级别 Level 3 的核心。**

```
Level 0: 单一 URI + POST（如 SOAP）
Level 1: 多 URI，标识资源
Level 2: 使用 HTTP 方法语义
Level 3: 超媒体驱动（HATEOAS）← 真正的 REST
```

**HATEOAS 响应示例：**

```json
{
    "id": 123,
    "name": "Alice",
    "email": "alice@example.com",
    "status": "active",
    "_links": {
        "self": {"href": "/api/v1/users/123", "method": "GET"},
        "update": {"href": "/api/v1/users/123", "method": "PUT"},
        "delete": {"href": "/api/v1/users/123", "method": "DELETE"},
        "orders": {"href": "/api/v1/users/123/orders", "method": "GET"},
        "deactivate": {"href": "/api/v1/users/123/deactivate", "method": "POST"}
    }
}
```

**Go — HATEOAS 封装：**

```go
// Link 超媒体链接
type Link struct {
    Href   string `json:"href"`
    Method string `json:"method"`
    Rel    string `json:"rel,omitempty"`
}

// HATEOASResource 带超媒体链接的资源
type HATEOASResource struct {
    Data  interface{}     `json:"data"`
    Links map[string]Link `json:"_links"`
}

func NewUserResource(user *User) HATEOASResource {
    base := fmt.Sprintf("/api/v1/users/%d", user.ID)
    links := map[string]Link{
        "self":   {Href: base, Method: "GET"},
        "update": {Href: base, Method: "PUT"},
        "delete": {Href: base, Method: "DELETE"},
        "orders": {Href: base + "/orders", Method: "GET"},
    }
    if user.Status == "active" {
        links["deactivate"] = Link{Href: base + "/deactivate", Method: "POST"}
    }
    return HATEOASResource{Data: user, Links: links}
}
```

### 1.6 版本管理策略

| 策略 | 示例 | 优点 | 缺点 |
|------|------|------|------|
| URL Path | `/api/v1/users` | 直观明了，缓存友好 | URL 变更，不够"RESTful" |
| Header | `Accept: application/vnd.api.v1+json` | URL 干净 | 调试不便，缓存需额外处理 |
| Query Param | `/api/users?version=1` | 简单 | 容易遗忘，缓存需额外处理 |

> 业界主流选择：**URL Path**（GitHub, Google, Stripe 都用这种方式）。

---

## 2. gRPC 深入

> ⭐⭐⭐ 核心考点：gRPC 是微服务间通信的主流选择，面试侧重 Protobuf 编码原理、四种通信模式、拦截器机制。

### 2.1 gRPC 核心概念

gRPC 是 Google 开源的高性能 RPC 框架，基于 HTTP/2 + Protocol Buffers。

**gRPC vs REST 对比：**

| 维度 | gRPC | REST |
|------|------|------|
| 协议 | HTTP/2 | HTTP/1.1 或 HTTP/2 |
| 序列化 | Protobuf（二进制） | JSON（文本） |
| 性能 | 高（二进制小、HTTP/2多路复用） | 较低 |
| 类型安全 | 强（.proto 文件定义） | 弱（依赖文档） |
| 流式传输 | 原生支持 | 需 WebSocket/SSE |
| 浏览器支持 | 需 gRPC-Web | 原生支持 |
| 可读性 | 低（二进制） | 高（JSON） |
| 代码生成 | 自动 | 需额外工具 |
| 适用场景 | 微服务内部通信 | 对外开放 API |

### 2.2 Protobuf 编码原理

**Protobuf 使用 Varint + Tag-Length-Value (TLV) 编码，极度紧凑。**

```protobuf
syntax = "proto3";

package user;

option go_package = "pb/user";

// 用户服务
service UserService {
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
    rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
    rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
    rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
    // 流式
    rpc WatchUsers(WatchUsersRequest) returns (stream UserEvent);
    rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateResponse);
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message GetUserRequest {
    int64 id = 1;
}

message GetUserResponse {
    User user = 1;
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    UserStatus status = 4;
    repeated string tags = 5;
    map<string, string> metadata = 6;
    google.protobuf.Timestamp created_at = 7;
}

enum UserStatus {
    USER_STATUS_UNSPECIFIED = 0;
    USER_STATUS_ACTIVE = 1;
    USER_STATUS_INACTIVE = 2;
    USER_STATUS_BANNED = 3;
}

message ListUsersRequest {
    int32 page = 1;
    int32 per_page = 2;
    string status_filter = 3;
}

message ListUsersResponse {
    repeated User users = 1;
    int32 total = 2;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message CreateUserResponse {
    User user = 1;
}

message UpdateUserRequest {
    int64 id = 1;
    string name = 2;
    string email = 3;
    google.protobuf.FieldMask update_mask = 4; // 部分更新
}

message UpdateUserResponse {
    User user = 1;
}

message DeleteUserRequest {
    int64 id = 1;
}

message DeleteUserResponse {}

message WatchUsersRequest {
    string status_filter = 1;
}

message UserEvent {
    string event_type = 1; // "created", "updated", "deleted"
    User user = 2;
}

message BatchCreateResponse {
    int32 created_count = 1;
}

message ChatMessage {
    string sender = 1;
    string content = 2;
}
```

**Varint 编码原理：**

```
数值 150 的 Varint 编码:
150 = 10010110 (二进制)

1. 每7位一组，低位在前:
   组1: 0010110  (低7位)
   组2: 0000001  (高位)

2. 非最后一组最高位设为1:
   组1: 1_0010110 = 0x96
   组2: 0_0000001 = 0x01

编码结果: [0x96, 0x01]  → 仅2字节（JSON "150" 需要3字节）
```

**Tag 编码：field_number << 3 | wire_type**

| Wire Type | 值 | 用途 |
|-----------|-----|------|
| Varint | 0 | int32, int64, uint32, bool, enum |
| 64-bit | 1 | fixed64, double |
| Length-delimited | 2 | string, bytes, repeated, message |
| 32-bit | 5 | fixed32, float |

### 2.3 四种通信模式

```
┌─────────────────────────────────────────────────────────────────┐
│                     gRPC 四种通信模式                             │
├──────────────────┬──────────────────────────────────────────────┤
│ Unary            │  Client ──req──> Server ──resp──> Client     │
│ Server Stream    │  Client ──req──> Server ══resp══> Client     │
│ Client Stream    │  Client ══req══> Server ──resp──> Client     │
│ Bidirectional    │  Client ══req══> Server ══resp══> Client     │
│                  │           (双向独立流)                         │
└──────────────────┴──────────────────────────────────────────────┘
  ── 单条消息    ══ 流式消息
```

| 模式 | 定义语法 | 使用场景 |
|------|----------|----------|
| Unary | `rpc Foo(Req) returns (Resp)` | 普通请求/响应 |
| Server Streaming | `rpc Foo(Req) returns (stream Resp)` | 实时通知、日志推送 |
| Client Streaming | `rpc Foo(stream Req) returns (Resp)` | 文件上传、批量写入 |
| Bidirectional Streaming | `rpc Foo(stream Req) returns (stream Resp)` | 聊天、实时协作 |

### 2.4 Go 实现完整 gRPC 服务

**服务端实现：**

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log"
    "net"
    "sync"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/status"

    pb "myapp/pb/user"
)

// userServer 实现 UserServiceServer 接口
type userServer struct {
    pb.UnimplementedUserServiceServer
    mu    sync.RWMutex
    users map[int64]*pb.User
    nextID int64
}

func newUserServer() *userServer {
    return &userServer{
        users:  make(map[int64]*pb.User),
        nextID: 1,
    }
}

// Unary RPC — 获取用户
func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    user, ok := s.users[req.Id]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
    }
    return &pb.GetUserResponse{User: user}, nil
}

// Unary RPC — 创建用户
func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserResponse, error) {
    if req.Name == "" {
        return nil, status.Errorf(codes.InvalidArgument, "name is required")
    }
    if req.Email == "" {
        return nil, status.Errorf(codes.InvalidArgument, "email is required")
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    // 检查邮箱是否已存在
    for _, u := range s.users {
        if u.Email == req.Email {
            return nil, status.Errorf(codes.AlreadyExists, "email %s already exists", req.Email)
        }
    }

    user := &pb.User{
        Id:     s.nextID,
        Name:   req.Name,
        Email:  req.Email,
        Status: pb.UserStatus_USER_STATUS_ACTIVE,
    }
    s.users[s.nextID] = user
    s.nextID++

    return &pb.CreateUserResponse{User: user}, nil
}

// Server Streaming — 监听用户变更
func (s *userServer) WatchUsers(req *pb.WatchUsersRequest, stream pb.UserService_WatchUsersServer) error {
    // 模拟每秒推送一个事件
    for i := 0; i < 10; i++ {
        select {
        case <-stream.Context().Done():
            return stream.Context().Err()
        default:
            event := &pb.UserEvent{
                EventType: "updated",
                User:      &pb.User{Id: int64(i), Name: fmt.Sprintf("user_%d", i)},
            }
            if err := stream.Send(event); err != nil {
                return err
            }
            time.Sleep(time.Second)
        }
    }
    return nil
}

// Client Streaming — 批量创建用户
func (s *userServer) BatchCreateUsers(stream pb.UserService_BatchCreateUsersServer) error {
    var count int32
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            // 客户端发送完毕，返回结果
            return stream.SendAndClose(&pb.BatchCreateResponse{
                CreatedCount: count,
            })
        }
        if err != nil {
            return err
        }

        // 创建用户
        s.mu.Lock()
        s.users[s.nextID] = &pb.User{
            Id:    s.nextID,
            Name:  req.Name,
            Email: req.Email,
        }
        s.nextID++
        count++
        s.mu.Unlock()
    }
}

// Bidirectional Streaming — 聊天
func (s *userServer) Chat(stream pb.UserService_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }

        // 回显消息
        reply := &pb.ChatMessage{
            Sender:  "server",
            Content: "Echo: " + msg.Content,
        }
        if err := stream.Send(reply); err != nil {
            return err
        }
    }
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    // 创建 gRPC Server，注册拦截器
    s := grpc.NewServer(
        grpc.UnaryInterceptor(unaryLoggingInterceptor),
        grpc.StreamInterceptor(streamLoggingInterceptor),
    )
    pb.RegisterUserServiceServer(s, newUserServer())

    log.Println("gRPC server listening on :50051")
    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

### 2.5 gRPC 拦截器（Interceptor）

拦截器是 gRPC 的中间件机制，分为 Unary 拦截器和 Stream 拦截器。

```go
// Unary 拦截器 — 日志 + 耗时统计
func unaryLoggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()

    // 提取 metadata
    md, ok := metadata.FromIncomingContext(ctx)
    if ok {
        log.Printf("metadata: %v", md)
    }

    // 执行实际处理
    resp, err := handler(ctx, req)

    // 记录日志
    duration := time.Since(start)
    st, _ := status.FromError(err)
    log.Printf("method=%s duration=%v code=%s error=%v",
        info.FullMethod, duration, st.Code(), err)

    return resp, err
}

// Stream 拦截器
func streamLoggingInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    start := time.Now()
    log.Printf("stream started: %s", info.FullMethod)

    err := handler(srv, ss)

    log.Printf("stream ended: %s duration=%v error=%v",
        info.FullMethod, time.Since(start), err)
    return err
}

// 认证拦截器
func authInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.Unauthenticated, "missing metadata")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Errorf(codes.Unauthenticated, "missing token")
    }

    // 验证 token
    token := tokens[0]
    userID, err := validateToken(token)
    if err != nil {
        return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
    }

    // 将用户信息注入 context
    ctx = context.WithValue(ctx, "user_id", userID)
    return handler(ctx, req)
}

func validateToken(token string) (int64, error) {
    // 模拟 token 验证
    if token == "valid-token" {
        return 123, nil
    }
    return 0, fmt.Errorf("invalid token")
}
```

### 2.6 gRPC 错误处理

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "google.golang.org/genproto/googleapis/rpc/errdetails"
)

// 带详细信息的错误
func richError() error {
    st := status.New(codes.InvalidArgument, "invalid parameters")

    // 添加字段级别的错误详情
    details, err := st.WithDetails(
        &errdetails.BadRequest{
            FieldViolations: []*errdetails.BadRequest_FieldViolation{
                {Field: "email", Description: "invalid email format"},
                {Field: "name", Description: "name must be 2-50 characters"},
            },
        },
        &errdetails.RetryInfo{
            RetryDelay: &durationpb.Duration{Seconds: 5},
        },
    )
    if err != nil {
        return st.Err()
    }
    return details.Err()
}

// 客户端解析错误详情
func handleError(err error) {
    st, ok := status.FromError(err)
    if !ok {
        log.Printf("non-gRPC error: %v", err)
        return
    }

    log.Printf("code: %s, message: %s", st.Code(), st.Message())

    for _, detail := range st.Details() {
        switch d := detail.(type) {
        case *errdetails.BadRequest:
            for _, v := range d.FieldViolations {
                log.Printf("  field: %s, error: %s", v.Field, v.Description)
            }
        case *errdetails.RetryInfo:
            log.Printf("  retry after: %v", d.RetryDelay.AsDuration())
        }
    }
}
```

**gRPC 错误码与 HTTP 状态码映射：**

| gRPC Code | HTTP Status | 含义 |
|-----------|-------------|------|
| OK | 200 | 成功 |
| InvalidArgument | 400 | 参数错误 |
| Unauthenticated | 401 | 未认证 |
| PermissionDenied | 403 | 无权限 |
| NotFound | 404 | 不存在 |
| AlreadyExists | 409 | 已存在 |
| ResourceExhausted | 429 | 限流 |
| Internal | 500 | 内部错误 |
| Unavailable | 503 | 不可用 |
| DeadlineExceeded | 504 | 超时 |

---

## 3. GraphQL

> ⭐⭐ 核心考点：GraphQL 的核心优势是按需查询，面试关注 Schema 设计、N+1 问题和 DataLoader。

### 3.1 GraphQL 核心概念

**GraphQL vs REST 对比：**

| 维度 | GraphQL | REST |
|------|---------|------|
| 端点 | 单一端点 `/graphql` | 多个端点 |
| 数据获取 | 按需获取，客户端决定 | 服务端决定 |
| Over-fetching | 无 | 常见 |
| Under-fetching | 无 | 需多次请求 |
| 类型系统 | 内置强类型 Schema | 依赖文档 |
| 缓存 | 复杂（需客户端库） | HTTP 缓存友好 |
| 文件上传 | 需额外处理 | 原生支持 |
| 学习曲线 | 较高 | 较低 |

### 3.2 Schema 设计

```graphql
# schema.graphql

# 标量类型扩展
scalar DateTime
scalar JSON

# 用户类型
type User {
    id: ID!
    name: String!
    email: String!
    avatar: String
    status: UserStatus!
    orders(first: Int = 10, after: String): OrderConnection!
    createdAt: DateTime!
}

enum UserStatus {
    ACTIVE
    INACTIVE
    BANNED
}

# 订单类型
type Order {
    id: ID!
    user: User!
    items: [OrderItem!]!
    totalAmount: Float!
    status: OrderStatus!
    createdAt: DateTime!
}

type OrderItem {
    id: ID!
    product: Product!
    quantity: Int!
    price: Float!
}

type Product {
    id: ID!
    name: String!
    price: Float!
    category: String!
}

enum OrderStatus {
    PENDING
    PAID
    SHIPPED
    DELIVERED
    CANCELLED
}

# Relay 风格分页
type OrderConnection {
    edges: [OrderEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
}

type OrderEdge {
    node: Order!
    cursor: String!
}

type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
}

# 输入类型
input CreateUserInput {
    name: String!
    email: String!
    avatar: String
}

input UpdateUserInput {
    name: String
    email: String
    avatar: String
}

input OrderFilterInput {
    status: OrderStatus
    minAmount: Float
    maxAmount: Float
    startDate: DateTime
    endDate: DateTime
}

# 查询
type Query {
    user(id: ID!): User
    users(first: Int = 20, after: String, status: UserStatus): UserConnection!
    order(id: ID!): Order
    orders(filter: OrderFilterInput, first: Int = 20, after: String): OrderConnection!
}

# 变更
type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
    createOrder(userId: ID!, items: [OrderItemInput!]!): Order!
}

input OrderItemInput {
    productId: ID!
    quantity: Int!
}

# 订阅
type Subscription {
    orderStatusChanged(userId: ID!): Order!
    newUser: User!
}
```

### 3.3 N+1 问题与 DataLoader

**N+1 问题：** 查询列表时，先查1次获取N条记录，再对每条记录各查1次关联数据，共 N+1 次查询。

```graphql
# 这个查询会导致 N+1 问题
query {
    users(first: 100) {
        edges {
            node {
                id
                name
                orders(first: 5) {  # 每个用户都查一次 orders → 100次SQL
                    edges {
                        node {
                            id
                            totalAmount
                        }
                    }
                }
            }
        }
    }
}
```

**DataLoader 解决方案原理：**

```
不用 DataLoader:
  Resolve user1 → SELECT * FROM orders WHERE user_id = 1
  Resolve user2 → SELECT * FROM orders WHERE user_id = 2
  ...
  Resolve user100 → SELECT * FROM orders WHERE user_id = 100
  共 100 次查询

用 DataLoader:
  收集所有 user_id → [1, 2, ..., 100]
  批量查询 → SELECT * FROM orders WHERE user_id IN (1, 2, ..., 100)
  按 user_id 分组返回
  仅 1 次查询
```

### 3.4 Go 实现 GraphQL 服务（gqlgen）

```go
// graph/resolver.go
package graph

import (
    "context"
    "fmt"
    "sync"

    "github.com/graph-gophers/dataloader/v7"
)

type Resolver struct {
    users  map[string]*model.User
    orders map[string]*model.Order
    mu     sync.RWMutex
}

// DataLoader — 批量加载用户
type userLoader struct {
    db *sql.DB
}

func (u *userLoader) BatchGetUsers(ctx context.Context, keys []string) []*dataloader.Result[*model.User] {
    // 批量查询：SELECT * FROM users WHERE id IN (?, ?, ...)
    results := make([]*dataloader.Result[*model.User], len(keys))

    placeholders := make([]string, len(keys))
    args := make([]interface{}, len(keys))
    for i, key := range keys {
        placeholders[i] = "?"
        args[i] = key
    }

    query := fmt.Sprintf(
        "SELECT id, name, email, status FROM users WHERE id IN (%s)",
        strings.Join(placeholders, ","),
    )
    rows, err := u.db.QueryContext(ctx, query, args...)
    if err != nil {
        for i := range results {
            results[i] = &dataloader.Result[*model.User]{Error: err}
        }
        return results
    }
    defer rows.Close()

    // 构建 id -> user 映射
    userMap := make(map[string]*model.User)
    for rows.Next() {
        var user model.User
        if err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.Status); err != nil {
            continue
        }
        userMap[user.ID] = &user
    }

    // 按 keys 顺序返回结果
    for i, key := range keys {
        user, ok := userMap[key]
        if !ok {
            results[i] = &dataloader.Result[*model.User]{
                Error: fmt.Errorf("user %s not found", key),
            }
        } else {
            results[i] = &dataloader.Result[*model.User]{Data: user}
        }
    }
    return results
}

// 在中间件中注入 DataLoader（每个请求独立）
func DataLoaderMiddleware(db *sql.DB, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        loader := &userLoader{db: db}
        userDataLoader := dataloader.NewBatchedLoader(
            loader.BatchGetUsers,
            dataloader.WithBatchCapacity[string, *model.User](100),
            dataloader.WithWait[string, *model.User](time.Millisecond),
        )

        ctx := context.WithValue(r.Context(), "userLoader", userDataLoader)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Resolver 中使用 DataLoader
func (r *orderResolver) User(ctx context.Context, obj *model.Order) (*model.User, error) {
    loader := ctx.Value("userLoader").(*dataloader.Loader[string, *model.User])
    // 这里不会立即执行查询，而是收集 key，等一个批次的 key 收齐后统一查询
    thunk := loader.Load(ctx, obj.UserID)
    user, err := thunk()
    return user, err
}
```

**GraphQL 查询复杂度限制（防止恶意查询）：**

```go
// 限制查询深度和复杂度
srv := handler.NewDefaultServer(generated.NewExecutableSchema(generated.Config{
    Resolvers: &graph.Resolver{},
}))

// 限制查询深度
srv.Use(extension.FixedComplexityLimit(200))

// 自定义复杂度计算
srv.Use(extension.ComplexityLimit(func(ctx context.Context) int {
    // 根据用户等级返回不同的复杂度上限
    role := auth.GetRole(ctx)
    switch role {
    case "admin":
        return 1000
    case "premium":
        return 500
    default:
        return 200
    }
}))
```

---

## 4. API 幂等性设计

> ⭐⭐⭐ 核心考点：幂等性是分布式系统的基石，防止重复提交和消息重复消费。

### 4.1 幂等性基本概念

**幂等性（Idempotency）：同一操作执行一次和执行多次产生的效果相同。**

**各 HTTP 方法幂等性分析：**

| 方法 | 幂等 | 安全 | 分析 |
|------|------|------|------|
| GET | ✅ | ✅ | 只读操作，天然幂等 |
| HEAD | ✅ | ✅ | 只读操作，天然幂等 |
| OPTIONS | ✅ | ✅ | 只读操作，天然幂等 |
| PUT | ✅ | ❌ | 全量替换，多次执行结果相同 |
| DELETE | ✅ | ❌ | 删除后再删返回 404，**但资源状态不变** |
| POST | ❌ | ❌ | 每次调用可能创建新资源 |
| PATCH | ❌* | ❌ | 依赖具体实现 |

> *重点理解：PUT 是幂等的，因为"替换为同一内容"多次执行后状态一致；POST 不幂等，因为每次可能创建新资源。

### 4.2 幂等键方案

```
客户端生成唯一幂等键（如 UUID），随请求发送。
服务端用这个 key 做去重判断。

请求1: POST /orders  Idempotency-Key: abc-123  → 创建订单，返回结果
请求2: POST /orders  Idempotency-Key: abc-123  → 发现已处理，直接返回缓存结果
```

**Go — 幂等性中间件实现：**

```go
package middleware

import (
    "context"
    "crypto/sha256"
    "encoding/json"
    "fmt"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/redis/go-redis/v9"
)

const (
    idempotencyKeyHeader = "Idempotency-Key"
    idempotencyTTL       = 24 * time.Hour
    lockTTL              = 30 * time.Second
)

// CachedResponse 缓存的响应
type CachedResponse struct {
    StatusCode int               `json:"status_code"`
    Headers    map[string]string `json:"headers"`
    Body       json.RawMessage   `json:"body"`
}

type IdempotencyMiddleware struct {
    rdb *redis.Client
}

func NewIdempotencyMiddleware(rdb *redis.Client) *IdempotencyMiddleware {
    return &IdempotencyMiddleware{rdb: rdb}
}

func (m *IdempotencyMiddleware) Handle() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 只对非幂等方法生效
        if c.Request.Method == http.MethodGet ||
            c.Request.Method == http.MethodHead ||
            c.Request.Method == http.MethodOptions {
            c.Next()
            return
        }

        key := c.GetHeader(idempotencyKeyHeader)
        if key == "" {
            c.Next() // 未提供幂等键，正常处理
            return
        }

        ctx := c.Request.Context()
        cacheKey := fmt.Sprintf("idempotency:%s:%s",
            c.Request.URL.Path, key)
        lockKey := cacheKey + ":lock"

        // 1. 先尝试获取已有响应
        cached, err := m.rdb.Get(ctx, cacheKey).Bytes()
        if err == nil {
            // 命中缓存，直接返回
            var resp CachedResponse
            if json.Unmarshal(cached, &resp) == nil {
                for k, v := range resp.Headers {
                    c.Header(k, v)
                }
                c.Data(resp.StatusCode, "application/json", resp.Body)
                c.Abort()
                return
            }
        }

        // 2. 获取分布式锁，防止并发重复请求
        locked, err := m.rdb.SetNX(ctx, lockKey, "1", lockTTL).Result()
        if err != nil || !locked {
            c.JSON(http.StatusConflict, gin.H{
                "code":    409,
                "message": "request is being processed",
            })
            c.Abort()
            return
        }
        defer m.rdb.Del(ctx, lockKey)

        // 3. 使用自定义 ResponseWriter 捕获响应
        writer := &responseCapture{
            ResponseWriter: c.Writer,
            body:           &bytes.Buffer{},
        }
        c.Writer = writer

        c.Next()

        // 4. 缓存响应
        if c.Writer.Status() >= 200 && c.Writer.Status() < 500 {
            resp := CachedResponse{
                StatusCode: writer.Status(),
                Headers:    make(map[string]string),
                Body:       writer.body.Bytes(),
            }
            if data, err := json.Marshal(resp); err == nil {
                m.rdb.Set(ctx, cacheKey, data, idempotencyTTL)
            }
        }
    }
}

// responseCapture 捕获响应体
type responseCapture struct {
    gin.ResponseWriter
    body *bytes.Buffer
}

func (w *responseCapture) Write(data []byte) (int, error) {
    w.body.Write(data)
    return w.ResponseWriter.Write(data)
}
```

### 4.3 去重表方案

适用于数据库事务场景，利用唯一索引保证幂等。

```go
// 去重表方案 — 利用数据库唯一索引
type IdempotencyRecord struct {
    ID         int64     `gorm:"primaryKey"`
    BizType    string    `gorm:"uniqueIndex:uk_biz_key;size:64"`
    BizKey     string    `gorm:"uniqueIndex:uk_biz_key;size:128"`
    Response   string    `gorm:"type:text"`
    CreatedAt  time.Time
    ExpiredAt  time.Time
}

// SQL:
// CREATE TABLE idempotency_records (
//     id         BIGINT PRIMARY KEY AUTO_INCREMENT,
//     biz_type   VARCHAR(64) NOT NULL,
//     biz_key    VARCHAR(128) NOT NULL,
//     response   TEXT,
//     created_at DATETIME NOT NULL,
//     expired_at DATETIME NOT NULL,
//     UNIQUE KEY uk_biz_key (biz_type, biz_key)
// );

func CreateOrderIdempotent(ctx context.Context, db *gorm.DB, idempotencyKey string, req *CreateOrderReq) (*Order, error) {
    // 1. 先查去重表
    var record IdempotencyRecord
    err := db.Where("biz_type = ? AND biz_key = ?", "create_order", idempotencyKey).
        First(&record).Error
    if err == nil {
        // 已有记录，返回缓存结果
        var order Order
        json.Unmarshal([]byte(record.Response), &order)
        return &order, nil
    }

    // 2. 在事务中同时插入业务数据和去重记录
    var order Order
    err = db.Transaction(func(tx *gorm.DB) error {
        // 创建订单
        order = Order{
            UserID:      req.UserID,
            TotalAmount: req.TotalAmount,
            Status:      "pending",
        }
        if err := tx.Create(&order).Error; err != nil {
            return err
        }

        // 插入去重记录（利用唯一索引防止并发）
        respData, _ := json.Marshal(order)
        dedup := IdempotencyRecord{
            BizType:   "create_order",
            BizKey:    idempotencyKey,
            Response:  string(respData),
            CreatedAt: time.Now(),
            ExpiredAt: time.Now().Add(24 * time.Hour),
        }
        if err := tx.Create(&dedup).Error; err != nil {
            // 唯一索引冲突 → 重复请求
            return err
        }

        return nil
    })

    return &order, err
}
```

### 4.4 Token 机制

适用于防止表单重复提交场景。

```go
// Token 防重复提交机制
// 1. 客户端先请求一个一次性 Token
// 2. 提交时携带 Token
// 3. 服务端验证并删除 Token（原子操作）

func GetSubmitToken(c *gin.Context) {
    token := uuid.New().String()
    rdb.Set(c, "submit_token:"+token, "1", 10*time.Minute)
    c.JSON(200, gin.H{"token": token})
}

func SubmitWithToken(c *gin.Context) {
    token := c.GetHeader("X-Submit-Token")
    if token == "" {
        c.JSON(400, gin.H{"message": "missing submit token"})
        return
    }

    // 原子性地检查并删除 Token（Lua脚本保证原子性）
    script := redis.NewScript(`
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `)

    result, err := script.Run(c, rdb, []string{"submit_token:" + token}, "1").Int()
    if err != nil || result == 0 {
        c.JSON(409, gin.H{"message": "duplicate submission"})
        return
    }

    // 正常处理业务逻辑
    // ...
}
```

**三种幂等方案对比：**

| 方案 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| 幂等键 + Redis | 通用 API 幂等 | 高性能、灵活 | 需客户端配合传 key |
| 去重表 | 数据库事务场景 | 强一致性 | 性能较低 |
| Token 机制 | 表单提交 | 简单直观 | 需两次请求，仅防重复提交 |

---

## 5. API 版本管理

> ⭐⭐ 核心考点：如何在不破坏已有客户端的前提下演进 API。

### 5.1 三种版本策略对比

| 策略 | 实现方式 | 优点 | 缺点 | 代表用户 |
|------|----------|------|------|----------|
| URL Path | `/api/v1/users` | 直观，缓存友好，调试方便 | URL 改变，不够 RESTful 纯粹 | GitHub, Google, Stripe |
| Header | `Accept: application/vnd.api.v1+json` | URL 干净，符合 REST 语义 | 调试不便，需特殊工具 | GitHub(也支持) |
| Query Param | `/api/users?version=1` | 简单，易于实现 | 参数可选易遗漏，缓存复杂 | Amazon |

### 5.2 向后兼容原则

```
✅ 向后兼容的变更（不需要新版本）:
  - 新增可选请求参数
  - 新增响应字段
  - 新增 API 端点
  - 新增枚举值（前端需能忽略未知值）

❌ 破坏性变更（需要新版本）:
  - 删除或重命名字段
  - 修改字段类型
  - 修改字段语义
  - 修改响应结构
  - 修改 URL 路径
  - 修改必选参数
```

**Go — 版本路由实现：**

```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()

    // 方式一：URL Path 版本
    v1 := r.Group("/api/v1")
    {
        v1.GET("/users", listUsersV1)
        v1.GET("/users/:id", getUserV1)
    }

    v2 := r.Group("/api/v2")
    {
        v2.GET("/users", listUsersV2)     // V2: 新增分页格式
        v2.GET("/users/:id", getUserV2)   // V2: 响应结构变化
    }

    // 方式二：Header 版本（用中间件路由）
    api := r.Group("/api")
    api.Use(VersionMiddleware())
    {
        api.GET("/users", func(c *gin.Context) {
            version := c.GetString("api_version")
            switch version {
            case "v2":
                listUsersV2(c)
            default:
                listUsersV1(c)
            }
        })
    }

    r.Run(":8080")
}

func VersionMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 从 Accept Header 中解析版本
        accept := c.GetHeader("Accept")
        version := "v1" // 默认版本

        // Accept: application/vnd.myapi.v2+json
        if strings.Contains(accept, "vnd.myapi.v2") {
            version = "v2"
        }

        // 也可以从自定义 Header 解析
        if v := c.GetHeader("X-API-Version"); v != "" {
            version = v
        }

        c.Set("api_version", version)
        c.Next()
    }
}

// V1 响应
func listUsersV1(c *gin.Context) {
    c.JSON(200, gin.H{
        "users": []gin.H{
            {"id": 1, "name": "Alice"},
        },
        "total": 1,
    })
}

// V2 响应 — 改用 Relay 风格分页
func listUsersV2(c *gin.Context) {
    c.JSON(200, gin.H{
        "data": []gin.H{
            {"id": 1, "name": "Alice", "email": "alice@a.com"},
        },
        "pagination": gin.H{
            "total":     1,
            "page":      1,
            "per_page":  20,
            "has_next":  false,
        },
    })
}

func getUserV1(c *gin.Context) { c.JSON(200, gin.H{"id": 1, "name": "Alice"}) }
func getUserV2(c *gin.Context) {
    c.JSON(200, gin.H{
        "id": 1, "name": "Alice", "email": "alice@a.com",
        "_links": gin.H{
            "self": "/api/v2/users/1",
        },
    })
}
```

### 5.3 版本废弃策略

```
1. Sunset Header 告知废弃时间
   Sunset: Sat, 01 Jan 2025 00:00:00 GMT
   Deprecation: true

2. 版本生命周期
   Alpha → Beta → GA → Deprecated → Sunset
   
3. 建议保持最多 2-3 个活跃版本

4. 废弃流程：
   a. 提前 6 个月通知（邮件/文档/Header）
   b. 标记 Deprecated, 日志告警
   c. 返回 Warning Header
   d. 到期后返回 410 Gone
```

---

## 6. API 网关

> ⭐⭐ 核心考点：API 网关作为统一入口，承载路由、限流、认证、熔断等横切关注点。

### 6.1 网关核心功能

```
                    ┌─────────────────────────────────┐
                    │           API Gateway            │
    Client ──────>  │                                   │
                    │  ┌─────────┐  ┌───────────────┐  │  ┌──────────┐
                    │  │  路由    │→ │  认证/鉴权    │  │→ │ Service A │
                    │  └─────────┘  └───────────────┘  │  └──────────┘
                    │  ┌─────────┐  ┌───────────────┐  │  ┌──────────┐
                    │  │  限流    │→ │  熔断/降级    │  │→ │ Service B │
                    │  └─────────┘  └───────────────┘  │  └──────────┘
                    │  ┌─────────┐  ┌───────────────┐  │  ┌──────────┐
                    │  │  日志    │→ │  协议转换     │  │→ │ Service C │
                    │  └─────────┘  └───────────────┘  │  └──────────┘
                    └─────────────────────────────────┘
```

**主流网关对比：**

| 特性 | Kong | APISIX | Envoy | Nginx |
|------|------|--------|-------|-------|
| 语言 | Lua/OpenResty | Lua/OpenResty | C++ | C |
| 配置 | Admin API + DB | etcd | xDS API | 配置文件 |
| 插件机制 | Lua 插件 | Lua/Wasm/多语言 | Wasm/Lua | Lua/C模块 |
| 性能 | 高 | 极高 | 极高 | 高 |
| 服务发现 | DNS/Consul/etc | DNS/etcd/Nacos/etc | EDS | 无内置 |
| Dashboard | Kong Manager | 内置 Dashboard | 无内置 | 无内置 |
| 社区 | 大 | 活跃 | 大 | 最大 |

### 6.2 Go 实现简易 API 网关

```go
package main

import (
    "context"
    "log"
    "math/rand"
    "net"
    "net/http"
    "net/http/httputil"
    "net/url"
    "strings"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

// ==================== 路由 ====================

type Route struct {
    PathPrefix string
    Backends   []string // 多个后端实例
    StripPath  bool
}

type Router struct {
    routes []Route
}

func (r *Router) Match(path string) (*Route, string) {
    for i, route := range r.routes {
        if strings.HasPrefix(path, route.PathPrefix) {
            newPath := path
            if route.StripPath {
                newPath = strings.TrimPrefix(path, route.PathPrefix)
                if newPath == "" {
                    newPath = "/"
                }
            }
            return &r.routes[i], newPath
        }
    }
    return nil, ""
}

// ==================== 负载均衡 ====================

type LoadBalancer struct {
    mu      sync.Mutex
    counter map[string]int
}

func NewLoadBalancer() *LoadBalancer {
    return &LoadBalancer{counter: make(map[string]int)}
}

// RoundRobin 轮询
func (lb *LoadBalancer) Pick(backends []string, key string) string {
    lb.mu.Lock()
    defer lb.mu.Unlock()

    idx := lb.counter[key] % len(backends)
    lb.counter[key]++
    return backends[idx]
}

// ==================== 限流 ====================

type RateLimiter struct {
    mu       sync.Mutex
    limiters map[string]*rate.Limiter
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(r rate.Limit, burst int) *RateLimiter {
    return &RateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    burst,
    }
}

func (rl *RateLimiter) GetLimiter(key string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    limiter, ok := rl.limiters[key]
    if !ok {
        limiter = rate.NewLimiter(rl.rate, rl.burst)
        rl.limiters[key] = limiter
    }
    return limiter
}

// ==================== 熔断器 ====================

type CircuitState int

const (
    StateClosed   CircuitState = iota // 正常
    StateOpen                          // 熔断
    StateHalfOpen                      // 半开
)

type CircuitBreaker struct {
    mu              sync.Mutex
    state           CircuitState
    failures        int
    successes       int
    threshold       int     // 失败阈值
    halfOpenMax     int     // 半开状态最大请求数
    resetTimeout    time.Duration
    lastFailureTime time.Time
}

func NewCircuitBreaker(threshold int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:        StateClosed,
        threshold:    threshold,
        halfOpenMax:  3,
        resetTimeout: resetTimeout,
    }
}

func (cb *CircuitBreaker) Allow() bool {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case StateClosed:
        return true
    case StateOpen:
        // 检查是否到了重试时间
        if time.Since(cb.lastFailureTime) > cb.resetTimeout {
            cb.state = StateHalfOpen
            cb.successes = 0
            return true
        }
        return false
    case StateHalfOpen:
        return cb.successes < cb.halfOpenMax
    }
    return false
}

func (cb *CircuitBreaker) RecordSuccess() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case StateHalfOpen:
        cb.successes++
        if cb.successes >= cb.halfOpenMax {
            cb.state = StateClosed
            cb.failures = 0
        }
    case StateClosed:
        cb.failures = 0
    }
}

func (cb *CircuitBreaker) RecordFailure() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    cb.lastFailureTime = time.Now()
    switch cb.state {
    case StateClosed:
        cb.failures++
        if cb.failures >= cb.threshold {
            cb.state = StateOpen
        }
    case StateHalfOpen:
        cb.state = StateOpen
    }
}

// ==================== 网关主体 ====================

type Gateway struct {
    router     *Router
    lb         *LoadBalancer
    limiter    *RateLimiter
    breakers   map[string]*CircuitBreaker
    mu         sync.RWMutex
}

func NewGateway() *Gateway {
    return &Gateway{
        router: &Router{
            routes: []Route{
                {PathPrefix: "/api/users", Backends: []string{"http://localhost:8081", "http://localhost:8082"}, StripPath: false},
                {PathPrefix: "/api/orders", Backends: []string{"http://localhost:8083"}, StripPath: false},
            },
        },
        lb:       NewLoadBalancer(),
        limiter:  NewRateLimiter(100, 200), // 100 req/s, burst 200
        breakers: make(map[string]*CircuitBreaker),
    }
}

func (g *Gateway) getBreaker(key string) *CircuitBreaker {
    g.mu.RLock()
    cb, ok := g.breakers[key]
    g.mu.RUnlock()
    if ok {
        return cb
    }

    g.mu.Lock()
    defer g.mu.Unlock()
    cb, ok = g.breakers[key]
    if !ok {
        cb = NewCircuitBreaker(5, 30*time.Second)
        g.breakers[key] = cb
    }
    return cb
}

func (g *Gateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 1. 限流
    clientIP, _, _ := net.SplitHostPort(r.RemoteAddr)
    limiter := g.limiter.GetLimiter(clientIP)
    if !limiter.Allow() {
        http.Error(w, `{"code":429,"message":"rate limit exceeded"}`, http.StatusTooManyRequests)
        return
    }

    // 2. 路由匹配
    route, newPath := g.router.Match(r.URL.Path)
    if route == nil {
        http.Error(w, `{"code":404,"message":"route not found"}`, http.StatusNotFound)
        return
    }

    // 3. 熔断检查
    cb := g.getBreaker(route.PathPrefix)
    if !cb.Allow() {
        http.Error(w, `{"code":503,"message":"service unavailable (circuit open)"}`, http.StatusServiceUnavailable)
        return
    }

    // 4. 负载均衡选取后端
    backend := g.lb.Pick(route.Backends, route.PathPrefix)
    target, _ := url.Parse(backend)

    // 5. 反向代理
    proxy := httputil.NewSingleHostReverseProxy(target)
    proxy.ErrorHandler = func(w http.ResponseWriter, r *http.Request, err error) {
        cb.RecordFailure()
        http.Error(w, `{"code":502,"message":"bad gateway"}`, http.StatusBadGateway)
    }

    // 修改请求路径
    r.URL.Path = newPath
    r.Host = target.Host

    // 添加网关头
    r.Header.Set("X-Forwarded-For", clientIP)
    r.Header.Set("X-Request-ID", generateRequestID())

    proxy.ServeHTTP(w, r)
    cb.RecordSuccess()
}

func generateRequestID() string {
    return fmt.Sprintf("%d-%d", time.Now().UnixNano(), rand.Int63())
}

func main() {
    gateway := NewGateway()
    log.Println("API Gateway listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", gateway))
}
```

### 6.3 Kong / APISIX 配置示例

```yaml
# Kong 声明式配置 (kong.yml)
_format_version: "3.0"

services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: user-route
        paths:
          - /api/v1/users
        strip_path: false
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
          redis_host: redis
      - name: jwt
        config:
          claims_to_verify:
            - exp
      - name: cors
        config:
          origins:
            - "https://example.com"
          methods:
            - GET
            - POST
            - PUT
            - DELETE

  - name: order-service
    url: http://order-service:8080
    routes:
      - name: order-route
        paths:
          - /api/v1/orders
    plugins:
      - name: request-transformer
        config:
          add:
            headers:
              - "X-Request-Source:gateway"
```

```yaml
# APISIX 配置 (apisix.yaml)
routes:
  - uri: /api/v1/users/*
    upstream:
      type: roundrobin
      nodes:
        "user-service-1:8080": 1
        "user-service-2:8080": 1
      checks:
        active:
          http_path: /health
          interval: 5
          healthy:
            successes: 2
          unhealthy:
            http_failures: 3
    plugins:
      limit-req:
        rate: 100
        burst: 50
        key: remote_addr
      jwt-auth:
        key: "user-key"
      prometheus:
        prefer_name: true
```

---

## 7. API 文档与测试

> ⭐⭐ 核心考点：好的 API 需要完善的文档和契约测试。

### 7.1 OpenAPI / Swagger

**OpenAPI 3.0 规范示例：**

```yaml
openapi: 3.0.3
info:
  title: User Service API
  description: 用户服务接口文档
  version: 1.0.0
  contact:
    name: API Support
    email: api@example.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

paths:
  /users:
    get:
      summary: 获取用户列表
      operationId: listUsers
      tags:
        - Users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: per_page
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
        - name: status
          in: query
          schema:
            $ref: '#/components/schemas/UserStatus'
      responses:
        '200':
          description: 成功返回用户列表
          content:
            application/json:
              schema:
                type: object
                properties:
                  code:
                    type: integer
                    example: 0
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  meta:
                    $ref: '#/components/schemas/Pagination'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '429':
          $ref: '#/components/responses/RateLimited'

    post:
      summary: 创建用户
      operationId: createUser
      tags:
        - Users
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: 用户创建成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/BadRequest'
        '409':
          description: 邮箱已存在

  /users/{id}:
    get:
      summary: 获取用户详情
      operationId: getUser
      tags:
        - Users
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
          format: int64
        name:
          type: string
        email:
          type: string
          format: email
        status:
          $ref: '#/components/schemas/UserStatus'
        created_at:
          type: string
          format: date-time

    UserStatus:
      type: string
      enum:
        - active
        - inactive
        - banned

    CreateUserRequest:
      type: object
      required:
        - name
        - email
      properties:
        name:
          type: string
          minLength: 2
          maxLength: 50
        email:
          type: string
          format: email

    Pagination:
      type: object
      properties:
        total:
          type: integer
        page:
          type: integer
        per_page:
          type: integer

  responses:
    BadRequest:
      description: 请求参数错误
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    Unauthorized:
      description: 未认证
    NotFound:
      description: 资源不存在
    RateLimited:
      description: 请求过多
      headers:
        Retry-After:
          schema:
            type: integer

    Error:
      type: object
      properties:
        code:
          type: integer
        message:
          type: string

  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - BearerAuth: []
```

### 7.2 Go Swagger 集成 (swag)

```go
package main

import (
    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger "github.com/swaggo/gin-swagger"

    _ "myapp/docs" // swag init 生成的文档
)

// @title           User Service API
// @version         1.0
// @description     用户服务接口
// @host            localhost:8080
// @BasePath        /api/v1
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization

func main() {
    r := gin.Default()

    // Swagger UI
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

    v1 := r.Group("/api/v1")
    {
        v1.GET("/users", listUsers)
        v1.POST("/users", createUser)
        v1.GET("/users/:id", getUser)
    }

    r.Run(":8080")
}

// listUsers godoc
// @Summary      获取用户列表
// @Description  分页获取用户列表，支持按状态过滤
// @Tags         Users
// @Accept       json
// @Produce      json
// @Param        page      query    int     false  "页码"      default(1)
// @Param        per_page  query    int     false  "每页条数"  default(20)  maximum(100)
// @Param        status    query    string  false  "状态过滤"  Enums(active, inactive, banned)
// @Success      200  {object}  Response{data=[]User}
// @Failure      401  {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /users [get]
func listUsers(c *gin.Context) { /* ... */ }

// createUser godoc
// @Summary      创建用户
// @Description  创建新用户
// @Tags         Users
// @Accept       json
// @Produce      json
// @Param        request  body      CreateUserRequest  true  "创建用户请求"
// @Success      201      {object}  Response{data=User}
// @Failure      400      {object}  ErrorResponse
// @Failure      409      {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /users [post]
func createUser(c *gin.Context) { /* ... */ }

// getUser godoc
// @Summary      获取用户详情
// @Tags         Users
// @Produce      json
// @Param        id   path      int  true  "用户ID"
// @Success      200  {object}  Response{data=User}
// @Failure      404  {object}  ErrorResponse
// @Security     BearerAuth
// @Router       /users/{id} [get]
func getUser(c *gin.Context) { /* ... */ }
```

### 7.3 契约测试（Contract Testing）

```go
// 契约测试 — 验证 API 符合约定
// 使用 Pact 框架的思路：Consumer 定义期望，Provider 验证实现

package api_test

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// Contract 定义接口契约
type Contract struct {
    Method         string
    Path           string
    RequestBody    string
    ExpectedStatus int
    ExpectedFields []string // 响应中必须存在的字段
}

var userContracts = []Contract{
    {
        Method:         "GET",
        Path:           "/api/v1/users",
        ExpectedStatus: 200,
        ExpectedFields: []string{"code", "data", "meta"},
    },
    {
        Method:         "POST",
        Path:           "/api/v1/users",
        RequestBody:    `{"name":"Alice","email":"alice@example.com"}`,
        ExpectedStatus: 201,
        ExpectedFields: []string{"code", "data"},
    },
    {
        Method:         "GET",
        Path:           "/api/v1/users/999999",
        ExpectedStatus: 404,
        ExpectedFields: []string{"code", "message"},
    },
}

func TestUserContracts(t *testing.T) {
    router := setupRouter() // 初始化路由

    for _, contract := range userContracts {
        t.Run(contract.Method+" "+contract.Path, func(t *testing.T) {
            var req *http.Request
            if contract.RequestBody != "" {
                req = httptest.NewRequest(contract.Method, contract.Path,
                    strings.NewReader(contract.RequestBody))
                req.Header.Set("Content-Type", "application/json")
            } else {
                req = httptest.NewRequest(contract.Method, contract.Path, nil)
            }

            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)

            // 验证状态码
            assert.Equal(t, contract.ExpectedStatus, w.Code)

            // 验证响应字段
            var body map[string]interface{}
            err := json.Unmarshal(w.Body.Bytes(), &body)
            require.NoError(t, err)

            for _, field := range contract.ExpectedFields {
                assert.Contains(t, body, field,
                    "response missing field: %s", field)
            }
        })
    }
}
```

---

## 8. 接口性能优化

> ⭐⭐ 核心考点：从多个维度优化 API 性能。

### 8.1 批量接口设计

```go
// ❌ 反模式：N次网络请求
// GET /api/v1/users/1
// GET /api/v1/users/2
// ...
// GET /api/v1/users/100

// ✅ 批量接口
// GET /api/v1/users?ids=1,2,3,...,100
// POST /api/v1/users/batch-get  {"ids": [1, 2, ..., 100]}

type BatchGetRequest struct {
    IDs []int64 `json:"ids" binding:"required,max=100"` // 限制单次最多100个
}

func batchGetUsers(c *gin.Context) {
    var req BatchGetRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"message": err.Error()})
        return
    }

    // 批量查询
    users, err := userRepo.FindByIDs(c, req.IDs)
    if err != nil {
        c.JSON(500, gin.H{"message": "internal error"})
        return
    }

    // 返回 map 格式，方便客户端按 ID 查找
    userMap := make(map[int64]*User)
    for _, u := range users {
        userMap[u.ID] = u
    }

    c.JSON(200, gin.H{
        "data":    userMap,
        "missing": findMissing(req.IDs, userMap),
    })
}

func findMissing(ids []int64, m map[int64]*User) []int64 {
    var missing []int64
    for _, id := range ids {
        if _, ok := m[id]; !ok {
            missing = append(missing, id)
        }
    }
    return missing
}
```

### 8.2 分页设计

| 方式 | 实现 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| Offset 分页 | `LIMIT 20 OFFSET 40` | 简单，支持跳页 | 深翻页性能差，数据漂移 | 后台管理 |
| Cursor 分页 | `WHERE id > cursor LIMIT 20` | 性能稳定，无漂移 | 不支持跳页 | 瀑布流、时间线 |
| Keyset 分页 | `WHERE (sort_key, id) > (v, k)` | 支持排序，性能好 | 实现复杂 | 排序列表 |

```go
// Cursor 分页实现
type CursorPagination struct {
    Cursor  string `form:"cursor"`
    Limit   int    `form:"limit,default=20" binding:"max=100"`
}

type PagedResult struct {
    Data       interface{} `json:"data"`
    NextCursor string      `json:"next_cursor,omitempty"`
    HasMore    bool        `json:"has_more"`
}

func listUsersWithCursor(c *gin.Context) {
    var page CursorPagination
    if err := c.ShouldBindQuery(&page); err != nil {
        c.JSON(400, gin.H{"message": err.Error()})
        return
    }

    // 解码 cursor（通常是 base64 编码的 ID 或时间戳）
    var lastID int64
    if page.Cursor != "" {
        decoded, err := base64.StdEncoding.DecodeString(page.Cursor)
        if err == nil {
            lastID, _ = strconv.ParseInt(string(decoded), 10, 64)
        }
    }

    // 查询 limit+1 条，多查一条判断是否有下一页
    query := db.Model(&User{}).Order("id ASC").Limit(page.Limit + 1)
    if lastID > 0 {
        query = query.Where("id > ?", lastID)
    }

    var users []User
    query.Find(&users)

    hasMore := len(users) > page.Limit
    if hasMore {
        users = users[:page.Limit]
    }

    var nextCursor string
    if hasMore && len(users) > 0 {
        lastUser := users[len(users)-1]
        nextCursor = base64.StdEncoding.EncodeToString(
            []byte(strconv.FormatInt(lastUser.ID, 10)))
    }

    c.JSON(200, PagedResult{
        Data:       users,
        NextCursor: nextCursor,
        HasMore:    hasMore,
    })
}
```

### 8.3 缓存策略

```go
// HTTP 缓存头
func cacheMiddleware(maxAge int) gin.HandlerFunc {
    return func(c *gin.Context) {
        if c.Request.Method == http.MethodGet {
            c.Header("Cache-Control", fmt.Sprintf("public, max-age=%d", maxAge))

            // ETag 支持
            c.Next()

            // 生成 ETag
            if c.Writer.Status() == 200 {
                // 简化示例：基于响应体生成 ETag
                // 实际应在业务层基于数据版本生成
            }
        } else {
            c.Next()
        }
    }
}

// 应用层缓存（Redis）
type CachedAPI struct {
    rdb    *redis.Client
    prefix string
}

func (c *CachedAPI) CacheGet(ctx context.Context, key string, ttl time.Duration,
    fetcher func() (interface{}, error)) ([]byte, error) {

    cacheKey := c.prefix + ":" + key

    // 1. 尝试从缓存获取
    cached, err := c.rdb.Get(ctx, cacheKey).Bytes()
    if err == nil {
        return cached, nil
    }

    // 2. 缓存未命中，查询数据源
    data, err := fetcher()
    if err != nil {
        return nil, err
    }

    // 3. 写入缓存
    jsonData, _ := json.Marshal(data)
    c.rdb.Set(ctx, cacheKey, jsonData, ttl)

    return jsonData, nil
}

// 缓存失效策略
// 1. 写后失效（Write-Through）
func (c *CachedAPI) InvalidateOnWrite(ctx context.Context, key string) {
    c.rdb.Del(ctx, c.prefix+":"+key)
}

// 2. 延迟双删（防缓存不一致）
func (c *CachedAPI) DelayedDoubleDelete(ctx context.Context, key string) {
    cacheKey := c.prefix + ":" + key
    c.rdb.Del(ctx, cacheKey)      // 第一次删除
    go func() {
        time.Sleep(500 * time.Millisecond) // 等待主从同步
        c.rdb.Del(ctx, cacheKey)           // 第二次删除
    }()
}
```

### 8.4 响应压缩与连接池

```go
// Gzip 压缩中间件
import "github.com/gin-contrib/gzip"

r := gin.Default()
r.Use(gzip.Gzip(gzip.DefaultCompression))

// 自定义 HTTP Client 连接池
var httpClient = &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,              // 最大空闲连接
        MaxIdleConnsPerHost: 20,               // 每个 Host 最大空闲连接
        MaxConnsPerHost:     50,               // 每个 Host 最大总连接
        IdleConnTimeout:     90 * time.Second, // 空闲连接超时
        TLSHandshakeTimeout: 10 * time.Second,
        DisableKeepAlives:   false,            // 启用 Keep-Alive
    },
    Timeout: 30 * time.Second,
}

// 数据库连接池配置
func setupDB() *sql.DB {
    db, _ := sql.Open("mysql", dsn)
    db.SetMaxOpenConns(50)                  // 最大打开连接
    db.SetMaxIdleConns(25)                  // 最大空闲连接
    db.SetConnMaxLifetime(5 * time.Minute)  // 连接最大生命周期
    db.SetConnMaxIdleTime(3 * time.Minute)  // 空闲连接最大生命周期
    return db
}
```

**性能优化总结表：**

| 优化手段 | 效果 | 复杂度 | 优先级 |
|----------|------|--------|--------|
| 批量接口 | 减少网络往返 | 低 | 高 |
| Cursor 分页 | 避免深翻页性能劣化 | 中 | 高 |
| HTTP 缓存 | 减少重复请求 | 低 | 高 |
| Redis 缓存 | 减少 DB 查询 | 中 | 高 |
| Gzip 压缩 | 减少传输大小 60-80% | 低 | 中 |
| 连接池 | 复用连接，减少握手开销 | 低 | 高 |
| 字段裁剪 | 减少响应体大小 | 中 | 中 |
| 异步处理 | 降低响应延迟 | 高 | 视场景 |

---

## 9. API 安全

> ⭐⭐ 核心考点：签名验证、防重放攻击是开放 API 安全的基础。

### 9.1 请求签名验证

```
签名算法流程:
1. 将所有请求参数按 key 字典序排序
2. 拼接为 key=value&key=value 格式
3. 拼接 timestamp + nonce + secret
4. 对拼接字符串做 HMAC-SHA256 签名
5. 服务端用同样的算法验证签名
```

```go
package security

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "fmt"
    "net/http"
    "sort"
    "strconv"
    "strings"
    "time"

    "github.com/gin-gonic/gin"
)

const (
    signatureHeader = "X-Signature"
    timestampHeader = "X-Timestamp"
    nonceHeader     = "X-Nonce"
    appKeyHeader    = "X-App-Key"
    maxTimeDiff     = 5 * time.Minute // 时间戳有效期
)

// 签名验证中间件
func SignatureAuth(getSecret func(appKey string) string) gin.HandlerFunc {
    return func(c *gin.Context) {
        appKey := c.GetHeader(appKeyHeader)
        signature := c.GetHeader(signatureHeader)
        timestamp := c.GetHeader(timestampHeader)
        nonce := c.GetHeader(nonceHeader)

        if appKey == "" || signature == "" || timestamp == "" || nonce == "" {
            c.JSON(http.StatusUnauthorized, gin.H{
                "code": 401, "message": "missing authentication headers",
            })
            c.Abort()
            return
        }

        // 1. 验证时间戳（防重放）
        ts, err := strconv.ParseInt(timestamp, 10, 64)
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"message": "invalid timestamp"})
            c.Abort()
            return
        }
        requestTime := time.Unix(ts, 0)
        if time.Since(requestTime).Abs() > maxTimeDiff {
            c.JSON(http.StatusUnauthorized, gin.H{
                "code": 401, "message": "timestamp expired",
            })
            c.Abort()
            return
        }

        // 2. 验证 nonce（防重放，与 Redis 结合使用）
        nonceKey := fmt.Sprintf("nonce:%s:%s", appKey, nonce)
        exists, _ := rdb.Exists(c, nonceKey).Result()
        if exists > 0 {
            c.JSON(http.StatusUnauthorized, gin.H{
                "code": 401, "message": "duplicate request (nonce reused)",
            })
            c.Abort()
            return
        }
        rdb.Set(c, nonceKey, "1", maxTimeDiff*2) // nonce 过期时间为时间窗口的2倍

        // 3. 计算签名并验证
        secret := getSecret(appKey)
        if secret == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"message": "invalid app key"})
            c.Abort()
            return
        }

        expectedSig := calculateSignature(c.Request, timestamp, nonce, secret)
        if !hmac.Equal([]byte(signature), []byte(expectedSig)) {
            c.JSON(http.StatusUnauthorized, gin.H{
                "code": 401, "message": "invalid signature",
            })
            c.Abort()
            return
        }

        c.Set("app_key", appKey)
        c.Next()
    }
}

func calculateSignature(req *http.Request, timestamp, nonce, secret string) string {
    // 收集所有查询参数
    params := make(map[string]string)
    for k, v := range req.URL.Query() {
        if len(v) > 0 {
            params[k] = v[0]
        }
    }

    // 按 key 排序
    keys := make([]string, 0, len(params))
    for k := range params {
        keys = append(keys, k)
    }
    sort.Strings(keys)

    // 拼接参数
    var parts []string
    for _, k := range keys {
        parts = append(parts, fmt.Sprintf("%s=%s", k, params[k]))
    }

    // 拼接 timestamp + nonce + path
    signStr := strings.Join(parts, "&")
    signStr += "&timestamp=" + timestamp
    signStr += "&nonce=" + nonce
    signStr += "&path=" + req.URL.Path

    // HMAC-SHA256
    h := hmac.New(sha256.New, []byte(secret))
    h.Write([]byte(signStr))
    return hex.EncodeToString(h.Sum(nil))
}
```

### 9.2 防重放攻击

```
防重放攻击三层防护:

1. 时间戳校验：请求时间与服务器时间差不超过 5 分钟
   → 防止使用过期请求重放

2. Nonce 校验：每个 nonce 只能使用一次（Redis记录）
   → 防止在时间窗口内重放

3. 签名校验：参数+时间戳+nonce一起签名
   → 防止篡改任何参数

三者配合，可以有效防止重放攻击。
```

### 9.3 IP 白名单

```go
func IPWhitelist(allowed []string) gin.HandlerFunc {
    allowedSet := make(map[string]bool)
    for _, ip := range allowed {
        allowedSet[ip] = true
    }

    return func(c *gin.Context) {
        clientIP := c.ClientIP()

        // 支持 CIDR 匹配
        for _, allowedIP := range allowed {
            if strings.Contains(allowedIP, "/") {
                _, network, err := net.ParseCIDR(allowedIP)
                if err == nil && network.Contains(net.ParseIP(clientIP)) {
                    c.Next()
                    return
                }
            } else if clientIP == allowedIP {
                c.Next()
                return
            }
        }

        c.JSON(http.StatusForbidden, gin.H{
            "code":    403,
            "message": "IP not allowed",
        })
        c.Abort()
    }
}

// 使用
r.Group("/internal").Use(IPWhitelist([]string{
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
}))
```

### 9.4 敏感数据脱敏

```go
// 脱敏工具
package mask

import "strings"

// Phone 手机号脱敏: 138****1234
func Phone(phone string) string {
    if len(phone) != 11 {
        return phone
    }
    return phone[:3] + "****" + phone[7:]
}

// Email 邮箱脱敏: a***e@example.com
func Email(email string) string {
    at := strings.Index(email, "@")
    if at <= 1 {
        return email
    }
    name := email[:at]
    domain := email[at:]
    if len(name) <= 2 {
        return name[:1] + "***" + domain
    }
    return name[:1] + "***" + name[len(name)-1:] + domain
}

// IDCard 身份证脱敏: 110***********1234
func IDCard(id string) string {
    if len(id) < 8 {
        return id
    }
    return id[:3] + strings.Repeat("*", len(id)-7) + id[len(id)-4:]
}

// BankCard 银行卡脱敏: **** **** **** 1234
func BankCard(card string) string {
    card = strings.ReplaceAll(card, " ", "")
    if len(card) < 4 {
        return card
    }
    return strings.Repeat("**** ", (len(card)-4)/4) + card[len(card)-4:]
}

// 响应拦截中间件 — 自动脱敏
type MaskConfig struct {
    Fields map[string]func(string) string
}

var defaultMaskConfig = MaskConfig{
    Fields: map[string]func(string) string{
        "phone":     Phone,
        "mobile":    Phone,
        "email":     Email,
        "id_card":   IDCard,
        "bank_card": BankCard,
    },
}
```

---

## 10. 实战案例：设计一套完整的开放平台 API 体系

> ⭐⭐⭐ 综合考点：开放平台 API 设计涵盖认证、限流、签名、版本、文档、SDK 等全方位设计。

### 10.1 整体架构

```
┌──────────────────────────────────────────────────────────────────────┐
│                        开放平台整体架构                                │
│                                                                      │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────────────────┐ │
│  │ 开发者    │───>│  开发者门户   │    │        管理后台              │ │
│  │ (第三方)  │    │  - 注册应用   │    │  - 应用审核                 │ │
│  └──────────┘    │  - 获取密钥   │    │  - 配额管理                 │ │
│       │          │  - 查看文档   │    │  - 监控告警                 │ │
│       │          │  - SDK下载    │    └─────────────────────────────┘ │
│       ▼          └──────────────┘                                    │
│  ┌──────────────────────────────────────────────────┐                │
│  │                  API Gateway                      │                │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐│                │
│  │  │ 认证    │ │ 限流    │ │ 签名    │ │ 日志/监控  ││                │
│  │  │ (OAuth) │ │(令牌桶) │ │ 验证    │ │ (追踪)    ││                │
│  │  └────────┘ └────────┘ └────────┘ └────────────┘│                │
│  └─────────────────────┬────────────────────────────┘                │
│                        │                                             │
│  ┌─────────────────────┼────────────────────────────┐                │
│  │     Service Mesh     │                            │                │
│  │  ┌─────────┐  ┌─────┴─────┐  ┌────────────────┐ │                │
│  │  │ 用户     │  │  订单      │  │  支付           │ │                │
│  │  │ Service  │  │  Service   │  │  Service       │ │                │
│  │  └─────────┘  └───────────┘  └────────────────┘ │                │
│  └──────────────────────────────────────────────────┘                │
└──────────────────────────────────────────────────────────────────────┘
```

### 10.2 应用注册与密钥管理

```go
// 开放平台应用模型
type App struct {
    ID          int64     `json:"id" gorm:"primaryKey"`
    AppKey      string    `json:"app_key" gorm:"uniqueIndex;size:32"`
    AppSecret   string    `json:"-" gorm:"size:64"` // 不返回给前端
    Name        string    `json:"name"`
    Description string    `json:"description"`
    OwnerID     int64     `json:"owner_id"`
    Status      string    `json:"status"` // pending/approved/rejected/suspended
    RateLimit   int       `json:"rate_limit"` // 每秒请求数
    Scopes      []string  `json:"scopes" gorm:"serializer:json"` // 权限范围
    IPWhitelist []string  `json:"ip_whitelist" gorm:"serializer:json"`
    CreatedAt   time.Time `json:"created_at"`
}

// 应用注册
func RegisterApp(ctx context.Context, req *RegisterAppRequest) (*App, error) {
    app := &App{
        AppKey:    generateAppKey(),    // 32 位随机字符串
        AppSecret: generateAppSecret(), // 64 位随机字符串
        Name:      req.Name,
        OwnerID:   req.OwnerID,
        Status:    "pending",
        RateLimit: 100, // 默认 100 QPS
        Scopes:    req.Scopes,
    }

    if err := db.Create(app).Error; err != nil {
        return nil, err
    }

    return app, nil
}

func generateAppKey() string {
    b := make([]byte, 16)
    rand.Read(b)
    return hex.EncodeToString(b)
}

func generateAppSecret() string {
    b := make([]byte, 32)
    rand.Read(b)
    return hex.EncodeToString(b)
}
```

### 10.3 OAuth 2.0 认证流程

```go
// OAuth 2.0 授权码模式
// 1. 第三方应用引导用户跳转授权页面
// 2. 用户同意授权，回调携带 code
// 3. 第三方用 code 换取 access_token
// 4. 用 access_token 调用 API

// 授权码端点
func authorize(c *gin.Context) {
    clientID := c.Query("client_id")
    redirectURI := c.Query("redirect_uri")
    scope := c.Query("scope")
    state := c.Query("state")

    // 验证应用
    app, err := findApp(clientID)
    if err != nil {
        c.JSON(400, gin.H{"error": "invalid_client"})
        return
    }

    // 验证 redirect_uri
    if !isValidRedirectURI(app, redirectURI) {
        c.JSON(400, gin.H{"error": "invalid_redirect_uri"})
        return
    }

    // 生成授权码（有效期 10 分钟，一次性）
    code := generateAuthCode()
    storeAuthCode(code, &AuthCodeData{
        AppKey:      clientID,
        UserID:      getCurrentUserID(c),
        Scope:       scope,
        RedirectURI: redirectURI,
    }, 10*time.Minute)

    // 重定向回第三方
    redirectURL := fmt.Sprintf("%s?code=%s&state=%s", redirectURI, code, state)
    c.Redirect(http.StatusFound, redirectURL)
}

// Token 端点
func token(c *gin.Context) {
    grantType := c.PostForm("grant_type")

    switch grantType {
    case "authorization_code":
        handleAuthCodeGrant(c)
    case "refresh_token":
        handleRefreshToken(c)
    case "client_credentials":
        handleClientCredentials(c)
    default:
        c.JSON(400, gin.H{"error": "unsupported_grant_type"})
    }
}

func handleAuthCodeGrant(c *gin.Context) {
    code := c.PostForm("code")
    clientID := c.PostForm("client_id")
    clientSecret := c.PostForm("client_secret")

    // 验证应用身份
    app, err := findAndVerifyApp(clientID, clientSecret)
    if err != nil {
        c.JSON(401, gin.H{"error": "invalid_client"})
        return
    }

    // 验证授权码
    codeData, err := getAndDeleteAuthCode(code) // 一次性使用
    if err != nil {
        c.JSON(400, gin.H{"error": "invalid_grant"})
        return
    }

    if codeData.AppKey != clientID {
        c.JSON(400, gin.H{"error": "invalid_grant"})
        return
    }

    // 生成 token
    accessToken := generateJWT(app, codeData.UserID, codeData.Scope, 2*time.Hour)
    refreshToken := generateRefreshToken()
    storeRefreshToken(refreshToken, codeData, 30*24*time.Hour)

    c.JSON(200, gin.H{
        "access_token":  accessToken,
        "token_type":    "Bearer",
        "expires_in":    7200,
        "refresh_token": refreshToken,
        "scope":         codeData.Scope,
    })
}
```

### 10.4 分级限流

```go
// 多级限流策略
type RateLimitConfig struct {
    // 全局限流
    GlobalQPS int

    // 应用级限流（不同套餐不同限额）
    AppLimits map[string]int // app_key -> QPS

    // 接口级限流
    APILimits map[string]int // path -> QPS

    // 用户级限流
    UserQPS int
}

func MultiLevelRateLimit(config *RateLimitConfig) gin.HandlerFunc {
    return func(c *gin.Context) {
        appKey := c.GetString("app_key")
        userID := c.GetString("user_id")
        path := c.Request.URL.Path

        // 1. 全局限流
        if !checkRate(c, "global", config.GlobalQPS) {
            c.JSON(429, gin.H{"error": "global rate limit exceeded"})
            c.Abort()
            return
        }

        // 2. 应用级限流
        appLimit := config.AppLimits[appKey]
        if appLimit == 0 {
            appLimit = 100 // 默认
        }
        if !checkRate(c, "app:"+appKey, appLimit) {
            c.JSON(429, gin.H{
                "error":       "app rate limit exceeded",
                "retry_after": 1,
            })
            c.Header("Retry-After", "1")
            c.Abort()
            return
        }

        // 3. 接口级限流
        if apiLimit, ok := config.APILimits[path]; ok {
            if !checkRate(c, "api:"+path, apiLimit) {
                c.JSON(429, gin.H{"error": "api rate limit exceeded"})
                c.Abort()
                return
            }
        }

        // 4. 用户级限流
        if userID != "" {
            if !checkRate(c, "user:"+userID, config.UserQPS) {
                c.JSON(429, gin.H{"error": "user rate limit exceeded"})
                c.Abort()
                return
            }
        }

        c.Next()
    }
}

// Redis 滑动窗口限流
func checkRate(ctx context.Context, key string, limit int) bool {
    now := time.Now().UnixMicro()
    windowStart := now - int64(time.Second/time.Microsecond)

    pipe := rdb.Pipeline()
    // 移除窗口外的记录
    pipe.ZRemRangeByScore(ctx, "ratelimit:"+key, "0", strconv.FormatInt(windowStart, 10))
    // 添加当前请求
    pipe.ZAdd(ctx, "ratelimit:"+key, redis.Z{Score: float64(now), Member: now})
    // 计数
    countCmd := pipe.ZCard(ctx, "ratelimit:"+key)
    // 设置过期
    pipe.Expire(ctx, "ratelimit:"+key, 2*time.Second)

    pipe.Exec(ctx)

    return countCmd.Val() <= int64(limit)
}
```

### 10.5 SDK 设计思路

```go
// Go SDK 示例
package openapi

import (
    "context"
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "sort"
    "strconv"
    "strings"
    "time"

    "github.com/google/uuid"
)

type Client struct {
    appKey    string
    appSecret string
    baseURL   string
    httpClient *http.Client
}

func NewClient(appKey, appSecret, baseURL string) *Client {
    return &Client{
        appKey:    appKey,
        appSecret: appSecret,
        baseURL:   baseURL,
        httpClient: &http.Client{Timeout: 30 * time.Second},
    }
}

func (c *Client) Get(ctx context.Context, path string, params map[string]string) ([]byte, error) {
    return c.doRequest(ctx, http.MethodGet, path, params, nil)
}

func (c *Client) Post(ctx context.Context, path string, body interface{}) ([]byte, error) {
    return c.doRequest(ctx, http.MethodPost, path, nil, body)
}

func (c *Client) doRequest(ctx context.Context, method, path string,
    params map[string]string, body interface{}) ([]byte, error) {

    // 构建 URL
    fullURL := c.baseURL + path
    if len(params) > 0 {
        parts := make([]string, 0, len(params))
        for k, v := range params {
            parts = append(parts, k+"="+v)
        }
        fullURL += "?" + strings.Join(parts, "&")
    }

    // 序列化请求体
    var bodyReader io.Reader
    if body != nil {
        data, err := json.Marshal(body)
        if err != nil {
            return nil, fmt.Errorf("marshal body: %w", err)
        }
        bodyReader = bytes.NewReader(data)
    }

    req, err := http.NewRequestWithContext(ctx, method, fullURL, bodyReader)
    if err != nil {
        return nil, err
    }

    // 添加签名头
    timestamp := strconv.FormatInt(time.Now().Unix(), 10)
    nonce := uuid.New().String()
    signature := c.sign(req, timestamp, nonce)

    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-App-Key", c.appKey)
    req.Header.Set("X-Timestamp", timestamp)
    req.Header.Set("X-Nonce", nonce)
    req.Header.Set("X-Signature", signature)

    // 发送请求
    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("http request failed: %w", err)
    }
    defer resp.Body.Close()

    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("read response: %w", err)
    }

    if resp.StatusCode >= 400 {
        return nil, fmt.Errorf("api error (status %d): %s", resp.StatusCode, respBody)
    }

    return respBody, nil
}

func (c *Client) sign(req *http.Request, timestamp, nonce string) string {
    params := make(map[string]string)
    for k, v := range req.URL.Query() {
        if len(v) > 0 {
            params[k] = v[0]
        }
    }

    keys := make([]string, 0, len(params))
    for k := range params {
        keys = append(keys, k)
    }
    sort.Strings(keys)

    var parts []string
    for _, k := range keys {
        parts = append(parts, k+"="+params[k])
    }
    signStr := strings.Join(parts, "&")
    signStr += "&timestamp=" + timestamp + "&nonce=" + nonce + "&path=" + req.URL.Path

    h := hmac.New(sha256.New, []byte(c.appSecret))
    h.Write([]byte(signStr))
    return hex.EncodeToString(h.Sum(nil))
}

// 使用示例
func example() {
    client := NewClient("my-app-key", "my-app-secret", "https://api.example.com/v1")

    // 获取用户列表
    data, err := client.Get(context.Background(), "/users", map[string]string{
        "page":     "1",
        "per_page": "20",
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(data))

    // 创建用户
    data, err = client.Post(context.Background(), "/users", map[string]interface{}{
        "name":  "Alice",
        "email": "alice@example.com",
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(data))
}
```

---

## 面试题自查

### 一、治理补充（3 条经验）

1. **API First 原则**：先设计 API 契约（OpenAPI/Proto），再编码实现。通过 Contract-First 开发可让前后端并行开发，减少联调成本。
2. **灰度发布与流量染色**：新版本 API 上线时先导入 1%-5% 流量，通过 Header 染色（`X-Traffic-Tag: canary`）追踪全链路日志，确认无异常后全量切换。
3. **API 度量黄金指标**：每个 API 至少监控 RED 指标 — Rate（请求速率）、Errors（错误率）、Duration（P50/P95/P99 延迟）。结合 SLO（如 P99 < 200ms, 可用性 > 99.9%）建立告警。

---

### 二、初级高频速刷（5 题简答）

**Q1：REST 中 PUT 和 PATCH 的区别是什么？**
PUT 是全量替换，PATCH 是部分更新。PUT 需要传递完整资源，未包含的字段会被清空；PATCH 只传需要修改的字段，其他字段保持不变。PUT 是幂等的，PATCH 不一定。

**Q2：HTTP 状态码 401 和 403 的区别？**
401 Unauthorized 表示"未认证"——请求未携带有效凭证（Token 缺失或过期）；403 Forbidden 表示"已认证但无权限"——身份已确认但没有访问该资源的权限。

**Q3：什么是 API 幂等性？为什么 POST 不是幂等的？**
幂等性指同一操作执行一次和多次效果相同。POST 通常用于创建资源，每次调用都可能创建一个新资源，因此不是幂等的。GET/PUT/DELETE 是幂等的。

**Q4：gRPC 和 REST 各适合什么场景？**
gRPC 适合微服务间内部通信（高性能、强类型、流式支持）；REST 适合面向外部的开放 API（浏览器友好、易调试、HTTP 缓存友好）。

**Q5：API 版本管理主流方式是什么？**
URL Path 版本（如 `/api/v1/users`）是业界主流（GitHub、Google、Stripe 均使用）。优点是直观明了、缓存友好、调试方便。

---

### 三、中级高频进阶（5 题）

**Q1：如何设计一个防重复提交的方案？**
三种方案：
- **幂等键**：客户端生成 UUID 作为 `Idempotency-Key` Header 传入，服务端用 Redis 缓存结果，相同 key 直接返回缓存。适合通用 API。
- **去重表**：在数据库中创建唯一索引的去重表，与业务在同一事务中写入。适合强一致性场景。
- **Token 机制**：先获取一次性 Token，提交时携带 Token，服务端用 Lua 脚本原子性验证并删除。适合表单场景。

**Q2：Protobuf 为什么比 JSON 快？**
- 二进制编码（vs JSON 文本编码），数据更紧凑
- 数字用 Varint 编码，小数字仅需 1-2 字节
- 字段用数字 Tag 标识（vs JSON 的字符串 key）
- 不传零值字段，节省空间
- 编解码逻辑编译期生成，无反射

**Q3：Cursor 分页相比 Offset 分页的优势是什么？**
Offset 分页的 `LIMIT 20 OFFSET 10000` 需扫描前 10000 行再跳过，深翻页性能极差。Cursor 分页用 `WHERE id > last_id LIMIT 20` 直接定位到起始行，走索引扫描，性能稳定不受页码影响。且 Offset 分页存在"数据漂移"问题（翻页期间数据新增/删除导致重复或遗漏），Cursor 分页没有此问题。

**Q4：API 网关的核心职责是什么？为什么不让服务直接暴露？**
核心职责：路由转发、认证鉴权、限流熔断、日志监控、协议转换、灰度发布。不让服务直接暴露是因为：(1) 横切关注点下沉避免每个服务重复实现；(2) 统一入口便于安全管控和审计；(3) 隐藏内部架构拓扑；(4) 客户端只需对接一个域名。

**Q5：如何防止 API 重放攻击？**
三层防护：(1) 时间戳校验——请求必须在 5 分钟时间窗口内；(2) Nonce 唯一性——每个 nonce 只能使用一次（Redis SET + TTL）；(3) HMAC 签名——参数+时间戳+nonce 一起签名，防止篡改。三者结合可有效杜绝重放攻击。

---

### 四、深入问答（20+ 题 Q&A 格式）

**Q1：设计 RESTful API 时，对于"非 CRUD 操作"（如"审批"、"发布"、"冻结"）如何处理？**

A：两种常见方案：

方案一：**子资源/动作资源**

```
POST /api/v1/orders/123/approve     # 审批
POST /api/v1/articles/456/publish   # 发布
POST /api/v1/users/789/freeze       # 冻结
```

方案二：**状态变更 PATCH**

```
PATCH /api/v1/orders/123
{"status": "approved"}

PATCH /api/v1/articles/456
{"status": "published"}
```

推荐方案一，因为它能更好地表达业务语义，可以携带审批意见等额外参数，且更容易做权限控制。方案二适合简单的状态切换。

---

**Q2：gRPC 的四种通信模式分别适用什么场景？请举例。**

A：

| 模式 | 场景 | 例子 |
|------|------|------|
| Unary | 普通请求响应 | 用户登录、查询订单详情 |
| Server Streaming | 服务端持续推送 | 股票行情推送、日志实时流、文件下载 |
| Client Streaming | 客户端持续发送 | 文件上传、传感器数据上报、批量导入 |
| Bidirectional | 双向实时通信 | 聊天、在线协作编辑、实时游戏同步 |

```go
// Server Streaming 示例：股票行情推送
func (s *StockServer) SubscribeQuotes(req *SubscribeReq, stream StockService_SubscribeQuotesServer) error {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-stream.Context().Done():
            return nil
        case <-ticker.C:
            quote := getLatestQuote(req.Symbol)
            if err := stream.Send(quote); err != nil {
                return err
            }
        }
    }
}
```

---

**Q3：GraphQL 的 N+1 问题本质是什么？DataLoader 的批处理窗口机制如何工作？**

A：N+1 问题的本质是 **Resolver 逐条执行关联查询**。每个 User 的 `orders` Resolver 独立执行一次 SQL 查询，100 个用户就执行 100 次 `SELECT * FROM orders WHERE user_id = ?`。

DataLoader 的工作原理：
1. **收集阶段**：在一个事件循环/协程调度周期内，DataLoader 不立即执行查询，而是将所有 `Load(key)` 调用收集到一个队列中
2. **批处理窗口**：等待一小段时间（如 1ms 或一个 tick），收集所有待加载的 key
3. **统一执行**：将收集到的 key 合并为一次批量查询 `SELECT * FROM orders WHERE user_id IN (...)`
4. **结果分发**：按 key 将结果拆分给各个等待的 Resolver

关键配置：`batchCapacity`（最大批次大小）和 `wait`（等待窗口时长），需根据业务权衡延迟和批量效率。

---

**Q4：如何实现 API 的优雅降级？服务不可用时返回什么？**

A：

```go
// 熔断 + 降级策略
type DegradedResponse struct {
    Data      interface{} `json:"data"`
    Degraded  bool        `json:"_degraded"`
    Message   string      `json:"_message,omitempty"`
}

func getUserWithFallback(c *gin.Context) {
    userID := c.Param("id")

    // 1. 先尝试正常调用
    user, err := userService.GetUser(c, userID)
    if err == nil {
        c.JSON(200, DegradedResponse{Data: user, Degraded: false})
        return
    }

    // 2. 降级：尝试从缓存获取（即使是过期数据）
    cached, err := cache.GetStale(c, "user:"+userID)
    if err == nil {
        c.JSON(200, DegradedResponse{
            Data:     cached,
            Degraded: true,
            Message:  "serving stale data due to upstream unavailability",
        })
        return
    }

    // 3. 兜底：返回默认值或部分数据
    c.JSON(200, DegradedResponse{
        Data:     map[string]interface{}{"id": userID, "name": "Unknown"},
        Degraded: true,
        Message:  "service temporarily unavailable, returning default data",
    })
}
```

降级策略优先级：热缓存 → 过期缓存 → 默认值 → 503。关键是携带 `_degraded` 标记让客户端知道数据可能不是最新的。

---

**Q5：如何设计一个支持多租户的 API 系统？租户隔离有哪些策略？**

A：

| 隔离级别 | 实现 | 优点 | 缺点 |
|----------|------|------|------|
| 数据库级别 | 每个租户独立数据库 | 强隔离，性能互不影响 | 成本高，运维复杂 |
| Schema 级别 | 同库不同 Schema | 较好隔离，成本适中 | Schema 迁移复杂 |
| 行级别 | 同表加 `tenant_id` 列 | 成本低，管理简单 | 需严格过滤，存在数据泄露风险 |

```go
// 行级隔离中间件
func TenantMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        tenantID := c.GetHeader("X-Tenant-ID")
        if tenantID == "" {
            // 从 token 中解析
            tenantID = extractTenantFromToken(c)
        }
        if tenantID == "" {
            c.JSON(400, gin.H{"error": "tenant_id required"})
            c.Abort()
            return
        }
        c.Set("tenant_id", tenantID)
        c.Next()
    }
}

// 所有查询自动注入 tenant_id 条件
func (r *UserRepo) List(ctx context.Context) ([]User, error) {
    tenantID := ctx.Value("tenant_id").(string)
    var users []User
    return users, r.db.Where("tenant_id = ?", tenantID).Find(&users).Error
}
```

---

**Q6：Protobuf 中 `optional` / `repeated` / `map` / `oneof` 各自的编码方式和使用场景？**

A：

```protobuf
message Example {
    optional string nickname = 1;  // proto3 中 optional 允许区分"未设置"和"零值"
    repeated string tags = 2;      // 数组，编码为多个同 tag 的 TLV
    map<string, int32> scores = 3; // 等价于 repeated MapEntry { string key = 1; int32 value = 2; }
    oneof contact {                // 互斥字段，同一时刻只能设置一个
        string email = 4;
        string phone = 5;
    }
}
```

| 类型 | Wire Type | 编码方式 | 场景 |
|------|-----------|----------|------|
| optional | 与字段类型一致 | has_xxx 位标记是否设置 | 区分空值与默认值 |
| repeated | 2 (Length-delimited) | Packed：`tag + length + v1v2v3...` | 列表/数组 |
| map | 2 (Length-delimited) | 等价于 `repeated Entry` 的嵌套消息 | 键值对 |
| oneof | 与各字段类型一致 | 仅编码已设置的字段 | 联合类型/多选一 |

---

**Q7：如何设计一个接口同时兼容 gRPC 和 REST（gRPC-Gateway）？**

A：gRPC-Gateway 通过 Protobuf 注解自动生成 REST 反向代理。

```protobuf
import "google/api/annotations.proto";

service UserService {
    rpc GetUser(GetUserRequest) returns (User) {
        option (google.api.http) = {
            get: "/api/v1/users/{id}"
        };
    }

    rpc CreateUser(CreateUserRequest) returns (User) {
        option (google.api.http) = {
            post: "/api/v1/users"
            body: "*"
        };
    }

    rpc UpdateUser(UpdateUserRequest) returns (User) {
        option (google.api.http) = {
            put: "/api/v1/users/{id}"
            body: "*"
        };
    }
}
```

```go
// 同时启动 gRPC 和 REST
func main() {
    // gRPC Server
    grpcServer := grpc.NewServer()
    pb.RegisterUserServiceServer(grpcServer, &userServer{})

    // gRPC-Gateway (REST → gRPC 的反向代理)
    ctx := context.Background()
    mux := runtime.NewServeMux()
    opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
    err := pb.RegisterUserServiceHandlerFromEndpoint(ctx, mux, "localhost:50051", opts)
    if err != nil {
        log.Fatal(err)
    }

    // 启动两个监听
    go func() {
        lis, _ := net.Listen("tcp", ":50051")
        grpcServer.Serve(lis)
    }()

    log.Println("REST gateway on :8080, gRPC on :50051")
    http.ListenAndServe(":8080", mux)
}
```

内部微服务通过 gRPC 通信（高性能），外部客户端通过 REST 调用（兼容性好），一份 Proto 定义同时生成两套接口。

---

**Q8：如何设计安全的文件上传 API？**

A：

```go
func uploadFile(c *gin.Context) {
    // 1. 限制文件大小（中间件层已限制 MaxMultipartMemory）
    file, header, err := c.Request.FormFile("file")
    if err != nil {
        c.JSON(400, gin.H{"error": "no file uploaded"})
        return
    }
    defer file.Close()

    // 2. 检查文件大小（二次校验）
    if header.Size > 10*1024*1024 { // 10MB
        c.JSON(400, gin.H{"error": "file too large (max 10MB)"})
        return
    }

    // 3. 检查文件类型（基于 Magic Number，不信任扩展名）
    buf := make([]byte, 512)
    file.Read(buf)
    contentType := http.DetectContentType(buf)
    file.Seek(0, io.SeekStart) // 重置读取位置

    allowedTypes := map[string]bool{
        "image/jpeg": true,
        "image/png":  true,
        "image/gif":  true,
        "application/pdf": true,
    }
    if !allowedTypes[contentType] {
        c.JSON(400, gin.H{"error": "unsupported file type: " + contentType})
        return
    }

    // 4. 生成安全的文件名（防止路径穿越）
    ext := filepath.Ext(header.Filename)
    safeFilename := uuid.New().String() + ext

    // 5. 上传到对象存储（不保存到本地）
    objectKey := fmt.Sprintf("uploads/%s/%s", time.Now().Format("2006/01/02"), safeFilename)
    url, err := objectStorage.Upload(c, objectKey, file, contentType)
    if err != nil {
        c.JSON(500, gin.H{"error": "upload failed"})
        return
    }

    // 6. 异步病毒扫描
    go virusScanner.Scan(objectKey)

    c.JSON(201, gin.H{
        "url":      url,
        "filename": safeFilename,
        "size":     header.Size,
        "type":     contentType,
    })
}
```

关键安全措施：限制大小、基于 Magic Number 检测类型、随机化文件名、不保存到 Web 根目录、异步病毒扫描。

---

**Q9：如何实现 API 的请求/响应日志审计，且不影响性能？**

A：

```go
// 异步日志审计中间件
type AuditLog struct {
    RequestID   string        `json:"request_id"`
    Method      string        `json:"method"`
    Path        string        `json:"path"`
    StatusCode  int           `json:"status_code"`
    Duration    time.Duration `json:"duration"`
    ClientIP    string        `json:"client_ip"`
    UserID      string        `json:"user_id"`
    RequestBody string        `json:"request_body,omitempty"` // 脱敏后
    RespSize    int           `json:"response_size"`
}

var auditChan = make(chan *AuditLog, 10000) // 缓冲通道

func init() {
    // 后台批量写入
    go func() {
        batch := make([]*AuditLog, 0, 100)
        ticker := time.NewTicker(time.Second)
        for {
            select {
            case log := <-auditChan:
                batch = append(batch, log)
                if len(batch) >= 100 {
                    flushAuditLogs(batch)
                    batch = batch[:0]
                }
            case <-ticker.C:
                if len(batch) > 0 {
                    flushAuditLogs(batch)
                    batch = batch[:0]
                }
            }
        }
    }()
}

func AuditMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }
        c.Header("X-Request-ID", requestID)

        c.Next()

        // 异步写入审计日志（不阻塞请求）
        select {
        case auditChan <- &AuditLog{
            RequestID:  requestID,
            Method:     c.Request.Method,
            Path:       c.Request.URL.Path,
            StatusCode: c.Writer.Status(),
            Duration:   time.Since(start),
            ClientIP:   c.ClientIP(),
            UserID:     c.GetString("user_id"),
            RespSize:   c.Writer.Size(),
        }:
        default:
            // 通道满了就丢弃（性能优先于审计完整性）
        }
    }
}

func flushAuditLogs(logs []*AuditLog) {
    // 批量写入 ES / ClickHouse / Kafka
}
```

关键：用 buffered channel 异步化、批量写入、通道满时丢弃（非阻塞）、敏感数据脱敏。

---

**Q10：如何处理 API 的向后兼容？字段重命名、类型变更、结构调整的迁移策略？**

A：

```
向后兼容变更（安全，不需新版本）:
  ✅ 新增可选字段
  ✅ 新增新端点
  ✅ 新增枚举值
  ✅ 放宽参数约束（如 max_length 从 50 放宽到 100）

破坏性变更（需要版本迁移）:
  ❌ 删除字段 → 先标记 deprecated，保留两个版本周期
  ❌ 重命名字段 → 新旧字段同时返回，逐步迁移
  ❌ 修改类型 → 新字段名，旧字段标记废弃
  ❌ 修改结构 → 新版本 API
```

```go
// 字段重命名迁移策略：同时返回新旧字段
type UserV1 struct {
    ID       int64  `json:"id"`
    UserName string `json:"user_name"` // 旧字段名
}

type UserV2 struct {
    ID       int64  `json:"id"`
    Name     string `json:"name"`      // 新字段名
    UserName string `json:"user_name"` // 保留旧字段，标记 deprecated
}

// 在响应中同时返回，直到确认所有客户端已迁移
func getUser(c *gin.Context) {
    user := fetchUser(c.Param("id"))
    c.JSON(200, gin.H{
        "id":        user.ID,
        "name":      user.Name,      // 新
        "user_name": user.Name,      // 旧（兼容）
    })
}
```

---

**Q11：gRPC 的 Deadline 传播机制是什么？为什么微服务调用链需要 Deadline？**

A：

```go
// Deadline 传播：调用链上的每个服务会继承并递减剩余时间
// Client → ServiceA → ServiceB → ServiceC
// 如果 Client 设 5s 超时，ServiceA 处理了 1s，
// 则传给 ServiceB 的 deadline 是 4s，以此类推。

// 客户端设置 Deadline
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 123})
if err != nil {
    st, _ := status.FromError(err)
    if st.Code() == codes.DeadlineExceeded {
        log.Println("request timed out")
    }
}

// 服务端检查 Deadline
func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    // 检查剩余时间
    deadline, ok := ctx.Deadline()
    if ok {
        remaining := time.Until(deadline)
        log.Printf("remaining time: %v", remaining)

        // 如果剩余时间不足以完成操作，提前失败
        if remaining < 100*time.Millisecond {
            return nil, status.Errorf(codes.DeadlineExceeded, "insufficient time remaining")
        }
    }

    // 将 ctx 传递给下游调用，Deadline 自动传播
    order, err := orderClient.GetUserOrders(ctx, &pb.GetOrdersReq{UserId: req.Id})
    // ...
}
```

没有 Deadline 的后果：一个慢服务会导致整条调用链上所有服务的 goroutine/线程被阻塞，最终资源耗尽（级联故障）。Deadline 确保"如果结果已经没用了（客户端已超时），就不要继续浪费资源"。

---

**Q12：如何设计一个通用的 API 错误码体系？**

A：

```go
// 错误码设计：5位数字 = 2位服务码 + 3位错误码
// 10xxx: 用户服务
// 20xxx: 订单服务
// 30xxx: 支付服务

type BizError struct {
    HTTPCode int    `json:"-"`
    Code     int    `json:"code"`
    Message  string `json:"message"`
    I18nKey  string `json:"-"` // 国际化 key
}

var (
    // 通用错误 00xxx
    ErrSuccess       = &BizError{200, 0, "success", ""}
    ErrBadRequest    = &BizError{400, 400, "bad request", "error.bad_request"}
    ErrUnauth        = &BizError{401, 401, "unauthorized", "error.unauthorized"}
    ErrForbidden     = &BizError{403, 403, "forbidden", "error.forbidden"}
    ErrNotFound      = &BizError{404, 404, "not found", "error.not_found"}
    ErrInternal      = &BizError{500, 500, "internal error", "error.internal"}

    // 用户服务 10xxx
    ErrUserNotFound     = &BizError{404, 10001, "user not found", "user.not_found"}
    ErrEmailDuplicate   = &BizError{409, 10002, "email already registered", "user.email_dup"}
    ErrPasswordWeak     = &BizError{400, 10003, "password too weak", "user.pwd_weak"}
    ErrAccountLocked    = &BizError{403, 10004, "account is locked", "user.locked"}

    // 订单服务 20xxx
    ErrOrderNotFound    = &BizError{404, 20001, "order not found", "order.not_found"}
    ErrInsufficientStock = &BizError{409, 20002, "insufficient stock", "order.no_stock"}
    ErrOrderCancelled   = &BizError{409, 20003, "order already cancelled", "order.cancelled"}
)

// 统一错误响应格式
// {
//     "code": 10002,
//     "message": "email already registered",
//     "detail": "alice@example.com is already registered",
//     "request_id": "abc-123"
// }
```

---

**Q13：GraphQL 如何限制恶意查询（查询深度、复杂度攻击）？**

A：

```graphql
# 恶意查询示例：深度嵌套
query {
    user(id: 1) {
        orders {
            edges {
                node {
                    user {          # 反向引用
                        orders {
                            edges {
                                node {
                                    user {   # 无限嵌套
                                        ...
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

防御措施：

```go
// 1. 限制查询深度（通常 ≤ 10 层）
srv.Use(extension.DepthLimit(10))

// 2. 限制查询复杂度（根据字段权重计算）
//    - 标量字段：1
//    - 对象字段：2
//    - 列表字段：乘以 first 参数
srv.Use(extension.FixedComplexityLimit(500))

// 3. 查询白名单（生产环境推荐）
//    - 开发期间收集所有合法查询的 hash
//    - 生产环境只接受已注册的查询（Persisted Queries）
// 4. 请求超时
// 5. 分页强制 limit 上限（max=100）
```

---

**Q14：如何实现 API 的灰度发布？流量从 v1 逐步切到 v2？**

A：

```go
// 基于权重的流量分配
func GrayReleaseMiddleware(v2Weight int) gin.HandlerFunc { // v2Weight: 0-100
    return func(c *gin.Context) {
        version := "v1"

        // 策略1：指定用户灰度
        if c.GetHeader("X-Gray-User") == "true" {
            version = "v2"
        }

        // 策略2：基于用户ID哈希灰度
        userID := c.GetString("user_id")
        if userID != "" {
            hash := crc32.ChecksumIEEE([]byte(userID))
            if int(hash%100) < v2Weight {
                version = "v2"
            }
        }

        // 策略3：随机流量百分比
        if rand.Intn(100) < v2Weight {
            version = "v2"
        }

        c.Set("api_version", version)
        c.Header("X-API-Version", version) // 响应中标明版本
        c.Next()
    }
}
```

灰度发布流程：`0% → 1% (内测) → 5% (小流量) → 20% → 50% → 100%`，每步观察 RED 指标，异常立即回滚到 0%。

---

**Q15：分布式系统中，如何保证"创建订单 + 扣减库存"的幂等性？**

A：

```go
// 分布式幂等：基于全局唯一业务ID + 状态机

func CreateOrderIdempotent(ctx context.Context, orderID string, req *CreateOrderReq) error {
    // 1. 检查订单是否已存在
    existing, err := orderRepo.FindByBizID(ctx, orderID)
    if err == nil && existing != nil {
        // 已存在，根据状态决定
        switch existing.Status {
        case "completed":
            return nil // 已完成，幂等返回
        case "failed":
            // 可以重试
        case "processing":
            return ErrOrderProcessing
        }
    }

    // 2. 创建订单（pending 状态）
    order := &Order{
        BizID:  orderID,
        Status: "pending",
    }
    if err := orderRepo.Create(ctx, order); err != nil {
        if isDuplicateKeyError(err) {
            return nil // 并发创建，幂等
        }
        return err
    }

    // 3. 扣减库存（带幂等键）
    err = inventoryClient.Deduct(ctx, &DeductReq{
        IdempotencyKey: orderID, // 用订单ID作为幂等键
        ProductID:      req.ProductID,
        Quantity:        req.Quantity,
    })
    if err != nil {
        order.Status = "failed"
        orderRepo.Update(ctx, order)
        return err
    }

    // 4. 更新订单状态
    order.Status = "completed"
    orderRepo.Update(ctx, order)
    return nil
}
```

关键：全局唯一业务 ID（如订单号）贯穿整条链路，每个环节用这个 ID 做幂等，状态机防止重复执行已完成的步骤。

---

**Q16：如何设计一个高可用的 API 限流方案？单机限流 vs 分布式限流？**

A：

| 方案 | 算法 | 优点 | 缺点 |
|------|------|------|------|
| 单机令牌桶 | `golang.org/x/time/rate` | 简单高效，无外部依赖 | 多实例不共享配额 |
| 分布式滑动窗口 | Redis ZSET | 精确，全局视角 | Redis 依赖，网络延迟 |
| 分布式令牌桶 | Redis Lua | 平滑限流 | 实现复杂 |

```go
// Redis Lua 令牌桶
local key = KEYS[1]
local rate = tonumber(ARGV[1])      -- 每秒填充速率
local capacity = tonumber(ARGV[2])  -- 桶容量
local now = tonumber(ARGV[3])       -- 当前时间(微秒)
local requested = tonumber(ARGV[4]) -- 请求的令牌数

local fill_time = capacity / rate
local ttl = math.floor(fill_time * 2)

local last_tokens = tonumber(redis.call("hget", key, "tokens"))
if last_tokens == nil then
    last_tokens = capacity
end

local last_refreshed = tonumber(redis.call("hget", key, "ts"))
if last_refreshed == nil then
    last_refreshed = 0
end

local delta = math.max(0, now - last_refreshed)
local filled_tokens = math.min(capacity, last_tokens + (delta * rate / 1000000))

local allowed = filled_tokens >= requested
local new_tokens = filled_tokens
if allowed then
    new_tokens = filled_tokens - requested
end

redis.call("hset", key, "tokens", new_tokens)
redis.call("hset", key, "ts", now)
redis.call("expire", key, ttl)

return { allowed and 1 or 0, new_tokens }
```

高可用策略：Redis 不可用时**降级为单机限流**，避免限流系统故障导致所有请求被拒或全部放行。

---

**Q17：如何设计一个支持字段级权限控制的 API？**

A：

```go
// 不同角色看到不同字段
type FieldPermission struct {
    Role    string
    Visible []string
    Masked  []string // 需要脱敏的字段
}

var userFieldPerms = map[string]FieldPermission{
    "admin": {
        Visible: []string{"*"}, // 所有字段
        Masked:  []string{},
    },
    "operator": {
        Visible: []string{"id", "name", "email", "phone", "status"},
        Masked:  []string{"phone", "email"}, // 需脱敏
    },
    "guest": {
        Visible: []string{"id", "name", "avatar"},
        Masked:  []string{},
    },
}

func FilterFields(data map[string]interface{}, role string) map[string]interface{} {
    perm, ok := userFieldPerms[role]
    if !ok {
        perm = userFieldPerms["guest"]
    }

    result := make(map[string]interface{})

    if len(perm.Visible) == 1 && perm.Visible[0] == "*" {
        for k, v := range data {
            result[k] = v
        }
    } else {
        visibleSet := make(map[string]bool)
        for _, f := range perm.Visible {
            visibleSet[f] = true
        }
        for k, v := range data {
            if visibleSet[k] {
                result[k] = v
            }
        }
    }

    // 脱敏
    maskedSet := make(map[string]bool)
    for _, f := range perm.Masked {
        maskedSet[f] = true
    }
    for k, v := range result {
        if maskedSet[k] {
            if str, ok := v.(string); ok {
                result[k] = maskString(k, str)
            }
        }
    }

    return result
}
```

---

**Q18：WebSocket 和 gRPC 双向流的异同？什么场景选择哪个？**

A：

| 维度 | WebSocket | gRPC 双向流 |
|------|-----------|-------------|
| 协议 | WebSocket (WS/WSS) | HTTP/2 |
| 序列化 | 自定义（通常 JSON） | Protobuf |
| 浏览器支持 | 原生支持 | 需 gRPC-Web 代理 |
| 类型安全 | 无 | Proto 强类型 |
| 重连/恢复 | 需自行实现 | 需自行实现 |
| 负载均衡 | L7 需特殊支持 | L7 需 HTTP/2 感知 |
| 适用场景 | 浏览器实时通信 | 服务间实时通信 |

选择建议：
- **前端 ↔ 后端**：WebSocket（浏览器原生支持）或 SSE（单向推送）
- **服务 ↔ 服务**：gRPC 双向流（类型安全、高性能）

---

**Q19：如何设计 API 的重试策略？什么情况该重试，什么不该重试？**

A：

```go
// 重试策略
type RetryConfig struct {
    MaxRetries  int
    InitialWait time.Duration
    MaxWait     time.Duration
    Multiplier  float64       // 指数退避因子
    Jitter      bool          // 随机抖动
}

func WithRetry(config RetryConfig, fn func() error) error {
    var lastErr error
    wait := config.InitialWait

    for attempt := 0; attempt <= config.MaxRetries; attempt++ {
        err := fn()
        if err == nil {
            return nil
        }

        // 判断是否应该重试
        if !isRetryable(err) {
            return err
        }

        lastErr = err
        if attempt < config.MaxRetries {
            sleepDuration := wait
            if config.Jitter {
                sleepDuration = time.Duration(float64(wait) * (0.5 + rand.Float64()))
            }
            time.Sleep(sleepDuration)
            wait = time.Duration(float64(wait) * config.Multiplier)
            if wait > config.MaxWait {
                wait = config.MaxWait
            }
        }
    }
    return fmt.Errorf("all %d retries failed: %w", config.MaxRetries, lastErr)
}

func isRetryable(err error) bool {
    // ✅ 应该重试
    // - 5xx 服务端错误（502, 503, 504）
    // - 网络超时
    // - 连接被拒绝
    // - 429 Too Many Requests（按 Retry-After 等待）

    // ❌ 不应重试
    // - 4xx 客户端错误（400, 401, 403, 404, 409）
    // - 非幂等操作的不确定失败（POST 超时）

    if st, ok := status.FromError(err); ok {
        switch st.Code() {
        case codes.Unavailable, codes.DeadlineExceeded, codes.ResourceExhausted:
            return true
        case codes.InvalidArgument, codes.NotFound, codes.AlreadyExists,
             codes.PermissionDenied, codes.Unauthenticated:
            return false
        }
    }
    return false
}
```

**关键原则**：只有**幂等操作**的**瞬时故障**才应重试。非幂等 POST 超时后不知道服务端是否已处理，盲目重试可能导致重复创建。

---

**Q20：如何监控 API 的健康度？SLI / SLO / SLA 如何定义？**

A：

```
SLI (Service Level Indicator) — 度量指标
  - 可用性：成功请求数 / 总请求数
  - 延迟：P50, P95, P99 响应时间
  - 错误率：5xx 响应数 / 总请求数
  - 吞吐量：RPS (Requests Per Second)

SLO (Service Level Objective) — 目标值
  - 可用性 ≥ 99.9%（月度，允许 ~43 分钟停机）
  - P99 延迟 ≤ 200ms
  - 错误率 ≤ 0.1%

SLA (Service Level Agreement) — 合同承诺
  - SLO + 违约赔偿条款
  - 例：可用性低于 99.9% 赔偿 10% 月费
```

```go
// Prometheus 指标采集
import "github.com/prometheus/client_golang/prometheus"

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
        },
        []string{"method", "path"},
    )

    httpRequestsInFlight = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "http_requests_in_flight",
        Help: "Current in-flight HTTP requests",
    })
)

func MetricsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        httpRequestsInFlight.Inc()
        start := time.Now()

        c.Next()

        duration := time.Since(start).Seconds()
        status := strconv.Itoa(c.Writer.Status())
        path := c.FullPath() // 使用路由模板而非实际路径，避免高基数

        httpRequestsTotal.WithLabelValues(c.Request.Method, path, status).Inc()
        httpRequestDuration.WithLabelValues(c.Request.Method, path).Observe(duration)
        httpRequestsInFlight.Dec()
    }
}
```

---

**Q21：如何实现 API 的请求合并（Request Coalescing / Singleflight）？**

A：多个并发请求查询同一资源时，只执行一次实际查询，其他请求复用结果。

```go
import "golang.org/x/sync/singleflight"

var sf singleflight.Group

func getUser(c *gin.Context) {
    userID := c.Param("id")

    // 多个并发请求 GET /users/123 只会执行一次 DB 查询
    result, err, shared := sf.Do("user:"+userID, func() (interface{}, error) {
        return userRepo.FindByID(c, userID)
    })

    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }

    if shared {
        log.Printf("request coalesced for user:%s", userID)
    }

    c.JSON(200, result)
}
```

适用场景：热点数据查询、缓存击穿防护。注意 singleflight 的结果会被所有等待者共享，确保返回的数据不会被某一个请求修改。

---

**Q22：如何设计长耗时 API（如报表导出、视频转码）？**

A：采用**异步轮询**或 **Webhook 回调**模式。

```go
// 方案一：异步轮询
// 1. 提交任务
// POST /api/v1/reports
// → 201 {"task_id": "abc-123", "status": "processing"}

// 2. 轮询状态
// GET /api/v1/reports/abc-123
// → 200 {"status": "processing", "progress": 60}
// → 200 {"status": "completed", "download_url": "https://..."}

func createReport(c *gin.Context) {
    taskID := uuid.New().String()

    // 异步处理
    go func() {
        result, err := generateReport(context.Background(), taskID)
        if err != nil {
            updateTaskStatus(taskID, "failed", err.Error())
            return
        }
        updateTaskStatus(taskID, "completed", result.URL)
    }()

    c.JSON(http.StatusAccepted, gin.H{ // 注意是 202 Accepted
        "task_id": taskID,
        "status":  "processing",
        "poll_url": fmt.Sprintf("/api/v1/reports/%s", taskID),
    })
}

// 方案二：Webhook 回调
// POST /api/v1/reports
// Body: {"callback_url": "https://client.com/webhook/report"}
// → 202 Accepted

// 任务完成后回调:
// POST https://client.com/webhook/report
// Body: {"task_id": "abc-123", "status": "completed", "download_url": "..."}
```

核心：返回 **202 Accepted**（而非 200），提供轮询 URL 或 Webhook 回调机制。

---

**Q23：OpenAPI/Swagger 文档如何保证与代码一致？**

A：三种策略：

| 策略 | 方式 | 优点 | 缺点 |
|------|------|------|------|
| Code First | 代码注解生成文档（swag） | 代码即文档，永远一致 | 注解侵入代码 |
| API First | 先写 OpenAPI YAML，生成代码 | 设计先行，前后端并行 | 需要代码生成工具 |
| 契约测试 | CI 中验证代码是否符合 spec | 双重保障 | 需额外维护测试 |

推荐组合：**API First + CI 契约测试**。先设计 OpenAPI 规范 → 代码生成 Server Stub → 实现业务逻辑 → CI 中运行契约测试验证一致性。

---

本文档覆盖了 API 设计与治理的核心知识点，从设计范式（REST/gRPC/GraphQL）到质量保障（幂等性/版本管理/文档测试），再到基础设施（网关/性能/安全），最后以开放平台实战串联所有知识。面试前重点复习 ⭐⭐⭐ 标记的章节和深入问答部分。

### 开放式设计题

**D1：设计一个面向第三方开发者的开放API平台（类似微信开放平台），需要考虑什么？**

**参考思路**：
- 接入管理：应用注册/审核、AppID+AppSecret分发、OAuth2.0授权（授权码模式+PKCE）
- API治理：版本管理（URL路径版本）、限流（按AppID多维度限流）、配额（免费/基础/企业分级）
- 安全：请求签名验证（HMAC-SHA256+时间戳+Nonce防重放）、IP白名单、敏感数据加密传输
- 文档与体验：OpenAPI自动生成文档、Sandbox环境、SDK多语言生成
- 运营：用量统计Dashboard、计费系统、开发者社区/工单系统
- 可用性：API网关多活部署、灰度发布新版本、降级策略

**D2：线上发现某个API的P99延迟突然从50ms涨到2s，且只影响部分用户，如何排查？**

**参考思路**：
- 分析维度：按用户/地域/设备/API版本切分看是否有规律 → 分布式追踪看慢在哪一跳
- "部分用户"线索：某个分片/某个DB从库慢、某个下游服务特定实例异常、CDN某个节点故障
- 排查路径：Trace → 找到慢Span → 是DB慢（慢查询日志）还是下游RPC慢（超时/重试叠加）还是GC停顿
- 快速止损：如果是特定实例问题则摘除、如果是DB则切主、如果是缓存失效则预热

### 开放式设计题

**D1：设计一个面向第三方开发者的开放API平台（类似微信开放平台），需要考虑什么？**

**参考思路**：
- 接入管理：应用注册/审核、AppID+AppSecret分发、OAuth2.0授权（授权码模式+PKCE）
- API治理：版本管理（URL路径版本）、限流（按AppID多维度限流）、配额（免费/基础/企业分级）
- 安全：请求签名验证（HMAC-SHA256+时间戳+Nonce防重放）、IP白名单、敏感数据加密传输
- 文档与体验：OpenAPI自动生成文档、Sandbox环境、SDK多语言生成
- 运营：用量统计Dashboard、计费系统、开发者社区/工单系统
- 可用性：API网关多活部署、灰度发布新版本、降级策略

**D2：线上发现某个API的P99延迟突然从50ms涨到2s，且只影响部分用户，如何排查？**

**参考思路**：
- 分析维度：按用户/地域/设备/API版本切分看是否有规律 → 分布式追踪看慢在哪一跳
- "部分用户"线索：某个分片/某个DB从库慢、某个下游服务特定实例异常、CDN某个节点故障
- 排查路径：Trace → 找到慢Span → 是DB慢（慢查询日志）还是下游RPC慢（超时/重试叠加）还是GC停顿
- 快速止损：如果是特定实例问题则摘除、如果是DB则切主、如果是缓存失效则预热
