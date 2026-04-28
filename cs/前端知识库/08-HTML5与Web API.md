# HTML5 与 Web API

---

## 📑 目录

### HTML5 基础
1. [HTML5 语义化](#html5-语义化) ⭐⭐
2. [Web Storage](#web-storage) ⭐⭐⭐

### Worker 与离线
3. [Web Worker](#web-worker) ⭐⭐⭐
4. [Service Worker 与 PWA](#service-worker-与-pwa) ⭐⭐

### 图形与媒体
5. [Canvas 与 WebGL](#canvas-与-webgl) ⭐⭐
6. [WebRTC](#webrtc) ⭐⭐

### 网络与观察者
7. [Fetch API 与 AbortController](#fetch-api-与-abortcontroller) ⭐⭐⭐
8. [Intersection Observer](#intersection-observer) ⭐⭐

### 动画与扩展
9. [Web Animations API](#web-animations-api)
10. [其他重要 API](#其他重要-api)

### 综合与自查
11. [实战案例](#实战案例)
12. [面试题自查](#面试题自查)

---

## HTML5 语义化

### 核心语义标签 ⭐⭐

**1. 语义化标签体系**

HTML5 引入了一系列语义化标签，使文档结构更加清晰，替代了传统的 `<div>` 嵌套。

```html
<!-- ❌ 传统 div 写法 -->
<div class="header">
  <div class="nav">...</div>
</div>
<div class="main">
  <div class="article">
    <div class="section">...</div>
  </div>
  <div class="sidebar">...</div>
</div>
<div class="footer">...</div>

<!-- ✅ 语义化写法 -->
<header>
  <nav aria-label="主导航">
    <ul>
      <li><a href="/">首页</a></li>
      <li><a href="/about">关于</a></li>
    </ul>
  </nav>
</header>

<main>
  <article>
    <header>
      <h1>文章标题</h1>
      <time datetime="2026-04-27">2026年4月27日</time>
    </header>
    <section>
      <h2>第一章节</h2>
      <p>内容...</p>
    </section>
    <section>
      <h2>第二章节</h2>
      <figure>
        <img src="chart.png" alt="数据统计图" />
        <figcaption>图1：2026年用户增长趋势</figcaption>
      </figure>
    </section>
    <footer>
      <p>作者：张三</p>
    </footer>
  </article>

  <aside aria-label="侧边栏">
    <section>
      <h3>相关推荐</h3>
      <ul>
        <li><a href="/post/2">推荐文章</a></li>
      </ul>
    </section>
  </aside>
</main>

<footer>
  <p>&copy; 2026 版权所有</p>
</footer>
```

**2. 各标签语义与使用规则**

| 标签 | 语义 | 可嵌套位置 | 注意事项 |
|------|------|-----------|---------|
| `<header>` | 头部区域 | `body`/`article`/`section` | 不可嵌套在 `<footer>`/`<address>` 内 |
| `<nav>` | 导航链接 | 任意流内容 | 主要导航使用，非所有链接组 |
| `<main>` | 主体内容 | `body` 直接子元素 | 每页仅一个，不含重复内容 |
| `<article>` | 独立内容 | 任意流内容 | 可独立分发/复用（博客、评论） |
| `<section>` | 主题分组 | 任意流内容 | 必须有标题（h1-h6） |
| `<aside>` | 辅助内容 | 任意流内容 | 侧边栏/广告/相关链接 |
| `<footer>` | 底部区域 | `body`/`article`/`section` | 版权/联系信息/相关链接 |
| `<figure>` | 独立引用 | 任意流内容 | 图片/代码/图表+figcaption |
| `<time>` | 时间标记 | 短语内容 | 必须有 datetime 属性 |
| `<mark>` | 高亮文本 | 短语内容 | 搜索结果高亮场景 |
| `<details>` | 折叠内容 | 任意流内容 | 配合 `<summary>` 使用 |

---

**3. SEO 影响**

语义化标签对搜索引擎优化有直接影响：

```javascript
// 用 JS 检测页面语义化程度的工具函数
function analyzeSemantics() {
  const semanticTags = [
    'header', 'nav', 'main', 'article', 'section',
    'aside', 'footer', 'figure', 'figcaption', 'time'
  ];

  const report = {};
  semanticTags.forEach(tag => {
    const elements = document.querySelectorAll(tag);
    report[tag] = elements.length;
  });

  // 检查 heading 层级是否合理
  const headings = document.querySelectorAll('h1, h2, h3, h4, h5, h6');
  let headingOrder = [];
  let hasSkip = false;

  headings.forEach(h => {
    const level = parseInt(h.tagName[1]);
    if (headingOrder.length > 0) {
      const prev = headingOrder[headingOrder.length - 1];
      if (level > prev + 1) hasSkip = true; // 跳级
    }
    headingOrder.push(level);
  });

  report.headingLevels = headingOrder;
  report.headingSkipped = hasSkip;
  report.h1Count = document.querySelectorAll('h1').length;

  // SEO 建议
  const suggestions = [];
  if (report.h1Count !== 1) suggestions.push('页面应只有一个 h1');
  if (hasSkip) suggestions.push('标题层级不应跳级（如 h1 直接到 h3）');
  if (report.main === 0) suggestions.push('缺少 <main> 标签');
  if (report.nav === 0) suggestions.push('缺少 <nav> 导航');

  return { report, suggestions };
}
```

SEO 关键要点：
- 搜索引擎根据语义标签理解页面结构，`<article>` 内容权重更高
- `<nav>` 帮助爬虫发现站点链接结构
- `<time datetime="">` 帮助搜索引擎理解时间信息，用于结构化数据
- `<h1>` 每页仅一个，与 `<title>` 对应

---

**4. 可访问性（a11y）**

语义化标签自动创建 ARIA 地标角色，屏幕阅读器依赖这些角色导航：

```html
<!-- 语义标签自动映射 ARIA 角色 -->
<header>   <!-- role="banner" -->
<nav>      <!-- role="navigation" -->
<main>     <!-- role="main" -->
<aside>    <!-- role="complementary" -->
<footer>   <!-- role="contentinfo" -->
<form>     <!-- role="form"（有 accessible name 时） -->
```

```javascript
// 可访问性增强示例
class A11yEnhancer {
  constructor(root = document) {
    this.root = root;
  }

  // 为图片添加缺失的 alt
  auditImages() {
    const images = this.root.querySelectorAll('img');
    const issues = [];

    images.forEach((img, i) => {
      if (!img.hasAttribute('alt')) {
        issues.push({
          element: img,
          issue: `第${i + 1}张图片缺少 alt 属性`,
          fix: '添加描述性 alt 或空 alt（装饰性图片）'
        });
      }
    });
    return issues;
  }

  // 检查 ARIA 使用是否正确
  auditAria() {
    const issues = [];

    // 检查 aria-label 与可见文本冲突
    const labeled = this.root.querySelectorAll('[aria-label]');
    labeled.forEach(el => {
      if (el.textContent.trim() && el.getAttribute('aria-label') !== el.textContent.trim()) {
        issues.push({
          element: el,
          issue: 'aria-label 与可见文本不一致',
        });
      }
    });

    // 检查按钮是否有可访问名称
    const buttons = this.root.querySelectorAll('button');
    buttons.forEach(btn => {
      if (!btn.textContent.trim() && !btn.getAttribute('aria-label')) {
        issues.push({
          element: btn,
          issue: '按钮缺少可访问名称（文本或 aria-label）',
        });
      }
    });

    return issues;
  }

  // 键盘导航增强 —— skip to content
  addSkipLink() {
    const main = this.root.querySelector('main');
    if (!main) return;

    if (!main.id) main.id = 'main-content';

    const skip = document.createElement('a');
    skip.href = `#${main.id}`;
    skip.className = 'skip-link';
    skip.textContent = '跳转到主要内容';
    skip.style.cssText = `
      position: absolute; top: -40px; left: 0;
      background: #000; color: #fff; padding: 8px 16px;
      z-index: 10000; transition: top 0.2s;
    `;
    skip.addEventListener('focus', () => { skip.style.top = '0'; });
    skip.addEventListener('blur', () => { skip.style.top = '-40px'; });

    document.body.prepend(skip);
  }
}

// 使用
const enhancer = new A11yEnhancer();
console.log(enhancer.auditImages());
console.log(enhancer.auditAria());
enhancer.addSkipLink();
```

**面试追问**：
- `<section>` 与 `<div>` 的区别是什么？何时用 `<div>`？
- 说出 5 个以上 ARIA 地标角色
- `<article>` 可以嵌套 `<article>` 吗？什么场景下？

---

## Web Storage

### 存储方案对比 ⭐⭐⭐

**1. 三种客户端存储方案**

| 特性 | localStorage | sessionStorage | IndexedDB |
|------|-------------|----------------|-----------|
| 容量 | 5-10 MB | 5-10 MB | 无硬性上限（通常数百 MB+） |
| 生命周期 | 永久（手动清除） | 页面会话结束 | 永久（手动清除） |
| 作用域 | 同源所有标签页 | 仅当前标签页 | 同源所有标签页 |
| API 类型 | 同步 | 同步 | 异步（基于事件） |
| 数据类型 | 仅字符串 | 仅字符串 | 几乎任何 JS 对象 |
| 索引查询 | ❌ | ❌ | ✅ 支持索引 |
| 事务支持 | ❌ | ❌ | ✅ |
| Web Worker 可用 | ❌ | ❌ | ✅ |
| 适用场景 | 用户偏好/token | 表单临时数据 | 大量结构化数据/离线 |

---

**2. localStorage / sessionStorage 深入**

```javascript
// ============ 基础使用 ============
localStorage.setItem('username', 'Alice');
localStorage.getItem('username');      // 'Alice'
localStorage.removeItem('username');
localStorage.clear();                  // 清除所有

// ============ 存储对象（必须序列化） ============
const user = { name: 'Alice', age: 30, roles: ['admin'] };
localStorage.setItem('user', JSON.stringify(user));
const stored = JSON.parse(localStorage.getItem('user'));

// ============ 带过期时间的封装 ============
class StorageWithExpiry {
  constructor(storage = localStorage) {
    this.storage = storage;
  }

  set(key, value, ttlMs) {
    const item = {
      value,
      expiry: ttlMs ? Date.now() + ttlMs : null,
    };
    this.storage.setItem(key, JSON.stringify(item));
  }

  get(key) {
    const raw = this.storage.getItem(key);
    if (!raw) return null;

    try {
      const item = JSON.parse(raw);
      if (item.expiry && Date.now() > item.expiry) {
        this.storage.removeItem(key);
        return null;
      }
      return item.value;
    } catch {
      return raw; // 兼容非本类写入的数据
    }
  }

  remove(key) {
    this.storage.removeItem(key);
  }
}

// 使用：缓存 1 小时
const cache = new StorageWithExpiry();
cache.set('api_data', { items: [1, 2, 3] }, 60 * 60 * 1000);
console.log(cache.get('api_data')); // { items: [1, 2, 3] }

// ============ 跨标签页通信 ============
// 标签页A：写入
localStorage.setItem('broadcast', JSON.stringify({
  type: 'USER_LOGOUT',
  timestamp: Date.now()
}));

// 标签页B：监听
window.addEventListener('storage', (event) => {
  if (event.key === 'broadcast') {
    const data = JSON.parse(event.newValue);
    if (data.type === 'USER_LOGOUT') {
      // 同步登出
      window.location.href = '/login';
    }
  }
});
```

**注意**：`storage` 事件只在**其他**标签页触发，不在写入的当前标签页触发。

---

**3. IndexedDB 深入**

```javascript
// ============ 基于 Promise 封装的 IndexedDB 工具 ============
class SimpleDB {
  constructor(dbName, version = 1) {
    this.dbName = dbName;
    this.version = version;
    this.db = null;
  }

  open(onUpgrade) {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, this.version);

      request.onupgradeneeded = (event) => {
        const db = event.target.result;
        if (onUpgrade) onUpgrade(db, event);
      };

      request.onsuccess = (event) => {
        this.db = event.target.result;
        resolve(this.db);
      };

      request.onerror = (event) => {
        reject(event.target.error);
      };
    });
  }

  _transaction(storeName, mode = 'readonly') {
    const tx = this.db.transaction(storeName, mode);
    return tx.objectStore(storeName);
  }

  _request(storeName, mode, callback) {
    return new Promise((resolve, reject) => {
      const store = this._transaction(storeName, mode);
      const request = callback(store);
      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  add(storeName, data) {
    return this._request(storeName, 'readwrite', store => store.add(data));
  }

  put(storeName, data) {
    return this._request(storeName, 'readwrite', store => store.put(data));
  }

  get(storeName, key) {
    return this._request(storeName, 'readonly', store => store.get(key));
  }

  delete(storeName, key) {
    return this._request(storeName, 'readwrite', store => store.delete(key));
  }

  getAll(storeName) {
    return this._request(storeName, 'readonly', store => store.getAll());
  }

  // 索引查询
  getByIndex(storeName, indexName, value) {
    return new Promise((resolve, reject) => {
      const store = this._transaction(storeName, 'readonly');
      const index = store.index(indexName);
      const request = index.getAll(value);
      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  // 游标遍历（适合大数据集）
  iterate(storeName, callback) {
    return new Promise((resolve, reject) => {
      const store = this._transaction(storeName, 'readonly');
      const request = store.openCursor();
      const results = [];

      request.onsuccess = (event) => {
        const cursor = event.target.result;
        if (cursor) {
          const shouldContinue = callback(cursor.value, cursor.key);
          if (shouldContinue !== false) {
            results.push(cursor.value);
            cursor.continue();
          } else {
            resolve(results);
          }
        } else {
          resolve(results);
        }
      };
      request.onerror = () => reject(request.error);
    });
  }
}

// ============ 使用示例：离线文章存储 ============
async function demo() {
  const db = new SimpleDB('MyApp', 1);

  await db.open((database) => {
    // 创建 object store（类似表）
    if (!database.objectStoreNames.contains('articles')) {
      const store = database.createObjectStore('articles', { keyPath: 'id' });
      store.createIndex('category', 'category', { unique: false });
      store.createIndex('date', 'publishDate', { unique: false });
    }
  });

  // 写入
  await db.put('articles', {
    id: 'a1',
    title: 'HTML5 语义化',
    category: 'frontend',
    publishDate: '2026-04-27',
    content: '...'
  });

  // 按索引查询
  const frontendArticles = await db.getByIndex('articles', 'category', 'frontend');
  console.log(frontendArticles);

  // 获取所有
  const all = await db.getAll('articles');
  console.log('所有文章:', all);
}
```

---

**4. 安全考量**

```javascript
// ============ 安全最佳实践 ============

// ❌ 绝对不要存储敏感数据
localStorage.setItem('password', '123456');      // 极度危险
localStorage.setItem('creditCard', '4111...');   // 极度危险

// ❌ JWT token 存 localStorage 有 XSS 风险
localStorage.setItem('token', 'eyJhbGciOiJI...'); // 可被 XSS 窃取

// ✅ 推荐：HttpOnly Cookie 存 token，localStorage 存非敏感偏好
document.cookie; // HttpOnly Cookie 无法被 JS 读取，更安全

// ✅ 如果必须用 localStorage 存 token，做好 XSS 防御
class SecureStorage {
  // 简单混淆（注意：这不是真正的加密，只是增加门槛）
  static encode(value) {
    return btoa(encodeURIComponent(value));
  }

  static decode(value) {
    return decodeURIComponent(atob(value));
  }

  static set(key, value) {
    localStorage.setItem(key, this.encode(JSON.stringify(value)));
  }

  static get(key) {
    const raw = localStorage.getItem(key);
    if (!raw) return null;
    try {
      return JSON.parse(this.decode(raw));
    } catch {
      return null;
    }
  }
}

// ============ 存储容量检测 ============
function getStorageUsage() {
  let total = 0;
  for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i);
    const value = localStorage.getItem(key);
    total += (key.length + value.length) * 2; // UTF-16 每字符 2 字节
  }
  return {
    usedBytes: total,
    usedKB: (total / 1024).toFixed(2),
    usedMB: (total / (1024 * 1024)).toFixed(2),
  };
}

// 测试可用容量
function testStorageLimit() {
  const testKey = '__storage_test__';
  let size = 0;
  try {
    // 每次写入 100KB
    const chunk = 'x'.repeat(100 * 1024);
    while (true) {
      localStorage.setItem(testKey + size, chunk);
      size += 100;
    }
  } catch (e) {
    // 清理
    for (let i = 0; i < size; i += 100) {
      localStorage.removeItem(testKey + i);
    }
    console.log(`可用容量约 ${size} KB`);
  }
}
```

**面试追问**：
- localStorage 何时会触发 QuotaExceededError？如何优雅处理？
- 如何在 Web Worker 中使用 IndexedDB？
- sessionStorage 在新标签页（`window.open` / `target="_blank"`）中的行为？

---

## Web Worker

### 多线程编程 ⭐⭐⭐

**1. Worker 类型对比**

| 特性 | Dedicated Worker | Shared Worker | Service Worker |
|------|-----------------|---------------|----------------|
| 作用域 | 单个页面 | 多个同源页面 | 同源所有页面 |
| 生命周期 | 随页面 | 最后连接断开时终止 | 独立于页面 |
| 通信方式 | postMessage | MessagePort | postMessage / FetchEvent |
| 使用场景 | 密集计算 | 跨标签页共享 | 离线缓存/推送 |
| DOM 访问 | ❌ | ❌ | ❌ |
| 可用 API | XHR/Fetch/IDB | XHR/Fetch/IDB | Fetch/IDB/CacheAPI |

---

**2. Dedicated Worker（专用 Worker）**

```javascript
// ============ worker.js ============
// Worker 线程中不能访问 DOM，但可以使用大部分 JS API
self.addEventListener('message', (event) => {
  const { type, data } = event.data;

  switch (type) {
    case 'HEAVY_CALC': {
      // 模拟耗时计算（如大数据排序）
      const result = heavySort(data);
      self.postMessage({ type: 'CALC_RESULT', data: result });
      break;
    }
    case 'FIBONACCI': {
      const result = fibonacci(data);
      self.postMessage({ type: 'FIB_RESULT', data: result });
      break;
    }
  }
});

function heavySort(arr) {
  // 对百万级数据排序
  return arr.sort((a, b) => a - b);
}

function fibonacci(n) {
  if (n <= 1) return n;
  let a = 0, b = 1;
  for (let i = 2; i <= n; i++) {
    [a, b] = [b, a + b];
  }
  return b;
}

// ============ main.js ============
const worker = new Worker('./worker.js');

// 监听结果
worker.addEventListener('message', (event) => {
  const { type, data } = event.data;
  switch (type) {
    case 'CALC_RESULT':
      console.log('排序完成，前10项:', data.slice(0, 10));
      break;
    case 'FIB_RESULT':
      console.log('斐波那契结果:', data);
      break;
  }
});

// 错误处理
worker.addEventListener('error', (event) => {
  console.error('Worker 错误:', event.message, event.filename, event.lineno);
});

// 发送任务
const bigArray = Array.from({ length: 1000000 }, () => Math.random() * 1000000);
worker.postMessage({ type: 'HEAVY_CALC', data: bigArray });
worker.postMessage({ type: 'FIBONACCI', data: 50 });

// 终止 Worker
// worker.terminate();
```

---

**3. Transferable Objects（零拷贝传输）**

```javascript
// ============ 大数据量传输优化 ============

// ❌ 普通 postMessage —— 结构化克隆，百兆数据可能卡顿
const hugeArray = new Float64Array(10_000_000);
worker.postMessage({ type: 'PROCESS', buffer: hugeArray });
// hugeArray 被拷贝，内存翻倍

// ✅ Transferable Objects —— 所有权转移，零拷贝
const hugeArray2 = new Float64Array(10_000_000);
worker.postMessage(
  { type: 'PROCESS', buffer: hugeArray2 },
  [hugeArray2.buffer]  // 转移 ArrayBuffer 的所有权
);
// 注意：transfer 后 hugeArray2.byteLength === 0（已不可用）

// Worker 端接收
self.addEventListener('message', (event) => {
  const { buffer } = event.data;
  // buffer 数据完整可用
  // 处理完后可以 transfer 回去
  const result = processData(buffer);
  self.postMessage({ result }, [result.buffer]);
});
```

---

**4. Shared Worker（共享 Worker）**

```javascript
// ============ shared-worker.js ============
const connections = new Set();

self.addEventListener('connect', (event) => {
  const port = event.ports[0];
  connections.add(port);

  port.addEventListener('message', (e) => {
    const { type, data } = e.data;

    switch (type) {
      case 'BROADCAST':
        // 广播给所有已连接的页面
        connections.forEach(p => {
          p.postMessage({ type: 'BROADCAST', data, from: 'shared-worker' });
        });
        break;

      case 'GET_CONNECTION_COUNT':
        port.postMessage({ type: 'COUNT', data: connections.size });
        break;
    }
  });

  port.start();

  // 通知所有页面有新连接
  connections.forEach(p => {
    p.postMessage({ type: 'PEER_JOINED', data: connections.size });
  });
});

// ============ page.js（多个页面共用） ============
const shared = new SharedWorker('./shared-worker.js');
shared.port.start();

shared.port.addEventListener('message', (event) => {
  const { type, data } = event.data;
  console.log(`收到 [${type}]:`, data);
});

// 广播消息
shared.port.postMessage({ type: 'BROADCAST', data: '来自页面A的消息' });
```

---

**5. 内联 Worker（无需额外文件）**

```javascript
// ============ 使用 Blob URL 创建内联 Worker ============
function createInlineWorker(fn) {
  const blob = new Blob(
    [`(${fn.toString()})()`],
    { type: 'application/javascript' }
  );
  const url = URL.createObjectURL(blob);
  const worker = new Worker(url);

  // 清理 Blob URL（Worker 已经加载了代码）
  URL.revokeObjectURL(url);
  return worker;
}

// 使用
const worker = createInlineWorker(function () {
  self.addEventListener('message', (e) => {
    const { numbers } = e.data;
    // 在 Worker 中执行素数筛
    const primes = sieveOfEratosthenes(numbers);
    self.postMessage({ primes });
  });

  function sieveOfEratosthenes(limit) {
    const sieve = new Uint8Array(limit + 1);
    const primes = [];
    for (let i = 2; i <= limit; i++) {
      if (!sieve[i]) {
        primes.push(i);
        for (let j = i * i; j <= limit; j += i) {
          sieve[j] = 1;
        }
      }
    }
    return primes;
  }
});

worker.onmessage = (e) => {
  console.log(`找到 ${e.data.primes.length} 个素数`);
};

worker.postMessage({ numbers: 10_000_000 });
```

---

**6. Worker 池（任务调度）**

```javascript
// ============ Worker Pool 实现 ============
class WorkerPool {
  constructor(workerScript, poolSize = navigator.hardwareConcurrency || 4) {
    this.workers = [];
    this.queue = [];
    this.activeCount = 0;
    this.poolSize = poolSize;

    for (let i = 0; i < poolSize; i++) {
      this.workers.push({
        worker: new Worker(workerScript),
        busy: false,
      });
    }
  }

  exec(data) {
    return new Promise((resolve, reject) => {
      const task = { data, resolve, reject };
      const idle = this.workers.find(w => !w.busy);

      if (idle) {
        this._run(idle, task);
      } else {
        this.queue.push(task); // 排队
      }
    });
  }

  _run(workerObj, task) {
    workerObj.busy = true;
    this.activeCount++;

    const onMessage = (e) => {
      workerObj.worker.removeEventListener('message', onMessage);
      workerObj.worker.removeEventListener('error', onError);
      workerObj.busy = false;
      this.activeCount--;
      task.resolve(e.data);
      this._next(workerObj);
    };

    const onError = (e) => {
      workerObj.worker.removeEventListener('message', onMessage);
      workerObj.worker.removeEventListener('error', onError);
      workerObj.busy = false;
      this.activeCount--;
      task.reject(e);
      this._next(workerObj);
    };

    workerObj.worker.addEventListener('message', onMessage);
    workerObj.worker.addEventListener('error', onError);
    workerObj.worker.postMessage(task.data);
  }

  _next(workerObj) {
    if (this.queue.length > 0) {
      const next = this.queue.shift();
      this._run(workerObj, next);
    }
  }

  terminate() {
    this.workers.forEach(w => w.worker.terminate());
    this.workers = [];
    this.queue = [];
  }
}

// 使用
const pool = new WorkerPool('./compute-worker.js', 4);
const tasks = Array.from({ length: 20 }, (_, i) => pool.exec({ taskId: i, data: i * 1000 }));
const results = await Promise.all(tasks);
console.log('所有任务完成:', results);
pool.terminate();
```

**面试追问**：
- Worker 中能使用哪些 API？不能使用哪些？
- `postMessage` 的结构化克隆算法有什么限制？
- 如何在 Worker 中使用 ES Module（`type: 'module'`）？

---

## Service Worker 与 PWA

### 离线优先架构 ⭐⭐

**1. Service Worker 生命周期**

```
注册 → 安装(install) → 等待(waiting) → 激活(activate) → 控制页面(fetch)
                                                          ↓
                                                    冗余(redundant)
```

```javascript
// ============ 注册 Service Worker ============
// main.js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const reg = await navigator.serviceWorker.register('/sw.js', {
        scope: '/', // 控制范围
      });

      // 更新检测
      reg.addEventListener('updatefound', () => {
        const newWorker = reg.installing;
        console.log('发现新版本 Service Worker');

        newWorker.addEventListener('statechange', () => {
          console.log('SW 状态变更:', newWorker.state);
          // installing → installed → activating → activated
        });
      });

      console.log('SW 注册成功，scope:', reg.scope);
    } catch (err) {
      console.error('SW 注册失败:', err);
    }
  });
}

// ============ sw.js —— Service Worker 核心 ============
const CACHE_NAME = 'app-v2';
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/styles/main.css',
  '/scripts/app.js',
  '/images/logo.png',
];

// 安装阶段：预缓存静态资源
self.addEventListener('install', (event) => {
  console.log('[SW] 安装中...');

  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(STATIC_ASSETS))
      .then(() => {
        console.log('[SW] 静态资源已缓存');
        // 跳过等待，立即激活
        return self.skipWaiting();
      })
  );
});

// 激活阶段：清理旧缓存
self.addEventListener('activate', (event) => {
  console.log('[SW] 激活中...');

  event.waitUntil(
    caches.keys()
      .then(names =>
        Promise.all(
          names
            .filter(name => name !== CACHE_NAME)
            .map(name => {
              console.log('[SW] 删除旧缓存:', name);
              return caches.delete(name);
            })
        )
      )
      .then(() => {
        // 立即控制所有页面
        return self.clients.claim();
      })
  );
});
```

---

**2. 缓存策略**

```javascript
// ============ 多种缓存策略 ============

// 策略1：Cache First（缓存优先） —— 适合静态资源
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  const response = await fetch(request);
  if (response.ok) {
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
  }
  return response;
}

// 策略2：Network First（网络优先） —— 适合 API 数据
async function networkFirst(request, timeout = 3000) {
  try {
    const response = await Promise.race([
      fetch(request),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('timeout')), timeout)
      ),
    ]);

    if (response.ok) {
      const cache = await caches.open(CACHE_NAME);
      cache.put(request, response.clone());
    }
    return response;
  } catch {
    const cached = await caches.match(request);
    return cached || new Response('离线且无缓存', { status: 503 });
  }
}

