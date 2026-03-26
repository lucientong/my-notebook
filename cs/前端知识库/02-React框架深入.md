# React框架深入 - 从原理到实战（完整版）

---

## 📑 目录

### 第一部分：组件化架构
1. [React组件树与架构](#1-react组件树与架构)
2. [组件通信的8种方式](#2-组件通信的8种方式)
3. [数据流与状态管理](#3-数据流与状态管理)

### 第二部分：渲染机制
4. [组件渲染与更新原理](#4-组件渲染与更新原理)
5. [虚拟DOM与Diff算法](#5-虚拟dom与diff算法)
6. [Fiber架构与并发渲染](#6-fiber架构与并发渲染)

### 第三部分：生命周期与Hooks
7. [完整的生命周期](#7-完整的生命周期)
8. [Hooks深入与实现原理](#8-hooks深入与实现原理)
9. [自定义Hooks最佳实践](#9-自定义hooks最佳实践)

### 第四部分：框架对比与设计哲学
10. [React vs Vue vs Angular](#10-react-vs-vue-vs-angular)
11. [React设计哲学](#11-react设计哲学)
12. [性能优化进阶](#12-性能优化进阶)

### 第五部分：实战、微前端与面试
13. [高级模式与技巧](#13-高级模式与技巧)
14. [源码级面试题](#14-源码级面试题)
15. [企业级实战案例](#15-企业级实战案例)
16. [React与微前端架构](#16-react与微前端架构) ⭐NEW⭐
    - 什么是微前端
    - React微前端解决方案（qiankun/Module Federation/iframe）
    - 微前端核心问题（样式隔离、JS沙箱、通信、路由）
    - 独立部署方案
    - 微前端最佳实践
    - 完整示例

---

# 第一部分：组件化架构

## 1. React组件树与架构

### 1.1 组件树的本质

React应用是一个**树形结构**，从根节点开始，层层嵌套：

```jsx
// 组件树结构示意
<App>                          // 根组件
  ├── <Header>                 // 头部
  │   ├── <Logo />
  │   └── <Nav>
  │       ├── <NavItem />
  │       └── <NavItem />
  ├── <Main>                   // 主体
  │   ├── <Sidebar>
  │   │   └── <Menu />
  │   └── <Content>
  │       ├── <Article />
  │       └── <Comment />
  └── <Footer>                 // 底部
```

**对应的代码结构**：

```jsx
function App() {
  return (
    <div className="app">
      <Header />
      <Main />
      <Footer />
    </div>
  );
}

function Header() {
  return (
    <header>
      <Logo />
      <Nav />
    </header>
  );
}

function Nav() {
  return (
    <nav>
      <NavItem title="首页" />
      <NavItem title="关于" />
    </nav>
  );
}
```

### 1.2 组件树的内部表示（Fiber树）

React内部用**Fiber节点**表示组件树：

```javascript
// 简化的Fiber节点结构
{
  type: 'div',              // 组件类型
  props: { className: 'app' },  // 属性
  child: FiberNode,         // 第一个子节点
  sibling: FiberNode,       // 兄弟节点
  return: FiberNode,        // 父节点
  stateNode: DOMElement,    // 真实DOM引用
  alternate: FiberNode,     // 指向旧Fiber树（用于Diff）
  effectTag: 'UPDATE',      // 标记需要的操作（增删改）
}
```

**三棵树**：
1. **Current Fiber树**：当前显示的树
2. **WorkInProgress Fiber树**：正在构建的新树
3. **DOM树**：真实的DOM树

```
渲染流程：
组件更新 → 构建WIP树 → Diff → 标记effectTag → Commit → 替换Current树
```

### 1.3 组件树的遍历

**深度优先遍历（DFS）**：

```javascript
function walkTree(fiber) {
  // 1. 处理当前节点
  console.log(fiber.type);
  
  // 2. 遍历子节点
  if (fiber.child) {
    walkTree(fiber.child);
  }
  
  // 3. 遍历兄弟节点
  if (fiber.sibling) {
    walkTree(fiber.sibling);
  }
}

// 遍历顺序：
// App → Header → Logo → Nav → NavItem(首页) → NavItem(关于) → Main → ...
```

**为什么是深度优先？**
- **自下而上的更新**：子组件更新完才能确定父组件是否需要更新
- **提前发现问题**：深入到叶子节点可以尽早发现错误
- **符合组件依赖关系**：子组件依赖父组件的props

### 1.4 组件树的挂载过程

```jsx
// 1. 代码层面
ReactDOM.createRoot(document.getElementById('root')).render(<App />);

// 2. 内部流程
createRoot
  → createFiberRoot           // 创建根Fiber
  → render(<App />)
  → scheduleUpdateOnFiber     // 调度更新
  → workLoopSync              // 开始工作循环
  → performUnitOfWork         // 处理每个Fiber单元
    → beginWork               // 向下遍历（创建子Fiber）
    → completeWork            // 向上回溯（创建DOM）
  → commitRoot                // 提交到DOM
    → commitBeforeMutationEffects  // DOM变更前
    → commitMutationEffects        // DOM变更
    → commitLayoutEffects          // DOM变更后
```

**详细的挂载流程**：

```javascript
// 阶段1：Render阶段（可中断）
function beginWork(fiber) {
  // 根据fiber.type创建子Fiber
  if (fiber.type === 'div') {
    // 创建子Fiber链表
    reconcileChildren(fiber, fiber.props.children);
  } else if (typeof fiber.type === 'function') {
    // 执行函数组件，获取返回的JSX
    const children = fiber.type(fiber.props);
    reconcileChildren(fiber, children);
  }
  return fiber.child;  // 返回第一个子节点
}

function completeWork(fiber) {
  // 创建真实DOM
  if (fiber.tag === 'HostComponent') {  // 原生标签
    const domElement = document.createElement(fiber.type);
    // 挂载props
    Object.keys(fiber.props).forEach(key => {
      if (key !== 'children') {
        domElement[key] = fiber.props[key];
      }
    });
    fiber.stateNode = domElement;
  }
}

// 阶段2：Commit阶段（不可中断）
function commitRoot(root) {
  // 遍历effectList，执行DOM操作
  let effect = root.firstEffect;
  while (effect) {
    if (effect.effectTag === 'PLACEMENT') {
      // 插入DOM
      effect.return.stateNode.appendChild(effect.stateNode);
    } else if (effect.effectTag === 'UPDATE') {
      // 更新DOM属性
      updateDOMProperties(effect.stateNode, effect.alternate.props, effect.props);
    }
    effect = effect.nextEffect;
  }
}
```

---

## 2. 组件通信的8种方式

### 2.1 父→子：Props

**最基础的通信方式**：

```jsx
// 父组件
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <Child count={count} onIncrement={() => setCount(count + 1)} />
    </div>
  );
}

// 子组件
function Child({ count, onIncrement }) {
  return (
    <div>
      <p>计数：{count}</p>
      <button onClick={onIncrement}>+1</button>
    </div>
  );
}
```

**Props的本质**：

```javascript
// JSX编译后
React.createElement(Child, { 
  count: count, 
  onIncrement: () => setCount(count + 1) 
});

// React内部处理
function renderChild(fiber) {
  const props = fiber.props;  // 从Fiber节点获取props
  const Component = fiber.type;
  const element = Component(props);  // 执行组件函数，传入props
  return element;
}
```

**Props的不可变性**：

```jsx
// ❌ 错误：直接修改props
function Child({ user }) {
  user.name = '李四';  // 违反了单向数据流
  return <div>{user.name}</div>;
}

// ✅ 正确：通知父组件修改
function Child({ user, onUpdateUser }) {
  return (
    <button onClick={() => onUpdateUser({ ...user, name: '李四' })}>
      修改名字
    </button>
  );
}
```

### 2.2 子→父：回调函数

```jsx
function Parent() {
  const handleChildData = (data) => {
    console.log('子组件传来的数据：', data);
  };
  
  return <Child onData={handleChildData} />;
}

function Child({ onData }) {
  const sendData = () => {
    onData({ message: 'Hello from child' });
  };
  
  return <button onClick={sendData}>发送数据</button>;
}
```

**为什么子组件不能直接修改父组件的state？**

```
1. 单向数据流：数据只能从父到子，保证数据流向清晰
2. 可预测性：所有状态变化都在父组件，便于调试
3. 避免副作用：子组件不能影响父组件，降低耦合
```

### 2.3 跨层级：Context

**Context的工作原理**：

```jsx
// 1. 创建Context
const ThemeContext = React.createContext('light');

// 2. Provider提供值（在组件树的某个位置）
function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Toolbar />  {/* Toolbar的所有子组件都能访问 */}
    </ThemeContext.Provider>
  );
}

// 3. Consumer消费值（在任意深度的子组件）
function Button() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>按钮</button>;
}
```

**Context的内部实现**：

```javascript
// 简化版实现
function createContext(defaultValue) {
  const context = {
    _currentValue: defaultValue,  // 存储当前值
    Provider: ({ value, children }) => {
      context._currentValue = value;  // 更新值
      return children;
    },
    Consumer: ({ children }) => {
      return children(context._currentValue);  // 传递值
    }
  };
  return context;
}

function useContext(context) {
  return context._currentValue;  // 读取值
}
```

**Context的性能问题**：

```jsx
// ❌ 问题：value是对象，每次render都创建新对象
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <ThemeContext.Provider value={{ theme: 'dark', count }}>
      <Child />  {/* 每次count变化，Child都会重新渲染 */}
    </ThemeContext.Provider>
  );
}

// ✅ 解决：useMemo缓存对象
function App() {
  const [count, setCount] = useState(0);
  const contextValue = useMemo(() => ({ theme: 'dark', count }), [count]);
  
  return (
    <ThemeContext.Provider value={contextValue}>
      <Child />
    </ThemeContext.Provider>
  );
}
```

### 2.4 兄弟组件：状态提升

```jsx
function Parent() {
  const [sharedData, setSharedData] = useState('');
  
  return (
    <div>
      <ChildA data={sharedData} onDataChange={setSharedData} />
      <ChildB data={sharedData} />
    </div>
  );
}

function ChildA({ data, onDataChange }) {
  return <input value={data} onChange={e => onDataChange(e.target.value)} />;
}

function ChildB({ data }) {
  return <div>接收到：{data}</div>;
}
```

**状态提升的原则**：

```
1. 找到共同父组件：需要共享数据的组件的最近公共祖先
2. 状态放在父组件：父组件管理状态
3. Props向下传递：通过props传给子组件
4. 回调向上通知：子组件通过回调修改状态
```

### 2.5 全局状态：Redux/Zustand

**Redux的工作流程**：

```jsx
// 1. 定义Reducer
function counterReducer(state = { count: 0 }, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };
    default:
      return state;
  }
}

// 2. 创建Store
const store = createStore(counterReducer);

// 3. 组件连接Store
function Counter() {
  const count = useSelector(state => state.count);  // 订阅
  const dispatch = useDispatch();  // 获取dispatch
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+1</button>
    </div>
  );
}
```

**Redux的核心原理**：

```javascript
// 简化版Redux实现
function createStore(reducer) {
  let state;
  let listeners = [];
  
  // 获取状态
  function getState() {
    return state;
  }
  
  // 派发action
  function dispatch(action) {
    state = reducer(state, action);  // 计算新状态
    listeners.forEach(listener => listener());  // 通知订阅者
  }
  
  // 订阅变化
  function subscribe(listener) {
    listeners.push(listener);
    return () => {
      listeners = listeners.filter(l => l !== listener);  // 取消订阅
    };
  }
  
  dispatch({ type: '@@INIT' });  // 初始化state
  
  return { getState, dispatch, subscribe };
}
```

**Zustand（更轻量）**：

```jsx
import create from 'zustand';

// 1. 创建Store（不需要Provider）
const useStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

// 2. 直接使用
function Counter() {
  const { count, increment } = useStore();
  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>+1</button>
    </div>
  );
}
```

### 2.6 Ref转发：访问子组件DOM

```jsx
// 父组件访问子组件的DOM节点
const Child = forwardRef((props, ref) => {
  return <input ref={ref} />;
});

function Parent() {
  const inputRef = useRef();
  
  const focusInput = () => {
    inputRef.current.focus();  // 直接操作子组件的DOM
  };
  
  return (
    <div>
      <Child ref={inputRef} />
      <button onClick={focusInput}>聚焦输入框</button>
    </div>
  );
}
```

**useImperativeHandle：暴露子组件方法**：

```jsx
const Child = forwardRef((props, ref) => {
  const [count, setCount] = useState(0);
  
  // 暴露指定的方法给父组件
  useImperativeHandle(ref, () => ({
    reset: () => setCount(0),
    increment: () => setCount(c => c + 1),
  }));
  
  return <div>计数：{count}</div>;
});

function Parent() {
  const childRef = useRef();
  
  return (
    <div>
      <Child ref={childRef} />
      <button onClick={() => childRef.current.reset()}>重置</button>
      <button onClick={() => childRef.current.increment()}>+1</button>
    </div>
  );
}
```

### 2.7 事件总线：EventEmitter

```jsx
// 1. 创建事件总线
class EventEmitter {
  constructor() {
    this.events = {};
  }
  
  on(event, listener) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(listener);
  }
  
  emit(event, ...args) {
    if (this.events[event]) {
      this.events[event].forEach(listener => listener(...args));
    }
  }
  
  off(event, listener) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(l => l !== listener);
    }
  }
}

const eventBus = new EventEmitter();

// 2. 组件A：发送事件
function ComponentA() {
  const sendMessage = () => {
    eventBus.emit('message', { text: 'Hello' });
  };
  
  return <button onClick={sendMessage}>发送消息</button>;
}

// 3. 组件B：监听事件
function ComponentB() {
  const [message, setMessage] = useState('');
  
  useEffect(() => {
    const handleMessage = (data) => {
      setMessage(data.text);
    };
    
    eventBus.on('message', handleMessage);
    
    return () => {
      eventBus.off('message', handleMessage);  // 清理监听器
    };
  }, []);
  
  return <div>收到消息：{message}</div>;
}
```

### 2.8 URL参数：React Router

```jsx
// 1. 通过URL传递数据
function UserList() {
  const navigate = useNavigate();
  
  const goToUser = (id) => {
    navigate(`/users/${id}`);  // 路径参数
    // 或
    navigate(`/users?id=${id}`);  // 查询参数
  };
  
  return <button onClick={() => goToUser(123)}>查看用户</button>;
}

// 2. 接收URL参数
function UserDetail() {
  // 路径参数
  const { id } = useParams();  // /users/123 → id=123
  
  // 查询参数
  const [searchParams] = useSearchParams();  // /users?id=123
  const id2 = searchParams.get('id');
  
  return <div>用户ID: {id}</div>;
}
```

**通信方式对比**：

| 方式 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| **Props** | 父→子 | 简单、清晰 | 层级深时繁琐 |
| **回调函数** | 子→父 | 直接 | 只能单向 |
| **Context** | 跨层级 | 避免props drilling | 性能开销 |
| **状态提升** | 兄弟组件 | 符合React哲学 | 状态可能过于集中 |
| **Redux/Zustand** | 全局状态 | 强大、可预测 | 学习成本高 |
| **Ref转发** | 访问DOM/方法 | 直接操作 | 破坏封装性 |
| **事件总线** | 任意组件 | 灵活 | 难以追踪、易出bug |
| **URL参数** | 跨页面 | 可分享、可刷新 | 只能传简单数据 |

---

## 3. 数据流与状态管理

### 3.1 单向数据流

**React的核心原则**：

```
数据流向：State → View → Event → State
         ↑____________________________↓

1. State（状态）决定View（视图）
2. View（视图）触发Event（事件）
3. Event（事件）修改State（状态）
4. State变化重新渲染View
```

**示例**：

```jsx
function Counter() {
  // 1. State
  const [count, setCount] = useState(0);
  
  // 3. Event Handler
  const increment = () => {
    setCount(count + 1);  // 修改State
  };
  
  // 2. View（由State决定）
  return (
    <div>
      <p>计数：{count}</p>
      <button onClick={increment}>+1</button>
    </div>
  );
}
```

**与双向绑定的对比（Vue）**：

```vue
<!-- Vue的双向绑定 -->
<template>
  <input v-model="message" />
</template>

<script>
export default {
  data() {
    return { message: '' }
  }
}
</script>
```

```jsx
// React的单向数据流（需要手动绑定）
function Input() {
  const [message, setMessage] = useState('');
  
  return (
    <input 
      value={message} 
      onChange={e => setMessage(e.target.value)} 
    />
  );
}
```

**为什么React选择单向数据流？**

```
优点：
1. 可预测性：数据流向清晰，易于追踪变化
2. 易于调试：状态变化有明确的路径
3. 易于测试：给定输入，输出固定

缺点：
1. 代码量多：需要手动处理onChange
2. 不够直观：没有双向绑定方便
```

### 3.2 State的更新机制

**State更新是异步的**：

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(count + 1);
    console.log(count);  // 输出0（旧值）
    
    setCount(count + 1);
    console.log(count);  // 输出0（旧值）
    
    // 最终count只增加1，不是2
  };
  
  return <button onClick={handleClick}>+1</button>;
}
```

**为什么是异步的？**

```javascript
// React的批处理机制（Batching）
function batchedUpdates() {
  const updates = [];
  
  // 收集所有setState调用
  setCount(count + 1);  // updates.push({ type: 'setState', value: count + 1 })
  setCount(count + 1);  // updates.push({ type: 'setState', value: count + 1 })
  
  // 批量处理（只触发一次重新渲染）
  processUpdates(updates);
}

// 优点：
// 1. 性能优化：多次setState只触发一次渲染
// 2. 避免中间状态：保证UI一致性
```

**函数式更新**：

```jsx
const handleClick = () => {
  // ✅ 正确：使用函数式更新
  setCount(c => c + 1);  // c是最新值
  setCount(c => c + 1);  // c是上一步的最新值
  
  // 最终count增加2
};
```

**内部实现原理**：

```javascript
// 简化版useState实现
let state;
let stateQueue = [];

function useState(initialValue) {
  state = state || initialValue;
  
  const setState = (newValue) => {
    if (typeof newValue === 'function') {
      // 函数式更新：传入最新的state
      stateQueue.push(newValue);
    } else {
      // 直接赋值：使用当前的state
      stateQueue.push(() => newValue);
    }
    
    // 调度更新
    scheduleUpdate();
  };
  
  return [state, setState];
}

function scheduleUpdate() {
  // 批量处理所有更新
  requestIdleCallback(() => {
    stateQueue.forEach(update => {
      state = update(state);  // 依次执行，传入最新state
    });
    stateQueue = [];
    rerender();  // 重新渲染
  });
}
```

### 3.3 State的存储位置

**Fiber节点上的memorizedState**：

```javascript
// Fiber节点结构
{
  type: Counter,
  stateNode: null,
  memorizedState: {
    // Hooks链表
    memorizedState: 0,  // useState的值
    next: {
      memorizedState: [],  // useEffect的依赖
      next: null
    }
  }
}
```

**多个useState的存储**：

```jsx
function Component() {
  const [count, setCount] = useState(0);      // Hook 1
  const [name, setName] = useState('张三');    // Hook 2
  const [age, setAge] = useState(25);         // Hook 3
  
  // 内部存储：
  // memorizedState = {
  //   memorizedState: 0,           // count
  //   next: {
  //     memorizedState: '张三',     // name
  //     next: {
  //       memorizedState: 25,      // age
  //       next: null
  //     }
  //   }
  // }
}
```

**为什么Hooks不能在条件语句中使用？**

```jsx
// ❌ 错误：条件语句中使用Hook
function Component({ showAge }) {
  const [count, setCount] = useState(0);
  
  if (showAge) {
    const [age, setAge] = useState(25);  // 违反规则
  }
  
  const [name, setName] = useState('张三');
}

// 问题：
// 首次渲染：Hook链表 [count] → [age] → [name]
// 二次渲染（showAge=false）：Hook链表 [count] → [name]
// React会错误地认为第二个Hook是age，但实际是name
```

---

（待续：第二部分 - 渲染机制）

---

**当前进度**：第一部分完成（组件化架构）  
**文件规模**：约600行  
**下一部分**：渲染机制、虚拟DOM、Fiber架构...# React框架深入 - Part 2：渲染机制

> 接续Part 1：组件化架构  
> 本部分：渲染原理、虚拟DOM、Fiber架构、并发渲染

---

## 4. 组件渲染与更新原理

### 4.1 首次渲染流程

**完整的挂载流程**：

```javascript
// 1. 入口
ReactDOM.createRoot(document.getElementById('root')).render(<App />);

// 2. 内部流程
createRoot(container)
  ↓
createFiberRoot()           // 创建FiberRoot（整个应用的根）
  ↓
createHostRootFiber()       // 创建HostRootFiber（根Fiber节点）
  ↓
render(<App />)
  ↓
scheduleUpdateOnFiber()     // 调度更新
  ↓
performSyncWorkOnRoot()     // 执行同步工作
  ↓
renderRootSync()            // 渲染根节点
  ↓
workLoopSync()              // 工作循环
  ↓
performUnitOfWork()         // 处理每个Fiber单元
  ├── beginWork()           // 向下遍历（递阶段）
  │   ├── 根据fiber.type创建子Fiber
  │   ├── 调和子节点（reconcileChildren）
  │   └── 返回第一个子节点
  └── completeWork()        // 向上回溯（归阶段）
      ├── 创建DOM实例
      ├── 处理props
      └── 收集副作用（effectList）
  ↓
commitRoot()                // 提交阶段
  ├── commitBeforeMutationEffects()   // DOM变更前
  │   └── 执行getSnapshotBeforeUpdate
  ├── commitMutationEffects()         // DOM变更
  │   ├── 插入DOM
  │   ├── 更新DOM
  │   └── 删除DOM
  └── commitLayoutEffects()           // DOM变更后
      ├── 执行componentDidMount
      └── 执行useLayoutEffect
```

**详细的beginWork流程**：

```javascript
function beginWork(current, workInProgress) {
  // current: 旧Fiber节点
  // workInProgress: 新Fiber节点
  
  switch (workInProgress.tag) {
    case FunctionComponent: {
      // 1. 执行函数组件
      const Component = workInProgress.type;
      const props = workInProgress.pendingProps;
      const children = Component(props);  // 得到JSX
      
      // 2. 调和子节点
      reconcileChildren(current, workInProgress, children);
      
      // 3. 返回第一个子节点
      return workInProgress.child;
    }
    
    case HostComponent: {  // 原生DOM标签
      // 1. 获取子节点
      const nextChildren = workInProgress.pendingProps.children;
      
      // 2. 调和子节点
      reconcileChildren(current, workInProgress, nextChildren);
      
      return workInProgress.child;
    }
    
    case HostText: {  // 文本节点
      return null;  // 文本节点没有子节点
    }
  }
}
```

**reconcileChildren（调和子节点）**：

```javascript
function reconcileChildren(current, workInProgress, nextChildren) {
  if (current === null) {
    // 首次挂载：直接创建Fiber
    workInProgress.child = mountChildFibers(workInProgress, null, nextChildren);
  } else {
    // 更新：执行Diff算法
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren
    );
  }
}

function mountChildFibers(returnFiber, currentFirstChild, newChild) {
  // 1. 处理单个子节点
  if (typeof newChild === 'object' && newChild !== null) {
    if (newChild.$$typeof === REACT_ELEMENT_TYPE) {
      return placeSingleChild(
        reconcileSingleElement(returnFiber, currentFirstChild, newChild)
      );
    }
  }
  
  // 2. 处理数组子节点
  if (Array.isArray(newChild)) {
    return reconcileChildrenArray(returnFiber, currentFirstChild, newChild);
  }
  
  // 3. 处理文本子节点
  if (typeof newChild === 'string' || typeof newChild === 'number') {
    return placeSingleChild(
      reconcileSingleTextNode(returnFiber, currentFirstChild, '' + newChild)
    );
  }
  
  return null;
}
```

**completeWork流程**：

```javascript
function completeWork(current, workInProgress) {
  const newProps = workInProgress.pendingProps;
  
  switch (workInProgress.tag) {
    case HostComponent: {  // 原生DOM标签
      if (current !== null && workInProgress.stateNode != null) {
        // 更新：标记需要更新的props
        updateHostComponent(current, workInProgress, workInProgress.type, newProps);
      } else {
        // 挂载：创建DOM实例
        const instance = createInstance(workInProgress.type, newProps);
        
        // 挂载子DOM
        appendAllChildren(instance, workInProgress);
        
        // 保存DOM实例
        workInProgress.stateNode = instance;
        
        // 初始化DOM属性
        finalizeInitialChildren(instance, workInProgress.type, newProps);
      }
      return null;
    }
    
    case HostText: {  // 文本节点
      const newText = newProps;
      if (current !== null && workInProgress.stateNode != null) {
        // 更新文本
        updateHostText(current, workInProgress, newText);
      } else {
        // 创建文本节点
        workInProgress.stateNode = createTextInstance(newText);
      }
      return null;
    }
  }
}

function appendAllChildren(parent, workInProgress) {
  let node = workInProgress.child;
  while (node !== null) {
    if (node.tag === HostComponent || node.tag === HostText) {
      // 挂载DOM
      parent.appendChild(node.stateNode);
    } else if (node.child !== null) {
      // 跳过组件节点，继续找DOM节点
      node.child.return = node;
      node = node.child;
      continue;
    }
    
    if (node === workInProgress) {
      return;
    }
    
    // 遍历兄弟节点
    while (node.sibling === null) {
      if (node.return === null || node.return === workInProgress) {
        return;
      }
      node = node.return;
    }
    
    node.sibling.return = node.return;
    node = node.sibling;
  }
}
```

### 4.2 更新渲染流程

**触发更新的方式**：

```javascript
// 1. setState
this.setState({ count: 1 });

// 2. forceUpdate
this.forceUpdate();

// 3. useState的setter
setCount(1);

// 4. useReducer的dispatch
dispatch({ type: 'INCREMENT' });

// 5. ReactDOM.render（根节点更新）
ReactDOM.render(<App />, container);
```

**更新流程**：

```javascript
// 1. 创建Update对象
const update = {
  eventTime,        // 事件时间
  lane,             // 优先级
  action: payload,  // setState的参数
  next: null        // 链表
};

// 2. 将Update加入队列
enqueueUpdate(fiber, update);

// 3. 调度更新
scheduleUpdateOnFiber(fiber, lane, eventTime);
  ↓
ensureRootIsScheduled(root, eventTime);  // 确保根节点被调度
  ↓
scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));  // 同步调度
// 或
scheduleCallback(NormalSchedulerPriority, performConcurrentWorkOnRoot.bind(null, root));  // 异步调度
  ↓
workLoopSync() / workLoopConcurrent()  // 工作循环
  ↓
performUnitOfWork()  // 处理每个Fiber
  ├── beginWork()    // 对比新旧props，决定是否复用
  └── completeWork() // 标记effectTag
  ↓
commitRoot()  // 提交到DOM
```

**beginWork中的优化**：

```javascript
function beginWork(current, workInProgress) {
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    
    // 优化：props没变化，直接复用
    if (oldProps === newProps && workInProgress.type === current.type) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress);
    }
  }
  
  // props变化了，继续reconcile
  // ...
}

function bailoutOnAlreadyFinishedWork(current, workInProgress) {
  // 跳过当前节点，直接返回子节点
  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

### 4.3 Commit阶段

**三个子阶段**：

```javascript
function commitRoot(root) {
  const finishedWork = root.finishedWork;
  
  // 阶段1：Before Mutation（DOM变更前）
  commitBeforeMutationEffects(finishedWork);
  
  // 阶段2：Mutation（DOM变更）
  commitMutationEffects(finishedWork, root);
  
  // 切换current指针
  root.current = finishedWork;
  
  // 阶段3：Layout（DOM变更后）
  commitLayoutEffects(finishedWork, root);
}
```

**阶段1：Before Mutation**：

```javascript
function commitBeforeMutationEffects(firstChild) {
  let fiber = firstChild;
  
  while (fiber !== null) {
    // 处理有Snapshot effect的节点
    if (fiber.effectTag & Snapshot) {
      if (fiber.tag === ClassComponent) {
        // 执行getSnapshotBeforeUpdate
        const instance = fiber.stateNode;
        const snapshot = instance.getSnapshotBeforeUpdate(
          fiber.elementType === fiber.type
            ? prevProps
            : resolveDefaultProps(fiber.type, prevProps),
          prevState,
        );
        instance.__reactInternalSnapshotBeforeUpdate = snapshot;
      }
    }
    
    fiber = fiber.nextEffect;
  }
}
```

**阶段2：Mutation**：

```javascript
function commitMutationEffects(firstChild, root) {
  let fiber = firstChild;
  
  while (fiber !== null) {
    const effectTag = fiber.effectTag;
    
    // 1. 重置文本内容
    if (effectTag & ContentReset) {
      commitResetTextContent(fiber);
    }
    
    // 2. 处理ref
    if (effectTag & Ref) {
      const current = fiber.alternate;
      if (current !== null) {
        commitDetachRef(current);  // 解绑旧ref
      }
    }
    
    // 3. 根据effectTag执行DOM操作
    const primaryEffectTag = effectTag & (Placement | Update | Deletion);
    
    switch (primaryEffectTag) {
      case Placement: {
        // 插入DOM
        commitPlacement(fiber);
        fiber.effectTag &= ~Placement;  // 清除标记
        break;
      }
      case Update: {
        // 更新DOM
        const current = fiber.alternate;
        commitWork(current, fiber);
        break;
      }
      case Deletion: {
        // 删除DOM
        commitDeletion(root, fiber);
        break;
      }
    }
    
    fiber = fiber.nextEffect;
  }
}

function commitPlacement(fiber) {
  // 1. 找到父DOM节点
  const parentFiber = getHostParentFiber(fiber);
  const parent = parentFiber.stateNode;
  
  // 2. 找到插入位置（兄弟节点）
  const before = getHostSibling(fiber);
  
  // 3. 执行DOM插入
  if (before) {
    parent.insertBefore(fiber.stateNode, before);
  } else {
    parent.appendChild(fiber.stateNode);
  }
}

function commitWork(current, fiber) {
  switch (fiber.tag) {
    case HostComponent: {
      const instance = fiber.stateNode;
      if (instance != null) {
        const newProps = fiber.memoizedProps;
        const oldProps = current !== null ? current.memoizedProps : newProps;
        const type = fiber.type;
        
        // 更新DOM属性
        updateProperties(instance, type, oldProps, newProps);
      }
      return;
    }
    case HostText: {
      const textInstance = fiber.stateNode;
      const newText = fiber.memoizedProps;
      const oldText = current !== null ? current.memoizedProps : newText;
      
      // 更新文本内容
      commitTextUpdate(textInstance, oldText, newText);
      return;
    }
  }
}
```

**阶段3：Layout**：

```javascript
function commitLayoutEffects(firstChild, root) {
  let fiber = firstChild;
  
  while (fiber !== null) {
    const effectTag = fiber.effectTag;
    
    // 1. 执行生命周期
    if (effectTag & (Update | Callback)) {
      if (fiber.tag === ClassComponent) {
        const instance = fiber.stateNode;
        if (fiber.effectTag & Update) {
          if (current === null) {
            // 挂载：执行componentDidMount
            instance.componentDidMount();
          } else {
            // 更新：执行componentDidUpdate
            instance.componentDidUpdate(prevProps, prevState, snapshot);
          }
        }
      } else if (fiber.tag === FunctionComponent) {
        // 执行useLayoutEffect的create函数
        commitHookEffectListMount(HookLayout | HookHasEffect, fiber);
      }
    }
    
    // 2. 绑定ref
    if (effectTag & Ref) {
      commitAttachRef(fiber);
    }
    
    fiber = fiber.nextEffect;
  }
  
  // 3. 调度useEffect（异步执行）
  scheduleCallback(NormalSchedulerPriority, () => {
    flushPassiveEffects();
    return null;
  });
}
```

---

## 5. 虚拟DOM与Diff算法

### 5.1 虚拟DOM的结构

**JSX → React Element → Fiber**：

```jsx
// 1. JSX
const element = (
  <div className="app">
    <h1>标题</h1>
    <p>内容</p>
  </div>
);

// 2. 编译成React.createElement
const element = React.createElement(
  'div',
  { className: 'app' },
  React.createElement('h1', null, '标题'),
  React.createElement('p', null, '内容')
);

// 3. 返回React Element（虚拟DOM）
{
  $$typeof: Symbol(react.element),
  type: 'div',
  key: null,
  ref: null,
  props: {
    className: 'app',
    children: [
      { $$typeof: Symbol(react.element), type: 'h1', props: { children: '标题' } },
      { $$typeof: Symbol(react.element), type: 'p', props: { children: '内容' } }
    ]
  }
}

// 4. 转换成Fiber节点
{
  type: 'div',
  key: null,
  stateNode: HTMLDivElement,
  child: Fiber(h1),
  sibling: null,
  return: Fiber(parent),
  memoizedProps: { className: 'app' },
  // ...
}
```

### 5.2 Diff算法的三大策略

**策略1：Tree Diff - 分层比较**：

```javascript
// 只比较同层级节点，不跨层级比较
旧树：              新树：
  A                  A
 / \                / \
B   C      ====>   D   C
   / \                / \
  D   E              B   E

// React的Diff：
// 1. A层：A相同，继续
// 2. B/C层：B→D（删除B，新增D），C相同
// 3. D/E层：删除D、E，新增B、E

// 实际上只是D节点移动了，但React会删除重建
// 原因：跨层级比较的时间复杂度是O(n³)，分层比较是O(n)
```

**策略2：Component Diff - 组件比较**：

```javascript
// 类型相同：复用实例，更新props
<Counter count={1} />  →  <Counter count={2} />  // 复用

// 类型不同：销毁重建
<Counter />  →  <Timer />  // 销毁Counter，新建Timer
```

**策略3：Element Diff - 元素比较**：

```javascript
// 使用key优化列表
// 旧列表：[A, B, C, D]
// 新列表：[B, A, D, C]

// 没有key：删除重建（4次操作）
删除A → 删除B → 删除C → 删除D → 新建B → 新建A → 新建D → 新建C

// 有key：移动复用（4次移动）
移动B → 保持A → 移动D → 保持C
```

### 5.3 单节点Diff

```javascript
function reconcileSingleElement(
  returnFiber,
  currentFirstChild,
  element
) {
  const key = element.key;
  let child = currentFirstChild;
  
  // 遍历旧子节点
  while (child !== null) {
    // 1. 比较key
    if (child.key === key) {
      // 2. 比较type
      if (child.elementType === element.type) {
        // key和type都相同：复用
        deleteRemainingChildren(returnFiber, child.sibling);  // 删除其他兄弟节点
        const existing = useFiber(child, element.props);  // 复用Fiber
        existing.return = returnFiber;
        return existing;
      } else {
        // key相同但type不同：删除所有旧节点
        deleteRemainingChildren(returnFiber, child);
        break;
      }
    } else {
      // key不同：删除当前节点，继续比较下一个
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }
  
  // 3. 创建新节点
  const created = createFiberFromElement(element);
  created.return = returnFiber;
  return created;
}
```

### 5.4 多节点Diff

**算法流程**：

```javascript
function reconcileChildrenArray(
  returnFiber,
  currentFirstChild,
  newChildren
) {
  let resultingFirstChild = null;  // 返回的第一个子Fiber
  let previousNewFiber = null;     // 上一个新Fiber
  let oldFiber = currentFirstChild;  // 当前处理的旧Fiber
  let lastPlacedIndex = 0;         // 最后一个可复用节点的位置
  let newIdx = 0;                  // 新节点的索引
  let nextOldFiber = null;
  
  // 第一轮：处理相同位置的节点
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }
    
    // 尝试复用节点（key和type都相同）
    const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx]);
    
    if (newFiber === null) {
      // 无法复用，跳出第一轮
      if (oldFiber === null) {
        oldFiber = nextOldFiber;
      }
      break;
    }
    
    if (shouldTrackSideEffects) {
      if (oldFiber && newFiber.alternate === null) {
        // 新节点没有复用旧节点，删除旧节点
        deleteChild(returnFiber, oldFiber);
      }
    }
    
    // 标记插入位置
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    
    // 构建新Fiber链表
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }
  
  // 第二轮：新节点已经遍历完，删除剩余旧节点
  if (newIdx === newChildren.length) {
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }
  
  // 第三轮：旧节点已经遍历完，插入剩余新节点
  if (oldFiber === null) {
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx]);
      if (newFiber === null) continue;
      
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
    return resultingFirstChild;
  }
  
  // 第四轮：新旧节点都没遍历完，使用key优化
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
  
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber,
      newIdx,
      newChildren[newIdx]
    );
    
    if (newFiber !== null) {
      if (shouldTrackSideEffects) {
        if (newFiber.alternate !== null) {
          // 节点被复用，从map中删除
          existingChildren.delete(
            newFiber.key === null ? newIdx : newFiber.key
          );
        }
      }
      
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
  }
  
  // 删除map中剩余的旧节点
  if (shouldTrackSideEffects) {
    existingChildren.forEach(child => deleteChild(returnFiber, child));
  }
  
  return resultingFirstChild;
}
```

**实际案例分析**：

```jsx
// 旧列表
<ul>
  <li key="A">A</li>
  <li key="B">B</li>
  <li key="C">C</li>
  <li key="D">D</li>
</ul>

// 新列表
<ul>
  <li key="B">B</li>
  <li key="A">A</li>
  <li key="D">D</li>
  <li key="C">C</li>
</ul>

// Diff过程：
// 第一轮：
// newIdx=0, oldFiber=A, newChild=B → key不同，跳出

// 第四轮（构建map）：
// map = { A: FiberA, B: FiberB, C: FiberC, D: FiberD }

// newIdx=0, newChild=B:
//   从map中找到FiberB，复用
//   FiberB.index=1 > lastPlacedIndex=0，不需要移动，lastPlacedIndex=1

// newIdx=1, newChild=A:
//   从map中找到FiberA，复用
//   FiberA.index=0 < lastPlacedIndex=1，需要移动（标记Placement）

// newIdx=2, newChild=D:
//   从map中找到FiberD，复用
//   FiberD.index=3 > lastPlacedIndex=1，不需要移动，lastPlacedIndex=3

// newIdx=3, newChild=C:
//   从map中找到FiberC，复用
//   FiberC.index=2 < lastPlacedIndex=3，需要移动（标记Placement）

// 结果：B保持不动，A移动到B后面，D保持不动，C移动到D后面
```

**placeChild函数**：

```javascript
function placeChild(newFiber, lastPlacedIndex, newIndex) {
  newFiber.index = newIndex;
  
  if (!shouldTrackSideEffects) {
    // 首次渲染，不需要移动
    return lastPlacedIndex;
  }
  
  const current = newFiber.alternate;
  if (current !== null) {
    // 节点复用
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // 需要移动
      newFiber.effectTag = Placement;
      return lastPlacedIndex;
    } else {
      // 不需要移动
      return oldIndex;
    }
  } else {
    // 新节点
    newFiber.effectTag = Placement;
    return lastPlacedIndex;
  }
}
```

---

## 6. Fiber架构与并发渲染

### 6.1 为什么需要Fiber？

**React 15的问题（Stack Reconciler）**：

```javascript
// 递归遍历组件树
function mountComponent(vnode) {
  // 1. 创建DOM
  const dom = document.createElement(vnode.type);
  
  // 2. 递归处理子节点
  vnode.children.forEach(child => {
    const childDom = mountComponent(child);  // 递归
    dom.appendChild(childDom);
  });
  
  return dom;
}

// 问题：
// 1. 递归无法中断：一旦开始就必须完成
// 2. 长任务阻塞主线程：导致页面卡顿
// 3. 无法实现优先级调度：无法区分紧急/不紧急的更新
```

**Fiber的解决方案**：

```
1. 链表结构：child/sibling/return，可以随时中断和恢复
2. 时间切片：workLoop中判断时间，超时就暂停
3. 优先级调度：不同更新有不同优先级
4. 并发模式：可以同时准备多个版本的UI
```

### 6.2 Fiber节点的完整结构

```javascript
function FiberNode(tag, pendingProps, key) {
  // ===== 实例属性 =====
  this.tag = tag;                    // Fiber类型（FunctionComponent/ClassComponent/HostComponent...）
  this.key = key;                    // key
  this.elementType = null;           // React元素类型
  this.type = null;                  // 组件类型（函数/类/标签名）
  this.stateNode = null;             // 真实DOM节点/类组件实例
  
  // ===== Fiber链表结构 =====
  this.return = null;                // 父Fiber
  this.child = null;                 // 第一个子Fiber
  this.sibling = null;               // 下一个兄弟Fiber
  this.index = 0;                    // 在父节点中的索引
  
  this.ref = null;                   // ref引用
  
  // ===== 工作单元 =====
  this.pendingProps = pendingProps;  // 新的props
  this.memoizedProps = null;         // 上一次渲染的props
  this.updateQueue = null;           // 更新队列（setState的参数）
  this.memoizedState = null;         // 上一次渲染的state
  this.dependencies = null;          // 依赖（Context/Subscription）
  
  this.mode = NoMode;                // 模式（ConcurrentMode/StrictMode...）
  
  // ===== 副作用 =====
  this.effectTag = NoEffect;         // 副作用标记（Placement/Update/Deletion...）
  this.nextEffect = null;            // 下一个有副作用的Fiber（链表）
  this.firstEffect = null;           // 第一个子Fiber的副作用
  this.lastEffect = null;            // 最后一个子Fiber的副作用
  
  // ===== 优先级 =====
  this.lanes = NoLanes;              // 本次更新的优先级
  this.childLanes = NoLanes;         // 子树的优先级
  
  // ===== Double Buffering =====
  this.alternate = null;             // 指向另一棵树的相同节点（Current ↔ WorkInProgress）
}
```

### 6.3 双缓冲机制（Double Buffering）

```javascript
// 两棵Fiber树
FiberRoot
  ├── current ───────→ Current Fiber Tree（当前显示的树）
  └── workInProgress → WorkInProgress Fiber Tree（正在构建的树）

// 每个Fiber节点都有alternate指针
CurrentFiber.alternate = WorkInProgressFiber;
WorkInProgressFiber.alternate = CurrentFiber;

// 渲染流程：
// 1. 基于Current树创建WorkInProgress树
// 2. 在WorkInProgress树上进行Diff和更新
// 3. Commit阶段：将WorkInProgress树应用到DOM
// 4. 交换指针：current = workInProgress

// 优点：
// 1. 可中断：WorkInProgress树的构建可以中断，Current树保持不变
// 2. 快速回滚：构建失败时直接丢弃WorkInProgress树
// 3. 内存复用：下次更新时复用Fiber节点
```

**createWorkInProgress实现**：

```javascript
function createWorkInProgress(current, pendingProps) {
  let workInProgress = current.alternate;
  
  if (workInProgress === null) {
    // 首次渲染：创建新Fiber
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode
    );
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;
    
    // 建立双向连接
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    // 更新：复用Fiber
    workInProgress.pendingProps = pendingProps;
    workInProgress.type = current.type;
    
    // 清空副作用
    workInProgress.effectTag = NoEffect;
    workInProgress.nextEffect = null;
    workInProgress.firstEffect = null;
    workInProgress.lastEffect = null;
  }
  
  // 复制其他属性
  workInProgress.childLanes = current.childLanes;
  workInProgress.lanes = current.lanes;
  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;
  
  return workInProgress;
}
```

### 6.4 时间切片（Time Slicing）

**workLoop的实现**：

```javascript
function workLoopConcurrent() {
  // 当有工作 && 还没到yield时间，继续工作
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function shouldYield() {
  const currentTime = performance.now();
  return currentTime >= deadline;  // 超过5ms就yield
}

// Scheduler的调度
function scheduleCallback(priorityLevel, callback) {
  const currentTime = performance.now();
  const timeout = timeoutForPriorityLevel(priorityLevel);
  const expirationTime = currentTime + timeout;
  
  const newTask = {
    callback,
    priorityLevel,
    expirationTime,
    sortIndex: expirationTime  // 用于排序
  };
  
  // 加入任务队列（小顶堆）
  push(taskQueue, newTask);
  
  // 请求调度
  requestHostCallback(flushWork);
}

function flushWork(hasTimeRemaining, initialTime) {
  return workLoop(hasTimeRemaining, initialTime);
}

function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  
  // 取出优先级最高的任务
  currentTask = peek(taskQueue);
  
  while (currentTask !== null) {
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // 时间到了或需要yield，暂停
      break;
    }
    
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      const continuationCallback = callback(currentTask.expirationTime <= currentTime);
      
      currentTime = performance.now();
      
      if (typeof continuationCallback === 'function') {
        // 任务没完成，继续执行
        currentTask.callback = continuationCallback;
      } else {
        // 任务完成，移除
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
    } else {
      pop(taskQueue);
    }
    
    currentTask = peek(taskQueue);
  }
  
  // 返回是否还有任务
  return currentTask !== null;
}
```

**浏览器空闲时间调度**：

```javascript
// requestIdleCallback的polyfill
const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = performance.now();
    const hasTimeRemaining = frameDeadline - currentTime > 0;
    
    try {
      const hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
      
      if (!hasMoreWork) {
        scheduledHostCallback = null;
      } else {
        // 还有工作，继续调度
        port.postMessage(null);
      }
    } catch (error) {
      // 重新调度并抛出错误
      port.postMessage(null);
      throw error;
    }
  }
};

function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  port.postMessage(null);  // 触发宏任务
}
```

### 6.5 优先级调度（Lane模型）

**Lane的定义**：

```javascript
// 使用二进制位表示优先级
const NoLanes = 0b0000000000000000000000000000000;
const SyncLane = 0b0000000000000000000000000000001;  // 同步（最高优先级）
const InputContinuousLane = 0b0000000000000000000000000000100;  // 连续输入
const DefaultLane = 0b0000000000000000000000000010000;  // 默认
const IdleLane = 0b0100000000000000000000000000000;  // 空闲（最低优先级）

// 优势：
// 1. 高效的位运算
// 2. 支持批量操作（多个更新合并成一个Lane）
// 3. 灵活的优先级管理
```

**不同优先级的触发场景**：

```javascript
// 1. SyncLane：同步更新（不可中断）
ReactDOM.flushSync(() => {
  setState(1);  // 立即同步执行
});

// 2. InputContinuousLane：用户输入
<input onChange={e => setState(e.target.value)} />  // 连续输入

// 3. DefaultLane：普通更新
useEffect(() => {
  setState(1);  // 默认优先级
}, []);

// 4. IdleLane：空闲更新
startTransition(() => {
  setState(1);  // 可被打断
});
```

**优先级的处理**：

```javascript
function ensureRootIsScheduled(root, currentTime) {
  const nextLanes = getNextLanes(root, NoLanes);
  
  if (nextLanes === NoLanes) {
    // 没有待处理的更新
    return;
  }
  
  const newCallbackPriority = getHighestPriorityLane(nextLanes);
  const existingCallbackPriority = root.callbackPriority;
  
  if (existingCallbackPriority === newCallbackPriority) {
    // 优先级相同，不需要重新调度
    return;
  }
  
  // 取消旧任务
  if (existingCallbackNode !== null) {
    cancelCallback(existingCallbackNode);
  }
  
  // 调度新任务
  let newCallbackNode;
  if (newCallbackPriority === SyncLane) {
    // 同步优先级
    scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    newCallbackNode = null;
  } else {
    // 异步优先级
    const schedulerPriorityLevel = lanePriorityToSchedulerPriority(newCallbackPriority);
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root)
    );
  }
  
  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

### 6.6 Suspense与并发渲染

**Suspense的使用**：

```jsx
import { lazy, Suspense } from 'react';

const LazyComponent = lazy(() => import('./LazyComponent'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <LazyComponent />
    </Suspense>
  );
}
```

**Suspense的原理**：

```javascript
// 1. 组件throw一个Promise
function LazyComponent() {
  const resource = fetchData();  // 如果数据没准备好，throw promise
  return <div>{resource.data}</div>;
}

// 2. React捕获Promise
function throwException(root, returnFiber, sourceFiber, value) {
  if (value !== null && typeof value === 'object' && typeof value.then === 'function') {
    // 这是一个Promise，进入Suspense流程
    const wakeable = value;
    
    // 标记为Incomplete
    sourceFiber.effectTag |= Incomplete;
    
    // 找到最近的Suspense边界
    const suspenseBoundary = findSuspenseBoundary(returnFiber);
    
    // 挂载Promise的回调
    wakeable.then(
      () => {
        // 数据准备好了，重新渲染
        retryTimedOutBoundary(suspenseBoundary);
      }
    );
    
    // 渲染fallback
    renderFallback(suspenseBoundary);
  }
}

// 3. Promise resolve后重新渲染
function retryTimedOutBoundary(boundaryFiber) {
  const retryLane = DefaultLane;
  scheduleUpdateOnFiber(boundaryFiber, retryLane);
}
```

**并发特性**：

```jsx
import { useTransition } from 'react';

function App() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('about');
  
  function selectTab(nextTab) {
    // 将更新标记为低优先级（可被打断）
    startTransition(() => {
      setTab(nextTab);
    });
  }
  
  return (
    <div>
      <button onClick={() => selectTab('about')}>About</button>
      <button onClick={() => selectTab('posts')}>Posts</button>
      {isPending && <Spinner />}
      <TabContent tab={tab} />
    </div>
  );
}
```

---

（待续：第三部分 - 生命周期与Hooks）

---

**当前进度**：第二部分完成（渲染机制）  
**文件规模**：约600行  
**下一部分**：生命周期、Hooks实现原理、自定义Hooks...# React框架深入 - Part 3：生命周期、Hooks与框架对比

> 接续Part 2：渲染机制  
> 本部分：完整生命周期、Hooks实现原理、React vs Vue vs Angular

---

## 7. 完整的生命周期

### 7.1 类组件的生命周期

**完整流程图**：

```
挂载阶段（Mounting）：
constructor()
  ↓
static getDerivedStateFromProps()
  ↓
render()
  ↓
componentDidMount()

更新阶段（Updating）：
static getDerivedStateFromProps()
  ↓
shouldComponentUpdate()
  ↓
render()
  ↓
getSnapshotBeforeUpdate()
  ↓
componentDidUpdate()

卸载阶段（Unmounting）：
componentWillUnmount()

错误处理：
static getDerivedStateFromError()
componentDidCatch()
```

**详细说明**：

```jsx
class LifeCycleDemo extends React.Component {
  // 1. 构造函数（只执行一次）
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    console.log('1. constructor');
  }
  
  // 2. 静态方法：根据props计算state
  static getDerivedStateFromProps(props, state) {
    console.log('2. getDerivedStateFromProps');
    // 返回新的state，或null表示不更新
    if (props.count !== state.count) {
      return { count: props.count };
    }
    return null;
  }
  
  // 3. 判断是否需要更新（性能优化）
  shouldComponentUpdate(nextProps, nextState) {
    console.log('3. shouldComponentUpdate');
    // 返回false可以阻止更新
    return nextState.count !== this.state.count;
  }
  
  // 4. 渲染（纯函数）
  render() {
    console.log('4. render');
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={() => this.setState({ count: this.state.count + 1 })}>
          +1
        </button>
      </div>
    );
  }
  
  // 5. DOM更新前的快照（用于记录滚动位置等）
  getSnapshotBeforeUpdate(prevProps, prevState) {
    console.log('5. getSnapshotBeforeUpdate');
    return { scrollTop: window.scrollY };  // 返回值会传给componentDidUpdate
  }
  
  // 6. 挂载完成
  componentDidMount() {
    console.log('6. componentDidMount');
    // 执行副作用：数据请求、订阅、DOM操作
  }
  
  // 7. 更新完成
  componentDidUpdate(prevProps, prevState, snapshot) {
    console.log('7. componentDidUpdate', snapshot);
    // 可以执行DOM操作、数据请求（需要判断props/state是否变化）
    if (this.state.count !== prevState.count) {
      console.log('count changed');
    }
  }
  
  // 8. 卸载前
  componentWillUnmount() {
    console.log('8. componentWillUnmount');
    // 清理：取消订阅、清除定时器
  }
  
  // 9. 错误捕获（静态方法）
  static getDerivedStateFromError(error) {
    console.log('9. getDerivedStateFromError', error);
    return { hasError: true };  // 更新state以显示错误UI
  }
  
  // 10. 错误处理
  componentDidCatch(error, errorInfo) {
    console.log('10. componentDidCatch', error, errorInfo);
    // 记录错误日志
  }
}
```

**执行顺序**：

```
首次渲染：
1. constructor
2. getDerivedStateFromProps
4. render
6. componentDidMount

setState触发更新：
2. getDerivedStateFromProps
3. shouldComponentUpdate（返回true）
4. render
5. getSnapshotBeforeUpdate
7. componentDidUpdate

父组件传入新props：
2. getDerivedStateFromProps
3. shouldComponentUpdate
4. render
5. getSnapshotBeforeUpdate
7. componentDidUpdate

组件卸载：
8. componentWillUnmount
```

**已废弃的生命周期**：

```jsx
// ❌ 已废弃（React 17移除）
componentWillMount()           // 用componentDidMount代替
componentWillReceiveProps()    // 用getDerivedStateFromProps代替
componentWillUpdate()          // 用getSnapshotBeforeUpdate代替

// 为什么废弃？
// 1. 与Fiber架构不兼容（可能被多次调用）
// 2. 不适合异步渲染
// 3. 容易被误用（在will*中setState导致死循环）
```

### 7.2 函数组件的生命周期（Hooks）

**Hooks与类组件生命周期的对应**：

| 类组件 | Hooks | 说明 |
|--------|-------|------|
| constructor | useState惰性初始化 | `useState(() => initialState)` |
| componentDidMount | useEffect(() => {}, []) | 空依赖数组 |
| componentDidUpdate | useEffect(() => {}) | 无依赖数组 |
| componentWillUnmount | useEffect(() => return cleanup, []) | 返回清理函数 |
| shouldComponentUpdate | React.memo | 包裹组件 |
| getDerivedStateFromProps | 在render中计算 | 直接使用props |
| getDerivedStateFromError | 无 | 只能用Error Boundary |
| componentDidCatch | 无 | 只能用Error Boundary |

**完整示例**：

```jsx
function LifeCycleHooks() {
  // === constructor ===
  const [count, setCount] = useState(() => {
    console.log('惰性初始化（只执行一次）');
    return 0;
  });
  
  // === componentDidMount ===
  useEffect(() => {
    console.log('挂载：执行一次');
    
    // === componentWillUnmount ===
    return () => {
      console.log('卸载：清理资源');
    };
  }, []);  // 空依赖
  
  // === componentDidUpdate ===
  useEffect(() => {
    console.log('更新：每次render都执行');
  });  // 无依赖
  
  // === componentDidUpdate（依赖特定值）===
  useEffect(() => {
    console.log('count变化时执行');
  }, [count]);  // 依赖count
  
  // === getDerivedStateFromProps ===
  // 不需要Hook，直接在render中计算
  const doubleCount = count * 2;
  
  // === render ===
  return (
    <div>
      <p>Count: {count}</p>
      <p>Double: {doubleCount}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}

// === shouldComponentUpdate ===
const MemoizedComponent = React.memo(LifeCycleHooks, (prevProps, nextProps) => {
  // 返回true表示不更新，false表示更新（与shouldComponentUpdate相反）
  return prevProps.count === nextProps.count;
});
```

**useLayoutEffect vs useEffect**：

```jsx
function LayoutEffectDemo() {
  const [count, setCount] = useState(0);
  
  // useEffect：DOM更新后异步执行（不阻塞渲染）
  useEffect(() => {
    console.log('useEffect:', count);
    // 适用场景：数据请求、订阅、日志
  }, [count]);
  
  // useLayoutEffect：DOM更新后同步执行（阻塞渲染）
  useLayoutEffect(() => {
    console.log('useLayoutEffect:', count);
    // 适用场景：测量DOM、同步DOM操作
    
    // 例如：避免闪烁
    const element = document.getElementById('box');
    if (element && count > 0) {
      element.style.backgroundColor = 'red';  // 用户看不到中间状态
    }
  }, [count]);
  
  return (
    <div>
      <div id="box" style={{ width: 100, height: 100 }}>Box</div>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}

// 执行顺序：
// 1. render
// 2. DOM更新
// 3. useLayoutEffect（同步）
// 4. 浏览器绘制
// 5. useEffect（异步）
```

---

## 8. Hooks深入与实现原理

### 8.1 Hooks的存储结构

**Fiber节点上的memoizedState**：

```javascript
// Fiber节点
{
  memoizedState: {
    // Hook链表的第一个节点
    memoizedState: 0,          // useState的值
    baseState: 0,
    baseQueue: null,
    queue: {                   // 更新队列
      pending: null,
      dispatch: setCount,
      lastRenderedReducer: basicStateReducer,
      lastRenderedState: 0
    },
    next: {                    // 下一个Hook节点
      memoizedState: [],       // useEffect的依赖数组
      baseState: null,
      baseQueue: null,
      queue: null,
      next: {                  // 再下一个Hook
        memoizedState: callback,  // useCallback的函数
        baseState: null,
        baseQueue: null,
        queue: null,
        next: null
      }
    }
  }
}
```

**Hook链表的构建**：

```jsx
function Component() {
  const [count, setCount] = useState(0);      // Hook1
  const [name, setName] = useState('张三');    // Hook2
  useEffect(() => {}, [count]);               // Hook3
  const increment = useCallback(() => {}, []); // Hook4
  
  // 对应的Hook链表：
  // Hook1(useState) → Hook2(useState) → Hook3(useEffect) → Hook4(useCallback)
}
```

### 8.2 useState的实现原理

**简化版实现**：

```javascript
// 全局变量
let workInProgressHook = null;  // 当前正在工作的Hook
let currentHook = null;         // 旧Hook链表
let currentlyRenderingFiber = null;  // 当前渲染的Fiber

function useState(initialState) {
  return useReducer(
    basicStateReducer,
    initialState
  );
}

function basicStateReducer(state, action) {
  return typeof action === 'function' ? action(state) : action;
}

function useReducer(reducer, initialState) {
  // 1. 创建或复用Hook对象
  const hook = updateWorkInProgressHook();
  
  // 2. 计算新state
  const queue = hook.queue;
  
  if (queue !== null) {
    // 有更新：计算新state
    const first = queue.pending;
    
    if (first !== null) {
      let newState = hook.memoizedState;
      let update = first;
      
      do {
        // 执行reducer
        const action = update.action;
        newState = reducer(newState, action);
        update = update.next;
      } while (update !== first);
      
      hook.memoizedState = newState;
      hook.baseState = newState;
      queue.pending = null;
    }
  } else {
    // 首次渲染：初始化state
    hook.memoizedState = hook.baseState =
      typeof initialState === 'function' ? initialState() : initialState;
    
    // 创建更新队列
    const queue = {
      pending: null,
      dispatch: null,
      lastRenderedReducer: reducer,
      lastRenderedState: hook.memoizedState
    };
    hook.queue = queue;
    
    // 创建dispatch函数
    const dispatch = dispatchAction.bind(null, currentlyRenderingFiber, queue);
    queue.dispatch = dispatch;
  }
  
  return [hook.memoizedState, queue.dispatch];
}

function updateWorkInProgressHook() {
  // 1. 从旧Hook链表中取出Hook
  let nextCurrentHook;
  if (currentHook === null) {
    // 链表的第一个Hook
    const current = currentlyRenderingFiber.alternate;
    nextCurrentHook = current !== null ? current.memoizedState : null;
  } else {
    // 链表的后续Hook
    nextCurrentHook = currentHook.next;
  }
  
  // 2. 创建新Hook
  let nextWorkInProgressHook;
  if (workInProgressHook === null) {
    // 链表的第一个Hook
    nextWorkInProgressHook = {
      memoizedState: null,
      baseState: null,
      baseQueue: null,
      queue: null,
      next: null
    };
    currentlyRenderingFiber.memoizedState = nextWorkInProgressHook;
  } else {
    // 链表的后续Hook
    nextWorkInProgressHook = {
      memoizedState: null,
      baseState: null,
      baseQueue: null,
      queue: null,
      next: null
    };
    workInProgressHook.next = nextWorkInProgressHook;
  }
  
  // 3. 复制旧Hook的值
  if (nextCurrentHook !== null) {
    nextWorkInProgressHook.memoizedState = nextCurrentHook.memoizedState;
    nextWorkInProgressHook.baseState = nextCurrentHook.baseState;
    nextWorkInProgressHook.baseQueue = nextCurrentHook.baseQueue;
    nextWorkInProgressHook.queue = nextCurrentHook.queue;
  }
  
  // 4. 移动指针
  workInProgressHook = nextWorkInProgressHook;
  currentHook = nextCurrentHook;
  
  return workInProgressHook;
}

function dispatchAction(fiber, queue, action) {
  // 1. 创建update对象
  const update = {
    action,
    next: null
  };
  
  // 2. 将update加入环形链表
  const pending = queue.pending;
  if (pending === null) {
    // 第一个update
    update.next = update;  // 指向自己
  } else {
    // 插入链表
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;
  
  // 3. 调度更新
  scheduleUpdateOnFiber(fiber);
}
```

**为什么useState是异步的？**

```javascript
// 批处理机制
let isBatchingUpdates = false;
let updateQueue = [];

function batchedUpdates(fn) {
  isBatchingUpdates = true;
  try {
    fn();  // 执行用户代码
  } finally {
    isBatchingUpdates = false;
    flushUpdates();  // 批量处理
  }
}

function setState(action) {
  if (isBatchingUpdates) {
    // 批处理中：加入队列
    updateQueue.push(action);
  } else {
    // 非批处理：立即执行
    executeUpdate(action);
  }
}

function flushUpdates() {
  updateQueue.forEach(action => executeUpdate(action));
  updateQueue = [];
}

// React 18的自动批处理
// 所有更新都会自动批处理（包括setTimeout、Promise等）
```

### 8.3 useEffect的实现原理

**简化版实现**：

```javascript
function useEffect(create, deps) {
  return updateEffectImpl(
    PassiveEffect,  // effect标记
    HookPassive,    // hook标记
    create,
    deps
  );
}

function useLayoutEffect(create, deps) {
  return updateEffectImpl(
    UpdateEffect,   // effect标记（同步执行）
    HookLayout,
    create,
    deps
  );
}

function updateEffectImpl(fiberEffectTag, hookEffectTag, create, deps) {
  // 1. 获取Hook
  const hook = updateWorkInProgressHook();
  
  // 2. 检查依赖是否变化
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;
  
  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 依赖没变化：不执行effect
        hook.memoizedState = pushEffect(hookEffectTag, create, destroy, nextDeps);
        return;
      }
    }
  }
  
  // 3. 依赖变化：标记需要执行effect
  currentlyRenderingFiber.effectTag |= fiberEffectTag;
  
  hook.memoizedState = pushEffect(
    HookHasEffect | hookEffectTag,  // 标记HookHasEffect
    create,
    destroy,
    nextDeps
  );
}

function pushEffect(tag, create, destroy, deps) {
  const effect = {
    tag,
    create,      // effect函数
    destroy,     // 清理函数
    deps,        // 依赖数组
    next: null
  };
  
  // 构建effect环形链表
  let componentUpdateQueue = currentlyRenderingFiber.updateQueue;
  if (componentUpdateQueue === null) {
    // 第一个effect
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = componentUpdateQueue;
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    // 插入链表
    const lastEffect = componentUpdateQueue.lastEffect;
    const firstEffect = lastEffect.next;
    lastEffect.next = effect;
    effect.next = firstEffect;
    componentUpdateQueue.lastEffect = effect;
  }
  
  return effect;
}

function areHookInputsEqual(nextDeps, prevDeps) {
  if (prevDeps === null) return false;
  
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) continue;
    return false;
  }
  
  return true;
}
```

**effect的执行时机**：

```javascript
// Commit阶段的Layout子阶段
function commitLayoutEffects(finishedWork) {
  // ...
  
  // 调度useEffect（异步）
  schedulePassiveEffects(finishedWork);
  
  // 执行useLayoutEffect（同步）
  commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
}

function schedulePassiveEffects(finishedWork) {
  const updateQueue = finishedWork.updateQueue;
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    
    do {
      if ((effect.tag & HookPassive) !== NoHookEffect) {
        // 将effect加入调度队列
        enqueuePendingPassiveHookEffectUnmount(finishedWork, effect);
        enqueuePendingPassiveHookEffectMount(finishedWork, effect);
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}

// 异步执行useEffect
function flushPassiveEffects() {
  // 1. 执行所有清理函数
  pendingPassiveHookEffectsUnmount.forEach(effect => {
    const destroy = effect.destroy;
    if (destroy !== undefined) {
      destroy();  // 执行清理函数
    }
  });
  
  // 2. 执行所有effect函数
  pendingPassiveHookEffectsMount.forEach(effect => {
    const create = effect.create;
    effect.destroy = create();  // 保存清理函数
  });
}
```

### 8.4 useMemo与useCallback的实现

```javascript
function useMemo(nextCreate, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  
  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps = prevState[1];
      
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 依赖没变化：返回缓存值
        return prevState[0];
      }
    }
  }
  
  // 依赖变化：重新计算
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function useCallback(callback, deps) {
  // useCallback就是useMemo的语法糖
  return useMemo(() => callback, deps);
}
```

### 8.5 useRef的实现

```javascript
function useRef(initialValue) {
  const hook = updateWorkInProgressHook();
  
  if (currentHook === null) {
    // 首次渲染：创建ref对象
    const ref = { current: initialValue };
    hook.memoizedState = ref;
  }
  
  // 返回同一个ref对象（引用不变）
  return hook.memoizedState;
}

// ref对象的特点：
// 1. 引用不变：整个组件生命周期都是同一个对象
// 2. 修改不触发渲染：ref.current = newValue不会导致重新渲染
// 3. 持久化存储：保存在Fiber节点的memoizedState中
```

### 8.6 自定义Hook的最佳实践

**规则**：

```
1. 只在顶层调用Hooks：不能在条件语句、循环、嵌套函数中调用
2. 只在React函数中调用：函数组件或自定义Hook中
3. 命名以use开头：useXxx
```

**常用自定义Hook**：

```jsx
// 1. useFetch：数据请求
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    setLoading(true);
    fetch(url)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      });
    
    return () => {
      cancelled = true;  // 清理：取消请求
    };
  }, [url]);
  
  return { data, loading, error };
}

