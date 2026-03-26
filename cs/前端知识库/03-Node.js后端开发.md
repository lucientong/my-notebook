# Node.js后端开发

---

## 📑 目录

1. [Node.js基础](#1-nodejs基础)
2. [Express框架](#2-express框架)
3. [Koa框架](#3-koa框架)
4. [中间件原理](#4-中间件原理)
5. [数据库集成](#5-数据库集成)
6. [身份认证](#6-身份认证)
7. [文件上传](#7-文件上传)
8. [WebSocket实时通信](#8-websocket实时通信)
9. [性能优化](#9-性能优化)
10. [常见面试题](#10-常见面试题)
11. [实战案例](#11-实战案例)

---

## 1. Node.js基础

### 1.1 Node.js特点

**单线程、异步I/O、事件驱动**

**优势**：
- ✅ 高并发（非阻塞I/O）
- ✅ JavaScript全栈
- ✅ NPM生态丰富

**劣势**：
- ❌ CPU密集型任务性能差
- ❌ 单线程，一个错误崩溃整个进程

### 1.2 Event Loop（事件循环）

```
┌───────────────────────────┐
│        timers             │  setTimeout/setInterval
├───────────────────────────┤
│     pending callbacks     │  I/O回调
├───────────────────────────┤
│       idle, prepare       │  内部使用
├───────────────────────────┤
│         poll              │  轮询（等待I/O）
├───────────────────────────┤
│        check              │  setImmediate
├───────────────────────────┤
│    close callbacks        │  关闭回调
└───────────────────────────┘
```

**执行顺序**：
```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

setImmediate(() => console.log('3'));

Promise.resolve().then(() => console.log('4'));

console.log('5');

// 输出：1 5 4 2 3
```

### 1.3 模块系统

**CommonJS**：
```javascript
// math.js
module.exports = {
  add: (a, b) => a + b,
};

// app.js
const math = require('./math');
console.log(math.add(1, 2));
```

**ES Module**：
```javascript
// math.js
export const add = (a, b) => a + b;

// app.js
import { add } from './math.js';
console.log(add(1, 2));
```

---

## 2. Express框架

### 2.1 快速开始

```javascript
const express = require('express');
const app = express();

// 中间件
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// 路由
app.get('/', (req, res) => {
  res.send('Hello World');
});

app.post('/api/users', (req, res) => {
  const { name, age } = req.body;
  res.json({ id: 1, name, age });
});

// 启动服务器
app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### 2.2 路由

**基本路由**：
```javascript
app.get('/users', (req, res) => {
  res.json([{ id: 1, name: '张三' }]);
});

app.get('/users/:id', (req, res) => {
  const { id } = req.params;
  res.json({ id, name: '张三' });
});

app.post('/users', (req, res) => {
  const { name } = req.body;
  res.status(201).json({ id: 1, name });
});
```

**路由分组**：
```javascript
const express = require('express');
const router = express.Router();

// /api/users
router.get('/', (req, res) => {
  res.json([]);
});

router.get('/:id', (req, res) => {
  res.json({ id: req.params.id });
});

// 挂载路由
app.use('/api/users', router);
```

### 2.3 中间件

**应用级中间件**：
```javascript
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();  // 必须调用next()
});
```

**路由级中间件**：
```javascript
const auth = (req, res, next) => {
  const token = req.headers.authorization;
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  req.user = verifyToken(token);
  next();
};

app.get('/api/profile', auth, (req, res) => {
  res.json(req.user);
});
```

**错误处理中间件**：
```javascript
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Internal Server Error' });
});
```

### 2.4 请求与响应

**请求对象（req）**：
```javascript
req.params     // 路径参数 /users/:id
req.query      // 查询参数 ?name=张三
req.body       // 请求体（需要body-parser）
req.headers    // 请求头
req.cookies    // Cookies（需要cookie-parser）
```

**响应对象（res）**：
```javascript
res.send('Hello')               // 发送文本
res.json({ name: '张三' })      // 发送JSON
res.status(404).send('Not Found')  // 设置状态码
res.redirect('/login')          // 重定向
res.download('/path/to/file')   // 下载文件
```

---

## 3. Koa框架

### 3.1 基本使用

```javascript
const Koa = require('koa');
const Router = require('@koa/router');
const bodyParser = require('koa-bodyparser');

const app = new Koa();
const router = new Router();

// 中间件
app.use(bodyParser());

// 路由
router.get('/', async (ctx) => {
  ctx.body = 'Hello World';
});

router.post('/api/users', async (ctx) => {
  const { name, age } = ctx.request.body;
  ctx.body = { id: 1, name, age };
});

app.use(router.routes());

// 启动服务器
app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### 3.2 Context对象

```javascript
app.use(async (ctx) => {
  ctx.request   // Koa Request对象
  ctx.response  // Koa Response对象
  ctx.req       // Node.js原生request
  ctx.res       // Node.js原生response
  
  // 快捷方式
  ctx.url       // 请求URL
  ctx.method    // 请求方法
  ctx.query     // 查询参数
  ctx.params    // 路径参数
  ctx.body      // 响应体
  ctx.status    // 状态码
});
```

### 3.3 洋葱模型

**中间件执行顺序**：
```javascript
app.use(async (ctx, next) => {
  console.log('1 - 开始');
  await next();
  console.log('1 - 结束');
});

app.use(async (ctx, next) => {
  console.log('2 - 开始');
  await next();
  console.log('2 - 结束');
});

app.use(async (ctx) => {
  console.log('3 - 核心');
  ctx.body = 'Hello';
});

// 输出：
// 1 - 开始
// 2 - 开始
// 3 - 核心
// 2 - 结束
// 1 - 结束
```

---

## 4. 中间件原理

### 4.1 Express中间件原理

```javascript
class Express {
  constructor() {
    this.middlewares = [];
  }
  
  use(fn) {
    this.middlewares.push(fn);
  }
  
  handle(req, res) {
    let index = 0;
    
    const next = () => {
      if (index >= this.middlewares.length) return;
      const middleware = this.middlewares[index++];
      middleware(req, res, next);
    };
    
    next();
  }
}

// 使用
const app = new Express();
app.use((req, res, next) => {
  console.log('中间件1');
  next();
});
app.use((req, res, next) => {
  console.log('中间件2');
  res.send('Hello');
});
```

### 4.2 Koa洋葱模型原理

```javascript
function compose(middlewares) {
  return (ctx) => {
    let index = -1;
    
    function dispatch(i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'));
      index = i;
      
      if (i >= middlewares.length) return Promise.resolve();
      
      const middleware = middlewares[i];
      try {
        return Promise.resolve(middleware(ctx, () => dispatch(i + 1)));
      } catch (err) {
        return Promise.reject(err);
      }
    }
    
    return dispatch(0);
  };
}

// 使用
const middlewares = [
  async (ctx, next) => {
    console.log('1');
    await next();
    console.log('6');
  },
  async (ctx, next) => {
    console.log('2');
    await next();
    console.log('5');
  },
  async (ctx) => {
    console.log('3');
    ctx.body = 'Hello';
    console.log('4');
  },
];

const fn = compose(middlewares);
fn({});  // 1 2 3 4 5 6
```

---

## 5. 数据库集成

### 5.1 MySQL（mysql2）

```javascript
const mysql = require('mysql2/promise');

// 创建连接池
const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  password: 'password',
  database: 'mydb',
  waitForConnections: true,
  connectionLimit: 10,
});

// 查询
app.get('/api/users', async (req, res) => {
  const [rows] = await pool.query('SELECT * FROM users');
  res.json(rows);
});

// 插入
app.post('/api/users', async (req, res) => {
  const { name, age } = req.body;
  const [result] = await pool.query(
    'INSERT INTO users (name, age) VALUES (?, ?)',
    [name, age]
  );
  res.json({ id: result.insertId, name, age });
});
```

### 5.2 MongoDB（mongoose）

```javascript
const mongoose = require('mongoose');

// 连接数据库
mongoose.connect('mongodb://localhost:27017/mydb');

// 定义Schema
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  age: { type: Number, min: 0 },
  email: { type: String, unique: true },
  createdAt: { type: Date, default: Date.now },
});

// 创建Model
const User = mongoose.model('User', userSchema);

// 查询
app.get('/api/users', async (req, res) => {
  const users = await User.find();
  res.json(users);
});

// 插入
app.post('/api/users', async (req, res) => {
  const user = new User(req.body);
  await user.save();
  res.json(user);
});

// 更新
app.put('/api/users/:id', async (req, res) => {
  const user = await User.findByIdAndUpdate(req.params.id, req.body, { new: true });
  res.json(user);
});

// 删除
app.delete('/api/users/:id', async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.status(204).send();
});
```

### 5.3 Redis

```javascript
const redis = require('redis');
const client = redis.createClient();

await client.connect();

// 设置
await client.set('key', 'value');

// 获取
const value = await client.get('key');

// 设置过期时间
await client.setEx('key', 60, 'value');  // 60秒后过期

// 哈希
await client.hSet('user:1001', { name: '张三', age: 25 });
const user = await client.hGetAll('user:1001');

// 列表
await client.lPush('queue', 'task1');
const task = await client.rPop('queue');
```

---

## 6. 身份认证

### 6.1 JWT认证

```javascript
const jwt = require('jsonwebtoken');
const SECRET = 'your-secret-key';

// 登录：生成Token
app.post('/api/login', async (req, res) => {
  const { username, password } = req.body;
  
  // 验证用户名密码
  const user = await User.findOne({ username, password });
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // 生成Token
  const token = jwt.sign({ id: user.id, username: user.username }, SECRET, {
    expiresIn: '7d',
  });
  
  res.json({ token });
});

// 认证中间件
const auth = (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  try {
    const decoded = jwt.verify(token, SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// 受保护的路由
app.get('/api/profile', auth, (req, res) => {
  res.json(req.user);
});
```

### 6.2 Session认证

```javascript
const session = require('express-session');

app.use(session({
  secret: 'your-secret-key',
  resave: false,
  saveUninitialized: false,
  cookie: { maxAge: 60000 },  // 60秒
}));

// 登录
app.post('/api/login', async (req, res) => {
  const { username, password } = req.body;
  const user = await User.findOne({ username, password });
  
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // 保存到session
  req.session.user = { id: user.id, username: user.username };
  res.json({ message: 'Login success' });
});

// 认证中间件
const auth = (req, res, next) => {
  if (!req.session.user) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
};

// 登出
app.post('/api/logout', (req, res) => {
  req.session.destroy();
  res.json({ message: 'Logout success' });
});
```

---

## 7. 文件上传

### 7.1 Multer

```javascript
const multer = require('multer');
const path = require('path');

// 配置存储
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/');
  },
  filename: (req, file, cb) => {
    const uniqueName = Date.now() + path.extname(file.originalname);
    cb(null, uniqueName);
  },
});

// 文件过滤
const fileFilter = (req, file, cb) => {
  const allowedTypes = ['image/jpeg', 'image/png'];
  if (allowedTypes.includes(file.mimetype)) {
    cb(null, true);
  } else {
    cb(new Error('Invalid file type'));
  }
};

const upload = multer({
  storage,
  fileFilter,
  limits: { fileSize: 5 * 1024 * 1024 },  // 5MB
});

// 单文件上传
app.post('/api/upload', upload.single('file'), (req, res) => {
  res.json({ filename: req.file.filename });
});

// 多文件上传
app.post('/api/upload-multiple', upload.array('files', 10), (req, res) => {
  const filenames = req.files.map(f => f.filename);
  res.json({ filenames });
});
```

---

## 8. WebSocket实时通信

### 8.1 使用ws库

```javascript
const express = require('express');
const { WebSocketServer } = require('ws');

const app = express();
const server = app.listen(3000);

// 创建WebSocket服务器
const wss = new WebSocketServer({ server });

wss.on('connection', (ws) => {
  console.log('客户端连接');
  
  ws.on('message', (data) => {
    console.log('收到消息:', data);
    
    // 回显消息
    ws.send(`服务器收到: ${data}`);
    
    // 广播给所有客户端
    wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(data);
      }
    });
  });
  
  ws.on('close', () => {
    console.log('客户端断开');
  });
});
```

### 8.2 Socket.io

```javascript
const express = require('express');
const { Server } = require('socket.io');

const app = express();
const server = app.listen(3000);
const io = new Server(server);

io.on('connection', (socket) => {
  console.log('客户端连接:', socket.id);
  
  // 监听事件
  socket.on('chat message', (msg) => {
    console.log('消息:', msg);
    
    // 广播给所有客户端
    io.emit('chat message', msg);
  });
  
  // 加入房间
  socket.on('join room', (room) => {
    socket.join(room);
    socket.to(room).emit('user joined', socket.id);
  });
  
  // 房间内广播
  socket.on('room message', (room, msg) => {
    io.to(room).emit('room message', msg);
  });
  
  socket.on('disconnect', () => {
    console.log('客户端断开:', socket.id);
  });
});
```

---

## 9. 性能优化

### 9.1 集群（Cluster）

```javascript
const cluster = require('cluster');
const os = require('os');
const express = require('express');

if (cluster.isMaster) {
  // 主进程：创建工作进程
  const numCPUs = os.cpus().length;
  
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker) => {
    console.log(`工作进程 ${worker.process.pid} 退出`);
    cluster.fork();  // 重启
  });
} else {
  // 工作进程：运行服务器
  const app = express();
  
  app.get('/', (req, res) => {
    res.send(`Hello from ${process.pid}`);
  });
  
  app.listen(3000);
}
```

### 9.2 缓存

```javascript
const redis = require('redis');
const client = redis.createClient();

// 缓存中间件
const cache = (duration) => async (req, res, next) => {
  const key = `cache:${req.url}`;
  
  // 检查缓存
  const cached = await client.get(key);
  if (cached) {
    return res.json(JSON.parse(cached));
  }
  
  // 缓存响应
  const originalJson = res.json.bind(res);
  res.json = (data) => {
    client.setEx(key, duration, JSON.stringify(data));
    return originalJson(data);
  };
  
  next();
};

// 使用缓存
app.get('/api/users', cache(60), async (req, res) => {
  const users = await User.find();
  res.json(users);
});
```

### 9.3 压缩

```javascript
const compression = require('compression');

app.use(compression());
```

---

## 10. 常见面试题

### Q1: Node.js为什么是单线程？

**答案**：
- JavaScript设计为单线程（避免DOM操作冲突）
- 单线程 + 异步I/O = 高并发
- 通过Cluster可以利用多核

### Q2: Event Loop执行顺序？

**答案**：见1.2

### Q3: Express和Koa的区别？

**答案**：
| 特性 | Express | Koa |
|------|---------|-----|
| **回调** | 回调函数 | async/await |
| **中间件** | 线性执行 | 洋葱模型 |
| **错误处理** | try-catch | 统一catch |
| **大小** | 较大 | 更小 |

### Q4: JWT和Session的区别？

**答案**：
| 特性 | JWT | Session |
|------|-----|---------|
| **存储** | 客户端 | 服务器 |
| **扩展性** | 好（无状态） | 差（有状态） |
| **安全性** | 可能被窃取 | 更安全 |
| **大小** | 较大 | 较小 |

---

## 11. 实战案例

### 案例：RESTful API（用户管理）

```javascript
const express = require('express');
const mongoose = require('mongoose');
const jwt = require('jsonwebtoken');

const app = express();
app.use(express.json());

// 连接数据库
mongoose.connect('mongodb://localhost:27017/mydb');

// User Schema
const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  email: String,
});

const User = mongoose.model('User', userSchema);

// 注册
app.post('/api/register', async (req, res) => {
  try {
    const user = new User(req.body);
    await user.save();
    res.status(201).json({ id: user._id, username: user.username });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// 登录
app.post('/api/login', async (req, res) => {
  const { username, password } = req.body;
  const user = await User.findOne({ username, password });
  
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  const token = jwt.sign({ id: user._id }, 'secret', { expiresIn: '7d' });
  res.json({ token });
});

// 认证中间件
const auth = (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  
  try {
    req.user = jwt.verify(token, 'secret');
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// 获取用户列表
app.get('/api/users', auth, async (req, res) => {
  const users = await User.find().select('-password');
  res.json(users);
});

// 获取单个用户
app.get('/api/users/:id', auth, async (req, res) => {
  const user = await User.findById(req.params.id).select('-password');
  res.json(user);
});

// 更新用户
app.put('/api/users/:id', auth, async (req, res) => {
  const user = await User.findByIdAndUpdate(req.params.id, req.body, { new: true });
  res.json(user);
});

// 删除用户
app.delete('/api/users/:id', auth, async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.status(204).send();
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

---

## 总结

Node.js后端核心：
1. **Event Loop**：异步I/O、事件驱动
2. **框架**：Express（简单）、Koa（洋葱模型）
3. **数据库**：MySQL、MongoDB、Redis
4. **认证**：JWT、Session
5. **实时通信**：WebSocket、Socket.io
6. **性能优化**：Cluster、缓存、压缩

面试时展示**对Node.js异步机制的理解**和**后端开发经验**！🚀

---

**文件状态**：✅ 已完成（约650行）  
**下一步**：继续创建 Nginx与Web服务器.md...