// 策略3：Stale While Revalidate（返回缓存的同时更新） —— 适合更新不频繁的资源
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);

  // 后台更新缓存
  const fetchPromise = fetch(request).then(response => {
    if (response.ok) {
      cache.put(request, response.clone());
    }
    return response;
  });

  // 有缓存就先返回缓存
  return cached || fetchPromise;
}

// 策略4：Network Only —— 适合实时性要求高的数据
async function networkOnly(request) {
  return fetch(request);
}

// 策略5：Cache Only —— 适合不变的静态资源
async function cacheOnly(request) {
  return caches.match(request);
}

// ============ 路由分发 ============
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // 只处理同源请求
  if (url.origin !== location.origin) return;

  // 静态资源 → Cache First
  if (request.destination === 'image' ||
      request.destination === 'style' ||
      request.destination === 'script') {
    event.respondWith(cacheFirst(request));
    return;
  }

  // API 请求 → Network First
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(request));
    return;
  }

  // HTML 页面 → Stale While Revalidate
  if (request.mode === 'navigate') {
    event.respondWith(staleWhileRevalidate(request));
    return;
  }
});
```

---

**3. Push 通知**

```javascript
// ============ 推送通知 ============

// 客户端：请求通知权限并订阅
async function subscribePush() {
  const permission = await Notification.requestPermission();
  if (permission !== 'granted') {
    console.log('用户拒绝通知权限');
    return;
  }

  const reg = await navigator.serviceWorker.ready;

  // VAPID 公钥（服务端生成）
  const publicKey = 'BEl62iUYgUivxIkv69yViEuiBIa-Ib9-SkvMeAtA3LFgDzkOs-...';

  const subscription = await reg.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(publicKey),
  });

  // 发送订阅信息给服务器
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription),
  });
}