// 使用
function UserList() {
  const { data, loading, error } = useFetch('/api/users');
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  return <ul>{data.map(user => <li key={user.id}>{user.name}</li>)}</ul>;
}

// 2. useDebounce：防抖
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => {
      clearTimeout(timer);
    };
  }, [value, delay]);
  
  return debouncedValue;
}

// 使用
function SearchInput() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 500);
  
  useEffect(() => {
    // 只有停止输入500ms后才会触发搜索
    if (debouncedSearchTerm) {
      searchAPI(debouncedSearchTerm);
    }
  }, [debouncedSearchTerm]);
  
  return (
    <input
      value={searchTerm}
      onChange={e => setSearchTerm(e.target.value)}
      placeholder="搜索..."
    />
  );
}

// 3. useLocalStorage：持久化存储
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.log(error);
      return initialValue;
    }
  });
  
  const setValue = value => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.log(error);
    }
  };
  
  return [storedValue, setValue];
}

// 使用
function Settings() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  
  return (
    <div>
      <p>当前主题：{theme}</p>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        切换主题
      </button>
    </div>
  );
}

// 4. usePrevious：保存上一次的值
function usePrevious(value) {
  const ref = useRef();
  
  useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}

// 使用
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);
  
  return (
    <div>
      <p>当前值：{count}</p>
      <p>上一次的值：{prevCount}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}

