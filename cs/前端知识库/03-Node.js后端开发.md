# Node.js后端开发

---

## 📑 目录

### 基础与框架
1. [Node.js基础](#1-nodejs基础)
2. [Express框架](#2-express框架)
3. [Koa框架](#3-koa框架)
4. [中间件原理](#4-中间件原理)

### 服务能力
5. [数据库集成](#5-数据库集成)
6. [身份认证](#6-身份认证)
7. [文件上传](#7-文件上传)
8. [WebSocket实时通信](#8-websocket实时通信)

### 工程实践与自查
9. [性能优化](#9-性能优化)
10. [进程管理与部署](#10-进程管理与部署)
11. [Stream与Buffer](#11-stream与buffer)
12. [面试题自查](#12-面试题自查)
13. [实战案例](#13-实战案例)

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

## 10. 进程管理与部署

### 10.1 PM2进程管理

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'my-app',
    script: './app.js',
    instances: 'max',        // 根据CPU核心数创建实例
    exec_mode: 'cluster',    // 集群模式
    watch: true,             // 监听文件变化
    max_memory_restart: '1G', // 内存超过1G重启
    env: {
      NODE_ENV: 'development',
    },
    env_production: {
      NODE_ENV: 'production',
    },
  }],
};

// 常用命令
// pm2 start ecosystem.config.js
// pm2 start app.js -i max      # 集群模式启动
// pm2 list                     # 查看所有进程
// pm2 logs                     # 查看日志
// pm2 monit                    # 监控面板
// pm2 restart all              # 重启所有
// pm2 stop all                 # 停止所有
// pm2 delete all               # 删除所有
```

### 10.2 日志处理

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    // 控制台输出
    new winston.transports.Console({
      format: winston.format.simple(),
    }),
    // 错误日志文件
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
    }),
    // 所有日志文件
    new winston.transports.File({
      filename: 'logs/combined.log',
    }),
  ],
});

// 使用
logger.info('服务器启动', { port: 3000 });
logger.error('数据库连接失败', { error: err.message });

// Express日志中间件
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info('HTTP请求', {
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration: `${duration}ms`,
    });
  });
  next();
});
```

### 10.3 错误处理最佳实践

```javascript
// 自定义错误类
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;  // 可预期的操作错误
    Error.captureStackTrace(this, this.constructor);
  }
}

// 异步错误包装器
const catchAsync = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// 使用
app.get('/api/users/:id', catchAsync(async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    throw new AppError('用户不存在', 404);
  }
  res.json(user);
}));

// 全局错误处理中间件
app.use((err, req, res, next) => {
  err.statusCode = err.statusCode || 500;
  
  if (process.env.NODE_ENV === 'development') {
    res.status(err.statusCode).json({
      status: 'error',
      error: err,
      message: err.message,
      stack: err.stack,
    });
  } else {
    // 生产环境：只返回简洁错误信息
    if (err.isOperational) {
      res.status(err.statusCode).json({
        status: 'error',
        message: err.message,
      });
    } else {
      // 未知错误
      console.error('ERROR:', err);
      res.status(500).json({
        status: 'error',
        message: '服务器内部错误',
      });
    }
  }
});

// 未捕获的异常处理
process.on('uncaughtException', (err) => {
  console.error('未捕获的异常:', err);
  process.exit(1);
});

process.on('unhandledRejection', (err) => {
  console.error('未处理的Promise拒绝:', err);
  process.exit(1);
});
```

---

## 11. Stream与Buffer

### 11.1 Stream流

```javascript
const fs = require('fs');

// 读取流
const readStream = fs.createReadStream('large-file.txt', {
  encoding: 'utf8',
  highWaterMark: 64 * 1024,  // 64KB缓冲区
});

readStream.on('data', (chunk) => {
  console.log('读取数据:', chunk.length);
});

readStream.on('end', () => {
  console.log('读取完成');
});

// 写入流
const writeStream = fs.createWriteStream('output.txt');
writeStream.write('Hello\n');
writeStream.write('World\n');
writeStream.end();

// 管道（Pipe）- 最常用
const readStream = fs.createReadStream('input.txt');
const writeStream = fs.createWriteStream('output.txt');
readStream.pipe(writeStream);  // 自动处理背压

// Express中使用流
app.get('/download', (req, res) => {
  const filePath = 'large-file.zip';
  res.setHeader('Content-Disposition', 'attachment; filename=file.zip');
  fs.createReadStream(filePath).pipe(res);
});

// 流式处理大文件
const zlib = require('zlib');
fs.createReadStream('input.txt')
  .pipe(zlib.createGzip())           // 压缩
  .pipe(fs.createWriteStream('input.txt.gz'));
```

### 11.2 Buffer操作

```javascript
// 创建Buffer
const buf1 = Buffer.alloc(10);           // 10字节，填充0
const buf2 = Buffer.from('Hello');       // 从字符串创建
const buf3 = Buffer.from([1, 2, 3]);     // 从数组创建

// 写入
buf1.write('Hi');

// 读取
console.log(buf2.toString());            // 'Hello'
console.log(buf2.toString('hex'));       // '48656c6c6f'

// 拼接
const combined = Buffer.concat([buf2, buf3]);

// 比较
buf1.equals(buf2);  // false
buf1.compare(buf2); // -1/0/1

// 切片
const slice = buf2.slice(0, 2);  // 'He'

// 场景：处理二进制数据（图片、加密等）
const crypto = require('crypto');
const hash = crypto.createHash('sha256');
hash.update(Buffer.from('password'));
console.log(hash.digest('hex'));
```

---

## 12. 面试题自查

### Q1: Node.js为什么是单线程？如何利用多核？

**答案**：
```javascript
// 1. 为什么是单线程？
// - JavaScript设计为单线程（避免DOM操作冲突）
// - 单线程 + 异步I/O = 高并发
// - 避免多线程的锁和上下文切换开销

// 2. 如何利用多核？
// 方法1：Cluster模块
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
} else {
  require('./app.js');
}

// 方法2：Worker Threads（CPU密集型任务）
const { Worker } = require('worker_threads');
const worker = new Worker('./heavy-task.js');
worker.on('message', (result) => console.log(result));

// 方法3：PM2集群模式
// pm2 start app.js -i max
```

### Q2: Node.js的Event Loop执行顺序？

**答案**：
```javascript
// Node.js Event Loop的6个阶段：
// 1. timers: setTimeout/setInterval
// 2. pending callbacks: I/O回调
// 3. idle, prepare: 内部使用
// 4. poll: 等待I/O事件
// 5. check: setImmediate
// 6. close callbacks: close事件

// 微任务在每个阶段之间执行
// process.nextTick > Promise.then > queueMicrotask

console.log('1');                          // 同步
setTimeout(() => console.log('2'), 0);     // timers阶段
setImmediate(() => console.log('3'));      // check阶段
Promise.resolve().then(() => console.log('4')); // 微任务
process.nextTick(() => console.log('5'));  // nextTick队列（最高优先级）
console.log('6');                          // 同步

// 输出：1 6 5 4 2 3
```

### Q3: Express和Koa的中间件机制有什么区别？

**答案**：
```javascript
// Express：线性执行
app.use((req, res, next) => {
  console.log('1');
  next();
  console.log('4');  // next()后的代码不一定在后续中间件执行完后执行
});
app.use((req, res, next) => {
  console.log('2');
  next();
  console.log('3');
});

// Koa：洋葱模型（基于async/await）
app.use(async (ctx, next) => {
  console.log('1');
  await next();     // 等待后续中间件执行完
  console.log('6'); // 一定在后续中间件执行完后执行
});
app.use(async (ctx, next) => {
  console.log('2');
  await next();
  console.log('5');
});
app.use(async (ctx) => {
  console.log('3');
  console.log('4');
});

// Koa输出：1 2 3 4 5 6

// 区别总结：
// | 特性 | Express | Koa |
// |------|---------|-----|
// | 回调 | 回调函数 | async/await |
// | 中间件 | 线性执行 | 洋葱模型 |
// | 错误处理 | 需要传给next(err) | try-catch统一捕获 |
```

### Q4: JWT和Session的区别及选型？

**答案**：
```javascript
// JWT特点：
// - 无状态：Token自包含用户信息，服务器不存储
// - 可扩展：适合分布式系统，无需共享Session
// - 无法主动失效：除非加黑名单

// Session特点：
// - 有状态：服务器存储Session数据
// - 可控：服务器可随时销毁Session
// - 需要共享：分布式系统需要Redis等共享存储

// 选型建议：
// JWT适合：
// - 分布式系统/微服务
// - 移动端应用
// - 第三方API认证

// Session适合：
// - 传统Web应用
// - 需要主动踢人/强制下线
// - 安全性要求高

// 最佳实践：JWT + Redis黑名单
const blacklist = new Set();
app.use((req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (blacklist.has(token)) {
    return res.status(401).json({ error: 'Token已失效' });
  }
  next();
});
```

### Q5: 如何处理Node.js中的内存泄漏？

**答案**：
```javascript
// 常见内存泄漏原因：
// 1. 全局变量
// 2. 闭包中的引用
// 3. 事件监听器未移除
// 4. 定时器未清除
// 5. 缓存无限增长

// 排查方法：
// 1. 使用--inspect启动，Chrome DevTools分析
// node --inspect app.js

// 2. 使用heapdump生成快照
const heapdump = require('heapdump');
heapdump.writeSnapshot('./heap.heapsnapshot');

// 3. 监控内存使用
setInterval(() => {
  const usage = process.memoryUsage();
  console.log({
    heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)}MB`,
    heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)}MB`,
  });
}, 5000);

// 预防措施：
// 1. 使用WeakMap/WeakSet
const cache = new WeakMap();

// 2. 及时移除事件监听器
const handler = () => {};
emitter.on('event', handler);
emitter.removeListener('event', handler);

// 3. 限制缓存大小
const LRU = require('lru-cache');
const cache = new LRU({ max: 500 });
```

### Q6: Node.js如何处理大文件上传？

**答案**：
```javascript
// 方法1：流式处理（避免内存占用）
const busboy = require('busboy');

app.post('/upload', (req, res) => {
  const bb = busboy({ headers: req.headers });
  
  bb.on('file', (name, file, info) => {
    const writeStream = fs.createWriteStream(`./uploads/${info.filename}`);
    file.pipe(writeStream);
    
    file.on('end', () => {
      console.log('文件上传完成');
    });
  });
  
  req.pipe(bb);
});

// 方法2：分片上传
// 前端分片
async function uploadChunks(file) {
  const chunkSize = 5 * 1024 * 1024; // 5MB
  const chunks = Math.ceil(file.size / chunkSize);
  
  for (let i = 0; i < chunks; i++) {
    const chunk = file.slice(i * chunkSize, (i + 1) * chunkSize);
    const formData = new FormData();
    formData.append('chunk', chunk);
    formData.append('index', i);
    formData.append('total', chunks);
    formData.append('hash', await calculateHash(file));
    
    await fetch('/upload/chunk', { method: 'POST', body: formData });
  }
}

// 后端合并
app.post('/upload/merge', async (req, res) => {
  const { hash, total, filename } = req.body;
  const chunkDir = `./uploads/chunks/${hash}`;
  const chunks = await fs.readdir(chunkDir);
  
  chunks.sort((a, b) => parseInt(a) - parseInt(b));
  
  const writeStream = fs.createWriteStream(`./uploads/${filename}`);
  for (const chunk of chunks) {
    const chunkPath = path.join(chunkDir, chunk);
    const readStream = fs.createReadStream(chunkPath);
    await new Promise((resolve) => {
      readStream.pipe(writeStream, { end: false });
      readStream.on('end', resolve);
    });
  }
  writeStream.end();
  
  // 清理临时文件
  await fs.rm(chunkDir, { recursive: true });
  res.json({ success: true });
});
```

### Q7: 如何实现Node.js的优雅关闭（Graceful Shutdown）？

**答案**：
```javascript
const server = app.listen(3000);

async function gracefulShutdown(signal) {
  console.log(`收到 ${signal} 信号，开始优雅关闭...`);
  
  // 1. 停止接收新请求
  server.close(async () => {
    console.log('HTTP服务器已关闭');
    
    // 2. 关闭数据库连接
    await mongoose.connection.close();
    console.log('数据库连接已关闭');
    
    // 3. 关闭Redis连接
    await redisClient.quit();
    console.log('Redis连接已关闭');
    
    // 4. 退出进程
    process.exit(0);
  });
  
  // 超时强制退出
  setTimeout(() => {
    console.error('强制退出');
    process.exit(1);
  }, 10000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

### Q8: Node.js中require的加载机制？

**答案**：
```javascript
// require加载顺序：
// 1. 缓存：检查是否已加载
// 2. 内置模块：如fs、http
// 3. 文件模块：以./、../、/开头
// 4. node_modules：从当前目录向上查找

// 文件扩展名解析顺序：
// 1. 精确匹配
// 2. .js
// 3. .json
// 4. .node（C++扩展）

// 循环依赖处理：
// a.js
console.log('a开始');
exports.done = false;
const b = require('./b.js');
console.log('在a中，b.done =', b.done);
exports.done = true;
console.log('a结束');

// b.js
console.log('b开始');
exports.done = false;
const a = require('./a.js');  // 获取a的部分导出
console.log('在b中，a.done =', a.done);  // false
exports.done = true;
console.log('b结束');

// 执行 require('./a.js')
// 输出：
// a开始
// b开始
// 在b中，a.done = false
// b结束
// 在a中，b.done = true
// a结束
```

### Q9: 如何在Node.js中实现请求限流？

**答案**：
```javascript
// 方法1：简单的令牌桶算法
class RateLimiter {
  constructor(maxTokens, refillRate) {
    this.maxTokens = maxTokens;
    this.tokens = maxTokens;
    this.refillRate = refillRate;  // 每秒补充的令牌数
    
    setInterval(() => {
      this.tokens = Math.min(this.maxTokens, this.tokens + this.refillRate);
    }, 1000);
  }
  
  tryConsume() {
    if (this.tokens > 0) {
      this.tokens--;
      return true;
    }
    return false;
  }
}

const limiter = new RateLimiter(100, 10);  // 100个令牌，每秒补充10个

app.use((req, res, next) => {
  if (limiter.tryConsume()) {
    next();
  } else {
    res.status(429).json({ error: 'Too Many Requests' });
  }
});

// 方法2：使用express-rate-limit
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 60 * 1000,  // 1分钟
  max: 100,             // 每个IP最多100个请求
  message: { error: '请求过于频繁，请稍后再试' },
});

app.use('/api/', limiter);

// 方法3：使用Redis实现分布式限流
async function rateLimitMiddleware(req, res, next) {
  const key = `rate:${req.ip}`;
  const current = await redis.incr(key);
  
  if (current === 1) {
    await redis.expire(key, 60);  // 60秒过期
  }
  
  if (current > 100) {
    return res.status(429).json({ error: 'Too Many Requests' });
  }
  
  next();
}
```

### Q10: Node.js中如何实现定时任务？

**答案**：
```javascript
// 方法1：node-cron
const cron = require('node-cron');

// 每天凌晨3点执行
cron.schedule('0 3 * * *', async () => {
  console.log('执行每日清理任务');
  await cleanupExpiredData();
});

// 每小时执行
cron.schedule('0 * * * *', () => {
  console.log('执行每小时统计');
});

// 方法2：agenda（基于MongoDB）
const Agenda = require('agenda');
const agenda = new Agenda({ db: { address: 'mongodb://localhost/agenda' } });

agenda.define('send email', async (job) => {
  const { to, subject, body } = job.attrs.data;
  await sendEmail(to, subject, body);
});

(async function() {
  await agenda.start();
  
  // 立即执行
  await agenda.now('send email', { to: 'user@example.com' });
  
  // 定时执行
  await agenda.schedule('in 20 minutes', 'send email', { to: 'user@example.com' });
  
  // 重复执行
  await agenda.every('1 hour', 'send email', { to: 'admin@example.com' });
})();

// 方法3：bull（基于Redis）
const Queue = require('bull');
const emailQueue = new Queue('email', 'redis://localhost:6379');

emailQueue.process(async (job) => {
  const { to, subject, body } = job.data;
  await sendEmail(to, subject, body);
});

// 添加任务
emailQueue.add({ to: 'user@example.com', subject: 'Hello' }, {
  delay: 60000,        // 延迟1分钟
  attempts: 3,         // 失败重试3次
  backoff: 5000,       // 重试间隔5秒
  repeat: { cron: '0 9 * * *' },  // 每天9点执行
});
```

### Q11: 如何保证Node.js应用的高可用？

**答案**：
```javascript
// 1. PM2集群模式
// pm2 start app.js -i max --name my-app

// 2. 负载均衡（Nginx）
// upstream nodejs {
//   server 127.0.0.1:3001;
//   server 127.0.0.1:3002;
//   server 127.0.0.1:3003;
// }

// 3. 健康检查端点
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    uptime: process.uptime(),
    timestamp: Date.now(),
  });
});

// 4. 进程守护（自动重启）
cluster.on('exit', (worker, code, signal) => {
  console.log(`Worker ${worker.process.pid} died`);
  cluster.fork();  // 自动重启
});

// 5. 优雅降级
const circuitBreaker = require('opossum');

const options = {
  timeout: 3000,     // 超时时间
  errorThresholdPercentage: 50,  // 错误率阈值
  resetTimeout: 30000,  // 熔断后30秒尝试恢复
};

const breaker = new circuitBreaker(asyncFunction, options);

breaker.on('open', () => console.log('熔断器打开'));
breaker.on('halfOpen', () => console.log('熔断器半开'));
breaker.on('close', () => console.log('熔断器关闭'));

app.get('/api/data', async (req, res) => {
  try {
    const result = await breaker.fire();
    res.json(result);
  } catch (err) {
    res.json({ cached: true, data: getCachedData() });  // 降级处理
  }
});
```

### Q12: Node.js中Buffer和String的区别？

**答案**：
```javascript
// String: 文本数据，UTF-16编码
// Buffer: 二进制数据，类似数组

// 内存分配
const str = 'Hello';        // V8堆内存
const buf = Buffer.from('Hello');  // V8堆外内存（C++层）

// 编码转换
const buf = Buffer.from('你好', 'utf8');
console.log(buf);  // <Buffer e4 bd a0 e5 a5 bd>
console.log(buf.toString('utf8'));  // '你好'
console.log(buf.toString('base64'));  // '5L2g5aW9'

// 性能差异
// Buffer适合：网络通信、文件I/O、加密、图片处理
// String适合：文本处理、JSON、模板渲染

// 安全问题
// Buffer.alloc(size): 安全，填充0
// Buffer.allocUnsafe(size): 不安全，可能包含旧数据（更快）
const safe = Buffer.alloc(10);        // <Buffer 00 00 00 00 00 00 00 00 00 00>
const unsafe = Buffer.allocUnsafe(10); // 可能包含旧数据

// 实际应用
// 1. 处理二进制协议
const header = Buffer.alloc(4);
header.writeUInt32BE(data.length, 0);  // 写入数据长度（大端序）

// 2. 文件读取
const content = fs.readFileSync('image.png');  // 返回Buffer
const text = fs.readFileSync('text.txt', 'utf8');  // 返回String
```

### Q13: 什么是背压（Backpressure）？如何处理？

**答案**：
```javascript
// 背压：生产者速度 > 消费者速度，导致内存溢出

// 问题示例
const readStream = fs.createReadStream('large-file.txt');
const writeStream = fs.createWriteStream('output.txt');

// ❌ 直接写入，可能内存溢出
readStream.on('data', (chunk) => {
  writeStream.write(chunk);  // 写入速度可能跟不上
});

// ✅ 方法1：使用pipe（自动处理背压）
readStream.pipe(writeStream);

// ✅ 方法2：手动处理背压
readStream.on('data', (chunk) => {
  const canContinue = writeStream.write(chunk);
  
  if (!canContinue) {
    // 暂停读取
    readStream.pause();
  }
});

writeStream.on('drain', () => {
  // 缓冲区已清空，继续读取
  readStream.resume();
});

// ✅ 方法3：使用pipeline（推荐）
const { pipeline } = require('stream');

pipeline(
  readStream,
  transformStream,
  writeStream,
  (err) => {
    if (err) console.error('Pipeline failed:', err);
    else console.log('Pipeline succeeded');
  }
);
```

### Q14: Node.js如何处理CPU密集型任务？

**答案**：
```javascript
// Node.js主线程不适合CPU密集型任务，会阻塞事件循环

// 方法1：Worker Threads
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
  // 主线程
  const worker = new Worker(__filename);
  worker.on('message', (result) => {
    console.log('计算结果:', result);
  });
  worker.postMessage({ n: 40 });
} else {
  // Worker线程
  parentPort.on('message', ({ n }) => {
    const result = fibonacci(n);  // CPU密集型计算
    parentPort.postMessage(result);
  });
}

// 方法2：Child Process
const { fork } = require('child_process');

const child = fork('./heavy-task.js');
child.send({ n: 40 });
child.on('message', (result) => {
  console.log('计算结果:', result);
});

// 方法3：使用线程池
const Piscina = require('piscina');

const pool = new Piscina({
  filename: './worker.js',
  maxThreads: 4,
});

async function main() {
  const result = await pool.run({ n: 40 });
  console.log(result);
}

// 方法4：使用原生addon（C++）
// 适合需要高性能计算的场景
```

### Q15: Node.js的安全最佳实践？

**答案**：
```javascript
// 1. 防止SQL/NoSQL注入
// 使用参数化查询
const [rows] = await pool.query(
  'SELECT * FROM users WHERE id = ?',
  [req.params.id]
);

// 2. 防止XSS
const xss = require('xss');
const sanitized = xss(userInput);

// 3. 设置安全响应头
const helmet = require('helmet');
app.use(helmet());  // 设置多个安全头

// 4. CORS配置
const cors = require('cors');
app.use(cors({
  origin: ['https://example.com'],
  methods: ['GET', 'POST'],
  credentials: true,
}));

// 5. 请求大小限制
app.use(express.json({ limit: '10kb' }));

// 6. 密码加密
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash(password, 12);
const isMatch = await bcrypt.compare(password, hash);

// 7. 敏感信息不要硬编码
// 使用环境变量
const secret = process.env.JWT_SECRET;

// 8. 定期更新依赖
// npm audit
// npm audit fix

// 9. 日志脱敏
const maskSensitiveData = (data) => {
  return {
    ...data,
    password: '***',
    token: '***',
    creditCard: data.creditCard?.slice(-4).padStart(16, '*'),
  };
};
```

### Q16: CommonJS和ES Module的区别？

**答案**：
```javascript
// CommonJS
// - 同步加载
// - 运行时执行
// - 值拷贝
// - this指向module.exports

const fs = require('fs');
module.exports = { foo: 'bar' };
exports.baz = 'qux';

// ES Module
// - 异步加载
// - 编译时静态分析
// - 值引用（实时绑定）
// - this是undefined

import fs from 'fs';
import { readFile } from 'fs';
export const foo = 'bar';
export default { baz: 'qux' };

// 互操作
// ES Module导入CommonJS
import cjs from './cjs-module.js';  // 获取module.exports
import { named } from './cjs-module.js';  // 可能不支持

// CommonJS导入ES Module（需要动态import）
const esm = await import('./es-module.mjs');

// 主要区别
// | 特性 | CommonJS | ES Module |
// |------|----------|-----------|
// | 加载时机 | 运行时 | 编译时 |
// | 加载方式 | 同步 | 异步 |
// | 值传递 | 值拷贝 | 值引用 |
// | 顶层this | module.exports | undefined |
// | 动态导入 | require() | import() |
// | Tree Shaking | 不支持 | 支持 |
```

---

## 13. 实战案例

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
1. **Event Loop**：异步I/O、事件驱动、微任务与宏任务
2. **框架**：Express（简单）、Koa（洋葱模型）
3. **数据库**：MySQL、MongoDB、Redis
4. **认证**：JWT（无状态）、Session（有状态）
5. **实时通信**：WebSocket、Socket.io
6. **性能优化**：Cluster、缓存、压缩、限流
7. **进程管理**：PM2、优雅关闭、高可用
8. **Stream与Buffer**：流式处理、背压、二进制数据
9. **安全实践**：SQL注入防护、XSS、CORS、helmet


---