function urlBase64ToUint8Array(base64String) {
  const padding = '='.repeat((4 - base64String.length % 4) % 4);
  const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
  const rawData = atob(base64);
  return Uint8Array.from([...rawData].map(char => char.charCodeAt(0)));
}

// sw.js：接收推送
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? { title: '新消息', body: '你收到了一条新消息' };

  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/images/icon-192.png',
      badge: '/images/badge-72.png',
      data: { url: data.url || '/' },
      actions: [
        { action: 'open', title: '查看' },
        { action: 'dismiss', title: '忽略' },
      ],
    })
  );
});

// 通知点击处理
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'dismiss') return;

  const url = event.notification.data.url;

  event.waitUntil(
    self.clients.matchAll({ type: 'window' }).then(clients => {
      // 尝试聚焦已有窗口
      const existing = clients.find(c => c.url === url);
      if (existing) return existing.focus();
      // 否则打开新窗口
      return self.clients.openWindow(url);
    })
  );
});
```

---

**4. Workbox 使用**

```javascript
// ============ 使用 Google Workbox 简化 SW 开发 ============
// sw.js（使用 Workbox）
importScripts('https://storage.googleapis.com/workbox-cdn/releases/7.0.0/workbox-sw.js');

const { precacheAndRoute } = workbox.precaching;
const { registerRoute } = workbox.routing;
const { CacheFirst, NetworkFirst, StaleWhileRevalidate } = workbox.strategies;
const { ExpirationPlugin } = workbox.expiration;
const { CacheableResponsePlugin } = workbox.cacheableResponse;

// 预缓存（由构建工具注入 manifest）
precacheAndRoute(self.__WB_MANIFEST || []);

// 图片 —— Cache First + 过期管理
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 100,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 天
      }),
      new CacheableResponsePlugin({
        statuses: [0, 200],
      }),
    ],
  })
);

// API —— Network First
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    networkTimeoutSeconds: 3,
    plugins: [
      new ExpirationPlugin({ maxEntries: 50 }),
    ],
  })
);

// CSS/JS —— Stale While Revalidate
registerRoute(
  ({ request }) => request.destination === 'style' || request.destination === 'script',
  new StaleWhileRevalidate({
    cacheName: 'static-resources',
  })
);

// 离线回退页
workbox.recipes.offlineFallback({
  pageFallback: '/offline.html',
});
```

**面试追问**：
- Service Worker 的更新流程是怎样的？如何强制更新？
- `skipWaiting()` 与 `clients.claim()` 的区别？
- Cache API 和 HTTP 缓存（Cache-Control）有什么区别？

---

## Canvas 与 WebGL

### 图形绘制 ⭐⭐

**1. Canvas 2D API 基础**

```javascript
// ============ Canvas 基础 ============
const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');

// 设置画布大小（考虑高 DPR 屏幕）
function setupCanvas(canvas, width, height) {
  const dpr = window.devicePixelRatio || 1;
  canvas.width = width * dpr;
  canvas.height = height * dpr;
  canvas.style.width = `${width}px`;
  canvas.style.height = `${height}px`;

  const ctx = canvas.getContext('2d');
  ctx.scale(dpr, dpr);
  return ctx;
}

const highResCtx = setupCanvas(canvas, 800, 600);

// ============ 基本图形绘制 ============
// 矩形
ctx.fillStyle = '#3498db';
ctx.fillRect(50, 50, 200, 100);

ctx.strokeStyle = '#e74c3c';
ctx.lineWidth = 3;
ctx.strokeRect(50, 50, 200, 100);

// 路径 —— 三角形
ctx.beginPath();
ctx.moveTo(300, 50);
ctx.lineTo(400, 150);
ctx.lineTo(200, 150);
ctx.closePath();
ctx.fillStyle = '#2ecc71';
ctx.fill();
ctx.stroke();

// 圆弧
ctx.beginPath();
ctx.arc(500, 100, 50, 0, Math.PI * 2);
ctx.fillStyle = '#f39c12';
ctx.fill();

// 文本
ctx.font = '24px Arial';
ctx.fillStyle = '#333';
ctx.textAlign = 'center';
ctx.fillText('Canvas 2D', 400, 250);

// 渐变
const gradient = ctx.createLinearGradient(0, 300, 800, 300);
gradient.addColorStop(0, '#667eea');
gradient.addColorStop(1, '#764ba2');
ctx.fillStyle = gradient;
ctx.fillRect(0, 280, 800, 60);

// 图片绘制
const img = new Image();
img.onload = () => {
  ctx.drawImage(img, 50, 350, 200, 150);
};
img.src = 'photo.jpg';
```

---

**2. Canvas 动画**

```javascript
// ============ 粒子动画系统 ============
class Particle {
  constructor(canvas) {
    this.canvas = canvas;
    this.reset();
  }

  reset() {
    this.x = Math.random() * this.canvas.width;
    this.y = Math.random() * this.canvas.height;
    this.vx = (Math.random() - 0.5) * 2;
    this.vy = (Math.random() - 0.5) * 2;
    this.radius = Math.random() * 3 + 1;
    this.alpha = Math.random() * 0.5 + 0.5;
  }

  update() {
    this.x += this.vx;
    this.y += this.vy;

    // 边界反弹
    if (this.x < 0 || this.x > this.canvas.width) this.vx *= -1;
    if (this.y < 0 || this.y > this.canvas.height) this.vy *= -1;
  }

  draw(ctx) {
    ctx.beginPath();
    ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
    ctx.fillStyle = `rgba(100, 200, 255, ${this.alpha})`;
    ctx.fill();
  }
}

class ParticleSystem {
  constructor(canvasId, count = 200) {
    this.canvas = document.getElementById(canvasId);
    this.ctx = this.canvas.getContext('2d');
    this.particles = Array.from({ length: count }, () => new Particle(this.canvas));
    this.animationId = null;
    this.maxDistance = 120;
  }

  drawConnections() {
    for (let i = 0; i < this.particles.length; i++) {
      for (let j = i + 1; j < this.particles.length; j++) {
        const dx = this.particles[i].x - this.particles[j].x;
        const dy = this.particles[i].y - this.particles[j].y;
        const dist = Math.sqrt(dx * dx + dy * dy);

        if (dist < this.maxDistance) {
          const alpha = 1 - dist / this.maxDistance;
          this.ctx.beginPath();
          this.ctx.moveTo(this.particles[i].x, this.particles[i].y);
          this.ctx.lineTo(this.particles[j].x, this.particles[j].y);
          this.ctx.strokeStyle = `rgba(100, 200, 255, ${alpha * 0.3})`;
          this.ctx.stroke();
        }
      }
    }
  }

  animate = () => {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    this.particles.forEach(p => {
      p.update();
      p.draw(this.ctx);
    });

    this.drawConnections();
    this.animationId = requestAnimationFrame(this.animate);
  };

  start() {
    this.animate();
  }

  stop() {
    if (this.animationId) {
      cancelAnimationFrame(this.animationId);
      this.animationId = null;
    }
  }
}

const system = new ParticleSystem('myCanvas', 150);
system.start();
```

---

**3. WebGL 基础与 Three.js**

```javascript
// ============ 原生 WebGL —— 绘制三角形 ============
const canvas = document.getElementById('glCanvas');
const gl = canvas.getContext('webgl2') || canvas.getContext('webgl');

if (!gl) {
  console.error('WebGL 不受支持');
}

// 顶点着色器
const vsSource = `
  attribute vec4 aPosition;
  attribute vec4 aColor;
  varying lowp vec4 vColor;
  void main() {
    gl_Position = aPosition;
    vColor = aColor;
  }
`;

// 片段着色器
const fsSource = `
  varying lowp vec4 vColor;
  void main() {
    gl_FragColor = vColor;
  }
`;

function createShader(gl, type, source) {
  const shader = gl.createShader(type);
  gl.shaderSource(shader, source);
  gl.compileShader(shader);
  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    console.error('着色器编译失败:', gl.getShaderInfoLog(shader));
    gl.deleteShader(shader);
    return null;
  }
  return shader;
}

function createProgram(gl, vsSource, fsSource) {
  const vs = createShader(gl, gl.VERTEX_SHADER, vsSource);
  const fs = createShader(gl, gl.FRAGMENT_SHADER, fsSource);
  const program = gl.createProgram();
  gl.attachShader(program, vs);
  gl.attachShader(program, fs);
  gl.linkProgram(program);
  return program;
}

const program = createProgram(gl, vsSource, fsSource);

// ============ Three.js 入门 ============
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

function createScene() {
  // 场景
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x1a1a2e);

  // 相机
  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 5;

  // 渲染器
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  document.body.appendChild(renderer.domElement);

  // 控制器
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  // 光源
  const ambientLight = new THREE.AmbientLight(0x404040);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
  directionalLight.position.set(5, 5, 5);
  scene.add(directionalLight);

  // 几何体
  const geometry = new THREE.TorusKnotGeometry(1, 0.3, 100, 16);
  const material = new THREE.MeshPhongMaterial({
    color: 0x667eea,
    shininess: 100,
  });
  const mesh = new THREE.Mesh(geometry, material);
  scene.add(mesh);

  // 动画循环
  function animate() {
    requestAnimationFrame(animate);
    mesh.rotation.x += 0.01;
    mesh.rotation.y += 0.01;
    controls.update();
    renderer.render(scene, camera);
  }

  animate();

  // 响应式
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });
}

createScene();
```

**面试追问**：
- Canvas 如何处理高分辨率屏幕（Retina）？
- Canvas 与 SVG 分别适合什么场景？
- 什么是离屏 Canvas（OffscreenCanvas）？有什么优势？

---

## WebRTC

### P2P 实时通信 ⭐⭐

**1. WebRTC 架构**

```
┌─────────┐    信令服务器     ┌─────────┐
│  Peer A │ ◄──────────────► │  Peer B │
│         │    (WebSocket)    │         │
│  Offer  │ ──────────────►  │         │
│         │ ◄────────────── │  Answer  │
│  ICE    │ ◄──────────────► │  ICE    │
│Candidate│                   │Candidate│
└─────┬───┘                   └───┬─────┘
      │                           │
      │    P2P (DTLS/SRTP)        │
      └───────────────────────────┘
              直接通信

STUN Server：获取公网 IP
TURN Server：P2P 失败时的中继
```

---

**2. 信令流程与实现**

```javascript
// ============ 简化的 WebRTC 视频通话 ============
class VideoCall {
  constructor(signalingUrl) {
    this.pc = null;
    this.localStream = null;
    this.remoteStream = null;

    // 信令通道（WebSocket）
    this.ws = new WebSocket(signalingUrl);
    this.ws.onmessage = (e) => this._onSignalingMessage(JSON.parse(e.data));
  }

  async init() {
    // 获取本地媒体流
    this.localStream = await navigator.mediaDevices.getUserMedia({
      video: { width: 1280, height: 720 },
      audio: true,
    });

    document.getElementById('localVideo').srcObject = this.localStream;

    this._createPeerConnection();
  }

  _createPeerConnection() {
    this.pc = new RTCPeerConnection({
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
        {
          urls: 'turn:your-turn-server.com:3478',
          username: 'user',
          credential: 'pass',
        },
      ],
    });

    // 添加本地媒体轨道
    this.localStream.getTracks().forEach(track => {
      this.pc.addTrack(track, this.localStream);
    });

    // 接收远端媒体流
    this.pc.ontrack = (event) => {
      this.remoteStream = event.streams[0];
      document.getElementById('remoteVideo').srcObject = this.remoteStream;
    };

    // ICE 候选
    this.pc.onicecandidate = (event) => {
      if (event.candidate) {
        this._send({ type: 'ice-candidate', candidate: event.candidate });
      }
    };

    // 连接状态
    this.pc.onconnectionstatechange = () => {
      console.log('连接状态:', this.pc.connectionState);
    };
  }

  // 发起呼叫
  async call() {
    const offer = await this.pc.createOffer();
    await this.pc.setLocalDescription(offer);
    this._send({ type: 'offer', sdp: offer });
  }

  // 接收呼叫
  async _handleOffer(offer) {
    await this.pc.setRemoteDescription(new RTCSessionDescription(offer));
    const answer = await this.pc.createAnswer();
    await this.pc.setLocalDescription(answer);
    this._send({ type: 'answer', sdp: answer });
  }

  async _onSignalingMessage(msg) {
    switch (msg.type) {
      case 'offer':
        await this._handleOffer(msg.sdp);
        break;
      case 'answer':
        await this.pc.setRemoteDescription(new RTCSessionDescription(msg.sdp));
        break;
      case 'ice-candidate':
        await this.pc.addIceCandidate(new RTCIceCandidate(msg.candidate));
        break;
    }
  }

  _send(data) {
    this.ws.send(JSON.stringify(data));
  }

  // 屏幕共享
  async shareScreen() {
    const screenStream = await navigator.mediaDevices.getDisplayMedia({
      video: { cursor: 'always' },
      audio: false,
    });

    const videoTrack = screenStream.getVideoTracks()[0];
    const sender = this.pc.getSenders().find(s => s.track?.kind === 'video');
    if (sender) {
      await sender.replaceTrack(videoTrack);
    }

    videoTrack.onended = async () => {
      // 恢复摄像头
      const camTrack = this.localStream.getVideoTracks()[0];
      if (sender) await sender.replaceTrack(camTrack);
    };
  }

  // 挂断
  hangup() {
    this.pc?.close();
    this.localStream?.getTracks().forEach(t => t.stop());
    this.ws?.close();
  }
}