// 5. useInterval：声明式定时器
function useInterval(callback, delay) {
  const savedCallback = useRef();
  
  // 保存最新的回调
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);
  
  // 设置定时器
  useEffect(() => {
    if (delay !== null) {
      const id = setInterval(() => savedCallback.current(), delay);
      return () => clearInterval(id);
    }
  }, [delay]);
}

// 使用
function Timer() {
  const [count, setCount] = useState(0);
  
  useInterval(() => {
    setCount(count + 1);
  }, 1000);
  
  return <div>计时：{count}秒</div>;
}
```

---

## 9. 自定义Hooks最佳实践

### 9.1 抽象逻辑

**原则**：将可复用的逻辑抽取成自定义Hook

```jsx
// ❌ 不好：逻辑重复
function UserProfile() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, []);
  
  // ...
}

function PostList() {
  const [posts, setPosts] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch('/api/posts')
      .then(res => res.json())
      .then(data => {
        setPosts(data);
        setLoading(false);
      });
  }, []);
  
  // ...
}

// ✅ 好：抽取成自定义Hook
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setLoading(false);
      });
  }, [url]);
  
  return { data, loading };
}

function UserProfile() {
  const { data: user, loading } = useFetch('/api/user');
  // ...
}

function PostList() {
  const { data: posts, loading } = useFetch('/api/posts');
  // ...
}
```

### 9.2 返回值设计

**选择合适的返回值类型**：

```jsx
// 1. 数组：适合解构重命名
const [count, setCount] = useState(0);
const [name, setName] = useState('');  // 可以任意命名

