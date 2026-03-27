# Nginx与Web服务器

---

## 📑 目录

### 基础与代理
1. [Nginx基础](#1-nginx基础)
2. [核心配置](#2-核心配置)
3. [反向代理](#3-反向代理)

### 负载与安全
4. [负载均衡](#4-负载均衡)
5. [静态资源服务](#5-静态资源服务)
6. [HTTPS配置](#6-https配置)

### 优化与自查
7. [性能优化](#7-性能优化)
8. [高级配置](#8-高级配置)
9. [面试题自查](#9-面试题自查)

---

## 1. Nginx基础

### 1.1 什么是Nginx？

**Nginx**：高性能HTTP服务器、反向代理服务器

**特点**：
- ✅ 高并发（Event-driven，支持百万级并发）
- ✅ 低内存消耗
- ✅ 反向代理、负载均衡
- ✅ 静态文件服务

**应用场景**：
- 静态资源服务器
- 反向代理（隐藏后端服务器）
- 负载均衡（分发请求）
- HTTP缓存

### 1.2 安装与启动

```bash
# Ubuntu/Debian
sudo apt-get install nginx

# CentOS/RHEL
sudo yum install nginx

# 启动
sudo systemctl start nginx

# 停止
sudo systemctl stop nginx

# 重启
sudo systemctl restart nginx

# 重载配置（不中断服务）
sudo nginx -s reload

# 测试配置
sudo nginx -t
```

---

## 2. 核心配置

### 2.1 配置文件结构

```nginx
# /etc/nginx/nginx.conf

# 全局块
user nginx;
worker_processes auto;  # 自动设置为CPU核心数
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# events块
events {
    worker_connections 1024;  # 每个worker进程最大连接数
    use epoll;  # Linux高效I/O模型
}

# http块
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    # server块
    server {
        listen 80;
        server_name example.com;
        
        # location块
        location / {
            root /var/www/html;
            index index.html;
        }
    }
}
```

### 2.2 虚拟主机

**基于域名**：
```nginx
# 网站1
server {
    listen 80;
    server_name www.example1.com;
    root /var/www/site1;
}

# 网站2
server {
    listen 80;
    server_name www.example2.com;
    root /var/www/site2;
}
```

**基于端口**：
```nginx
server {
    listen 8080;
    server_name localhost;
    root /var/www/site1;
}

server {
    listen 8081;
    server_name localhost;
    root /var/www/site2;
}
```

---

## 3. 反向代理

### 3.1 基本反向代理

```nginx
server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://127.0.0.1:3000;  # 转发到后端服务器
        
        # 传递真实客户端IP
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### 3.2 路径转发

```nginx
server {
    listen 80;
    server_name example.com;
    
    # /api/* → http://api-server:8080/*
    location /api/ {
        proxy_pass http://api-server:8080/;
    }
    
    # /admin/* → http://admin-server:9000/*
    location /admin/ {
        proxy_pass http://admin-server:9000/;
    }
    
    # 其他请求 → 静态文件
    location / {
        root /var/www/html;
        index index.html;
    }
}
```

---

## 4. 负载均衡

### 4.1 轮询（默认）

```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
    }
}
```

### 4.2 权重

```nginx
upstream backend {
    server 192.168.1.101:8080 weight=3;  # 权重3
    server 192.168.1.102:8080 weight=2;  # 权重2
    server 192.168.1.103:8080 weight=1;  # 权重1
}
```

### 4.3 IP Hash（同一IP固定到同一服务器）

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

### 4.4 最少连接

```nginx
upstream backend {
    least_conn;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

### 4.5 健康检查

```nginx
upstream backend {
    server 192.168.1.101:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.102:8080 max_fails=3 fail_timeout=30s;
    # max_fails: 失败3次标记为down
    # fail_timeout: 30秒后重新尝试
}
```

---

## 5. 静态资源服务

### 5.1 基本配置

```nginx
server {
    listen 80;
    server_name static.example.com;
    
    location / {
        root /var/www/static;
        index index.html;
        
        # 开启gzip压缩
        gzip on;
        gzip_types text/plain text/css application/json application/javascript;
        
        # 缓存控制
        expires 7d;
        add_header Cache-Control "public, immutable";
    }
}
```

### 5.2 目录浏览

```nginx
location /downloads/ {
    alias /var/www/downloads/;
    autoindex on;  # 开启目录浏览
    autoindex_exact_size off;  # 显示文件大小（KB、MB）
    autoindex_localtime on;  # 显示本地时间
}
```

### 5.3 防盗链

```nginx
location ~* \.(jpg|jpeg|png|gif)$ {
    valid_referers none blocked example.com *.example.com;
    if ($invalid_referer) {
        return 403;
    }
}
```

---

## 6. HTTPS配置

### 6.1 SSL证书配置

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    
    # 证书文件
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    
    # SSL协议
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # SSL Session缓存
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    location / {
        root /var/www/html;
        index index.html;
    }
}

# HTTP重定向到HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

### 6.2 Let's Encrypt免费证书

```bash
# 安装certbot
sudo apt-get install certbot python3-certbot-nginx

# 自动配置
sudo certbot --nginx -d example.com -d www.example.com

# 自动续期
sudo certbot renew --dry-run
```

---

## 7. 性能优化

### 7.1 Gzip压缩

```nginx
http {
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;  # 小于1KB不压缩
    gzip_comp_level 6;  # 压缩级别1-9
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
}
```

### 7.2 HTTP缓存

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}

location ~* \.(html)$ {
    expires 1h;
    add_header Cache-Control "public, must-revalidate";
}
```

### 7.3 Sendfile

```nginx
http {
    sendfile on;  # 开启高效文件传输
    tcp_nopush on;  # 数据包累积到一定大小再发送
    tcp_nodelay on;  # 禁用Nagle算法（实时性优先）
}
```

### 7.4 连接优化

```nginx
http {
    keepalive_timeout 65;  # 保持连接65秒
    keepalive_requests 100;  # 单个连接最多100个请求
}

upstream backend {
    server 192.168.1.101:8080;
    keepalive 32;  # 保持32个到后端的连接
}
```

### 7.5 限流

```nginx
# 限制请求速率
http {
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
}

server {
    location /api/ {
        limit_req zone=mylimit burst=20 nodelay;
        # rate=10r/s: 每秒10个请求
        # burst=20: 突发最多20个请求
        # nodelay: 立即处理（不排队）
    }
}

# 限制连接数
http {
    limit_conn_zone $binary_remote_addr zone=addr:10m;
}

server {
    location / {
        limit_conn addr 10;  # 单IP最多10个并发连接
    }
}
```

---

## 8. 高级配置

### 8.1 URL重写

```nginx
server {
    listen 80;
    server_name example.com;
    
    # 重写规则
    # rewrite regex replacement [flag];
    # flag: last(继续匹配), break(停止), redirect(302), permanent(301)
    
    # 将旧URL重定向到新URL
    rewrite ^/old-page$ /new-page permanent;
    
    # 将/article/123重写为/article.php?id=123
    rewrite ^/article/(\d+)$ /article.php?id=$1 last;
    
    # 强制HTTPS
    if ($scheme = http) {
        rewrite ^(.*)$ https://$host$1 permanent;
    }
    
    # 移除.html后缀
    rewrite ^/(.+)\.html$ /$1 permanent;
    
    # 添加尾部斜杠
    rewrite ^([^.]*[^/])$ $1/ permanent;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### 8.2 WebSocket代理

```nginx
# WebSocket需要特殊的header处理
upstream websocket_backend {
    server 127.0.0.1:8080;
}

server {
    listen 80;
    server_name ws.example.com;
    
    location /ws/ {
        proxy_pass http://websocket_backend;
        
        # 必须的WebSocket头
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # 其他头
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # 超时设置（WebSocket连接可能持续很长时间）
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

### 8.3 日志配置与分析

```nginx
http {
    # 自定义日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    '$request_time $upstream_response_time';
    
    # JSON格式日志（便于ELK分析）
    log_format json_log escape=json
        '{"time": "$time_iso8601", '
        '"remote_addr": "$remote_addr", '
        '"request": "$request", '
        '"status": "$status", '
        '"body_bytes_sent": "$body_bytes_sent", '
        '"request_time": "$request_time", '
        '"upstream_response_time": "$upstream_response_time", '
        '"http_referer": "$http_referer", '
        '"http_user_agent": "$http_user_agent"}';
    
    access_log /var/log/nginx/access.log json_log;
    
    # 按条件记录日志
    map $status $loggable {
        ~^[23]  0;  # 2xx和3xx不记录
        default 1;
    }
    
    access_log /var/log/nginx/error_only.log main if=$loggable;
}
```

### 8.4 动静分离

```nginx
server {
    listen 80;
    server_name example.com;
    
    # 静态资源直接返回
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf)$ {
        root /var/www/static;
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;  # 静态资源不记录日志
    }
    
    # 动态请求转发到后端
    location / {
        proxy_pass http://app_server;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # API请求
    location /api/ {
        proxy_pass http://api_server;
        proxy_set_header Host $host;
    }
}
```

---

## 9. 面试题自查

### Q1: Nginx为什么高性能？

**答案**：
```
1. Event-driven（事件驱动）
   - 使用epoll（Linux）/kqueue（BSD）实现高效I/O多路复用
   - 异步非阻塞，单进程可处理数千个并发连接
   
2. Master-Worker架构
   - Master进程：管理Worker进程，读取配置，绑定端口
   - Worker进程：处理实际请求，数量通常等于CPU核心数
   - 各Worker进程独立，一个崩溃不影响其他

3. 内存池机制
   - 预分配内存池，减少malloc/free开销
   - 请求处理完毕统一释放，减少内存碎片

4. Sendfile零拷贝
   - 传统：磁盘→内核缓冲→用户缓冲→Socket缓冲→网络
   - Sendfile：磁盘→内核缓冲→网络（少两次拷贝）

5. 高效的数据结构
   - 红黑树管理定时器
   - 哈希表存储配置
   - 链表管理连接
```

### Q2: Nginx与Apache的区别？

**答案**：
```
| 特性 | Nginx | Apache |
|------|-------|--------|
| 并发模型 | 异步非阻塞（事件驱动） | 多进程/多线程（阻塞） |
| 并发能力 | 高（C10K问题解决者） | 较低（并发高时性能下降） |
| 内存消耗 | 低（静态内存消耗） | 高（每个连接一个进程/线程） |
| 静态资源 | 极快 | 一般 |
| 动态内容 | 需要FastCGI/uwsgi | 原生支持（mod_php） |
| 配置语法 | 简洁 | 复杂（.htaccess） |
| 模块加载 | 编译时加载 | 运行时动态加载 |
| 使用场景 | 高并发、反向代理 | 动态网站、传统PHP |
```

### Q3: 负载均衡算法有哪些？各有什么特点？

**答案**：
```nginx
# 1. 轮询（Round Robin）- 默认
upstream backend {
    server 192.168.1.101;
    server 192.168.1.102;
}
# 特点：简单公平，不考虑服务器性能差异

# 2. 加权轮询（Weighted Round Robin）
upstream backend {
    server 192.168.1.101 weight=3;
    server 192.168.1.102 weight=1;
}
# 特点：根据服务器性能分配权重

# 3. IP Hash
upstream backend {
    ip_hash;
    server 192.168.1.101;
    server 192.168.1.102;
}
# 特点：同一客户端IP固定到同一服务器，解决Session一致性

# 4. 最少连接（Least Connections）
upstream backend {
    least_conn;
    server 192.168.1.101;
    server 192.168.1.102;
}
# 特点：优先选择当前连接数最少的服务器

# 5. URL Hash（需要第三方模块）
upstream backend {
    hash $request_uri;
    server 192.168.1.101;
    server 192.168.1.102;
}
# 特点：相同URL固定到同一服务器，提高缓存命中率
```

### Q4: Nginx如何处理跨域？

**答案**：
```nginx
server {
    listen 80;
    server_name api.example.com;
    
    location /api/ {
        # 允许的源
        add_header Access-Control-Allow-Origin $http_origin;
        # 或指定域名：add_header Access-Control-Allow-Origin https://example.com;
        
        # 允许的方法
        add_header Access-Control-Allow-Methods 'GET, POST, PUT, DELETE, OPTIONS';
        
        # 允许的请求头
        add_header Access-Control-Allow-Headers 'Content-Type, Authorization, X-Requested-With';
        
        # 允许携带Cookie
        add_header Access-Control-Allow-Credentials true;
        
        # 预检请求缓存时间
        add_header Access-Control-Max-Age 86400;
        
        # 处理OPTIONS预检请求
        if ($request_method = 'OPTIONS') {
            return 204;
        }
        
        proxy_pass http://backend;
    }
}
```

### Q5: location匹配规则是什么？

**答案**：
```nginx
# 匹配优先级（从高到低）
# 1. 精确匹配 =
location = /api {
    # 只匹配/api
}

# 2. 前缀匹配（优先） ^~
location ^~ /static/ {
    # 匹配/static/开头，不再检查正则
}

# 3. 正则匹配（区分大小写） ~
location ~ \.php$ {
    # 匹配.php结尾
}

# 4. 正则匹配（不区分大小写） ~*
location ~* \.(jpg|png|gif)$ {
    # 匹配图片
}

# 5. 普通前缀匹配
location /api/ {
    # 匹配/api/开头
}

# 6. 通用匹配
location / {
    # 匹配所有
}

# 匹配流程：
# 1. 先检查精确匹配（=）
# 2. 检查前缀匹配，记录最长匹配
# 3. 如果最长匹配有^~，使用它
# 4. 按顺序检查正则匹配，使用第一个匹配的
# 5. 使用最长的前缀匹配
```

### Q6: Nginx如何实现高可用？

**答案**：
```bash
# 方案1：Keepalived + VIP（虚拟IP）
# 主备模式：主节点故障时，备节点接管VIP

# keepalived.conf（主节点）
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    virtual_ipaddress {
        192.168.1.100  # VIP
    }
}

# keepalived.conf（备节点）
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90
    virtual_ipaddress {
        192.168.1.100  # 相同VIP
    }
}

# 方案2：LVS + Nginx集群
# LVS在第四层实现负载均衡，后端是多个Nginx

# 方案3：DNS轮询
# 多个IP对应同一域名，DNS层面分流
```

### Q7: Nginx限流的实现原理？

**答案**：
```nginx
# 1. 令牌桶算法（limit_req）
# 以固定速率产生令牌，请求消耗令牌

http {
    # 定义限流区域
    # zone=mylimit:10m - 10MB内存存储状态
    # rate=10r/s - 每秒10个请求
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
}

server {
    location /api/ {
        # burst=20 - 允许突发20个请求
        # nodelay - 不延迟处理突发请求
        limit_req zone=mylimit burst=20 nodelay;
        
        # 超过限制返回503
        limit_req_status 503;
    }
}

# 2. 漏桶算法（limit_conn）
# 控制并发连接数

http {
    limit_conn_zone $binary_remote_addr zone=addr:10m;
}

server {
    location / {
        limit_conn addr 10;  # 每IP最多10个并发连接
    }
}

# 3. 按地理位置/IP限流
geo $limit {
    default 1;
    10.0.0.0/8 0;  # 内网不限制
}

map $limit $limit_key {
    0 "";
    1 $binary_remote_addr;
}

limit_req_zone $limit_key zone=mylimit:10m rate=5r/s;
```

### Q8: proxy_pass末尾有无斜杠的区别？

**答案**：
```nginx
# 假设请求URL：http://example.com/api/users

# 1. 无斜杠：保留location匹配的路径
location /api/ {
    proxy_pass http://backend;
}
# 转发到：http://backend/api/users

# 2. 有斜杠：替换location匹配的路径
location /api/ {
    proxy_pass http://backend/;
}
# 转发到：http://backend/users

# 3. 带路径无斜杠
location /api/ {
    proxy_pass http://backend/v1;
}
# 转发到：http://backend/v1users（注意没有斜杠）

# 4. 带路径有斜杠
location /api/ {
    proxy_pass http://backend/v1/;
}
# 转发到：http://backend/v1/users

# 总结规则：
# - proxy_pass没有URI部分（无斜杠）：请求URI原样传递
# - proxy_pass有URI部分（有斜杠或路径）：替换location匹配部分
```

### Q9: Nginx如何配置HTTP/2？

**答案**：
```nginx
server {
    # HTTP/2必须配合SSL使用
    listen 443 ssl http2;
    server_name example.com;
    
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    
    # SSL优化
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    
    # HTTP/2服务器推送
    location / {
        root /var/www/html;
        
        # 推送CSS和JS
        http2_push /css/style.css;
        http2_push /js/main.js;
        
        # 或使用preload
        add_header Link "</css/style.css>; rel=preload; as=style";
    }
}

# HTTP/2特性：
# 1. 多路复用：单连接并行多请求
# 2. 头部压缩：HPACK算法
# 3. 服务器推送：主动推送资源
# 4. 二进制分帧：更高效的解析
```

### Q10: Nginx如何实现A/B测试？

**答案**：
```nginx
# 方法1：基于Cookie的A/B测试
upstream backend_a {
    server 192.168.1.101;
}

upstream backend_b {
    server 192.168.1.102;
}

# 根据Cookie分流
map $cookie_ab_test $backend {
    default backend_a;
    "B" backend_b;
}

server {
    location / {
        proxy_pass http://$backend;
    }
}

# 方法2：基于权重的随机分流
split_clients "${remote_addr}${uri}" $variant {
    50% "A";
    50% "B";
}

server {
    location / {
        if ($variant = "A") {
            proxy_pass http://backend_a;
            break;
        }
        proxy_pass http://backend_b;
    }
}

# 方法3：使用sticky模块保持一致性
upstream backend {
    sticky cookie srv_id expires=1h path=/;
    server 192.168.1.101;
    server 192.168.1.102;
}
```

### Q11: root与alias的区别？

**答案**：
```nginx
# root: location的URI会追加到root路径后
location /images/ {
    root /var/www;
}
# 请求 /images/logo.png → /var/www/images/logo.png

# alias: location的URI会被alias路径替换
location /images/ {
    alias /var/www/static/;
}
# 请求 /images/logo.png → /var/www/static/logo.png

# 注意事项：
# 1. alias必须以/结尾（如果location以/结尾）
# 2. alias可以在任意location中使用
# 3. root是追加，alias是替换
# 4. 正则匹配location中使用alias更灵活

location ~ ^/images/(.+)$ {
    alias /var/www/static/$1;
}
# 请求 /images/logo.png → /var/www/static/logo.png
```

### Q12: Nginx如何防止DDoS攻击？

**答案**：
```nginx
http {
    # 1. 限制请求速率
    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;
    
    # 2. 限制连接数
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
    
    # 3. 限制请求体大小
    client_max_body_size 10m;
    
    # 4. 设置超时
    client_body_timeout 10s;
    client_header_timeout 10s;
    keepalive_timeout 5s 5s;
    send_timeout 10s;
    
    # 5. 限制缓冲区大小
    client_body_buffer_size 1k;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    server {
        listen 80;
        
        location / {
            limit_req zone=req_limit burst=20 nodelay;
            limit_conn conn_limit 10;
            
            # 6. 拒绝特定User-Agent
            if ($http_user_agent ~* "BadBot|Crawler") {
                return 403;
            }
            
            # 7. 拒绝特定请求方法
            if ($request_method !~ ^(GET|POST|HEAD)$) {
                return 444;  # 直接关闭连接
            }
            
            proxy_pass http://backend;
        }
    }
}

# 8. 使用fail2ban自动封禁
# 9. 使用第三方WAF（如ModSecurity）
# 10. 启用SYN Cookie防止SYN Flood
```

### Q13: Nginx如何配置灰度发布？

**答案**：
```nginx
upstream stable {
    server 192.168.1.101;  # 稳定版
}

upstream canary {
    server 192.168.1.102;  # 金丝雀版
}

# 方法1：按比例灰度
split_clients "${remote_addr}" $backend {
    10% "canary";   # 10%流量到金丝雀版
    90% "stable";   # 90%流量到稳定版
}

# 方法2：按用户ID灰度
map $cookie_user_id $backend {
    default "stable";
    ~^[0-9]$ "canary";  # user_id以0-9开头的用户
}

# 方法3：按请求头灰度
map $http_x_canary $backend {
    default "stable";
    "true" "canary";
}

server {
    location / {
        proxy_pass http://$backend;
    }
}

# 方法4：使用Lua脚本实现复杂逻辑
location / {
    access_by_lua_block {
        local user_id = ngx.var.cookie_user_id
        if user_id and tonumber(user_id) % 10 < 1 then
            ngx.var.backend = "canary"
        else
            ngx.var.backend = "stable"
        end
    }
    proxy_pass http://$backend;
}
```

### Q14: Nginx正向代理与反向代理的区别？

**答案**：
```
正向代理：
- 代理客户端，帮助客户端访问目标服务器
- 客户端知道代理存在，需要配置代理
- 用途：翻墙、缓存、访问控制

反向代理：
- 代理服务器，帮助服务器接收客户端请求
- 客户端不知道代理存在
- 用途：负载均衡、安全防护、SSL卸载

配置示例：

# 反向代理（常用）
server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://backend;
    }
}

# 正向代理（较少用）
server {
    listen 8888;
    resolver 8.8.8.8;  # DNS服务器
    
    location / {
        proxy_pass http://$host$request_uri;
    }
}
```

### Q15: try_files的作用是什么？

**答案**：
```nginx
# try_files按顺序检查文件是否存在

# 基本用法：检查文件→检查目录→返回fallback
location / {
    try_files $uri $uri/ /index.html;
}
# 1. 尝试$uri（如/about）
# 2. 尝试$uri/（如/about/）
# 3. 都不存在则返回/index.html

# SPA应用配置
location / {
    root /var/www/app;
    try_files $uri $uri/ /index.html;
}

# 检查多个文件
location / {
    try_files $uri $uri.html $uri.php /fallback.html;
}

# 结合@命名location
location / {
    try_files $uri $uri/ @backend;
}

location @backend {
    proxy_pass http://app_server;
}

# 文件存在才代理
location / {
    try_files $uri @proxy;
}

location @proxy {
    proxy_pass http://backend;
}

# 返回状态码
location / {
    try_files $uri $uri/ =404;
}

# 常见场景：
# 1. SPA路由（Vue/React）
# 2. 静态文件优先，不存在则转发后端
# 3. 实现伪静态URL
```

### Q16: Nginx upstream中的server参数有哪些？

**答案**：
```nginx
upstream backend {
    # weight: 权重，默认1
    server 192.168.1.101 weight=5;
    
    # max_fails: 失败次数阈值，默认1
    # fail_timeout: 失败后暂停时间，默认10s
    server 192.168.1.102 max_fails=3 fail_timeout=30s;
    
    # backup: 备用服务器，只有主服务器都down时才启用
    server 192.168.1.103 backup;
    
    # down: 标记为永久不可用
    server 192.168.1.104 down;
    
    # max_conns: 最大并发连接数（商业版）
    server 192.168.1.105 max_conns=100;
    
    # slow_start: 慢启动，从0逐渐增加到正常权重（商业版）
    server 192.168.1.106 slow_start=30s;
    
    # resolve: 动态解析域名（商业版）
    server backend.example.com resolve;
    
    # keepalive: 保持到upstream的长连接数
    keepalive 32;
    
    # keepalive_requests: 单连接最大请求数
    keepalive_requests 100;
    
    # keepalive_timeout: 长连接超时
    keepalive_timeout 60s;
}

server {
    location / {
        proxy_pass http://backend;
        
        # 长连接需要设置HTTP/1.1
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

---

## 总结

Nginx核心：
1. **高性能原理**：Event-driven、Sendfile、Master-Worker
2. **反向代理**：隐藏后端、统一入口、proxy_pass
3. **负载均衡**：轮询、权重、IP Hash、最少连接
4. **静态资源**：Gzip、缓存、Sendfile、防盗链
5. **HTTPS**：SSL证书、HTTP/2、Let's Encrypt
6. **性能优化**：限流（令牌桶/漏桶）、缓存、连接池
7. **高级配置**：URL重写、WebSocket代理、灰度发布、A/B测试
8. **安全防护**：DDoS防护、CORS、访问控制


---