// 使用
const vc = new VideoCall('wss://signaling.example.com');
await vc.init();
// 发起端
await vc.call();
```

---

**3. DataChannel（数据通道）**

```javascript
// ============ P2P 数据传输 ============
// 发起端创建 DataChannel
const dataChannel = pc.createDataChannel('fileTransfer', {
  ordered: true,          // 保序
  maxRetransmits: 3,      // 最大重传次数
});

dataChannel.onopen = () => console.log('DataChannel 已打开');
dataChannel.onclose = () => console.log('DataChannel 已关闭');

// 发送文件
async function sendFile(file) {
  const CHUNK_SIZE = 16384; // 16KB
  const reader = file.stream().getReader();
  let offset = 0;

  // 先发元信息
  dataChannel.send(JSON.stringify({
    type: 'file-meta',
    name: file.name,
    size: file.size,
    mimeType: file.type,
  }));

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    // 流控：等待缓冲区有空间
    while (dataChannel.bufferedAmount > 65536) {
      await new Promise(r => setTimeout(r, 10));
    }

    dataChannel.send(value);
    offset += value.byteLength;
    console.log(`发送进度: ${((offset / file.size) * 100).toFixed(1)}%`);
  }

  dataChannel.send(JSON.stringify({ type: 'file-end' }));
}

// 接收端
pc.ondatachannel = (event) => {
  const channel = event.channel;
  const chunks = [];
  let fileMeta = null;

  channel.onmessage = (e) => {
    if (typeof e.data === 'string') {
      const msg = JSON.parse(e.data);
      if (msg.type === 'file-meta') {
        fileMeta = msg;
        chunks.length = 0;
      } else if (msg.type === 'file-end') {
        const blob = new Blob(chunks, { type: fileMeta.mimeType });
        const url = URL.createObjectURL(blob);
        // 触发下载
        const a = document.createElement('a');
        a.href = url;
        a.download = fileMeta.name;
        a.click();
        URL.revokeObjectURL(url);
      }
    } else {
      chunks.push(e.data);
    }
  };
};
```

**面试追问**：
- WebRTC 如何穿越 NAT？STUN 与 TURN 的区别？
- SDP（Session Description Protocol）包含哪些信息？
- WebRTC 的安全机制是什么？（DTLS, SRTP）

---

## Fetch API 与 AbortController

### 现代网络请求 ⭐⭐⭐

**1. Fetch vs XMLHttpRequest**

| 特性 | Fetch API | XMLHttpRequest |
|------|-----------|----------------|
| Promise 支持 | ✅ 原生 | ❌ 回调/事件 |
| 流式响应 | ✅ ReadableStream | ❌ |
| 请求取消 | ✅ AbortController | ✅ abort() |
| 上传进度 | ❌（需要 workaround） | ✅ onprogress |
| Cookie 发送 | 需要 `credentials` 配置 | 默认同源发送 |
| 错误处理 | 仅网络错误 reject | 更精细的状态 |
| Service Worker | ✅ 可拦截 | ✅ 可拦截 |

---

**2. Fetch 深入使用**

```javascript
// ============ 基础请求 ============
// GET
const response = await fetch('/api/users');
if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${response.statusText}`);
}
const data = await response.json();

// POST
const res = await fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'Alice', email: 'alice@example.com' }),
});

// FormData（文件上传）
const formData = new FormData();
formData.append('file', fileInput.files[0]);
formData.append('description', '头像');
await fetch('/api/upload', { method: 'POST', body: formData });
// 注意：使用 FormData 时不要手动设置 Content-Type

// ============ AbortController —— 请求取消 ============
const controller = new AbortController();

// 超时自动取消
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch('/api/slow-endpoint', {
    signal: controller.signal,
  });
  clearTimeout(timeoutId);
  const data = await response.json();
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('请求被取消（超时或手动）');
  } else {
    throw err;
  }
}

// 手动取消（如用户切换页面）
// controller.abort();
```

---

**3. 流式响应（SSE / Streaming）**

```javascript
// ============ 流式读取响应体 ============
async function fetchWithProgress(url, onProgress) {
  const response = await fetch(url);
  const contentLength = +response.headers.get('Content-Length');
  const reader = response.body.getReader();

  let received = 0;
  const chunks = [];

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    chunks.push(value);
    received += value.length;

    if (onProgress && contentLength) {
      onProgress(received / contentLength);
    }
  }

  // 合并 chunks
  const allChunks = new Uint8Array(received);
  let offset = 0;
  for (const chunk of chunks) {
    allChunks.set(chunk, offset);
    offset += chunk.length;
  }

  return new TextDecoder().decode(allChunks);
}

// 使用
const text = await fetchWithProgress('/api/large-file', (progress) => {
  console.log(`下载进度: ${(progress * 100).toFixed(1)}%`);
});

// ============ 读取 SSE（Server-Sent Events）风格的流 ============
async function readSSEStream(url, onMessage) {
  const response = await fetch(url);
  const reader = response.body
    .pipeThrough(new TextDecoderStream())
    .getReader();

  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += value;
    const lines = buffer.split('\n');
    buffer = lines.pop(); // 保留不完整的行

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6);
        if (data === '[DONE]') return;
        try {
          onMessage(JSON.parse(data));
        } catch {
          onMessage(data);
        }
      }
    }
  }
}

// 使用（如 AI 流式对话）
await readSSEStream('/api/chat/stream', (chunk) => {
  process.stdout.write(chunk.content || '');
});
```

---

**4. 请求拦截器封装**

```javascript
// ============ Fetch 拦截器（类似 Axios） ============
class HttpClient {
  constructor(baseURL = '') {
    this.baseURL = baseURL;
    this.requestInterceptors = [];
    this.responseInterceptors = [];
    this.defaultHeaders = {};
  }

  // 添加请求拦截器
  useRequestInterceptor(fulfilled, rejected) {
    this.requestInterceptors.push({ fulfilled, rejected });
    return this.requestInterceptors.length - 1;
  }

  // 添加响应拦截器
  useResponseInterceptor(fulfilled, rejected) {
    this.responseInterceptors.push({ fulfilled, rejected });
    return this.responseInterceptors.length - 1;
  }

  async request(url, options = {}) {
    let config = {
      url: `${this.baseURL}${url}`,
      headers: { ...this.defaultHeaders, ...options.headers },
      ...options,
    };

    // 执行请求拦截器
    for (const interceptor of this.requestInterceptors) {
      try {
        config = await interceptor.fulfilled(config);
      } catch (err) {
        if (interceptor.rejected) interceptor.rejected(err);
        throw err;
      }
    }

    let response;
    try {
      response = await fetch(config.url, config);
    } catch (err) {
      // 网络错误
      for (const interceptor of this.responseInterceptors) {
        if (interceptor.rejected) interceptor.rejected(err);
      }
      throw err;
    }

    // 执行响应拦截器
    for (const interceptor of this.responseInterceptors) {
      try {
        response = await interceptor.fulfilled(response);
      } catch (err) {
        if (interceptor.rejected) interceptor.rejected(err);
        throw err;
      }
    }

    return response;
  }

  get(url, options) { return this.request(url, { ...options, method: 'GET' }); }
  post(url, body, options) {
    return this.request(url, {
      ...options,
      method: 'POST',
      body: JSON.stringify(body),
      headers: { 'Content-Type': 'application/json', ...options?.headers },
    });
  }
  put(url, body, options) {
    return this.request(url, {
      ...options,
      method: 'PUT',
      body: JSON.stringify(body),
      headers: { 'Content-Type': 'application/json', ...options?.headers },
    });
  }
  delete(url, options) { return this.request(url, { ...options, method: 'DELETE' }); }
}

// ============ 使用 ============
const http = new HttpClient('https://api.example.com');

// 请求拦截：注入 Token
http.useRequestInterceptor((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers = { ...config.headers, Authorization: `Bearer ${token}` };
  }
  return config;
});

// 响应拦截：统一错误处理 + 自动 JSON
http.useResponseInterceptor(
  async (response) => {
    if (response.status === 401) {
      // Token 过期，跳转登录
      window.location.href = '/login';
      throw new Error('Unauthorized');
    }
    if (!response.ok) {
      const error = await response.json().catch(() => ({}));
      throw new Error(error.message || `HTTP ${response.status}`);
    }
    return response.json(); // 自动解析 JSON
  },
  (error) => {
    console.error('网络请求失败:', error);
    throw error;
  }
);

// 发起请求
const users = await http.get('/users');
await http.post('/users', { name: 'Bob', email: 'bob@example.com' });
```

---

**5. 超时控制与竞态**

```javascript
// ============ 封装带超时的 fetch ============
function fetchWithTimeout(url, options = {}, timeout = 5000) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  return fetch(url, {
    ...options,
    signal: options.signal
      ? AbortSignal.any([options.signal, controller.signal])  // 合并多个信号
      : controller.signal,
  }).finally(() => clearTimeout(timeoutId));
}

// ============ 防止竞态条件（Race Condition） ============
// 场景：快速输入搜索词，只要最后一次结果
function createLatestOnly() {
  let currentController = null;

  return async function fetchLatest(url) {
    // 取消之前的请求
    if (currentController) {
      currentController.abort();
    }

    currentController = new AbortController();

    try {
      const response = await fetch(url, { signal: currentController.signal });
      return await response.json();
    } catch (err) {
      if (err.name === 'AbortError') {
        return null; // 被取消，忽略
      }
      throw err;
    }
  };
}

// 使用
const searchLatest = createLatestOnly();

searchInput.addEventListener('input', async (e) => {
  const query = e.target.value;
  if (!query) return;

  const result = await searchLatest(`/api/search?q=${encodeURIComponent(query)}`);
  if (result) {
    renderResults(result); // 只渲染最新的结果
  }
});
```

**面试追问**：
- Fetch 在什么情况下 reject？4xx/5xx 会 reject 吗？
- `AbortSignal.any()` 是什么？如何用？
- 如何用 Fetch 实现请求重试？

---

## Intersection Observer

### 可视区域观察 ⭐⭐

**1. 核心原理**

Intersection Observer API 异步观察目标元素与祖先元素（或视口）的交叉状态，避免了传统 `scroll` 事件 + `getBoundingClientRect()` 的性能问题。

```javascript
// ============ 基础用法 ============
const observer = new IntersectionObserver(
  (entries, observer) => {
    entries.forEach(entry => {
      console.log({
        target: entry.target,              // 被观察的元素
        isIntersecting: entry.isIntersecting, // 是否可见
        intersectionRatio: entry.intersectionRatio, // 可见比例 0-1
        boundingClientRect: entry.boundingClientRect, // 元素矩形
        rootBounds: entry.rootBounds,      // root 矩形
        time: entry.time,                  // 触发时间
      });
    });
  },
  {
    root: null,           // 观察相对于谁（null = viewport）
    rootMargin: '0px',    // root 的扩展/收缩（类似 CSS margin）
    threshold: [0, 0.25, 0.5, 0.75, 1], // 触发回调的可见比例
  }
);

observer.observe(document.querySelector('.target'));
// observer.unobserve(element);
// observer.disconnect();
```

---

**2. 图片懒加载**

```javascript
// ============ 懒加载实现 ============
class LazyLoader {
  constructor(options = {}) {
    this.observer = new IntersectionObserver(
      this._onIntersect.bind(this),
      {
        rootMargin: options.rootMargin || '200px 0px', // 提前 200px 加载
        threshold: 0,
      }
    );
  }

  _onIntersect(entries) {
    entries.forEach(entry => {
      if (!entry.isIntersecting) return;

      const el = entry.target;

      if (el.tagName === 'IMG') {
        this._loadImage(el);
      } else if (el.dataset.bgSrc) {
        el.style.backgroundImage = `url(${el.dataset.bgSrc})`;
        el.removeAttribute('data-bg-src');
      }

      this.observer.unobserve(el);
    });
  }

  _loadImage(img) {
    const src = img.dataset.src;
    const srcset = img.dataset.srcset;

    if (src) {
      img.src = src;
      img.removeAttribute('data-src');
    }
    if (srcset) {
      img.srcset = srcset;
      img.removeAttribute('data-srcset');
    }

    img.classList.add('loaded');
  }

  observe(selector) {
    document.querySelectorAll(selector).forEach(el => {
      this.observer.observe(el);
    });
  }

  destroy() {
    this.observer.disconnect();
  }
}

// HTML: <img data-src="photo.jpg" src="placeholder.svg" alt="..." />
const lazy = new LazyLoader({ rootMargin: '300px 0px' });
lazy.observe('img[data-src]');
```

---

**3. 无限滚动**