// 2. 对象：适合只使用部分值
const { data, loading, error, refetch } = useFetch('/api/users');
// 只使用部分值
const { data } = useFetch('/api/users');

// 3. 单个值：简单场景
const isOnline = useOnlineStatus();

// 自定义Hook的返回值建议：
// - 2个值以下：用数组
// - 3个值以上：用对象
```

### 9.3 处理清理

**总是清理副作用**：

```jsx
// ❌ 不好：没有清理
function useFetch(url) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => setData(data));  // 如果组件已卸载，会报错
  }, [url]);
  
  return data;
}

// ✅ 好：清理
function useFetch(url) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    fetch(url)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) {  // 检查是否已取消
          setData(data);
        }
      });
    
    return () => {
      cancelled = true;  // 清理：标记为已取消
    };
  }, [url]);
  
  return data;
}
```

---

（待续：第四部分 - 框架对比与性能优化）

---

**当前进度**：第三部分完成（生命周期与Hooks）  
**文件规模**：约600行  
**下一部分**：React vs Vue vs Angular、设计哲学、性能优化...# React框架深入 - Part 4：框架对比、设计哲学与实战

> 接续Part 3：生命周期与Hooks  
> 本部分：React vs Vue vs Angular、设计哲学、性能优化、面试题

---

## 10. React vs Vue vs Angular

### 10.1 渲染机制对比

#### React：虚拟DOM + Diff

**特点**：
- **推**模式（Push）：组件主动触发更新
- **不可变数据**：setState创建新对象
- **全量Diff**：对比整个组件树
- **手动优化**：需要React.memo/useMemo

**渲染流程**：

```javascript
// React的更新流程
setState(newValue)
  ↓
