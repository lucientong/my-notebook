# Nginx与Web服务器

---

## 📑 目录

1. [Nginx基础](#1-nginx基础)
2. [核心配置](#2-核心配置)
3. [反向代理](#3-反向代理)
4. [负载均衡](#4-负载均衡)
5. [静态资源服务](#5-静态资源服务)
6. [HTTPS配置](#6-https配置)
7. [性能优化](#7-性能优化)
8. [常见面试题](#8-常见面试题)

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

## 8. 常见面试题

### Q1: Nginx为什么高性能？

**答案**：
1. **Event-driven**：epoll/kqueue异步非阻塞
2. **Master-Worker**：多进程模型，充分利用多核
3. **Sendfile**：零拷贝，减少用户态/内核态切换
4. **内存池**：减少内存分配开销

### Q2: Nginx与Apache的区别？

**答案**：
| 特性 | Nginx | Apache |
|------|-------|--------|
| **并发模型** | 异步非阻塞 | 多进程/多线程 |
| **并发性能** | 高（百万级） | 低（万级） |
| **内存消耗** | 低 | 高 |
| **配置** | 简单 | 复杂 |
| **动态内容** | 需要FastCGI | 原生支持 |

### Q3: 负载均衡算法有哪些？

**答案**：
1. **轮询**：依次分配
2. **权重**：按权重分配
3. **IP Hash**：同一IP固定到同一服务器
4. **最少连接**：选择连接数最少的服务器

### Q4: 如何处理跨域？

**答案**：
```nginx
location /api/ {
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'Content-Type, Authorization';
    
    if ($request_method = 'OPTIONS') {
        return 204;
    }
    
    proxy_pass http://backend;
}
```

---

## 总结

Nginx核心：
1. **高性能**：Event-driven、Sendfile
2. **反向代理**：隐藏后端、统一入口
3. **负载均衡**：轮询、权重、IP Hash
4. **静态资源**：Gzip、缓存、Sendfile
5. **HTTPS**：SSL证书、HTTP/2
6. **性能优化**：限流、缓存、连接池

面试时展示**对Nginx核心原理的理解**和**实战优化经验**！🚀

---

**文件状态**：✅ 已完成（约500行）