```javascript
// ============ 无限滚动列表 ============
class InfiniteScroll {
  constructor(container, loadMore) {
    this.container = container;
    this.loadMore = loadMore;
    this.loading = false;
    this.hasMore = true;
    this.page = 1;

    // 哨兵元素
    this.sentinel = document.createElement('div');
    this.sentinel.className = 'scroll-sentinel';
    this.sentinel.style.height = '1px';
    this.container.appendChild(this.sentinel);

    this.observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && !this.loading && this.hasMore) {
          this._load();
        }
      },
      { rootMargin: '400px' } // 提前 400px 触发
    );

    this.observer.observe(this.sentinel);
  }

  async _load() {
    this.loading = true;
    this._showLoader();

    try {
      const { items, hasMore } = await this.loadMore(this.page);
      this.hasMore = hasMore;
      this.page++;

      // 在哨兵之前插入新内容
      const fragment = document.createDocumentFragment();
      items.forEach(item => {
        const el = this._createItem(item);
        fragment.appendChild(el);
      });
      this.container.insertBefore(fragment, this.sentinel);

      if (!hasMore) {
        this.observer.disconnect();
        this.sentinel.textContent = '已加载全部';
      }
    } catch (err) {
      console.error('加载失败:', err);
    } finally {
      this.loading = false;
      this._hideLoader();
    }
  }

  _createItem(item) {
    const div = document.createElement('div');
    div.className = 'list-item';
    div.textContent = item.title;
    return div;
  }

  _showLoader() { this.sentinel.textContent = '加载中...'; }
  _hideLoader() { this.sentinel.textContent = ''; }
}

// 使用
new InfiniteScroll(document.getElementById('list'), async (page) => {
  const res = await fetch(`/api/posts?page=${page}&limit=20`);
  const data = await res.json();
  return { items: data.items, hasMore: data.hasMore };
});
```

---

**4. 曝光统计**

```javascript
// ============ 元素曝光埋点 ============
class ExposureTracker {
  constructor(options = {}) {
    this.reported = new WeakSet(); // 防止重复上报
    this.minVisibleTime = options.minVisibleTime || 1000; // 最少可见 1 秒
    this.minVisibleRatio = options.minVisibleRatio || 0.5; // 至少可见 50%
    this.timers = new Map();

    this.observer = new IntersectionObserver(
      this._onIntersect.bind(this),
      { threshold: [0, this.minVisibleRatio] }
    );
  }

  _onIntersect(entries) {
    entries.forEach(entry => {
      const el = entry.target;

      if (entry.intersectionRatio >= this.minVisibleRatio) {
        // 开始计时
        if (!this.timers.has(el) && !this.reported.has(el)) {
          const timer = setTimeout(() => {
            this._report(el);
            this.timers.delete(el);
          }, this.minVisibleTime);
          this.timers.set(el, timer);
        }
      } else {
        // 离开视口，取消计时
        if (this.timers.has(el)) {
          clearTimeout(this.timers.get(el));
          this.timers.delete(el);
        }
      }
    });
  }

  _report(el) {
    if (this.reported.has(el)) return;
    this.reported.add(el);

    const data = {
      elementId: el.id || el.dataset.trackId,
      timestamp: Date.now(),
      page: location.pathname,
    };

    // 使用 sendBeacon 保证页面关闭时也能发送
    navigator.sendBeacon('/api/analytics/exposure', JSON.stringify(data));
    console.log('曝光上报:', data);
  }

  track(selector) {
    document.querySelectorAll(selector).forEach(el => {
      this.observer.observe(el);
    });
  }

  destroy() {
    this.timers.forEach(t => clearTimeout(t));
    this.timers.clear();
    this.observer.disconnect();
  }
}

// 使用
const tracker = new ExposureTracker({ minVisibleTime: 1500, minVisibleRatio: 0.5 });
tracker.track('[data-track-id]');
```

**面试追问**：
- IntersectionObserver 的回调在什么线程执行？
- rootMargin 支持百分比吗？
- 如何观察一个在 iframe 中的元素？

---

## Web Animations API

### 编程式动画

**1. WAAPI vs CSS 动画**

| 特性 | CSS Animations | Web Animations API |
|------|---------------|-------------------|
| 声明方式 | CSS @keyframes | JavaScript |
| 动态控制 | 有限（类名切换） | 完整（暂停/倒放/速率） |
| 运行时修改 | 困难 | 灵活 |
| 性能 | 可合成层优化 | 同样可合成层优化 |
| 事件监听 | animationend 等 | Promise（finished） |
| 复杂编排 | 困难 | 容易 |

---

**2. 基础用法**

```javascript
// ============ 基本动画 ============
const element = document.querySelector('.box');

// element.animate() 返回 Animation 对象
const animation = element.animate(
  [
    { transform: 'translateX(0) scale(1)', opacity: 1 },
    { transform: 'translateX(300px) scale(1.2)', opacity: 0.5, offset: 0.7 },
    { transform: 'translateX(500px) scale(1)', opacity: 1 },
  ],
  {
    duration: 2000,
    easing: 'cubic-bezier(0.25, 0.1, 0.25, 1)',
    iterations: Infinity,
    direction: 'alternate',
    fill: 'forwards',    // 保持结束状态
  }
);

// ============ 动画控制 ============
animation.pause();
animation.play();
animation.reverse();
animation.cancel();
animation.playbackRate = 2;      // 2 倍速
animation.currentTime = 1000;    // 跳到 1 秒处

// 等待动画完成
await animation.finished;
console.log('动画完成');

// ============ 复杂动画编排 ============
async function sequenceAnimation(element) {
  // 1. 淡入
  await element.animate(
    [{ opacity: 0 }, { opacity: 1 }],
    { duration: 500, fill: 'forwards' }
  ).finished;

  // 2. 放大
  await element.animate(
    [{ transform: 'scale(1)' }, { transform: 'scale(1.5)' }],
    { duration: 300, fill: 'forwards' }
  ).finished;

  // 3. 摇晃
  await element.animate(
    [
      { transform: 'scale(1.5) rotate(0deg)' },
      { transform: 'scale(1.5) rotate(5deg)' },
      { transform: 'scale(1.5) rotate(-5deg)' },
      { transform: 'scale(1.5) rotate(0deg)' },
    ],
    { duration: 400, iterations: 3 }
  ).finished;

  // 4. 恢复
  await element.animate(
    [{ transform: 'scale(1.5)' }, { transform: 'scale(1)' }],
    { duration: 300, fill: 'forwards' }
  ).finished;
}

// 并行动画
function parallelAnimation(elements) {
  const animations = elements.map((el, i) =>
    el.animate(
      [
        { transform: 'translateY(50px)', opacity: 0 },
        { transform: 'translateY(0)', opacity: 1 },
      ],
      {
        duration: 600,
        delay: i * 100, // 交错延迟
        fill: 'forwards',
        easing: 'ease-out',
      }
    )
  );

  return Promise.all(animations.map(a => a.finished));
}
```

---

**3. 性能优化**

```javascript
// ============ 合成层动画（高性能） ============
// ✅ 仅使用 transform 和 opacity（在合成器线程执行）
element.animate(
  [{ transform: 'translateX(0)' }, { transform: 'translateX(200px)' }],
  { duration: 300, composite: 'replace' }
);

// ❌ 触发布局的属性（慢）
element.animate(
  [{ width: '100px' }, { width: '200px' }],  // 触发 Layout
  { duration: 300 }
);

// ============ 检测动画是否在合成层 ============
// 使用 Animation.commitStyles() 将动画效果写入元素内联样式
const anim = element.animate(
  [{ transform: 'translateX(200px)' }],
  { duration: 500, fill: 'forwards' }
);

anim.finished.then(() => {
  anim.commitStyles(); // 将最终状态写入 style
  anim.cancel();       // 释放动画资源
});

// ============ 使用 will-change 提示浏览器 ============
element.style.willChange = 'transform, opacity';
const anim2 = element.animate(/* ... */);
anim2.finished.then(() => {
  element.style.willChange = 'auto'; // 动画完成后释放
});
```

**面试追问**：
- WAAPI 和 `requestAnimationFrame` 的区别？
- `composite: 'add'` 和 `'accumulate'` 有什么区别？
- 如何实现弹簧动画（spring）？

---

## 其他重要 API

### 常用 Web API 速查

**1. Clipboard API（剪贴板）**

```javascript
// ============ 现代剪贴板 API ============
// 写入文本
async function copyText(text) {
  try {
    await navigator.clipboard.writeText(text);
    console.log('复制成功');
  } catch (err) {
    // 降级方案
    const textarea = document.createElement('textarea');
    textarea.value = text;
    textarea.style.position = 'fixed';
    textarea.style.left = '-9999px';
    document.body.appendChild(textarea);
    textarea.select();
    document.execCommand('copy');
    document.body.removeChild(textarea);
  }
}

// 读取文本
async function pasteText() {
  return await navigator.clipboard.readText();
}

// 复制图片
async function copyImage(imgElement) {
  const canvas = document.createElement('canvas');
  canvas.width = imgElement.naturalWidth;
  canvas.height = imgElement.naturalHeight;
  canvas.getContext('2d').drawImage(imgElement, 0, 0);

  const blob = await new Promise(r => canvas.toBlob(r, 'image/png'));
  await navigator.clipboard.write([
    new ClipboardItem({ 'image/png': blob }),
  ]);
}
```

---

**2. Drag & Drop API**

```javascript
// ============ 拖拽排序 ============
class DragSortable {
  constructor(container) {
    this.container = container;
    this.dragItem = null;

    container.querySelectorAll('.sortable-item').forEach(item => {
      item.draggable = true;
      item.addEventListener('dragstart', this._onDragStart.bind(this));
      item.addEventListener('dragover', this._onDragOver.bind(this));
      item.addEventListener('dragend', this._onDragEnd.bind(this));
    });
  }

  _onDragStart(e) {
    this.dragItem = e.target;
    e.target.classList.add('dragging');
    e.dataTransfer.effectAllowed = 'move';
    e.dataTransfer.setData('text/plain', ''); // Firefox 兼容
  }

  _onDragOver(e) {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';

    const target = e.target.closest('.sortable-item');
    if (!target || target === this.dragItem) return;

    const rect = target.getBoundingClientRect();
    const midY = rect.top + rect.height / 2;

    if (e.clientY < midY) {
      this.container.insertBefore(this.dragItem, target);
    } else {
      this.container.insertBefore(this.dragItem, target.nextSibling);
    }
  }

  _onDragEnd(e) {
    e.target.classList.remove('dragging');
    this.dragItem = null;
    // 触发排序完成回调
    this._emitOrder();
  }

  _emitOrder() {
    const order = [...this.container.querySelectorAll('.sortable-item')]
      .map(el => el.dataset.id);
    this.container.dispatchEvent(new CustomEvent('sort', { detail: { order } }));
  }
}

new DragSortable(document.getElementById('sortable-list'));
```

---

**3. File API**

```javascript
// ============ 文件读取与处理 ============
// 读取文件内容
function readFile(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result);
    reader.onerror = () => reject(reader.error);

    if (file.type.startsWith('text/')) {
      reader.readAsText(file);
    } else if (file.type.startsWith('image/')) {
      reader.readAsDataURL(file); // Base64
    } else {
      reader.readAsArrayBuffer(file);
    }
  });
}

// 现代方式：File System Access API
async function openFile() {
  const [fileHandle] = await window.showOpenFilePicker({
    types: [
      {
        description: 'Text Files',
        accept: { 'text/*': ['.txt', '.md', '.json'] },
      },
    ],
  });

  const file = await fileHandle.getFile();
  const contents = await file.text();
  return { fileHandle, contents };
}

async function saveFile(fileHandle, contents) {
  const writable = await fileHandle.createWritable();
  await writable.write(contents);
  await writable.close();
}

// 拖拽上传
const dropZone = document.getElementById('drop-zone');
dropZone.addEventListener('dragover', (e) => {
  e.preventDefault();
  dropZone.classList.add('drag-over');
});
dropZone.addEventListener('dragleave', () => {
  dropZone.classList.remove('drag-over');
});
dropZone.addEventListener('drop', async (e) => {
  e.preventDefault();
  dropZone.classList.remove('drag-over');
  const files = [...e.dataTransfer.files];
  for (const file of files) {
    console.log(`文件: ${file.name}, 大小: ${file.size}, 类型: ${file.type}`);
  }
});
```

---

**4. Geolocation API**

```javascript
// ============ 地理定位 ============
function getCurrentPosition(options = {}) {
  return new Promise((resolve, reject) => {
    if (!navigator.geolocation) {
      reject(new Error('Geolocation 不受支持'));
      return;
    }

    navigator.geolocation.getCurrentPosition(resolve, reject, {
      enableHighAccuracy: true,  // 高精度（GPS，耗电更多）
      timeout: 10000,
      maximumAge: 60000,         // 缓存 1 分钟
      ...options,
    });
  });
}

// 使用
try {
  const { coords } = await getCurrentPosition();
  console.log(`纬度: ${coords.latitude}, 经度: ${coords.longitude}`);
  console.log(`精度: ${coords.accuracy}米`);
} catch (err) {
  switch (err.code) {
    case 1: console.log('用户拒绝授权'); break;
    case 2: console.log('位置不可用'); break;
    case 3: console.log('请求超时'); break;
  }
}

// 持续追踪
const watchId = navigator.geolocation.watchPosition(
  (pos) => console.log('位置更新:', pos.coords),
  (err) => console.error(err),
  { enableHighAccuracy: true }
);
// navigator.geolocation.clearWatch(watchId);
```

---

**5. Notification API**

```javascript
// ============ 桌面通知 ============
async function showNotification(title, options = {}) {
  if (!('Notification' in window)) {
    console.log('浏览器不支持通知');
    return;
  }

  let permission = Notification.permission;
  if (permission === 'default') {
    permission = await Notification.requestPermission();
  }

  if (permission !== 'granted') {
    console.log('用户拒绝通知');
    return;
  }

  const notification = new Notification(title, {
    body: options.body || '',
    icon: options.icon || '/favicon.png',
    tag: options.tag,       // 相同 tag 会替换
    requireInteraction: options.sticky || false,
    ...options,
  });

  notification.onclick = () => {
    window.focus();
    notification.close();
    options.onClick?.();
  };

  // 自动关闭
  if (options.autoClose) {
    setTimeout(() => notification.close(), options.autoClose);
  }

  return notification;
}

showNotification('新消息', {
  body: '你收到了一条来自 Alice 的消息',
  tag: 'chat-alice',
  autoClose: 5000,
  onClick: () => navigateTo('/chat/alice'),
});
```