标记Fiber为dirty
  ↓
调度更新（Scheduler）
  ↓
重新执行render函数
  ↓
生成新的虚拟DOM
  ↓
Diff算法对比新旧虚拟DOM
  ↓
收集变化（effectList）
  ↓
批量更新DOM
```

**示例**：

```jsx
// React
function Counter() {
  const [count, setCount] = useState(0);
  
  // 每次setCount都会重新执行整个函数
  console.log('render');
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

#### Vue：响应式系统 + 细粒度依赖追踪

**特点**：
- **拉**模式（Pull）：响应式数据变化自动触发更新
- **可变数据**：直接修改对象属性
- **精准更新**：只更新依赖变化的组件
- **自动优化**：依赖追踪自动优化

**渲染流程**：

```javascript
// Vue 3的更新流程
count.value++  // 修改响应式数据
  ↓
触发setter（Proxy的set）
  ↓
通知依赖（trigger）
  ↓
将effect加入队列
  ↓
批量执行effect
  ↓
重新执行render函数
  ↓
生成新的虚拟DOM
  ↓
Diff算法对比（与React类似）
  ↓
更新DOM
```

**响应式系统实现**：

```javascript
// Vue 3的响应式系统（简化版）
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key) {
      // 收集依赖
      track(target, key);
      return target[key];
    },
    set(target, key, value) {
      target[key] = value;
      // 触发更新
      trigger(target, key);
      return true;
    }
  });
}

let activeEffect = null;
const targetMap = new WeakMap();  // { target: { key: [effect1, effect2] } }

function track(target, key) {
  if (activeEffect) {
    let depsMap = targetMap.get(target);
    if (!depsMap) {
      targetMap.set(target, (depsMap = new Map()));
    }
    
    let dep = depsMap.get(key);
    if (!dep) {
      depsMap.set(key, (dep = new Set()));
    }
    
    dep.add(activeEffect);  // 收集依赖
  }
}

function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  
  const dep = depsMap.get(key);
  if (dep) {
    dep.forEach(effect => effect());  // 触发所有依赖
  }
}

function effect(fn) {
  activeEffect = fn;
  fn();  // 执行，触发get，收集依赖
  activeEffect = null;
}
```

**示例**：

```vue
<!-- Vue 3 -->
<script setup>
import { ref } from 'vue';

const count = ref(0);

// count变化时，只有使用count的地方会更新
// Vue会精准追踪依赖，不需要手动优化
</script>

<template>
  <div>
    <p>{{ count }}</p>
    <button @click="count++">+1</button>
  </div>
</template>
```

#### Angular：脏检查（Change Detection）

**特点**：
- **Zone.js劫持异步**：自动触发变更检测
- **自顶向下检查**：从根组件开始检查整个树
- **可配置策略**：OnPush策略优化性能

**渲染流程**：

```javascript
// Angular的更新流程
异步事件（click/setTimeout/Promise）
  ↓
Zone.js捕获
  ↓
触发变更检测（Change Detection）
  ↓
从根组件开始，递归检查整个组件树
  ↓
对比每个组件的属性（脏检查）
  ↓
更新变化的DOM
```

**示例**：

```typescript
// Angular
@Component({
  selector: 'app-counter',
  template: `
    <div>
      <p>{{ count }}</p>
      <button (click)="increment()">+1</button>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush  // 优化：只在输入变化时检查
})
export class CounterComponent {
  count = 0;
  
