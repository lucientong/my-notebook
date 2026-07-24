Language: English | [中文](../前端知识库/04-Nginx与Web服务器.md)

# Nginx and Web Servers

---

## Table of Contents

1. [Nginx Fundamentals](#1-nginx-fundamentals)
2. [Core Configuration](#2-core-configuration)
3. [Reverse Proxy](#3-reverse-proxy)
4. [Load Balancing](#4-load-balancing)
5. [Static Asset Serving](#5-static-asset-serving)
6. [HTTPS Configuration](#6-https-configuration)
7. [Performance Optimization](#7-performance-optimization)
8. [Advanced Configuration](#8-advanced-configuration)
9. [Interview Self-check](#9-interview-self-check)
10. [Production Scenarios](#10-production-scenarios)

---

## 1. Nginx Fundamentals

Nginx is an event-driven web server, reverse proxy, and load balancer. It uses a
master-worker architecture and non-blocking I/O, which makes it efficient for
large numbers of concurrent connections.

Why it is fast:

- Event-driven worker processes.
- Non-blocking network I/O.
- Efficient static file serving.
- Low memory overhead per connection.
- Mature reverse proxy and caching capabilities.

## 2. Core Configuration

Nginx configuration is hierarchical:

```nginx
worker_processes auto;

events {
  worker_connections 1024;
}

http {
  include mime.types;

  server {
    listen 80;
    server_name example.com;

    location / {
      root /var/www/html;
      index index.html;
    }
  }
}
```

Important contexts:

- Global context: process-level settings.
- `events`: connection handling.
- `http`: HTTP server configuration.
- `server`: virtual host.
- `location`: URI matching and request handling.

## 3. Reverse Proxy

Nginx often sits in front of application servers.

```nginx
location /api/ {
  proxy_pass http://backend/;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
}
```

### `proxy_pass` Slash Rule

```nginx
location /api/ {
  proxy_pass http://backend/;
}
```

Request `/api/users` becomes `http://backend/users`.

```nginx
location /api/ {
  proxy_pass http://backend;
}
```

Request `/api/users` becomes `http://backend/api/users`.

This is a common production misconfiguration and a frequent interview question.

## 4. Load Balancing

Common algorithms:

- Round robin: simple and default.
- Weighted round robin: capacity-aware distribution.
- IP hash: sticky routing by client IP.
- Least connections: useful when request duration varies.
- Consistent hashing: useful for cache locality.

```nginx
upstream app_servers {
  least_conn;
  server 10.0.0.1:3000 weight=2 max_fails=3 fail_timeout=30s;
  server 10.0.0.2:3000;
}

server {
  location / {
    proxy_pass http://app_servers;
  }
}
```

Production concerns:

- Health checks.
- Connection timeouts.
- Retry policy.
- Sticky-session requirements.
- Observability per upstream.

## 5. Static Asset Serving

For frontend deployments, Nginx commonly serves immutable hashed assets and
routes SPA fallback to `index.html`.

```nginx
location /assets/ {
  root /var/www/app;
  add_header Cache-Control "public, max-age=31536000, immutable";
}

location / {
  root /var/www/app;
  try_files $uri $uri/ /index.html;
}
```

Use `root` when the full URI should be appended to the filesystem path. Use
`alias` when the location prefix should be replaced.

## 6. HTTPS Configuration

HTTPS provides confidentiality, integrity, and server authentication.

```nginx
server {
  listen 443 ssl http2;
  server_name example.com;

  ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  add_header Strict-Transport-Security "max-age=31536000" always;
}

server {
  listen 80;
  server_name example.com;
  return 301 https://$host$request_uri;
}
```

Production practice:

- Automate certificate renewal.
- Enable HSTS only after HTTPS is stable.
- Keep TLS versions and ciphers modern.
- Monitor certificate expiry.

## 7. Performance Optimization

### Compression

```nginx
gzip on;
gzip_types text/plain text/css application/javascript application/json image/svg+xml;
gzip_min_length 1024;
```

Brotli can provide better compression than gzip, but requires module support and
CPU budget consideration.

### Caching

- HTML: no-cache or short cache, because it points to asset versions.
- Hashed JS/CSS: long cache with `immutable`.
- API responses: cache only when semantics are clear.

### Connection and File I/O

- `sendfile on` for efficient static file transfer.
- Tune keep-alive carefully.
- Avoid excessive worker connections without checking OS limits.

## 8. Advanced Configuration

### WebSocket Proxy

```nginx
location /ws/ {
  proxy_pass http://websocket_backend;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
}
```

### Rate Limiting

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api/ {
  limit_req zone=api burst=20 nodelay;
  proxy_pass http://backend;
}
```

### Gray Release

Gray release can be implemented by cookie, header, user ID hash, or weighted
traffic splitting. The important production issue is version consistency between
HTML and static assets.

### Troubleshooting 499, 502, and 504

- `499`: client closed the request before Nginx returned a response.
- `502`: upstream returned an invalid response or connection failed.
- `504`: upstream did not respond before timeout.

Check Nginx access/error logs, upstream logs, network connectivity, timeout
settings, deployment version, and resource saturation.

## 9. Interview Self-check

1. Why is Nginx high performance?
2. How does Nginx differ from Apache?
3. What load-balancing algorithms does Nginx support?
4. How do `root` and `alias` differ?
5. How does `location` matching priority work?
6. What is the difference between `return` and `rewrite`?
7. How do you configure SPA fallback?
8. How do you preserve the real client IP through proxy chains?
9. How would you diagnose 499, 502, and 504?
10. How do you implement safe gray release for frontend assets?
11. Why should `if` be used carefully in Nginx configuration?
12. How would you protect an API from traffic spikes?
13. How does `proxy_pass` URI replacement change with and without a trailing slash?
14. When would you use `return` instead of `rewrite`?
15. How do you configure cache headers for HTML versus hashed assets?
16. What upstream parameters matter for health, failover, and slow backends?
17. How do HTTP/2 and HTTP/3 change frontend asset delivery assumptions?
18. How do you roll out HSTS without locking users into a broken HTTPS setup?
19. How would you validate that real client IPs remain trustworthy behind multiple proxies?
20. What observability would you require before changing timeout or retry policy?

## 10. Production Scenarios

### Scenario 1: SPA Refresh Returns 404

Check `try_files` and the static root path. The usual frontend fallback is
`try_files $uri $uri/ /index.html`, while API paths must be excluded so backend
404s are not masked as a valid SPA page.

### Scenario 2: Gray Release Loads Mixed Asset Versions

Keep old hashed assets available, cache HTML with `no-cache`, avoid deleting
previous release artifacts immediately, and route users consistently by cookie or
header so HTML, chunks, and feature flags come from compatible versions.

### Scenario 3: Upstream 502 After Deployment

Inspect error logs, upstream health, connection refusal, DNS changes, container
readiness, TLS or protocol mismatch, and `proxy_read_timeout`. Correlate failures
with deployment time and upstream instance identity before changing global
timeouts.

### Scenario 4: API Traffic Spike

Use layered protection: CDN or WAF for edge filtering, `limit_req` for request
rate, `limit_conn` for concurrency, upstream circuit breaking where available,
and clear dashboards for rejection rate, upstream latency, and saturation.

## Summary

For frontend engineers, Nginx knowledge is most valuable at deployment and
incident boundaries: static asset caching, reverse proxy behavior, HTTPS,
fallback routing, rate limiting, gray release, and status-code diagnosis.