---

**6. WebSocket（复习）**

```javascript
// ============ 带自动重连和心跳的 WebSocket ============
class ReconnectingWebSocket {
  constructor(url, options = {}) {
    this.url = url;
    this.maxRetries = options.maxRetries || 10;
    this.retryDelay = options.retryDelay || 1000;
    this.heartbeatInterval = options.heartbeatInterval || 30000;
    this.retries = 0;
    this.handlers = new Map();
    this.ws = null;
    this._heartbeatTimer = null;

    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('[WS] 已连接');
      this.retries = 0;
      this._startHeartbeat();
      this._emit('open');
    };

    this.ws.onmessage = (e) => {
      const data = JSON.parse(e.data);
      if (data.type === 'pong') return; // 心跳响应
      this._emit('message', data);
    };

    this.ws.onclose = (e) => {
      console.log('[WS] 连接关闭:', e.code, e.reason);
      this._stopHeartbeat();
      this._emit('close', e);

      if (e.code !== 1000 && this.retries < this.maxRetries) {
        const delay = this.retryDelay * Math.pow(2, this.retries); // 指数退避
        console.log(`[WS] ${delay}ms 后重连（第 ${this.retries + 1} 次）`);
        setTimeout(() => this.connect(), delay);
        this.retries++;
      }
    };

    this.ws.onerror = (err) => {
      console.error('[WS] 错误:', err);
      this._emit('error', err);
    };
  }

  send(data) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    }
  }

  on(event, handler) {
    if (!this.handlers.has(event)) this.handlers.set(event, []);
    this.handlers.get(event).push(handler);
  }

  _emit(event, data) {
    this.handlers.get(event)?.forEach(h => h(data));
  }

  _startHeartbeat() {
    this._heartbeatTimer = setInterval(() => {
      this.send({ type: 'ping', timestamp: Date.now() });
    }, this.heartbeatInterval);
  }

  _stopHeartbeat() {
    clearInterval(this._heartbeatTimer);
  }

  close() {
    this.maxRetries = 0; // 阻止重连
    this.ws?.close(1000, '正常关闭');
    this._stopHeartbeat();
  }
}

// 使用
const ws = new ReconnectingWebSocket('wss://api.example.com/ws');
ws.on('message', (data) => console.log('收到:', data));
ws.send({ type: 'subscribe', channel: 'notifications' });
```

---

## 实战案例

### 综合运用：在线协作白板

以下案例综合运用 **Canvas、WebSocket、Web Worker、File API、Intersection Observer** 等多个 Web API。

```javascript
// ============ 在线协作白板 ============

// 1. 白板核心（Canvas + 事件处理）
class WhiteboardCanvas {
  constructor(canvasId) {
    this.canvas = document.getElementById(canvasId);
    this.ctx = this.canvas.getContext('2d');
    this.isDrawing = false;
    this.paths = [];
    this.currentPath = null;
    this.tool = 'pen';          // pen / eraser / rectangle
    this.color = '#333333';
    this.lineWidth = 2;

    this._setupHiDPI();
    this._bindEvents();
  }

  _setupHiDPI() {
    const dpr = window.devicePixelRatio || 1;
    const rect = this.canvas.getBoundingClientRect();
    this.canvas.width = rect.width * dpr;
    this.canvas.height = rect.height * dpr;
    this.ctx.scale(dpr, dpr);
    this.canvas.style.width = `${rect.width}px`;
    this.canvas.style.height = `${rect.height}px`;
  }

  _bindEvents() {
    this.canvas.addEventListener('pointerdown', this._onStart.bind(this));
    this.canvas.addEventListener('pointermove', this._onMove.bind(this));
    this.canvas.addEventListener('pointerup', this._onEnd.bind(this));
    this.canvas.addEventListener('pointerleave', this._onEnd.bind(this));
  }

  _getPos(e) {
    const rect = this.canvas.getBoundingClientRect();
    return { x: e.clientX - rect.left, y: e.clientY - rect.top };
  }

  _onStart(e) {
    this.isDrawing = true;
    const pos = this._getPos(e);
    this.currentPath = {
      tool: this.tool,
      color: this.tool === 'eraser' ? '#ffffff' : this.color,
      lineWidth: this.tool === 'eraser' ? 20 : this.lineWidth,
      points: [pos],
      id: crypto.randomUUID(),
    };
  }

  _onMove(e) {
    if (!this.isDrawing || !this.currentPath) return;
    const pos = this._getPos(e);
    this.currentPath.points.push(pos);
    this._drawPath(this.currentPath);

    // 广播给协作者
    this.onDraw?.(this.currentPath);
  }

  _onEnd() {
    if (this.currentPath) {
      this.paths.push(this.currentPath);
      this.onPathComplete?.(this.currentPath);
    }
    this.isDrawing = false;
    this.currentPath = null;
  }

  _drawPath(path) {
    const { points, color, lineWidth } = path;
    if (points.length < 2) return;

    this.ctx.beginPath();
    this.ctx.strokeStyle = color;
    this.ctx.lineWidth = lineWidth;
    this.ctx.lineCap = 'round';
    this.ctx.lineJoin = 'round';

    this.ctx.moveTo(points[0].x, points[0].y);
    for (let i = 1; i < points.length; i++) {
      this.ctx.lineTo(points[i].x, points[i].y);
    }
    this.ctx.stroke();
  }

  redrawAll() {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    this.paths.forEach(path => this._drawPath(path));
  }

  // 接收远程路径
  addRemotePath(path) {
    this.paths.push(path);
    this._drawPath(path);
  }

  // 导出为图片
  exportAsBlob(type = 'image/png') {
    return new Promise(r => this.canvas.toBlob(r, type));
  }

  // 撤销
  undo() {
    this.paths.pop();
    this.redrawAll();
  }
}

// 2. WebSocket 协作同步
class CollaborationSync {
  constructor(wsUrl, whiteboard) {
    this.ws = new ReconnectingWebSocket(wsUrl);
    this.whiteboard = whiteboard;

    // 接收远程绘制
    this.ws.on('message', (data) => {
      switch (data.type) {
        case 'draw':
          this.whiteboard.addRemotePath(data.path);
          break;
        case 'clear':
          this.whiteboard.paths = [];
          this.whiteboard.redrawAll();
          break;
        case 'sync':
          this.whiteboard.paths = data.paths;
          this.whiteboard.redrawAll();
          break;
      }
    });

    // 本地绘制完成时广播
    this.whiteboard.onPathComplete = (path) => {
      this.ws.send({ type: 'draw', path });
    };
  }
}

// 3. Worker 处理图像导出（不阻塞 UI）
const exportWorker = createInlineWorker(function () {
  self.addEventListener('message', async (e) => {
    const { imageData, format } = e.data;
    // 在 Worker 中进行图像处理
    // OffscreenCanvas 允许在 Worker 中使用 Canvas
    const canvas = new OffscreenCanvas(imageData.width, imageData.height);
    const ctx = canvas.getContext('2d');
    ctx.putImageData(imageData, 0, 0);
    const blob = await canvas.convertToBlob({ type: format, quality: 0.9 });
    self.postMessage({ blob }, []);
  });
});

// 4. 文件导入
async function importImage(file, whiteboard) {
  const img = new Image();
  const url = URL.createObjectURL(file);

  return new Promise((resolve) => {
    img.onload = () => {
      whiteboard.ctx.drawImage(img, 0, 0);
      URL.revokeObjectURL(url);
      resolve();
    };
    img.src = url;
  });
}

// 5. 组合使用
const whiteboard = new WhiteboardCanvas('whiteboard');
const sync = new CollaborationSync('wss://collab.example.com/ws', whiteboard);

// 工具栏交互
document.getElementById('btn-pen').onclick = () => { whiteboard.tool = 'pen'; };
document.getElementById('btn-eraser').onclick = () => { whiteboard.tool = 'eraser'; };
document.getElementById('btn-undo').onclick = () => { whiteboard.undo(); };
document.getElementById('btn-export').onclick = async () => {
  const blob = await whiteboard.exportAsBlob();
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'whiteboard.png';
  a.click();
  URL.revokeObjectURL(url);
};
document.getElementById('file-import').onchange = async (e) => {
  if (e.target.files[0]) {
    await importImage(e.target.files[0], whiteboard);
  }
};
```

---

## 面试题自查

### 治理补充

**常见误区澄清：**

1. **localStorage 是同步阻塞的** —— 大量读写会卡住主线程，生产环境存大数据要用 IndexedDB
2. **Fetch 不会因 404/500 reject** —— 只有网络错误（DNS 失败、断网）才 reject，需手动检查 `response.ok`
3. **Service Worker 作用域陷阱** —— `/sw.js` 放在根目录才能控制全站，放子目录只控制该子目录
4. **Canvas 默认坐标不考虑 DPR** —— 高分屏必须手动处理 `devicePixelRatio`，否则图像模糊
5. **Web Worker 无法访问 DOM** —— 但可以使用 fetch、IndexedDB、WebSocket 等

---

### 初级高频速刷（5 题）

**Q1：localStorage 和 sessionStorage 有什么区别？**
> localStorage 持久存储（手动清除），同源所有标签页共享。sessionStorage 仅在当前标签页会话期间有效，关闭标签页即清除，不跨标签页共享。

**Q2：什么是 HTML5 语义化？有什么好处？**
> 使用 `<header>`/`<nav>`/`<article>` 等标签代替无语义的 `<div>`。好处：① 提高可读性 ② 利于 SEO ③ 增强可访问性（屏幕阅读器）④ 便于维护。

**Q3：Web Worker 有什么用？和主线程怎么通信？**
> 在后台线程执行耗时计算，不阻塞 UI。通过 `postMessage()` 发送消息，`onmessage` 接收消息，数据通过结构化克隆传递。

**Q4：Fetch 和 XMLHttpRequest 的主要区别？**
> Fetch 基于 Promise、支持流式响应、用 AbortController 取消、更简洁。XHR 支持上传进度事件、默认发送同源 Cookie。

**Q5：IntersectionObserver 解决了什么问题？**
> 异步检测元素是否进入视口，替代 scroll 事件 + getBoundingClientRect() 的方案，性能更好。常用于懒加载、无限滚动、曝光埋点。

---

### 中级高频进阶（5 题）

**Q1：Service Worker 的生命周期是怎样的？更新策略有哪些？**
> 注册 → install → waiting → activate → fetch。更新时新 SW 进入 waiting，旧 SW 仍在控制页面，直到所有页面关闭。可用 `skipWaiting()` 强制激活，配合 `clients.claim()` 立即控制页面。Workbox 提供了 `workbox-window` 的 `promptForUpdate` 来通知用户刷新。

**Q2：IndexedDB 与 localStorage 在实际项目中如何选择？**
> 小于 2KB 的简单键值对（偏好设置）→ localStorage。大数据量、需要索引查询、需要在 Worker 中使用、离线数据 → IndexedDB。注意 localStorage 是同步阻塞的，频繁读写大数据会影响性能。

**Q3：如何防止 Fetch 请求的竞态条件？**
> 使用 AbortController：每次新请求前 abort 上一个。或维护一个请求 ID，回调中检查是否是最新请求。React 中可在 useEffect cleanup 中 abort。

**Q4：Canvas 和 SVG 分别适合什么场景？**
> Canvas：大量图形（粒子/游戏）、位图操作、实时渲染。SVG：少量可交互图形、可缩放不失真、需要 DOM 事件、可 CSS 样式化。性能上 Canvas 在图形数量多时优势明显，SVG 在少量图形时更灵活。

**Q5：Transferable Objects 的原理是什么？**
> postMessage 默认使用结构化克隆（深拷贝），数据大时开销高。Transferable Objects（ArrayBuffer、MessagePort、OffscreenCanvas 等）可以「转移所有权」——数据零拷贝，但原线程不再持有该对象（byteLength 变为 0）。

---

### 深入问答（15+ 题）

**Q1：localStorage 的 storage 事件在什么场景下触发？如何利用它实现跨标签页通信？**

> `storage` 事件在 **其他同源标签页** 修改 localStorage 时触发（当前标签页不触发）。可以利用这一特性实现跨标签页通信：

```javascript
// 发送端（标签页 A）
function broadcast(channel, data) {
  localStorage.setItem(`__broadcast_${channel}`, JSON.stringify({
    data,
    timestamp: Date.now(),
  }));
  // 发完立即清除，避免重复触发
  localStorage.removeItem(`__broadcast_${channel}`);
}

// 接收端（标签页 B/C/...）
window.addEventListener('storage', (e) => {
  if (e.key?.startsWith('__broadcast_') && e.newValue) {
    const channel = e.key.replace('__broadcast_', '');
    const { data } = JSON.parse(e.newValue);
    console.log(`频道 [${channel}] 收到:`, data);
  }
});

// 更现代的方案：BroadcastChannel API
const bc = new BroadcastChannel('app-events');
bc.postMessage({ type: 'LOGOUT' });
bc.onmessage = (e) => console.log('收到:', e.data);
```

---

**Q2：Web Worker 中可以使用哪些 API？哪些不能用？为什么？**

> Worker 运行在独立线程，没有 `window` 对象和 DOM 访问权限。

```javascript
// ✅ Worker 中可用的 API
// - fetch / XMLHttpRequest
// - setTimeout / setInterval / requestAnimationFrame（部分）
// - IndexedDB
// - WebSocket
// - crypto
// - TextEncoder / TextDecoder
// - structuredClone
// - importScripts()（经典 Worker）
// - ES Modules（type: 'module' 的 Worker）
// - navigator（部分属性）
// - location（只读）
// - performance
// - OffscreenCanvas

// ❌ Worker 中不可用
// - DOM（document、window、alert、confirm）
// - localStorage / sessionStorage（同步阻塞，设计上不适合多线程）
// - 直接访问页面元素

// 使用 ES Module Worker
const worker = new Worker('./worker.mjs', { type: 'module' });

// worker.mjs 中可以使用 import
import { heavyCompute } from './compute.mjs';
```