  increment() {
    this.count++;  // 直接修改
  }
}
```

### 10.2 数据流对比

#### React：单向数据流 + 不可变数据

```jsx
// React
function Parent() {
  const [count, setCount] = useState(0);
  
  // 数据向下流动
  return <Child count={count} onIncrement={() => setCount(count + 1)} />;
}

function Child({ count, onIncrement }) {
  // 子组件不能直接修改props
  return (
    <div>
      <p>{count}</p>
      <button onClick={onIncrement}>+1</button>  {/* 通过回调通知父组件 */}
    </div>
  );
}

// 不可变数据
const user = { name: '张三', age: 25 };
// ❌ 错误：直接修改
user.age = 26;
setUser(user);  // React不会检测到变化

// ✅ 正确：创建新对象
setUser({ ...user, age: 26 });
```

**为什么React选择不可变数据？**

```
优点：
1. 简单的相等性检查：Object.is(prev, next)，O(1)
2. 易于追踪变化：直接比较引用
3. 时间旅行：保存历史状态（Redux DevTools）
4. 易于并发：不同版本的state互不影响

缺点：
1. 性能开销：频繁创建新对象
2. 深拷贝复杂：嵌套对象需要展开运算符
3. 学习成本：需要理解不可变概念
```

#### Vue：双向绑定 + 可变数据

```vue
<!-- Vue -->
<script setup>
import { ref } from 'vue';

const count = ref(0);
const user = reactive({ name: '张三', age: 25 });

// 可以直接修改
count.value++;
user.age = 26;  // Vue会自动检测到变化
</script>

<template>
  <!-- 双向绑定 -->
  <input v-model="user.name" />
  <p>{{ user.name }}</p>
  
  <!-- 父子组件双向绑定 -->
  <Child v-model="count" />
</template>
```

**为什么Vue选择可变数据？**

```
优点：
1. 直观：符合直觉，直接修改属性
2. 性能：不需要创建新对象
3. 易用：不需要深拷贝

缺点：
1. 复杂的依赖追踪：需要Proxy/Getter/Setter
2. 难以调试：不知道谁修改了数据
3. 不支持时间旅行：历史状态被覆盖
```

#### Angular：双向绑定 + 脏检查

```typescript
// Angular
@Component({
  template: `
    <!-- 双向绑定 -->
    <input [(ngModel)]="user.name" />
    <p>{{ user.name }}</p>
  `
})
export class UserComponent {
  user = { name: '张三', age: 25 };
  