> 原因：Worker 的设计目标是「无共享并发」，通过消息传递通信，避免了多线程共享内存的复杂性（锁、竞态条件）。DOM 是单线程设计的，多线程操作 DOM 会导致不可预期的行为。

---

**Q3：Service Worker 中 skipWaiting() 和 clients.claim() 的区别及潜在问题？**

```javascript
// skipWaiting()：让新 SW 跳过 waiting 阶段，立即进入 activate
// 问题：页面可能一半请求由旧 SW 处理，一半由新 SW 处理

// clients.claim()：让已激活的 SW 立即控制当前所有打开的页面
// 正常情况下 SW 只控制在它之后打开的页面

// 组合使用的风险示例：
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('v2')
      .then(cache => cache.addAll(newAssets))
      .then(() => self.skipWaiting()) // 立即激活
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.delete('v1')               // 删除旧缓存
      .then(() => self.clients.claim()) // 立即控制
  );
});

// 风险：页面已加载了 v1 的 CSS/JS，此时 v2 的 SW 开始控制
// 后续请求获取的是 v2 资源，可能导致不兼容

// 安全做法：通知用户刷新
// 客户端
navigator.serviceWorker.addEventListener('controllerchange', () => {
  if (confirm('应用已更新，是否刷新页面？')) {
    window.location.reload();
  }
});
```

---

**Q4：如何使用 Fetch API 实现请求重试和指数退避？**

```javascript
async function fetchWithRetry(url, options = {}, retryConfig = {}) {
  const {
    maxRetries = 3,
    baseDelay = 1000,
    maxDelay = 10000,
    retryOn = [500, 502, 503, 504], // 服务端错误才重试
    backoffFactor = 2,
  } = retryConfig;

  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);

      if (response.ok || !retryOn.includes(response.status)) {
        return response;
      }

      lastError = new Error(`HTTP ${response.status}`);
    } catch (err) {
      // 网络错误
      if (err.name === 'AbortError') throw err; // 取消不重试
      lastError = err;
    }

    if (attempt < maxRetries) {
      // 指数退避 + 随机抖动
      const delay = Math.min(
        baseDelay * Math.pow(backoffFactor, attempt) + Math.random() * 500,
        maxDelay
      );
      console.log(`请求失败，${delay.toFixed(0)}ms 后重试（第 ${attempt + 1} 次）`);
      await new Promise(r => setTimeout(r, delay));
    }
  }

  throw lastError;
}

// 使用
const data = await fetchWithRetry('/api/data', {
  headers: { 'Content-Type': 'application/json' },
}, { maxRetries: 3 });
```

---

**Q5：解释 IntersectionObserver 的 rootMargin 和 threshold 参数，给出实际应用。**

```javascript
// rootMargin: 扩展或收缩 root 边界
// "100px 0px" → 上下各扩展 100px（提前加载）
// "-50px"     → 收缩 50px（元素需要更深入视口才触发）
// 支持 px 和 %，格式类似 CSS margin

// threshold: 触发回调的可见比例节点
// 0    → 刚进入或完全离开时触发
// 1    → 完全可见或开始离开时触发
// [0, 0.5, 1] → 在 0%、50%、100% 可见时各触发一次

// 实际应用：视频自动播放/暂停
const videoObserver = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      const video = entry.target;
      if (entry.isIntersecting && entry.intersectionRatio >= 0.75) {
        video.play();
      } else {
        video.pause();
      }
    });
  },
  {
    threshold: [0, 0.75],
    rootMargin: '0px',
  }
);

document.querySelectorAll('video[data-autoplay]').forEach(v => videoObserver.observe(v));
```

---

**Q6：Canvas 在高 DPR 屏幕上为什么会模糊？如何解决？**

```javascript
// 原因：Canvas 的内部像素和 CSS 像素 1:1 映射
// 在 2x DPR 屏幕上，1 CSS 像素 = 4 物理像素
// Canvas 的 1 像素被拉伸为 4 像素 → 模糊

// 解决方案
function createHiDPICanvas(width, height) {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');
  const dpr = window.devicePixelRatio || 1;

  // 画布实际大小 = CSS 大小 × DPR
  canvas.width = width * dpr;
  canvas.height = height * dpr;

  // CSS 大小保持不变
  canvas.style.width = `${width}px`;
  canvas.style.height = `${height}px`;

  // 缩放绘图上下文，这样后续代码用 CSS 坐标即可
  ctx.scale(dpr, dpr);

  return { canvas, ctx };
}

// OffscreenCanvas 方案（在 Worker 中使用）
// 主线程
const offscreen = canvas.transferControlToOffscreen();
worker.postMessage({ canvas: offscreen }, [offscreen]);

// Worker 中
self.addEventListener('message', (e) => {
  const canvas = e.data.canvas;
  const ctx = canvas.getContext('2d');
  // 在 Worker 中绘制，不阻塞主线程
});
```

---

**Q7：WebRTC 的 ICE（Interactive Connectivity Establishment）流程是什么？**

```
1. Gathering 阶段：
   - 收集本地候选（host candidate）：本机 IP
   - 通过 STUN 获取反射候选（srflx）：NAT 后的公网 IP
   - 通过 TURN 获取中继候选（relay）：中继服务器地址

2. Connectivity Check：
   - 两端交换 ICE 候选（通过信令服务器）
   - 按优先级逐对尝试连接（STUN Binding Request）
   - 找到可用的候选对后建立连接

3. 优先级：host > srflx > relay
   - 尽量使用直连（低延迟）
   - NAT 穿越失败才使用 TURN 中继（高延迟但保证连通）
```

```javascript
// 监控 ICE 状态
pc.oniceconnectionstatechange = () => {
  console.log('ICE 连接状态:', pc.iceConnectionState);
  // new → checking → connected → completed → disconnected → closed
};

pc.onicegatheringstatechange = () => {
  console.log('ICE 收集状态:', pc.iceGatheringState);
  // new → gathering → complete
};

// 获取连接统计
async function getConnectionStats() {
  const stats = await pc.getStats();
  stats.forEach(report => {
    if (report.type === 'candidate-pair' && report.state === 'succeeded') {
      console.log('候选对:', report);
      console.log('往返时间:', report.currentRoundTripTime, 's');
    }
  });
}
```

---

**Q8：如何使用 Fetch 的 ReadableStream 实现 AI 对话的流式输出？**

```javascript
async function streamChat(messages, onToken) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ messages, stream: true }),
  });

  if (!response.ok) throw new Error(`HTTP ${response.status}`);

  const reader = response.body
    .pipeThrough(new TextDecoderStream())
    .getReader();

  let buffer = '';
  let fullText = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += value;

    // 按行解析 SSE 格式
    while (buffer.includes('\n')) {
      const newlineIdx = buffer.indexOf('\n');
      const line = buffer.slice(0, newlineIdx).trim();
      buffer = buffer.slice(newlineIdx + 1);

      if (!line || !line.startsWith('data: ')) continue;
      const data = line.slice(6);
      if (data === '[DONE]') return fullText;

      try {
        const parsed = JSON.parse(data);
        const token = parsed.choices?.[0]?.delta?.content || '';
        if (token) {
          fullText += token;
          onToken(token, fullText);
        }
      } catch {
        // 忽略解析错误
      }
    }
  }

  return fullText;
}

// 使用
const chatContainer = document.getElementById('chat');
await streamChat(
  [{ role: 'user', content: '解释什么是闭包' }],
  (token, fullText) => {
    chatContainer.textContent = fullText; // 逐字输出
  }
);
```

---

**Q9：IndexedDB 事务模型是怎样的？与 SQL 数据库事务有什么区别？**

```javascript
// IndexedDB 使用自动提交事务模型：
// 1. 事务在创建时指定涉及的 store 和模式（readonly/readwrite）
// 2. 所有操作必须在事务的回调中完成
// 3. 事务自动提交（当所有请求完成且没有新请求时）
// 4. 不支持手动 commit/rollback（只能 abort）

const db = event.target.result;

// readwrite 事务（同一 store 同时只能有一个）
const tx = db.transaction(['users', 'orders'], 'readwrite');
const userStore = tx.objectStore('users');
const orderStore = tx.objectStore('orders');

// 事务中的多步操作
userStore.put({ id: 1, name: 'Alice', balance: 900 });
orderStore.add({ id: 101, userId: 1, amount: 100 });

// ⚠️ 事务中不能有异步等待（会导致事务自动提交）
// ❌ 这样写事务可能已经关闭
// await someAsyncOperation();
// userStore.put(...); // TransactionInactiveError

tx.oncomplete = () => console.log('事务提交成功');
tx.onerror = (e) => console.error('事务失败:', e.target.error);
tx.onabort = () => console.log('事务被中止');

// 手动回滚
// tx.abort();

// 与 SQL 数据库事务的区别：
// 1. IndexedDB 不支持跨 store 的 JOIN
// 2. 没有显式的 BEGIN/COMMIT/ROLLBACK
// 3. 基于事件驱动而非同步阻塞
// 4. 隔离级别固定（readwrite 串行，readonly 可并行）
```

---

**Q10：如何判断用户处于离线状态？离线优先应用的设计要点？**

```javascript
// ============ 检测网络状态 ============
// 1. navigator.onLine（不完全可靠）
console.log('在线:', navigator.onLine);

window.addEventListener('online', () => {
  console.log('网络恢复');
  syncPendingData(); // 同步待上传数据
});

window.addEventListener('offline', () => {
  console.log('网络断开');
  showOfflineBanner();
});

// 2. 更可靠的检测：实际请求
async function checkConnection() {
  try {
    const res = await fetch('/api/health', {
      method: 'HEAD',
      cache: 'no-store',
    });
    return res.ok;
  } catch {
    return false;
  }
}

// ============ 离线优先设计要点 ============
class OfflineFirstApp {
  constructor() {
    this.db = new SimpleDB('offline-app', 1);
    this.syncQueue = []; // 待同步的操作
  }

  // 写操作先写本地，再同步
  async createItem(item) {
    // 1. 立即写入 IndexedDB
    item._syncStatus = 'pending';
    item._createdAt = Date.now();
    await this.db.put('items', item);

    // 2. 尝试同步到服务器
    try {
      const response = await fetch('/api/items', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(item),
      });

      if (response.ok) {
        const serverItem = await response.json();
        serverItem._syncStatus = 'synced';
        await this.db.put('items', serverItem);
        return serverItem;
      }
    } catch {
      // 离线：加入同步队列
      this.syncQueue.push({
        method: 'POST',
        url: '/api/items',
        body: item,
      });
    }

    return item; // 返回本地版本
  }

  // 读操作优先本地
  async getItems() {
    const localItems = await this.db.getAll('items');
    // 后台尝试更新
    this._backgroundSync().catch(() => {});
    return localItems;
  }

  async _backgroundSync() {
    while (this.syncQueue.length > 0) {
      const op = this.syncQueue[0];
      const res = await fetch(op.url, {
        method: op.method,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(op.body),
      });
      if (res.ok) {
        this.syncQueue.shift();
      } else {
        break; // 下次再试
      }
    }
  }
}
```

---

**Q11：AbortSignal.any() 的使用场景是什么？如何实现一个 polyfill？**

```javascript
// AbortSignal.any() 组合多个 signal，任一触发即中止
// 场景：同时支持用户取消和超时取消

async function fetchWithUserCancelAndTimeout(url, userSignal) {
  const timeoutController = new AbortController();
  const timeoutId = setTimeout(() => timeoutController.abort(), 5000);

  try {
    return await fetch(url, {
      signal: AbortSignal.any([userSignal, timeoutController.signal]),
    });
  } finally {
    clearTimeout(timeoutId);
  }
}

// AbortSignal.timeout()（更简洁的超时）
fetch(url, { signal: AbortSignal.timeout(5000) });

// Polyfill（简易版）
function abortSignalAny(signals) {
  const controller = new AbortController();

  for (const signal of signals) {
    if (signal.aborted) {
      controller.abort(signal.reason);
      return controller.signal;
    }
    signal.addEventListener('abort', () => {
      controller.abort(signal.reason);
    }, { once: true });
  }

  return controller.signal;
}
```

---

**Q12：Shared Worker 为什么没有被广泛使用？有什么替代方案？**

> Shared Worker 的局限性：
> - 调试困难（Chrome 需要 `chrome://inspect/#workers`）
> - 部分浏览器不支持（Safari 近年才支持）
> - 生命周期管理复杂
> - 不如 BroadcastChannel / localStorage 事件简单

```javascript
// 替代方案对比

// 1. BroadcastChannel —— 最简单的跨标签页通信
const bc = new BroadcastChannel('my-channel');
bc.postMessage({ type: 'data-update', payload: { id: 1 } });
bc.onmessage = (e) => console.log(e.data);

// 2. Service Worker + MessageChannel
// Service Worker 可以充当共享状态中心
navigator.serviceWorker.controller.postMessage({
  type: 'SHARE_STATE',
  data: sharedData,
});

// 3. window.postMessage（跨域场景）
window.opener?.postMessage(data, 'https://target-origin.com');

// 4. SharedArrayBuffer + Atomics（真正的共享内存）
// 需要跨域隔离头：Cross-Origin-Opener-Policy / Cross-Origin-Embedder-Policy
const sab = new SharedArrayBuffer(1024);
const view = new Int32Array(sab);
Atomics.store(view, 0, 42);
Atomics.load(view, 0); // 42
```

---

**Q13：Canvas 动画中 requestAnimationFrame 和 setInterval 有什么区别？**

```javascript
// setInterval 的问题：
// 1. 固定间隔，不与屏幕刷新率同步
// 2. 标签页后台时仍在运行（浪费资源）
// 3. 可能出现掉帧或过度绘制

// requestAnimationFrame 的优势：
// 1. 与屏幕刷新率同步（通常 60fps）
// 2. 标签页不可见时自动暂停
// 3. 浏览器可以优化合并绘制
// 4. 回调参数提供高精度时间戳

// 时间基础的动画（帧率无关）
let lastTime = 0;

function animate(timestamp) {
  const deltaTime = timestamp - lastTime;
  lastTime = timestamp;

  // 基于 deltaTime 更新，而非固定步长
  // 这样无论 60fps 还是 144fps，动画速度一致
  const speed = 200; // 200px/秒
  object.x += speed * (deltaTime / 1000);

  render();
  requestAnimationFrame(animate);
}

requestAnimationFrame(animate);

// 固定时间步长 + 插值（物理模拟最佳实践）
const FIXED_STEP = 1000 / 60; // 16.67ms
let accumulator = 0;
let previousState = null;

function gameLoop(timestamp) {
  const dt = timestamp - lastTime;
  lastTime = timestamp;
  accumulator += dt;

  // 固定步长更新物理
  while (accumulator >= FIXED_STEP) {
    previousState = { ...state };
    updatePhysics(FIXED_STEP / 1000);
    accumulator -= FIXED_STEP;
  }

  // 插值渲染（平滑）
  const alpha = accumulator / FIXED_STEP;
  renderInterpolated(previousState, state, alpha);

  requestAnimationFrame(gameLoop);
}
```

---

**Q14：Web Animations API 的 composite 属性有什么作用？**

```javascript
// composite 控制多个动画效果如何叠加

// 'replace'（默认）—— 完全替换
element.animate([{ transform: 'translateX(100px)' }], { duration: 1000 });
element.animate([{ transform: 'translateY(50px)' }], { duration: 1000 });
// 结果：只有 translateY（后者替换前者）

// 'add' —— 叠加
element.animate(
  [{ transform: 'translateX(100px)' }],
  { duration: 1000, composite: 'add' }
);
element.animate(
  [{ transform: 'translateY(50px)' }],
  { duration: 1000, composite: 'add' }
);
// 结果：同时 translateX + translateY

// 'accumulate' —— 累积（数值相加）
element.animate(
  [{ transform: 'translateX(100px)' }],
  { duration: 1000, composite: 'accumulate' }
);
element.animate(
  [{ transform: 'translateX(50px)' }],
  { duration: 1000, composite: 'accumulate' }
);
// 结果：translateX(150px)

// 实际应用：鼠标悬停附加额外动画
element.addEventListener('mouseenter', () => {
  element.animate(
    [{ transform: 'scale(1.1)' }],
    {
      duration: 200,
      fill: 'forwards',
      composite: 'add', // 不覆盖正在进行的其他动画
    }
  );
});
```

---

**Q15：如何实现一个完整的图片懒加载方案，包括 loading="lazy" 和 IntersectionObserver 的降级？**

```javascript
// ============ 完整的图片懒加载方案 ============

class ImageLazyLoader {
  constructor(options = {}) {
    this.rootMargin = options.rootMargin || '200px';
    this.loaded = new WeakSet();
    this.observer = null;

    // 检测原生支持
    this.nativeLazySupported = 'loading' in HTMLImageElement.prototype;
  }

  init(selector = 'img[data-src]') {
    const images = document.querySelectorAll(selector);

    if (this.nativeLazySupported && !this._needsCustomBehavior()) {
      // 原生 lazy loading
      images.forEach(img => {
        img.loading = 'lazy';
        img.src = img.dataset.src;
        if (img.dataset.srcset) img.srcset = img.dataset.srcset;
      });
    } else if ('IntersectionObserver' in window) {
      // IntersectionObserver 方案
      this.observer = new IntersectionObserver(
        (entries) => this._handleEntries(entries),
        { rootMargin: this.rootMargin, threshold: 0 }
      );
      images.forEach(img => this.observer.observe(img));
    } else {
      // 最终降级：scroll 事件
      this._fallbackScrollListener(images);
    }
  }

  _needsCustomBehavior() {
    // 如果需要自定义 rootMargin 或回调，不使用原生方案
    return this.rootMargin !== '200px';
  }

  _handleEntries(entries) {
    entries.forEach(entry => {
      if (!entry.isIntersecting) return;
      this._loadImage(entry.target);
      this.observer.unobserve(entry.target);
    });
  }

  _loadImage(img) {
    if (this.loaded.has(img)) return;

    // 预加载：用 Image 对象预加载，完成后替换
    const preloader = new Image();
    preloader.onload = () => {
      img.src = img.dataset.src;
      if (img.dataset.srcset) img.srcset = img.dataset.srcset;
      img.classList.add('loaded');
      img.removeAttribute('data-src');
      this.loaded.add(img);
    };
    preloader.onerror = () => {
      img.classList.add('error');
      console.warn('图片加载失败:', img.dataset.src);
    };
    preloader.src = img.dataset.src;
  }

  _fallbackScrollListener(images) {
    const check = () => {
      images.forEach(img => {
        if (this.loaded.has(img)) return;
        const rect = img.getBoundingClientRect();
        if (rect.top < window.innerHeight + 200 && rect.bottom > -200) {
          this._loadImage(img);
        }
      });
    };

    // 节流
    let ticking = false;
    const onScroll = () => {
      if (!ticking) {
        requestAnimationFrame(() => {
          check();
          ticking = false;
        });
        ticking = true;
      }
    };

    window.addEventListener('scroll', onScroll, { passive: true });
    window.addEventListener('resize', onScroll, { passive: true });
    check(); // 初始检查
  }

  destroy() {
    this.observer?.disconnect();
  }
}

// 使用
const lazyLoader = new ImageLazyLoader({ rootMargin: '300px' });
lazyLoader.init('img[data-src]');
```

---

**Q16：WebSocket 和 Server-Sent Events（SSE）如何选择？**

```javascript
// 对比
// WebSocket: 全双工，客户端和服务端都能发送
// SSE: 单向（服务端 → 客户端），基于 HTTP，自动重连

// SSE 适合：通知推送、股票行情、日志流
// WebSocket 适合：聊天室、协作编辑、游戏

// SSE 使用（EventSource）
const source = new EventSource('/api/events');

source.addEventListener('message', (e) => {
  console.log('消息:', e.data);
});

source.addEventListener('notification', (e) => {
  // 自定义事件类型
  const data = JSON.parse(e.data);
  showNotification(data);
});

source.onerror = () => {
  console.log('SSE 连接断开，自动重连...');
  // EventSource 会自动重连
};

// 服务端（Node.js）
// res.writeHead(200, {
//   'Content-Type': 'text/event-stream',
//   'Cache-Control': 'no-cache',
//   'Connection': 'keep-alive',
// });
// res.write(`event: notification\ndata: ${JSON.stringify(data)}\n\n`);
```

---

**Q17：如何用 Web Worker + OffscreenCanvas 实现不卡顿的 Canvas 渲染？**

```javascript
// ============ OffscreenCanvas 在 Worker 中渲染 ============

// main.js
const canvas = document.getElementById('myCanvas');
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker('./render-worker.js');
worker.postMessage({ canvas: offscreen, width: 800, height: 600 }, [offscreen]);

// 主线程完全不参与渲染，UI 操作不受影响
worker.postMessage({ type: 'updateData', data: bigDataSet });

// render-worker.js
let canvas, ctx;

self.addEventListener('message', (e) => {
  if (e.data.canvas) {
    canvas = e.data.canvas;
    ctx = canvas.getContext('2d');
    canvas.width = e.data.width;
    canvas.height = e.data.height;
    startRenderLoop();
  }

  if (e.data.type === 'updateData') {
    currentData = e.data.data;
  }
});

let currentData = [];

function startRenderLoop() {
  function render() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // 在 Worker 中执行所有绘制操作
    currentData.forEach(item => {
      ctx.beginPath();
      ctx.arc(item.x, item.y, item.radius, 0, Math.PI * 2);
      ctx.fillStyle = item.color;
      ctx.fill();
    });

    // Worker 中也可以使用 requestAnimationFrame
    requestAnimationFrame(render);
  }

  render();
}
```

---

**Q18：Notification API 在不同平台上的表现差异？如何处理权限？**

```javascript
// 权限状态
// 'default' —— 用户还未选择
// 'granted' —— 已授权
// 'denied'  —— 已拒绝（无法再次请求）

async function ensureNotificationPermission() {
  if (!('Notification' in window)) {
    return { supported: false };
  }

  const permission = Notification.permission;

  if (permission === 'granted') {
    return { supported: true, granted: true };
  }

  if (permission === 'denied') {
    // 用户已拒绝，无法程序化重新请求
    // 需引导用户到浏览器设置手动开启
    return { supported: true, granted: false, canAsk: false };
  }

  // 'default' —— 可以请求
  // 最佳实践：在用户操作（如点击按钮）后请求，不要在页面加载时直接请求
  return { supported: true, granted: false, canAsk: true };
}

// 平台差异：
// - macOS：显示在通知中心，支持 actions
// - Windows：显示为 Toast，支持 actions
// - Android Chrome：通过 Service Worker 的 showNotification
// - iOS Safari：PWA 模式下支持（iOS 16.4+），普通网页不支持
// - 部分浏览器在 HTTP 下禁用（需 HTTPS）
```

---

**Q19：如何检测浏览器对各种 Web API 的支持，实现优雅降级？**

```javascript
// ============ 特性检测工具 ============
const features = {
  serviceWorker: 'serviceWorker' in navigator,
  webWorker: typeof Worker !== 'undefined',
  sharedWorker: typeof SharedWorker !== 'undefined',
  intersectionObserver: 'IntersectionObserver' in window,
  resizeObserver: 'ResizeObserver' in window,
  mutationObserver: 'MutationObserver' in window,
  fetch: 'fetch' in window,
  abortController: 'AbortController' in window,
  webSocket: 'WebSocket' in window,
  indexedDB: 'indexedDB' in window,
  notification: 'Notification' in window,
  clipboard: navigator.clipboard !== undefined,
  webRTC: 'RTCPeerConnection' in window,
  webGL: (() => {
    try {
      return !!document.createElement('canvas').getContext('webgl2');
    } catch { return false; }
  })(),
  offscreenCanvas: 'OffscreenCanvas' in window,
  webAnimations: 'animate' in HTMLElement.prototype,
  broadcastChannel: 'BroadcastChannel' in window,
  storageEstimate: navigator.storage?.estimate !== undefined,
};

console.table(features);

// 存储容量估算
async function estimateStorage() {
  if (navigator.storage?.estimate) {
    const { usage, quota } = await navigator.storage.estimate();
    return {
      usedMB: (usage / 1024 / 1024).toFixed(2),
      quotaMB: (quota / 1024 / 1024).toFixed(2),
      percentage: ((usage / quota) * 100).toFixed(2) + '%',
    };
  }
  return null;
}
```

---

**Q20：File System Access API 和传统 File API 的区别？实际应用场景？**

```javascript
// 传统 File API：只能通过 <input type="file"> 或拖拽获取文件
// File System Access API：可以直接打开文件/目录、保存文件

// ============ 简易文本编辑器 ============
class SimpleEditor {
  constructor(textareaId) {
    this.textarea = document.getElementById(textareaId);
    this.fileHandle = null;
  }

  async open() {
    try {
      [this.fileHandle] = await window.showOpenFilePicker({
        types: [{
          description: 'Text Files',
          accept: { 'text/plain': ['.txt', '.md', '.js', '.ts'] },
        }],
      });

      const file = await this.fileHandle.getFile();
      this.textarea.value = await file.text();
      document.title = file.name;
    } catch (err) {
      if (err.name !== 'AbortError') throw err;
    }
  }

  async save() {
    if (!this.fileHandle) return this.saveAs();

    const writable = await this.fileHandle.createWritable();
    await writable.write(this.textarea.value);
    await writable.close();
  }

  async saveAs() {
    try {
      this.fileHandle = await window.showSaveFilePicker({
        suggestedName: 'untitled.txt',
        types: [{
          description: 'Text Files',
          accept: { 'text/plain': ['.txt'] },
        }],
      });
      await this.save();
    } catch (err) {
      if (err.name !== 'AbortError') throw err;
    }
  }

  // 读取整个目录
  async openDirectory() {
    const dirHandle = await window.showDirectoryPicker();
    const files = [];

    for await (const [name, handle] of dirHandle) {
      if (handle.kind === 'file') {
        const file = await handle.getFile();
        files.push({ name, size: file.size, type: file.type, handle });
      }
    }

    return files;
  }
}

// 注意：File System Access API 目前仅 Chromium 系浏览器支持
// Firefox / Safari 可降级使用传统 File API + Blob 下载
```

---

> **复习建议**：
> 1. 每个 API 都要动手写 Demo，而非只看代码
> 2. 重点掌握 Fetch + AbortController、Web Worker、Service Worker 的实战应用
> 3. 理解各 API 的浏览器兼容性，能给出降级方案
> 4. 面试时能画出 Service Worker 生命周期图、WebRTC 信令流程图

### 开放式设计题

**D1：设计一个支持离线使用的PWA应用（如笔记应用），数据同步策略怎么做？**

**参考思路**：
- 离线存储：IndexedDB存笔记内容（结构化数据）、Cache API缓存静态资源
- Service Worker：Stale-While-Revalidate策略（先返回缓存再后台更新）
- 冲突解决：乐观锁（版本号）+ Last-Write-Wins或CRDTs（无冲突复制数据类型）
- 同步时机：navigator.onLine检测网络恢复 → Background Sync API排队同步 → 失败重试
- 用户体验：离线状态提示、同步进度指示、冲突手动合并UI

**D2：Web Worker中进行大量计算导致主线程Transferable Object传输开销大，如何优化？**

**参考思路**：
- Transferable Objects：ArrayBuffer/MessagePort/OffscreenCanvas的所有权转移（零拷贝）
- SharedArrayBuffer + Atomics：多Worker共享内存，无需传输，但需要COOP/COEP Headers
- 计算分片：将大任务拆成多个小任务，每帧只传输增量结果
- Worker池化：复用Worker实例避免创建销毁开销，任务队列分发
- 方案选择：数据量<1MB用结构化克隆、>1MB用Transferable、需要频繁读写用SharedArrayBuffer