  // 直接修改
  updateUser() {
    this.user.age = 26;  // Angular会在下次变更检测时发现
  }
}
```

### 10.3 组件通信对比

| 方式 | React | Vue | Angular |
|------|-------|-----|---------|
| **父→子** | props | props | @Input() |
| **子→父** | 回调函数 | $emit | @Output() EventEmitter |
| **跨组件** | Context | provide/inject | 依赖注入（DI） |
| **全局状态** | Redux/Zustand | Pinia/Vuex | Service单例 |
| **Ref** | forwardRef | ref | @ViewChild |
| **事件总线** | 第三方库 | mitt/EventBus | Subject/Observable |

### 10.4 性能优化对比

#### React的性能优化

```jsx
// 1. React.memo：缓存组件
const Child = React.memo(function Child({ count }) {
  return <div>{count}</div>;
});

// 2. useMemo：缓存计算结果
const expensiveValue = useMemo(() => {
  return computeExpensiveValue(a, b);
}, [a, b]);

// 3. useCallback：缓存函数
const handleClick = useCallback(() => {
  doSomething(a);
}, [a]);

// 4. 虚拟列表
import { FixedSizeList } from 'react-window';

// 5. 代码分割
const LazyComponent = lazy(() => import('./Component'));
```

#### Vue的性能优化

```vue
<!-- 1. v-once：只渲染一次 -->
<div v-once>{{ staticContent }}</div>

<!-- 2. v-memo：条件缓存 -->
<div v-memo="[count]">
  <!-- count不变时不重新渲染 -->
</div>

<!-- 3. keep-alive：缓存组件 -->
<keep-alive>
  <component :is="currentComponent" />
</keep-alive>

<!-- 4. 异步组件 -->
<script setup>
import { defineAsyncComponent } from 'vue';

const AsyncComp = defineAsyncComponent(() =>
  import('./Component.vue')
);
</script>

<!-- 5. 虚拟列表 -->
<virtual-list :items="items" :item-height="50" />
```

#### Angular的性能优化

```typescript
// 1. OnPush策略
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OptimizedComponent {}

// 2. trackBy：列表优化
<div *ngFor="let item of items; trackBy: trackByFn">
  {{ item.name }}
</div>

trackByFn(index, item) {
  return item.id;  // 使用id追踪
}

// 3. Lazy Loading
const routes: Routes = [
  {
    path: 'lazy',
    loadChildren: () => import('./lazy/lazy.module').then(m => m.LazyModule)
  }
];

// 4. Pure Pipe
@Pipe({ name: 'myPipe', pure: true })
export class MyPipe {}

// 5. Detach Change Detection
constructor(private cd: ChangeDetectorRef) {}

ngOnInit() {
  this.cd.detach();  // 手动控制变更检测
}
```

### 10.5 框架选型建议

| 框架 | 适用场景 | 优势 | 劣势 |
|------|----------|------|------|
| **React** | 大型应用、跨平台 | 生态丰富、灵活、React Native | 学习曲线、需要手动优化 |
| **Vue** | 中小型应用、快速开发 | 易学易用、渐进式、中文文档 | 生态较小、大型应用复杂度高 |
| **Angular** | 企业级应用 | 完整框架、TypeScript原生、依赖注入 | 学习曲线陡峭、体积大 |

---

## 11. React设计哲学

### 11.1 声明式编程

**命令式 vs 声明式**：

```javascript
// 命令式：告诉计算机"怎么做"
const list = document.getElementById('list');
const item = document.createElement('li');
item.textContent = '新项目';
list.appendChild(item);  // 手动操作DOM

// 声明式：告诉计算机"要什么"
function List({ items }) {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.text}</li>)}
    </ul>
  );
}
```

**优点**：
- 可读性强：代码即文档
- 易于维护：关注"what"而不是"how"
- 易于测试：输入输出明确

### 11.2 组件化思想

**UI = f(state)**：

```jsx
// UI是状态的函数
function Counter({ initialCount }) {
  const [count, setCount] = useState(initialCount);
  
  // 给定count，UI确定
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

**单一职责**：

```jsx
// ❌ 不好：一个组件做太多事
function UserDashboard() {
  return (
    <div>
      {/* 用户信息 */}
      <div>{user.name}</div>
      
      {/* 订单列表 */}
      <ul>{orders.map(...)}</ul>
      
      {/* 统计图表 */}
      <canvas />
    </div>
  );
}

// ✅ 好：拆分成多个组件
function UserDashboard() {
  return (
    <div>
      <UserProfile user={user} />
      <OrderList orders={orders} />
      <Statistics data={stats} />
    </div>
  );
}
```

### 11.3 数据驱动视图

**数据是唯一的真相来源（Single Source of Truth）**：

```jsx
// ❌ 不好：从DOM读取状态
function Counter() {
  const increment = () => {
    const countElement = document.getElementById('count');
    const count = parseInt(countElement.textContent);
    countElement.textContent = count + 1;  // 直接修改DOM
  };
  
  return (
    <div>
      <p id="count">0</p>
      <button onClick={increment}>+1</button>
    </div>
  );
}

// ✅ 好：从state读取状态
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>{count}</p>  {/* state是唯一真相来源 */}
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

### 11.4 函数式编程

**纯函数**：

```jsx
// ✅ 纯函数：相同输入，相同输出，无副作用
function add(a, b) {
  return a + b;
}

// ✅ React组件应该是纯函数
function Greeting({ name }) {
  return <div>Hello, {name}</div>;
}

// ❌ 非纯函数：有副作用
let total = 0;
function addToTotal(value) {
  total += value;  // 修改外部变量
  return total;
}
```

**不可变数据**：

```jsx
// ❌ 错误：直接修改
const user = { name: '张三', age: 25 };
user.age = 26;
setUser(user);  // React不会检测到变化

// ✅ 正确：创建新对象
setUser({ ...user, age: 26 });

// ✅ 数组的不可变操作
const items = [1, 2, 3];

// 添加
const newItems = [...items, 4];

// 删除
const filtered = items.filter(item => item !== 2);

// 修改
const updated = items.map(item => item === 2 ? 20 : item);
```

**高阶组件（HOC）**：

```jsx
// 高阶函数：接收函数，返回函数
const withLogging = (fn) => {
  return (...args) => {
    console.log('调用函数', args);
    return fn(...args);
  };
};

// 高阶组件：接收组件，返回组件
const withAuth = (Component) => {
  return function AuthComponent(props) {
    const isLoggedIn = useAuth();
    
    if (!isLoggedIn) {
      return <Redirect to="/login" />;
    }
    
    return <Component {...props} />;
  };
};

// 使用
const ProtectedPage = withAuth(Dashboard);
```

### 11.5 关注点分离

**逻辑与UI分离**：

```jsx
// ❌ 不好：逻辑和UI混在一起
function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(data => {
        setUsers(data);
        setLoading(false);
      });
  }, []);
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}

// ✅ 好：抽取逻辑到自定义Hook
function useFetchUsers() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(data => {
        setUsers(data);
        setLoading(false);
      });
  }, []);
  
  return { users, loading };
}

function UserList() {
  const { users, loading } = useFetchUsers();
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

---

## 12. 性能优化进阶

### 12.1 避免不必要的渲染

**问题诊断**：

```jsx
// 使用React DevTools Profiler
// 或添加console.log
function ExpensiveComponent({ data }) {
  console.log('ExpensiveComponent render');  // 观察渲染次数
  
  return <div>{/* 复杂UI */}</div>;
}
```

**优化方案**：

```jsx
// 1. React.memo
const ExpensiveComponent = React.memo(function ExpensiveComponent({ data }) {
  return <div>{data}</div>;
});

// 2. 自定义比较函数
const ExpensiveComponent = React.memo(
  function ExpensiveComponent({ user }) {
    return <div>{user.name}</div>;
  },
  (prevProps, nextProps) => {
    // 返回true表示不更新，false表示更新
    return prevProps.user.id === nextProps.user.id;
  }
);

// 3. useMemo缓存子组件
function Parent() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');
  
  // 只有name变化时才重新创建Child
  const child = useMemo(() => {
    return <ExpensiveChild name={name} />;
  }, [name]);
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      {child}
    </div>
  );
}
```

### 12.2 避免内联对象和函数

**问题**：

```jsx
// ❌ 每次render都创建新对象/函数
function Parent() {
  return (
    <Child
      style={{ color: 'red' }}  // 新对象
      onClick={() => console.log('click')}  // 新函数
    />
  );
}

// Child会每次重新渲染，即使props语义上没变
const Child = React.memo(function Child({ style, onClick }) {
  console.log('Child render');
  return <div style={style} onClick={onClick}>Child</div>;
});
```

**解决方案**：

```jsx
// ✅ 提取到外部
const style = { color: 'red' };

function Parent() {
  const handleClick = useCallback(() => {
    console.log('click');
  }, []);
  
  return <Child style={style} onClick={handleClick} />;
}
```

### 12.3 列表优化

**使用稳定的key**：

```jsx
// ❌ 错误：使用index作为key
{items.map((item, index) => (
  <li key={index}>{item.text}</li>
))}

// 问题：删除第一项时，所有项的index都变了，导致全部重新渲染

// ✅ 正确：使用唯一ID
{items.map(item => (
  <li key={item.id}>{item.text}</li>
))}
```

**虚拟列表**：

```jsx
import { FixedSizeList } from 'react-window';

function VirtualList({ items }) {
  return (
    <FixedSizeList
      height={600}        // 可见区域高度
      itemCount={items.length}
      itemSize={50}       // 每项高度
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          {items[index].text}
        </div>
      )}
    </FixedSizeList>
  );
}

// 优势：只渲染可见区域的项，性能提升10-100倍
```

### 12.4 代码分割

**路由级分割**：

```jsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// 懒加载组件
const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<Loading />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

**组件级分割**：

```jsx
// 大型组件懒加载
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  const [showHeavy, setShowHeavy] = useState(false);
  
  return (
    <div>
      <button onClick={() => setShowHeavy(true)}>显示重组件</button>
      
      {showHeavy && (
        <Suspense fallback={<div>加载中...</div>}>
          <HeavyComponent />
        </Suspense>
      )}
    </div>
  );
}
```

### 12.5 防抖和节流

```jsx
// 防抖：延迟执行
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debouncedValue;
}

// 使用
function SearchInput() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 500);
  
  useEffect(() => {
    if (debouncedSearchTerm) {
      // 只有停止输入500ms后才会触发搜索
      fetch(`/api/search?q=${debouncedSearchTerm}`);
    }
  }, [debouncedSearchTerm]);
  
  return (
    <input
      value={searchTerm}
      onChange={e => setSearchTerm(e.target.value)}
    />
  );
}

// 节流：限制执行频率
function useThrottle(value, limit) {
  const [throttledValue, setThrottledValue] = useState(value);
  const lastRan = useRef(Date.now());
  
  useEffect(() => {
    const handler = setTimeout(() => {
      if (Date.now() - lastRan.current >= limit) {
        setThrottledValue(value);
        lastRan.current = Date.now();
      }
    }, limit - (Date.now() - lastRan.current));
    
    return () => clearTimeout(handler);
  }, [value, limit]);
  
  return throttledValue;
}
```

---

## 13. 高级模式与技巧

### 13.1 Render Props

```jsx
// 通过函数prop渲染内容
function MouseTracker({ render }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMouseMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);
  
  return render(position);  // 调用render prop
}

// 使用
function App() {
  return (
    <MouseTracker
      render={({ x, y }) => (
        <div>鼠标位置：({x}, {y})</div>
      )}
    />
  );
}

// 或使用children prop
function MouseTracker({ children }) {
  // ...
  return children(position);
}

<MouseTracker>
  {({ x, y }) => <div>({x}, {y})</div>}
</MouseTracker>
```

### 13.2 Compound Components（复合组件）

```jsx
// Tab组件
const TabsContext = createContext();

function Tabs({ children, defaultValue }) {
  const [activeTab, setActiveTab] = useState(defaultValue);
  
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div className="tab-list">{children}</div>;
}

function Tab({ value, children }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  
  return (
    <button
      className={activeTab === value ? 'active' : ''}
      onClick={() => setActiveTab(value)}
    >
      {children}
    </button>
  );
}

function TabPanels({ children }) {
  return <div className="tab-panels">{children}</div>;
}

function TabPanel({ value, children }) {
  const { activeTab } = useContext(TabsContext);
  
  return activeTab === value ? <div>{children}</div> : null;
}

// 组合使用
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;

// 使用
function App() {
  return (
    <Tabs defaultValue="tab1">
      <Tabs.List>
        <Tabs.Tab value="tab1">Tab 1</Tabs.Tab>
        <Tabs.Tab value="tab2">Tab 2</Tabs.Tab>
      </Tabs.List>
      
      <Tabs.Panels>
        <Tabs.Panel value="tab1">Content 1</Tabs.Panel>
        <Tabs.Panel value="tab2">Content 2</Tabs.Panel>
      </Tabs.Panels>
    </Tabs>
  );
}
```

### 13.3 Controlled vs Uncontrolled

```jsx
// Controlled组件（受控）：值由React state控制
function ControlledInput() {
  const [value, setValue] = useState('');
  
  return (
    <input
      value={value}  // 值由state控制
      onChange={e => setValue(e.target.value)}
    />
  );
}

// Uncontrolled组件（非受控）：值由DOM控制
function UncontrolledInput() {
  const inputRef = useRef();
  
  const handleSubmit = () => {
    console.log(inputRef.current.value);  // 从DOM读取值
  };
  
  return (
    <div>
      <input ref={inputRef} defaultValue="默认值" />
      <button onClick={handleSubmit}>提交</button>
    </div>
  );
}

// 何时使用？
// Controlled：需要验证、格式化、同步到其他组件
// Uncontrolled：简单表单、第三方库集成、性能优化
```

### 13.4 Portal（传送门）

```jsx
import { createPortal } from 'react-dom';

// 将子组件渲染到DOM树的其他位置
function Modal({ children, isOpen }) {
  if (!isOpen) return null;
  
  // 渲染到body下，而不是当前组件的位置
  return createPortal(
    <div className="modal-overlay">
      <div className="modal">{children}</div>
    </div>,
    document.body  // 挂载点
  );
}

// 使用
function App() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsOpen(true)}>打开弹窗</button>
      
      <Modal isOpen={isOpen}>
        <h1>弹窗内容</h1>
        <button onClick={() => setIsOpen(false)}>关闭</button>
      </Modal>
    </div>
  );
}

// 优势：
// 1. 避免z-index问题
// 2. 避免overflow: hidden影响
// 3. 事件冒泡仍然按React树进行
```

---

## 14. 源码级面试题

### Q1：为什么不能在条件语句中使用Hooks？

**答案**：

```javascript
// Hooks存储在Fiber节点的memoizedState链表中
// 顺序必须一致，否则会读取错误的Hook

// ❌ 错误
function Component({ condition }) {
  if (condition) {
    const [state, setState] = useState(0);  // Hook的位置不固定
  }
  
  // 首次渲染（condition=true）：Hook链表 [useState]
  // 二次渲染（condition=false）：Hook链表 []
  // React会错误地认为没有Hook了
}

// ✅ 正确
function Component({ condition }) {
  const [state, setState] = useState(0);  // Hook的位置固定
  
  if (condition) {
    // 在条件语句中使用state
  }
}
```

### Q2：setState是同步还是异步的？

**答案**：

```javascript
// React 18之前：看场景
// 1. 合成事件中：异步（批处理）
<button onClick={() => {
  setCount(count + 1);
  console.log(count);  // 旧值（异步）
}}>

// 2. 原生事件中：同步
useEffect(() => {
  document.addEventListener('click', () => {
    setCount(count + 1);
    console.log(count);  // 新值（同步）
  });
}, []);

// 3. setTimeout中：同步
setTimeout(() => {
  setCount(count + 1);
  console.log(count);  // 新值（同步）
}, 0);

// React 18：自动批处理
// 所有更新都是异步的（包括原生事件、setTimeout）
// 除非使用flushSync强制同步
import { flushSync } from 'react-dom';

flushSync(() => {
  setCount(count + 1);
});
console.log(count);  // 新值（同步）
```

### Q3：React的Diff算法时间复杂度？

**答案**：

```
传统Diff算法：O(n³)
- 树的编辑距离问题
- 需要遍历所有节点对

React的Diff算法：O(n)
- 同层比较（不跨层级）
- 类型不同直接替换
- 使用key优化列表

具体优化：
1. Tree Diff：只比较同层节点
2. Component Diff：类型不同直接替换
3. Element Diff：使用key复用节点
```

### Q4：Fiber解决了什么问题？

**答案**：

```
React 15的问题（Stack Reconciler）：
1. 递归不可中断：长任务阻塞主线程
2. 无优先级：紧急更新和普通更新一样处理
3. 卡顿：超过16ms导致掉帧

Fiber的解决方案：
1. 链表结构：可以中断和恢复
2. 时间切片：workLoop检查时间，超时就yield
3. 优先级调度：高优先级任务优先执行
4. 并发模式：可以同时准备多个版本的UI

Fiber节点的双向链表：
child: 第一个子节点
sibling: 下一个兄弟节点
return: 父节点

可以随时中断和恢复遍历
```

### Q5：useEffect和useLayoutEffect的区别？

**答案**：

```
执行时机：
useEffect：DOM更新后异步执行（不阻塞渲染）
useLayoutEffect：DOM更新后同步执行（阻塞渲染）

执行顺序：
render → DOM更新 → useLayoutEffect → 浏览器绘制 → useEffect

使用场景：
useEffect：数据请求、订阅、日志（大部分场景）
useLayoutEffect：测量DOM、同步DOM操作、避免闪烁

示例：
// 避免闪烁
useLayoutEffect(() => {
  const element = ref.current;
  element.style.height = `${element.scrollHeight}px`;
  // 用户看不到中间状态（高度为0 → 高度为scrollHeight）
}, []);
```

---

## 15. 企业级实战案例

### 案例1：无限滚动列表

```jsx
function InfiniteScrollList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const observerRef = useRef();
  
  // 加载数据
  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return;
    
    setLoading(true);
    try {
      const res = await fetch(`/api/items?page=${page}`);
      const newItems = await res.json();
      
      if (newItems.length === 0) {
        setHasMore(false);
      } else {
        setItems(prev => [...prev, ...newItems]);
        setPage(prev => prev + 1);
      }
    } finally {
      setLoading(false);
    }
  }, [page, loading, hasMore]);
  
  // Intersection Observer监听
  const lastItemRef = useCallback(node => {
    if (loading) return;
    
    if (observerRef.current) {
      observerRef.current.disconnect();
    }
    
    observerRef.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting && hasMore) {
        loadMore();
      }
    });
    
    if (node) {
      observerRef.current.observe(node);
    }
  }, [loading, hasMore, loadMore]);
  
  // 首次加载
  useEffect(() => {
    loadMore();
  }, []);
  
  return (
    <div>
      {items.map((item, index) => {
        if (index === items.length - 1) {
          // 最后一项，添加ref
          return (
            <div key={item.id} ref={lastItemRef}>
              {item.text}
            </div>
          );
        }
        return <div key={item.id}>{item.text}</div>;
      })}
      
      {loading && <div>加载中...</div>}
      {!hasMore && <div>没有更多了</div>}
    </div>
  );
}
```

### 案例2：拖拽排序

```jsx
function DraggableList() {
  const [items, setItems] = useState([
    { id: 1, text: 'Item 1' },
    { id: 2, text: 'Item 2' },
    { id: 3, text: 'Item 3' },
  ]);
  const [draggedItem, setDraggedItem] = useState(null);
  
  const handleDragStart = (item) => {
    setDraggedItem(item);
  };
  
  const handleDragOver = (e, targetItem) => {
    e.preventDefault();
    
    if (!draggedItem || draggedItem.id === targetItem.id) return;
    
    // 重新排序
    const draggedIndex = items.findIndex(item => item.id === draggedItem.id);
    const targetIndex = items.findIndex(item => item.id === targetItem.id);
    
    const newItems = [...items];
    newItems.splice(draggedIndex, 1);
    newItems.splice(targetIndex, 0, draggedItem);
    
    setItems(newItems);
  };
  
  const handleDragEnd = () => {
    setDraggedItem(null);
  };
  
  return (
    <ul>
      {items.map(item => (
        <li
          key={item.id}
          draggable
          onDragStart={() => handleDragStart(item)}
          onDragOver={e => handleDragOver(e, item)}
          onDragEnd={handleDragEnd}
          style={{
            opacity: draggedItem?.id === item.id ? 0.5 : 1,
            cursor: 'move'
          }}
        >
          {item.text}
        </li>
      ))}
    </ul>
  );
}
```

### 案例3：表单验证

```jsx
function useForm(initialValues, validate) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  
  const handleChange = (name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
    
    // 实时验证
    if (touched[name]) {
      const fieldErrors = validate({ ...values, [name]: value });
      setErrors(prev => ({ ...prev, [name]: fieldErrors[name] }));
    }
  };
  
  const handleBlur = (name) => {
    setTouched(prev => ({ ...prev, [name]: true }));
    
    // 失焦验证
    const fieldErrors = validate(values);
    setErrors(prev => ({ ...prev, [name]: fieldErrors[name] }));
  };
  
  const handleSubmit = (onSubmit) => (e) => {
    e.preventDefault();
    
    // 验证所有字段
    const allErrors = validate(values);
    setErrors(allErrors);
    
    // 标记所有字段为touched
    const allTouched = Object.keys(values).reduce((acc, key) => {
      acc[key] = true;
      return acc;
    }, {});
    setTouched(allTouched);
    
    // 无错误时提交
    if (Object.keys(allErrors).length === 0) {
      onSubmit(values);
    }
  };
  
  return {
    values,
    errors,
    touched,
    handleChange,
    handleBlur,
    handleSubmit
  };
}

// 使用
function LoginForm() {
  const validate = (values) => {
    const errors = {};
    
    if (!values.email) {
      errors.email = '邮箱不能为空';
    } else if (!/\S+@\S+\.\S+/.test(values.email)) {
      errors.email = '邮箱格式不正确';
    }
    
    if (!values.password) {
      errors.password = '密码不能为空';
    } else if (values.password.length < 6) {
      errors.password = '密码至少6位';
    }
    
    return errors;
  };
  
  const { values, errors, touched, handleChange, handleBlur, handleSubmit } = useForm(
    { email: '', password: '' },
    validate
  );
  
  const onSubmit = (values) => {
    console.log('提交', values);
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input
          name="email"
          value={values.email}
          onChange={e => handleChange('email', e.target.value)}
          onBlur={() => handleBlur('email')}
          placeholder="邮箱"
        />
        {touched.email && errors.email && <span>{errors.email}</span>}
      </div>
      
      <div>
        <input
          name="password"
          type="password"
          value={values.password}
          onChange={e => handleChange('password', e.target.value)}
          onBlur={() => handleBlur('password')}
          placeholder="密码"
        />
        {touched.password && errors.password && <span>{errors.password}</span>}
      </div>
      
      <button type="submit">登录</button>
    </form>
  );
}
```

---

## 16. React与微前端架构

> 📌 **本节为React视角的微前端摘要**，完整的微前端架构知识请参考：
> 
> 👉 [06-前端框架与工程化综合.md §3 微前端架构](./06-前端框架与工程化综合.md#3-微前端架构)

### 16.1 微前端核心概念

```
微前端架构：
┌────────────────────────────────┐
│      Main Application          │  ← 主应用（React）
│        (Container)             │
├────────────┬─────────┬─────────┤
│  SubApp1   │ SubApp2 │ SubApp3 │  ← 子应用（可以是React/Vue/Angular）
│  (React)   │  (Vue)  │(Angular)│
└────────────┴─────────┴─────────┘
```

**核心特性**：技术栈无关、独立开发部署、运行时集成、样式隔离、状态隔离

### 16.2 React微前端方案对比

| 方案 | 特点 | 适用场景 |
|------|------|----------|
| **qiankun** | 最成熟、JS沙箱+样式隔离、社区生态好 | 大型企业级应用 |
| **Module Federation** | Webpack 5原生、按模块共享、构建时集成 | 微模块共享场景 |
| **iframe** | 天然隔离、简单可靠 | 简单集成、安全要求高 |

### 16.3 React微前端关键问题

| 问题 | 解决方案 |
|------|----------|
| **CSS样式隔离** | Shadow DOM / CSS Modules / 命名空间前缀 |
| **JS沙箱隔离** | Proxy沙箱（qiankun自带） / Snapshot沙箱 |
| **子应用通信** | 全局状态(initGlobalState) / CustomEvent / Props传递 |
| **路由同步** | 主应用路由监听 + 子应用路由同步 |

### 16.4 何时使用微前端？

✅ **适合**：大型应用多团队协作、渐进式重构老项目、不同模块不同技术栈  
❌ **不适合**：小型应用、单一团队、性能要求极高

> 💡 完整代码示例（qiankun集成、Module Federation配置、沙箱实现等）请参考 [06-前端框架与工程化综合.md](./06-前端框架与工程化综合.md#3-微前端架构)

---

## 总结

React的核心概念：

1. **组件化架构**：组件树、Fiber树、8种通信方式
2. **渲染机制**：虚拟DOM、Diff算法、Reconciliation流程
3. **Fiber架构**：时间切片、优先级调度（Lane模型）、并发渲染
4. **生命周期**：类组件完整生命周期、Hooks生命周期映射
5. **Hooks原理**：链表存储、useState/useEffect实现、闭包陷阱
6. **框架对比**：React vs Vue vs Angular（渲染机制、数据流、通信）
7. **设计哲学**：声明式、组件化（UI = f(state)）、函数式编程
8. **性能优化**：memo、useMemo、虚拟列表、代码分割、防抖节流
9. **高级模式**：Render Props、Compound Components、Portal、Controlled/Uncontrolled
10. **实战案例**：无限滚动、拖拽排序、表单验证
11. **微前端架构**：qiankun/Module Federation、样式隔离、JS沙箱、子应用通信、独立部署

---

