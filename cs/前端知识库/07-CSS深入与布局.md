# CSS 深入与布局

---

## 📑 目录

### 盒模型与格式化上下文
1. [CSS 盒模型](#css-盒模型)
2. [BFC / IFC / FFC / GFC](#bfc--ifc--ffc--gfc)

### 布局体系
3. [布局体系总览](#布局体系总览)
4. [Flexbox 深入](#flexbox-深入)
5. [Grid 深入](#grid-深入)

### 响应式与主题
6. [响应式设计](#响应式设计)
7. [CSS 变量与主题](#css-变量与主题)

### 动画与选择器
8. [CSS 动画与性能](#css-动画与性能)
9. [CSS 选择器与优先级](#css-选择器与优先级)

### 现代 CSS 与工程化
10. [现代 CSS 新特性](#现代-css-新特性)
11. [CSS 工程化](#css-工程化)

### 实战与自查
12. [实战案例](#实战案例)
13. [面试题自查](#面试题自查)

---

## CSS 盒模型

> ⭐⭐⭐ 高频考点 — 标准盒模型 / IE 盒模型 / box-sizing / margin 折叠

### 1. 标准盒模型 vs IE 盒模型

**标准盒模型（W3C Box Model）**

在标准盒模型中，`width` / `height` 仅指 **content** 区域的尺寸，**不包含** padding 和 border。

```
┌───────────────────────────── margin ──────────────────────────┐
│  ┌──────────────────────── border ─────────────────────────┐  │
│  │  ┌───────────────────── padding ─────────────────────┐  │  │
│  │  │  ┌──────────────── content ────────────────────┐  │  │  │
│  │  │  │                                             │  │  │  │
│  │  │  │         width × height (仅 content)         │  │  │  │
│  │  │  │                                             │  │  │  │
│  │  │  └─────────────────────────────────────────────┘  │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

实际占据宽度 = `width + padding-left + padding-right + border-left + border-right + margin-left + margin-right`

**IE 盒模型（怪异盒模型）**

在 IE 盒模型中，`width` / `height` 包含了 **content + padding + border**。

```
┌───────────────────────────── margin ──────────────────────────┐
│  ┌──────────────────────── border ─────────────────────────┐  │
│  │  ┌───────────────────── padding ─────────────────────┐  │  │
│  │  │  ┌──────────────── content ────────────────────┐  │  │  │
│  │  │  │                                             │  │  │  │
│  │  │  │  width 包含 content + padding + border      │  │  │  │
│  │  │  │                                             │  │  │  │
│  │  │  └─────────────────────────────────────────────┘  │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

实际占据宽度 = `width + margin-left + margin-right`（padding 和 border 已包含在 width 中）

```css
/* 标准盒模型 — 默认 */
.standard-box {
  box-sizing: content-box; /* 默认值 */
  width: 200px;
  padding: 20px;
  border: 5px solid #333;
  /* 实际内容宽度 = 200px */
  /* 盒子总宽度 = 200 + 20*2 + 5*2 = 250px */
}

/* IE 盒模型 */
.ie-box {
  box-sizing: border-box;
  width: 200px;
  padding: 20px;
  border: 5px solid #333;
  /* 盒子总宽度 = 200px（padding 和 border 挤压 content） */
  /* 实际内容宽度 = 200 - 20*2 - 5*2 = 150px */
}
```

**面试追问**：
- 为什么现代项目推荐全局使用 `box-sizing: border-box`？
- `box-sizing: border-box` 是否影响 margin？

---

### 2. box-sizing 全局重置最佳实践

```css
/* 方式一：直接重置所有元素（简单但继承性差） */
* {
  box-sizing: border-box;
}

/* 方式二：利用继承（推荐 — 第三方组件可覆盖） */
html {
  box-sizing: border-box;
}
*, *::before, *::after {
  box-sizing: inherit;
}

/* 第三方组件可覆盖为 content-box */
.third-party-widget {
  box-sizing: content-box;
}
.third-party-widget *, 
.third-party-widget *::before, 
.third-party-widget *::after {
  box-sizing: inherit;
}
```

**效果说明**：方式二使用 `inherit` 关键字，让子元素继承父元素的 `box-sizing`，当需要嵌入第三方库（如地图组件）时，可在该组件容器上重置为 `content-box`，其后代自动继承。

---

### 3. Margin 折叠（Margin Collapsing） ⭐⭐⭐

Margin 折叠仅发生在 **块级元素** 的 **垂直方向**（水平方向不会折叠）。

**三种折叠场景**：

```css
/* ① 相邻兄弟元素 — 上下 margin 取较大值 */
.box-a { margin-bottom: 30px; }
.box-b { margin-top: 20px; }
/* 实际间距 = max(30, 20) = 30px，而非 50px */

/* ② 父子元素 — 子元素 margin-top 穿透父元素 */
.parent {
  /* 没有 border / padding / overflow:hidden / BFC */
  background: #f0f0f0;
}
.child {
  margin-top: 20px;
  /* 这个 margin 会"穿透"到 parent 外面 */
}

/* ③ 空元素自身 — 上下 margin 折叠 */
.empty {
  margin-top: 20px;
  margin-bottom: 30px;
  /* 实际占据 margin = max(20, 30) = 30px */
}
```

**阻止 margin 折叠的方法**：

```css
/* 方法一：创建 BFC */
.parent {
  overflow: hidden; /* 或 auto / display: flow-root */
}

/* 方法二：添加 border 或 padding */
.parent {
  border-top: 1px solid transparent;
  /* 或 padding-top: 1px; */
}

/* 方法三：使用 Flex / Grid 布局 */
.parent {
  display: flex;
  flex-direction: column;
}
```

**折叠规则总结**：
| 场景 | 正正 | 正负 | 负负 |
|------|------|------|------|
| 计算方式 | 取较大值 | 相加 | 取绝对值较大的负值 |
| 示例 | max(30, 20) = 30 | 30 + (-20) = 10 | min(-30, -20) = -30 |

**面试追问**：
- 为什么 inline 元素 / 水平方向不会发生 margin 折叠？
- `display: flex` 的子元素之间还会发生 margin 折叠吗？（不会）
- 如何不增加额外 DOM 元素解决父子 margin 穿透？

---

### 4. 内联元素的盒模型特殊行为

```css
/* 内联元素（inline）对盒模型属性的响应 */
span {
  width: 200px;      /* ❌ 无效 */
  height: 100px;     /* ❌ 无效 */
  margin-top: 20px;  /* ❌ 无效（垂直 margin 不生效） */
  margin-left: 20px; /* ✅ 有效（水平 margin 生效） */
  padding-top: 20px; /* ⚠️ 视觉上有效，但不影响布局（不撑开行高） */
  border-top: 1px solid red; /* ⚠️ 视觉上有效，但不影响布局 */
}

/* inline-block 是解决方案 */
span.inline-block {
  display: inline-block;
  width: 200px;      /* ✅ 有效 */
  height: 100px;     /* ✅ 有效 */
  margin-top: 20px;  /* ✅ 有效 */
}
```

**面试追问**：
- `inline-block` 元素之间的空白间隙如何消除？
- `img` 是 inline 元素却能设置宽高，为什么？（替换元素）

---

## BFC / IFC / FFC / GFC

> ⭐⭐⭐ 高频考点 — 格式化上下文是 CSS 布局的底层机制

### 1. BFC（Block Formatting Context）块级格式化上下文 ⭐⭐⭐

**BFC 是一个独立的渲染区域，内部元素的布局不会影响外部元素。**

**触发 BFC 的条件（任一即可）**：

```css
/* ① 根元素 <html> 天然是 BFC */

/* ② float 不为 none */
.bfc { float: left; }          /* 或 right */

/* ③ position 为 absolute 或 fixed */
.bfc { position: absolute; }

/* ④ overflow 不为 visible 且不为 clip */
.bfc { overflow: hidden; }     /* 或 auto / scroll */

/* ⑤ display 为特定值 */
.bfc { display: inline-block; }
.bfc { display: flow-root; }   /* 推荐 — 无副作用创建 BFC */
.bfc { display: table-cell; }
.bfc { display: flex; }        /* 自身为 BFC，子元素为 FFC */
.bfc { display: grid; }        /* 自身为 BFC，子元素为 GFC */

/* ⑥ contain 为 layout / content / paint */
.bfc { contain: layout; }
```

**BFC 的布局规则**：
1. 内部的块级元素会在垂直方向一个接一个排列
2. 同一个 BFC 内相邻块级元素的 margin 会折叠
3. BFC 区域不会与 float 元素重叠
4. BFC 可以包含浮动元素（清除浮动）
5. BFC 是一个独立的容器，内部元素不影响外部

---

#### 应用一：清除浮动（包含浮动子元素）

```css
/* 问题：浮动元素导致父容器高度塌陷 */
.container {
  background: #eee;
  /* 没有高度 — 因为子元素全部浮动 */
}
.float-child {
  float: left;
  width: 200px;
  height: 100px;
  background: coral;
}

/* 解决方案一：父元素创建 BFC（推荐 display: flow-root） */
.container {
  display: flow-root; /* 最佳方案 — 无副作用 */
  background: #eee;
}

/* 解决方案二：clearfix 伪元素（经典方案） */
.clearfix::after {
  content: '';
  display: table; /* 或 block */
  clear: both;
}

/* 解决方案三：overflow: hidden（有副作用 — 可能裁剪内容） */
.container {
  overflow: hidden;
}
```

**效果说明**：`display: flow-root` 专门为创建 BFC 设计，不会改变元素的 display 行为，也不会产生 `overflow: hidden` 的裁剪副作用。

---

#### 应用二：阻止 margin 折叠

```css
/* 问题：兄弟元素 margin 折叠 */
/* HTML:
  <div class="wrapper">
    <div class="box" style="margin-bottom: 30px;">A</div>
    <div class="box" style="margin-top: 20px;">B</div>
    <!-- 实际间距 30px 而非 50px -->
  </div>
*/

/* 解决方案：将其中一个包裹在 BFC 容器中 */
/* HTML:
  <div class="wrapper">
    <div class="box" style="margin-bottom: 30px;">A</div>
    <div style="display: flow-root;">
      <div class="box" style="margin-top: 20px;">B</div>
    </div>
    <!-- 间距 = 30 + 20 = 50px -->
  </div>
*/
```

---

#### 应用三：自适应两栏布局

```css
/* BFC 不与浮动元素重叠 */
.sidebar {
  float: left;
  width: 200px;
  height: 300px;
  background: #4fc3f7;
}

.main {
  overflow: hidden; /* 创建 BFC — 不与左侧 float 重叠 */
  height: 300px;
  background: #81c784;
  /* 自动填满剩余空间 */
}
```

**面试追问**：
- `display: flow-root` 和 `overflow: hidden` 创建 BFC 的区别？
- 为什么 `overflow: visible` 不能创建 BFC？
- BFC 内部为什么还会 margin 折叠？（同一个 BFC 内的规则）

---

### 2. IFC（Inline Formatting Context）内联格式化上下文

**IFC 是内联级元素参与的格式化上下文。**

```css
/* IFC 特性 */
.ifc-container {
  /* 块级容器只包含内联级元素时，会创建 IFC */
  font-size: 16px;
  line-height: 1.5;
  text-align: center; /* 在 IFC 中水平居中 */
}

/* 行框（Line Box）— IFC 的核心概念 */
/*
  每行内容形成一个 line box
  line box 的高度由最高的 inline box 决定
  vertical-align 控制 inline box 在 line box 中的对齐
*/

.ifc-container span {
  vertical-align: middle; /* 在 line box 中垂直居中 */
}

/* 经典问题：图片下方的空白间隙 */
.img-container {
  background: #eee;
}
.img-container img {
  /* img 是 inline 元素，底部对齐基线，基线下方有 descender 空间 */
  /* 解决方案一 */
  display: block;
  /* 解决方案二 */
  vertical-align: bottom; /* 或 middle / top */
}
```

**面试追问**：
- `line-height` 和 `height` 的区别？
- 为什么 `vertical-align` 只对 inline / inline-block / table-cell 元素有效？

---

### 3. FFC（Flex Formatting Context）

```css
/* 当 display 为 flex / inline-flex 时，子元素参与 FFC */
.ffc-container {
  display: flex; /* 子元素成为 flex item */
}

/* FFC 特性 */
/* 1. 子元素的 float 失效 */
/* 2. 子元素的 vertical-align 失效 */
/* 3. 子元素自动成为块级（即使是 span） */
/* 4. 不存在 margin 折叠 */
```

---

### 4. GFC（Grid Formatting Context）

```css
/* 当 display 为 grid / inline-grid 时，子元素参与 GFC */
.gfc-container {
  display: grid;
  grid-template-columns: 1fr 2fr 1fr;
}

/* GFC 特性 */
/* 1. 子元素的 float 失效 */
/* 2. 子元素的 display: inline-block 等失效 */
/* 3. 子元素的 vertical-align 失效 */
/* 4. 不存在 margin 折叠 */
```

**四种格式化上下文对比**：

| 特性 | BFC | IFC | FFC | GFC |
|------|-----|-----|-----|-----|
| 触发条件 | overflow/float/position 等 | 块容器仅含内联元素 | display: flex | display: grid |
| 排列方向 | 垂直 | 水平（换行） | 主轴方向 | 行+列二维 |
| margin 折叠 | ✅ 会折叠 | ❌ | ❌ | ❌ |
| float 有效 | ✅ | ✅ | ❌ | ❌ |

---

## 布局体系总览

> ⭐⭐⭐ 四种核心布局方式的对比与适用场景

### 1. 浮动布局（Float）

**原理**：元素脱离文档流，向左/右浮动，文本和内联元素环绕。

```css
/* 经典两栏浮动布局 */
.sidebar {
  float: left;
  width: 250px;
}
.main {
  margin-left: 260px; /* sidebar 宽度 + 间距 */
}
/* 清除浮动 */
.row::after {
  content: '';
  display: table;
  clear: both;
}
```

**缺点**：
- 需要手动清除浮动
- 等高列难以实现
- 响应式适配困难
- 已被 Flex/Grid 替代，现代项目不推荐

---

### 2. 定位布局（Position）

```css
/* position 五个值 */
.static    { position: static; }    /* 默认 — 文档流 */
.relative  { position: relative; }  /* 相对自身偏移，不脱离文档流 */
.absolute  { position: absolute; }  /* 脱离文档流，相对最近定位祖先 */
.fixed     { position: fixed; }     /* 脱离文档流，相对视口 */
.sticky    { position: sticky; }    /* 滚动到阈值前为 relative，之后为 fixed */

/* Sticky 定位 — 常用于吸顶导航 */
.nav {
  position: sticky;
  top: 0;             /* 触发阈值 */
  z-index: 100;
  background: white;
}

/* 绝对定位居中 */
.center-absolute {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
```

**sticky 注意事项**：
1. 父容器不能设置 `overflow: hidden / auto / scroll`
2. 必须指定 `top / bottom / left / right` 之一
3. 父容器的高度必须大于 sticky 元素

**面试追问**：
- `position: absolute` 的包含块是谁？（最近的 position 不为 static 的祖先，或 ICB）
- `transform` 会影响 `position: fixed` 吗？（会 — 创建新包含块）
- `position: sticky` 的兼容性以及 polyfill 方案？

---

### 3. 布局方式对比总结

| 特性 | Float | Position | Flex | Grid |
|------|-------|----------|------|------|
| 维度 | 一维 | 自由 | 一维 | 二维 |
| 脱离文档流 | 半脱离 | absolute/fixed 脱离 | 否 | 否 |
| 等高列 | 难 | 难 | 天然支持 | 天然支持 |
| 垂直居中 | 难 | 可以 | 简单 | 简单 |
| 响应式 | 难 | 难 | 好 | 最好 |
| 适用场景 | 文字环绕 | 覆盖层/弹窗 | 一维布局 | 二维布局 |
| 推荐度 | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |

---

## Flexbox 深入

> ⭐⭐⭐ Flex 是目前使用最广泛的 CSS 布局方案

### 1. Flex 容器属性（6 个）

```css
.flex-container {
  display: flex; /* 或 inline-flex */

  /* ① 主轴方向 */
  flex-direction: row;            /* 默认 → 水平从左到右 */
  /* row-reverse | column | column-reverse */

  /* ② 换行 */
  flex-wrap: nowrap;              /* 默认 → 不换行 */
  /* wrap | wrap-reverse */

  /* ③ 简写：flex-direction + flex-wrap */
  flex-flow: row wrap;

  /* ④ 主轴对齐 */
  justify-content: flex-start;    /* 默认 */
  /* flex-end | center | space-between | space-around | space-evenly */

  /* ⑤ 交叉轴对齐（单行） */
  align-items: stretch;           /* 默认 → 拉伸填满 */
  /* flex-start | flex-end | center | baseline */

  /* ⑥ 交叉轴对齐（多行） */
  align-content: stretch;         /* 默认 */
  /* flex-start | flex-end | center | space-between | space-around | space-evenly */
}
```

**`justify-content` 对齐方式图示**：

```
/* space-between: 两端对齐，中间等分 */
|A       B       C|

/* space-around: 每项两侧等距（两端间距 = 中间一半） */
| A     B     C |

/* space-evenly: 所有间距完全相等 */
|  A    B    C  |
```

---

### 2. Flex 子项属性（6 个）

```css
.flex-item {
  /* ① 排列顺序 — 数值越小越靠前，默认 0 */
  order: 0;

  /* ② 放大比例 — 剩余空间的分配比例，默认 0（不放大） */
  flex-grow: 0;

  /* ③ 缩小比例 — 空间不足时的缩小比例，默认 1（等比缩小） */
  flex-shrink: 1;

  /* ④ 基础尺寸 — 分配剩余空间前的初始大小 */
  flex-basis: auto;  /* 默认 — 取 width/height */
  /* 可以是具体值：200px / 30% / 0 */

  /* ⑤ 简写：flex-grow flex-shrink flex-basis */
  flex: 0 1 auto;    /* 默认值 */
  flex: 1;           /* = 1 1 0%（放大+缩小+不占初始空间） */
  flex: auto;        /* = 1 1 auto */
  flex: none;        /* = 0 0 auto（完全不伸缩） */

  /* ⑥ 单独交叉轴对齐（覆盖 align-items） */
  align-self: auto;  /* 默认 — 继承容器的 align-items */
  /* flex-start | flex-end | center | stretch | baseline */
}
```

**flex-grow 计算详解**：

```css
/* 容器宽度 600px，三个子项 */
.container { display: flex; width: 600px; }
.a { flex: 1; width: 100px; } /* flex-basis: 0% → 忽略 width */
.b { flex: 2; width: 100px; }
.c { flex: 1; width: 100px; }

/*
  flex-basis 都是 0% → 全部空间 600px 参与分配
  a = 600 × 1/(1+2+1) = 150px
  b = 600 × 2/(1+2+1) = 300px
  c = 600 × 1/(1+2+1) = 150px
*/
```

**flex-shrink 计算详解**：

```css
.container { display: flex; width: 500px; }
.a { flex: 0 1 200px; } /* flex-shrink: 1, basis: 200 */
.b { flex: 0 2 200px; } /* flex-shrink: 2, basis: 200 */
.c { flex: 0 1 200px; } /* flex-shrink: 1, basis: 200 */

/*
  总 basis = 200 + 200 + 200 = 600px
  溢出 = 600 - 500 = 100px
  加权总和 = 1*200 + 2*200 + 1*200 = 800
  a 缩小 = 100 × (1*200)/800 = 25px → 175px
  b 缩小 = 100 × (2*200)/800 = 50px → 150px
  c 缩小 = 100 × (1*200)/800 = 25px → 175px
*/
```

**面试追问**：
- `flex: 1` 和 `flex: auto` 的区别？（basis 是 0% vs auto）
- `flex-shrink` 为什么要考虑 `flex-basis` 权重？（尺寸越大缩小越多才合理）
- `flex-basis` 和 `width` 同时存在时谁优先？（flex-basis 优先，除非 basis 为 auto）

---

### 3. Flex 经典布局实战

#### 圣杯布局（Holy Grail Layout）

```html
<!-- HTML 结构 -->
<div class="holy-grail">
  <header>Header</header>
  <div class="body">
    <main class="content">Main</main>
    <nav class="left">Left Nav</nav>
    <aside class="right">Right Aside</aside>
  </div>
  <footer>Footer</footer>
</div>
```

```css
.holy-grail {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

header, footer {
  flex: none;      /* 固定高度，不伸缩 */
  height: 60px;
  background: #333;
  color: white;
}

.body {
  display: flex;
  flex: 1;         /* 占满中间剩余空间 */
}

.content {
  flex: 1;         /* 自适应宽度 */
  padding: 20px;
}

.left {
  flex: 0 0 200px; /* 固定 200px */
  order: -1;       /* 排在 main 前面 */
  background: #e8f5e9;
}

.right {
  flex: 0 0 150px; /* 固定 150px */
  background: #e3f2fd;
}
```

**效果说明**：header/footer 固定高度，中间区域自适应铺满视口高度。左右两栏固定宽度，中间内容区自适应。DOM 中 main 优先渲染，通过 order 调整视觉顺序。

---

#### 双飞翼布局

```html
<div class="wings-layout">
  <div class="main-wrapper">
    <div class="main">Main Content</div>
  </div>
  <div class="left">Left</div>
  <div class="right">Right</div>
</div>
```

```css
.wings-layout {
  display: flex;
}

.main-wrapper {
  flex: 1;
}

.main {
  margin: 0 160px 0 210px; /* 留出左右空间 */
}

.left {
  flex: 0 0 200px;
  order: -1;
}

.right {
  flex: 0 0 150px;
}
```

---

#### 等高列布局

```css
/* Flex 天然等高 — 因为 align-items 默认 stretch */
.equal-height {
  display: flex;
  gap: 20px;
}

.equal-height .col {
  flex: 1;
  padding: 20px;
  background: #f5f5f5;
  border-radius: 8px;
  /* 所有列会自动等高（高度取决于最高的列） */
}
```

---

#### Sticky Footer

```html
<div class="app">
  <header>Header</header>
  <main>Content</main>
  <footer>Footer</footer>
</div>
```

```css
.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

header {
  flex-shrink: 0;    /* 不缩小 */
}

main {
  flex: 1;           /* 占满剩余空间 */
}

footer {
  flex-shrink: 0;
}
```

**效果说明**：当 `main` 内容不够一屏时，`flex: 1` 让它撑开，将 footer 推到视口底部。内容超过一屏时正常滚动。

---

#### 常见 Flex 居中方案

```css
/* 水平垂直居中 — 最简方式 */
.center-flex {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 或者用 margin: auto */
.center-parent {
  display: flex;
}
.center-child {
  margin: auto; /* flex 子项 margin: auto 会吸收所有剩余空间 */
}

/* 导航栏：logo 左 / 菜单中 / 按钮右 */
.navbar {
  display: flex;
  align-items: center;
}
.navbar .logo { margin-right: auto; }  /* 推开右侧 */
.navbar .menu { margin-right: auto; }  /* 居中 */
.navbar .actions { /* 自动到最右 */ }
```

**面试追问**：
- `margin: auto` 在 flex 和 block 布局中的行为区别？
- 如何实现最后一行左对齐的 flex 列表？

---

## Grid 深入

> ⭐⭐ Grid 是最强大的二维布局系统

### 1. Grid 基础概念

```css
.grid-container {
  display: grid; /* 或 inline-grid */

  /* 定义列（3 列） */
  grid-template-columns: 200px 1fr 200px;

  /* 定义行 */
  grid-template-rows: 80px 1fr 60px;

  /* 间距 */
  gap: 20px;            /* row-gap + column-gap */
  row-gap: 20px;
  column-gap: 15px;
}
```

**核心术语**：
- **网格线（Grid Line）**：分割网格的线，列有 n+1 条，行有 m+1 条
- **网格轨道（Grid Track）**：两条相邻网格线之间的空间（行或列）
- **网格单元（Grid Cell）**：行和列交叉形成的最小单位
- **网格区域（Grid Area）**：一个或多个网格单元组成的矩形区域

---

### 2. fr 单位与 repeat()

```css
/* fr — 可用空间的等分单位 */
.grid {
  grid-template-columns: 1fr 2fr 1fr;
  /* 等分比例 1:2:1 */
}

/* repeat() — 重复模式 */
.grid {
  grid-template-columns: repeat(3, 1fr);           /* 3 等分 */
  grid-template-columns: repeat(3, 100px 1fr);     /* 重复模式 */
  grid-template-columns: repeat(4, minmax(200px, 1fr)); /* 4列最小200px */
}

/* auto-fill vs auto-fit */
.grid-auto-fill {
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  /* 自动填充尽可能多的列 — 空轨道保留空间 */
}

.grid-auto-fit {
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  /* 自动适配 — 空轨道折叠为 0，已有项目拉伸填满 */
}
```

**auto-fill vs auto-fit 区别**：

```css
/* 假设容器 900px，每列 minmax(200px, 1fr)，只有 2 个子项 */

/* auto-fill: 创建 4 列（900/200=4），2 个有内容，2 个空轨道 */
/* | item1 | item2 | (空) | (空) | */

/* auto-fit: 折叠空轨道，2 个 item 拉伸填满容器 */
/* |   item1   |   item2   | */
```

---

### 3. minmax() 与 auto 关键字

```css
.grid {
  /* minmax(最小值, 最大值) */
  grid-template-columns: minmax(200px, 1fr) 3fr;
  /* 第一列最小 200px，最大分到 1fr 空间 */

  /* auto — 由内容决定 */
  grid-template-columns: auto 1fr auto;
  /* 两侧根据内容宽度，中间填满 */

  /* fit-content(limit) — 内容自适应但不超限 */
  grid-template-columns: fit-content(300px) 1fr;
}
```

---

### 4. 命名网格线与网格区域

```css
/* 命名网格线 */
.grid {
  grid-template-columns: [sidebar-start] 200px [sidebar-end main-start] 1fr [main-end];
  grid-template-rows: [header-start] 80px [header-end body-start] 1fr [body-end footer-start] 60px [footer-end];
}

.header {
  grid-column: sidebar-start / main-end;
  grid-row: header-start / header-end;
}

/* grid-template-areas — 直观的区域命名 */
.page-layout {
  display: grid;
  grid-template-columns: 200px 1fr 150px;
  grid-template-rows: 80px 1fr 60px;
  grid-template-areas:
    "header  header  header"
    "sidebar content aside"
    "footer  footer  footer";
  gap: 10px;
  min-height: 100vh;
}

.page-header  { grid-area: header; }
.page-sidebar { grid-area: sidebar; }
.page-content { grid-area: content; }
.page-aside   { grid-area: aside; }
.page-footer  { grid-area: footer; }

/* 用 . 表示空单元 */
.dashboard {
  grid-template-areas:
    "nav    nav    nav"
    "side   main   ."
    "side   main   .";
}
```

---

### 5. Grid 子项定位

```css
.grid-item {
  /* 通过网格线编号定位 */
  grid-column-start: 1;
  grid-column-end: 3;   /* 或 span 2 */
  grid-row-start: 1;
  grid-row-end: 2;

  /* 简写 */
  grid-column: 1 / 3;    /* 第1条线到第3条线 */
  grid-column: 1 / span 2; /* 从第1条线跨 2 列 */
  grid-row: 1 / 2;

  /* 超简写 grid-area: row-start / col-start / row-end / col-end */
  grid-area: 1 / 1 / 3 / 3; /* 占据 2×2 的区域 */
}
```

---

### 6. Grid 对齐

```css
.grid-container {
  /* 所有子项在单元格内的对齐 */
  justify-items: stretch;   /* 水平：start | end | center | stretch */
  align-items: stretch;     /* 垂直：start | end | center | stretch */
  place-items: center;      /* 简写：align-items justify-items */

  /* 整个网格在容器内的对齐（容器有剩余空间时） */
  justify-content: start;   /* start | end | center | space-between | space-around | space-evenly */
  align-content: start;
  place-content: center;
}

.grid-item {
  /* 单个子项覆盖对齐 */
  justify-self: center;
  align-self: end;
  place-self: center end;
}
```

---

### 7. 隐式网格与 auto 放置

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: 100px 100px;

  /* 超出模板定义的行/列会创建隐式轨道 */
  grid-auto-rows: 80px;           /* 隐式行的高度 */
  grid-auto-columns: 1fr;         /* 隐式列的宽度 */
  grid-auto-flow: row;            /* 自动放置方向：row | column | dense */
}

/* dense — 紧密填充，可能打乱 DOM 顺序 */
.grid-dense {
  grid-auto-flow: row dense;
  /* 后面的小项目会填充前面的空洞 */
}
```

**面试追问**：
- `auto-fit` 和 `auto-fill` 在什么情况下表现一致？（项目数量足够多时）
- Grid 和 Flex 如何选择？（一维用 Flex，二维用 Grid；或按内容驱动 vs 布局驱动选）
- Subgrid 解决了什么问题？

---

## 响应式设计

> ⭐⭐⭐ 现代 Web 开发必备能力

### 1. 媒体查询（Media Queries）

```css
/* 移动优先策略（推荐） — 先写移动端，再用 min-width 扩展 */
.container {
  width: 100%;
  padding: 10px;
}

/* 平板 */
@media (min-width: 768px) {
  .container {
    max-width: 720px;
    margin: 0 auto;
  }
}

/* 桌面 */
@media (min-width: 1024px) {
  .container {
    max-width: 960px;
  }
}

/* 大屏 */
@media (min-width: 1200px) {
  .container {
    max-width: 1140px;
  }
}

/* 常见断点（参考 Tailwind / Bootstrap） */
/*
  sm:  640px
  md:  768px
  lg:  1024px
  xl:  1280px
  2xl: 1536px
*/

/* 其他媒体特性 */
@media (prefers-color-scheme: dark) {
  /* 用户偏好暗色模式 */
  :root { --bg: #1a1a1a; --text: #e0e0e0; }
}

@media (prefers-reduced-motion: reduce) {
  /* 用户偏好减少动画 */
  * { animation-duration: 0.01ms !important; }
}

@media (hover: hover) {
  /* 设备支持 hover */
  .card:hover { transform: scale(1.02); }
}

@media print {
  .no-print { display: none; }
}
```

---

### 2. 相对单位详解

```css
/* rem — 相对根元素（html）字体大小 */
html { font-size: 16px; }       /* 1rem = 16px */
.box { font-size: 1.5rem; }     /* 24px */
.box .child { font-size: 1rem; } /* 16px（始终相对 html） */

/* em — 相对当前元素（或父元素）字体大小 */
.parent { font-size: 20px; }
.parent .child { 
  font-size: 0.8em;              /* 16px (20 × 0.8) */
  padding: 1em;                   /* 16px (相对自身 font-size) */
}

/* vw / vh — 相对视口 */
.hero {
  width: 100vw;       /* 视口宽度 */
  height: 100vh;      /* 视口高度 */
}

/* vmin / vmax */
.square {
  width: 50vmin;  /* 视口宽高中较小值的 50% */
  height: 50vmin;
}

/* svh / lvh / dvh — 移动端视口单位（推荐） */
.mobile-full {
  min-height: 100dvh; /* 动态视口高度 — 考虑浏览器 UI 变化 */
}

/* % — 相对父元素 */
.child {
  width: 50%;          /* 父元素 content-box 宽度的 50% */
  height: 50%;         /* 父元素 content-box 高度的 50% */
  padding-top: 10%;    /* ⚠️ 相对父元素宽度，不是高度！ */
  margin-top: 10%;     /* ⚠️ 同样相对父元素宽度 */
}
```

**面试追问**：
- `padding-top: 50%` 为什么相对的是父元素宽度？（CSS 规范：保证百分比计算的一致性）
- `100vh` 在移动端的坑是什么？（地址栏收缩导致实际高度变化，用 `100dvh`）
- `rem` 和 `em` 分别适合什么场景？（rem 用于全局尺寸，em 用于组件内比例）

---

### 3. Clamp() 函数 ⭐⭐

```css
/* clamp(最小值, 首选值, 最大值) */
.responsive-text {
  /* 字体在 16px~32px 之间，根据视口宽度流式变化 */
  font-size: clamp(1rem, 2.5vw, 2rem);
}

.responsive-container {
  /* 宽度在 300px~1200px 之间流式变化 */
  width: clamp(300px, 80vw, 1200px);
  margin: 0 auto;
}

.responsive-gap {
  gap: clamp(10px, 2vw, 30px);
}

/* 等价写法 — min(max(最小, 首选), 最大) */
.equivalent {
  font-size: min(max(1rem, 2.5vw), 2rem);
}

/* 实用公式：流体排版 */
/* font-size: clamp(minSize, preferredVw, maxSize) */
h1 { font-size: clamp(1.5rem, 4vw + 0.5rem, 3rem); }
h2 { font-size: clamp(1.25rem, 3vw + 0.25rem, 2.25rem); }
p  { font-size: clamp(0.875rem, 1.5vw + 0.25rem, 1.125rem); }
```

---

### 4. Container Queries ⭐⭐

```css
/* 容器查询 — 基于容器尺寸而非视口 */

/* 1. 定义容器 */
.card-wrapper {
  container-type: inline-size;   /* 开启内联方向（宽度）容器查询 */
  container-name: card;          /* 命名（可选） */
  /* 简写 */
  container: card / inline-size;
}

/* 2. 根据容器宽度变化布局 */
@container card (min-width: 400px) {
  .card {
    display: flex;
    gap: 20px;
  }
  .card-image {
    width: 40%;
  }
}

@container card (min-width: 700px) {
  .card {
    flex-direction: row;
  }
  .card-image {
    width: 30%;
  }
  .card-body {
    font-size: 1.1rem;
  }
}

/* 容器查询单位 */
.card-title {
  font-size: clamp(1rem, 3cqi, 1.5rem); /* cqi = 容器内联尺寸的 1% */
}
/* cqw / cqh / cqi / cqb / cqmin / cqmax */
```

**效果说明**：与媒体查询不同，容器查询响应的是 **组件容器** 的尺寸而非视口。这使得同一个组件放在不同宽度的区域时能自适应布局。

**面试追问**：
- 容器查询和媒体查询的核心区别？（容器 vs 视口）
- `container-type: inline-size` 为什么不用 `size`？（`size` 会在两个方向都建立包含，可能导致循环依赖）
- 容器查询单位（cqi/cqw）与视口单位（vw/vh）的区别？

---

## CSS 变量与主题

> ⭐⭐ 自定义属性是现代 CSS 主题方案的基础

### 1. CSS 自定义属性（CSS Variables）

```css
/* 声明 — 在 :root 中定义全局变量 */
:root {
  --color-primary: #1976d2;
  --color-secondary: #ff6f00;
  --color-bg: #ffffff;
  --color-text: #212121;
  --font-base: 16px;
  --spacing-unit: 8px;
  --radius-sm: 4px;
  --radius-md: 8px;
  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.12);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
  --transition-fast: 150ms ease;
  --transition-normal: 300ms ease;
}

/* 使用 — var(变量名, 兜底值) */
.button {
  background: var(--color-primary);
  color: white;
  padding: calc(var(--spacing-unit) * 1.5) calc(var(--spacing-unit) * 3);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-sm);
  transition: all var(--transition-fast);
}

.button:hover {
  box-shadow: var(--shadow-md);
}

/* 兜底值 */
.card {
  color: var(--card-color, var(--color-text)); /* 嵌套兜底 */
}

/* 局部覆盖 — 变量可在任何选择器中覆盖 */
.sidebar {
  --color-primary: #e91e63; /* 只在 .sidebar 内生效 */
}

/* 与 calc() 结合 */
.grid {
  --columns: 3;
  grid-template-columns: repeat(var(--columns), 1fr);
  gap: calc(var(--spacing-unit) * 2);
}
```

---

### 2. 暗色模式实现

```css
/* 方案一：媒体查询自动适配 */
:root {
  --bg: #ffffff;
  --text: #212121;
  --surface: #f5f5f5;
  --border: #e0e0e0;
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg: #121212;
    --text: #e0e0e0;
    --surface: #1e1e1e;
    --border: #333333;
  }
}

/* 方案二：class 手动切换（用户可控） */
:root,
[data-theme="light"] {
  --bg: #ffffff;
  --text: #212121;
  --surface: #f5f5f5;
}

[data-theme="dark"] {
  --bg: #121212;
  --text: #e0e0e0;
  --surface: #1e1e1e;
}

/* 方案三：两者结合 — 默认跟随系统，允许手动覆盖 */
:root {
  color-scheme: light dark;
  --bg: #ffffff;
  --text: #212121;
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --bg: #121212;
    --text: #e0e0e0;
  }
}

[data-theme="dark"] {
  --bg: #121212;
  --text: #e0e0e0;
}

/* 使用 */
body {
  background: var(--bg);
  color: var(--text);
  transition: background-color 0.3s ease, color 0.3s ease;
}
```

```javascript
// JavaScript 切换主题
const toggle = document.getElementById('theme-toggle');
toggle.addEventListener('click', () => {
  const current = document.documentElement.getAttribute('data-theme');
  const next = current === 'dark' ? 'light' : 'dark';
  document.documentElement.setAttribute('data-theme', next);
  localStorage.setItem('theme', next);
});

// 初始化时读取
const saved = localStorage.getItem('theme');
if (saved) {
  document.documentElement.setAttribute('data-theme', saved);
}
```

---

### 3. CSS 方案对比

| 特性 | CSS Modules | CSS-in-JS (styled-components) | Tailwind CSS | Sass/Less |
|------|-------------|------------------------------|-------------|-----------|
| 作用域隔离 | ✅ 自动 hash | ✅ 运行时生成 | ✅ 原子类 | ❌ 需 BEM |
| 运行时开销 | ❌ 无 | ⚠️ 有 | ❌ 无 | ❌ 无 |
| 动态样式 | ⚠️ 需 JS | ✅ 原生支持 | ⚠️ class 切换 | ❌ 编译时 |
| 包体积 | 中 | 中 | ✅ 小（按需） | 中 |
| 开发体验 | 中 | ✅ 强 | ✅ 快 | 中 |
| SSR 支持 | ✅ | ⚠️ 需额外配置 | ✅ | ✅ |
| 学习成本 | 低 | 中 | 中 | 低 |

```css
/* CSS Modules 示例 */
/* Button.module.css */
.button {
  background: var(--color-primary);
  padding: 8px 16px;
  border-radius: 4px;
}
.button:hover {
  opacity: 0.9;
}
```

```jsx
// React 中使用 CSS Modules
import styles from './Button.module.css';

function Button({ children }) {
  return <button className={styles.button}>{children}</button>;
  // 编译后: <button class="Button_button__x7f2a">
}
```

```jsx
// styled-components 示例
import styled from 'styled-components';

const StyledButton = styled.button`
  background: ${props => props.primary ? '#1976d2' : '#ccc'};
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
  
  &:hover {
    opacity: 0.9;
  }
`;
```

```html
<!-- Tailwind CSS 示例 -->
<button class="bg-blue-600 text-white px-4 py-2 rounded hover:opacity-90 
               transition-opacity dark:bg-blue-500">
  Click me
</button>
```

**面试追问**：
- CSS Variables 和 Sass 变量的区别？（运行时 vs 编译时，CSS Variables 可用 JS 操作）
- CSS-in-JS 的运行时开销从何而来？（动态生成 style 标签、序列化 CSS、className hash）
- 如何在 Tailwind 中做主题切换？（`dark:` 变体 + CSS 变量）

---

## CSS 动画与性能

> ⭐⭐ 动画是用户体验的核心，性能优化是进阶必考

### 1. Transition（过渡）

```css
.button {
  background: #1976d2;
  color: white;
  padding: 10px 20px;
  border-radius: 4px;

  /* 单独指定 */
  transition-property: background-color, transform, box-shadow;
  transition-duration: 300ms;
  transition-timing-function: ease;
  transition-delay: 0ms;

  /* 简写 */
  transition: background-color 300ms ease,
              transform 200ms ease-out,
              box-shadow 300ms ease;
}

.button:hover {
  background: #1565c0;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

/* 常用缓动函数 */
.ease-examples {
  transition-timing-function: linear;          /* 匀速 */
  transition-timing-function: ease;            /* 先慢后快再慢（默认） */
  transition-timing-function: ease-in;         /* 先慢后快 */
  transition-timing-function: ease-out;        /* 先快后慢 */
  transition-timing-function: ease-in-out;     /* 两端慢中间快 */
  transition-timing-function: cubic-bezier(0.68, -0.55, 0.27, 1.55); /* 自定义弹性 */
}
```

**transition 的局限**：
- 只能在两个状态之间过渡，不支持多关键帧
- 需要触发条件（:hover、class 切换等）
- 不能自动播放，不能循环
- 不能对 `display` 进行过渡

---

### 2. Animation + @keyframes

```css
/* 定义关键帧 */
@keyframes fadeSlideIn {
  0% {
    opacity: 0;
    transform: translateY(20px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 多关键帧 */
@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  25%      { transform: translateY(-15px); }
  50%      { transform: translateY(0); }
  75%      { transform: translateY(-8px); }
}

/* 应用动画 */
.animated-element {
  animation-name: fadeSlideIn;
  animation-duration: 600ms;
  animation-timing-function: ease-out;
  animation-delay: 0ms;
  animation-iteration-count: 1;         /* 播放次数，infinite 无限 */
  animation-direction: normal;           /* normal | reverse | alternate | alternate-reverse */
  animation-fill-mode: forwards;         /* none | forwards | backwards | both */
  animation-play-state: running;         /* running | paused */

  /* 简写 */
  animation: fadeSlideIn 600ms ease-out forwards;
}

/* fill-mode 详解 */
.fill-none     { animation-fill-mode: none; }      /* 动画前后都不保持状态 */
.fill-forwards { animation-fill-mode: forwards; }   /* 动画结束后保持最后一帧 */
.fill-backwards { animation-fill-mode: backwards; } /* 动画开始前应用第一帧 */
.fill-both     { animation-fill-mode: both; }       /* 两者兼顾 */

/* 加载动画示例 */
@keyframes spin {
  to { transform: rotate(360deg); }
}

.spinner {
  width: 40px;
  height: 40px;
  border: 3px solid #e0e0e0;
  border-top-color: #1976d2;
  border-radius: 50%;
  animation: spin 800ms linear infinite;
}

/* 骨架屏闪烁效果 */
@keyframes shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}

.skeleton {
  background: linear-gradient(
    90deg,
    #f0f0f0 25%,
    #e0e0e0 50%,
    #f0f0f0 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s ease-in-out infinite;
}
```

---

### 3. 动画性能优化 ⭐⭐

**浏览器渲染管线**：
```
JavaScript → Style → Layout → Paint → Composite
```

**性能分层**：

| 属性 | 触发阶段 | 性能 | 示例 |
|------|---------|------|------|
| transform | Composite | ✅ 最好 | translate, scale, rotate |
| opacity | Composite | ✅ 最好 | 透明度变化 |
| filter | Paint + Composite | ⚠️ 一般 | blur, brightness |
| background-color | Paint | ⚠️ 一般 | 颜色变化 |
| width/height | Layout | ❌ 最差 | 尺寸变化 |
| top/left | Layout | ❌ 最差 | 位置变化 |

```css
/* ❌ 差 — 触发 Layout（重排） */
.bad-animation {
  transition: width 300ms, left 300ms, top 300ms;
}
.bad-animation:hover {
  width: 200px;
  left: 10px;
  top: 10px;
}

/* ✅ 好 — 只触发 Composite（合成） */
.good-animation {
  transition: transform 300ms, opacity 300ms;
}
.good-animation:hover {
  transform: scale(1.1) translateX(10px);
  opacity: 0.8;
}
```

---

### 4. GPU 加速与 will-change

```css
/* will-change — 提前告知浏览器即将变化的属性 */
.will-animate {
  will-change: transform, opacity;
  /* 浏览器会为该元素提前创建独立的合成层 */
}

/* ⚠️ 注意事项 */
/* 1. 不要滥用 — 每个合成层都消耗内存 */
/* ❌ 错误 — 全局使用 */
* { will-change: transform; }

/* ✅ 正确 — 按需使用 */
.card {
  transition: transform 300ms;
}
.card:hover {
  will-change: transform; /* hover 时才开启 */
  transform: scale(1.05);
}

/* 2. 动画结束后移除 */
.card.animating {
  will-change: transform;
}

/* 3. 旧方案：translateZ(0) / translate3d(0,0,0) 强制 GPU 加速 */
.force-gpu {
  transform: translateZ(0); /* hack — 创建合成层 */
  /* 现代浏览器应使用 will-change 替代 */
}
```

**`contain` 属性 — 限制渲染范围**：

```css
.widget {
  contain: layout style paint;
  /* layout: 内部布局不影响外部 */
  /* style: 内部计数器等不影响外部 */
  /* paint: 内部绘制不会溢出容器 */
  /* size: 自身尺寸不依赖子元素 */
  /* content = layout + style + paint */
  /* strict = layout + style + paint + size */
}
```

**面试追问**：
- 什么属性的变化会触发重排（Reflow）？（几何属性：width/height/margin/padding/top/left 等）
- `transform` 动画为什么不触发重排？（在合成层上操作，不影响布局树）
- `will-change` 为什么不能写在所有元素上？（内存开销、可能导致更多合成层）
- 什么是隐式合成（Implicit Compositing）？（低层级元素被提升到合成层，可能导致层爆炸）

---

## CSS 选择器与优先级

> ⭐⭐⭐ 理解优先级是解决样式冲突的基础

### 1. 选择器分类

```css
/* 基础选择器 */
*            { }  /* 通配符 */
div          { }  /* 元素/类型 */
.class       { }  /* 类 */
#id          { }  /* ID */
[attr]       { }  /* 属性 */

/* 组合选择器 */
A B          { }  /* 后代（空格） */
A > B        { }  /* 直接子元素 */
A + B        { }  /* 相邻兄弟（紧挨着的下一个） */
A ~ B        { }  /* 通用兄弟（后面的所有兄弟） */

/* 伪类 — 表示元素的状态 */
:hover       { }  /* 鼠标悬停 */
:focus       { }  /* 获得焦点 */
:active      { }  /* 激活（按下） */
:visited     { }  /* 已访问链接 */
:first-child { }  /* 第一个子元素 */
:last-child  { }  /* 最后一个子元素 */
:nth-child(n){ }  /* 第 n 个子元素 */
:not(.cls)   { }  /* 否定 */
:is(.a, .b)  { }  /* 匹配列表中的任一 */
:where(.a)   { }  /* 同 :is()，但权重为 0 */
:has(.child) { }  /* 父元素选择器 ⭐ */

/* 伪元素 — 创建虚拟元素 */
::before     { content: ''; } /* 元素前插入 */
::after      { content: ''; } /* 元素后插入 */
::first-line { }               /* 第一行文本 */
::first-letter { }             /* 首字母 */
::placeholder { }              /* 占位符文本 */
::selection  { }               /* 用户选中文本 */
```

---

### 2. 权重计算 ⭐⭐⭐

**优先级从高到低**：

```
1. !important                             → 最高（慎用）
2. 内联样式 style=""                       → (1, 0, 0, 0)
3. ID 选择器 #id                           → (0, 1, 0, 0)
4. 类/伪类/属性选择器 .class :hover [attr] → (0, 0, 1, 0)
5. 元素/伪元素选择器 div ::before           → (0, 0, 0, 1)
6. 通配符 * 、组合符 > + ~                  → (0, 0, 0, 0)
```

**计算示例**：

```css
/* (0, 0, 0, 1) — 1 个元素选择器 */
p { color: red; }

/* (0, 0, 1, 0) — 1 个类选择器 */
.text { color: blue; }

/* (0, 0, 1, 1) — 1 个类 + 1 个元素 */
p.text { color: green; }

/* (0, 1, 0, 0) — 1 个 ID */
#main { color: orange; }

/* (0, 1, 1, 1) — 1 个 ID + 1 个类 + 1 个元素 */
#main p.text { color: purple; }

/* (0, 0, 2, 1) — 2 个类 + 1 个元素 */
.container .card p { color: teal; }

/* 比较方式：从左到右逐位比较，不进位 */
/* (0, 1, 0, 0) > (0, 0, 99, 99) — ID 永远大于任意数量的类 */
```

**特殊规则**：
```css
/* :not() 本身不计权重，但括号内的选择器计算 */
p:not(.special) { } /* (0, 0, 1, 1) — .special 贡献一个类 */

/* :is() 取括号内权重最高的 */
:is(#id, .class) p { } /* (0, 1, 0, 1) — 取 #id 的权重 */

/* :where() 权重始终为 0 */
:where(#id, .class) p { } /* (0, 0, 0, 1) — :where() 贡献 0 */

/* :has() 与 :is() 相同规则 */
div:has(> .active) { } /* (0, 0, 1, 1) */
```

---

### 3. CSS 层叠规则（Cascade）

**完整层叠优先级（从高到低）**：

```
1. 用户代理 !important（浏览器强制）
2. 用户 !important
3. 作者 !important（开发者 CSS 中的 !important）
4. 作者 @layer（Cascade Layers，后面的层优先）
5. 作者普通样式
6. 用户普通样式
7. 用户代理普通样式（浏览器默认样式）
```

**同优先级时的决定因素**：
1. 选择器权重（Specificity）
2. 出现顺序（后出现的覆盖先出现的）
3. 来源（同一文件中，后面的规则覆盖前面的）

```css
/* !important 的正确使用场景 */
/* ✅ 工具类 / 覆盖第三方库样式 */
.hidden { display: none !important; }
.sr-only {
  position: absolute !important;
  width: 1px !important;
  height: 1px !important;
  overflow: hidden !important;
  clip: rect(0, 0, 0, 0) !important;
}

/* ❌ 不要用 !important 解决优先级问题 */
.button { color: red !important; } /* 不推荐 */
```

**面试追问**：
- 如何覆盖一个 `!important` 样式？（更高权重的 `!important`，或使用 Cascade Layers）
- `:is()` 和 `:where()` 的使用场景区别？（`:where` 用于低权重重置样式）
- 内联样式 vs `!important` 谁优先？（`!important` 更高）

---

## 现代 CSS 新特性

> 现代浏览器支持的新一代 CSS 能力

### 1. Cascade Layers（@layer） ⭐⭐

```css
/* 声明层顺序 — 后面的层优先级更高 */
@layer reset, base, components, utilities;

/* 定义层内容 */
@layer reset {
  *, *::before, *::after {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }
}

@layer base {
  body {
    font-family: system-ui, sans-serif;
    line-height: 1.5;
    color: #333;
  }
  a { color: #1976d2; }
}

@layer components {
  .button {
    padding: 8px 16px;
    border-radius: 4px;
    background: #1976d2;
    color: white;
  }
  .card {
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    padding: 16px;
  }
}

@layer utilities {
  .mt-4  { margin-top: 1rem; }
  .mb-4  { margin-bottom: 1rem; }
  .hidden { display: none; }
}

/* 未分层的样式优先级 > 所有层 */
.special-button {
  background: red; /* 优先于 @layer 中的样式 */
}
```

**效果说明**：Cascade Layers 让开发者可以显式控制样式层叠顺序，解决了传统 CSS 中选择器权重战争的问题。第三方库样式可放在低优先级层中，方便覆盖。

---

### 2. CSS Nesting（原生嵌套）

```css
/* 原生 CSS 嵌套（已被主流浏览器支持） */
.card {
  padding: 16px;
  border-radius: 8px;

  /* 嵌套子选择器 */
  .title {
    font-size: 1.25rem;
    font-weight: bold;
  }

  .body {
    margin-top: 12px;
    color: #666;
  }

  /* 嵌套伪类 */
  &:hover {
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  }

  /* 嵌套媒体查询 */
  @media (min-width: 768px) {
    padding: 24px;
  }

  /* & 在复杂选择器中 */
  .dark & {
    background: #1e1e1e;
  }
}

/* 等价于传统写法 */
.card { padding: 16px; border-radius: 8px; }
.card .title { font-size: 1.25rem; font-weight: bold; }
.card .body { margin-top: 12px; color: #666; }
.card:hover { box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1); }
@media (min-width: 768px) { .card { padding: 24px; } }
.dark .card { background: #1e1e1e; }
```

---

### 3. :has() 选择器 — "父元素选择器" ⭐⭐

```css
/* 选择包含特定子元素的父元素 */
.card:has(img) {
  /* 有图片的卡片 */
  grid-template-rows: auto 1fr;
}

.card:has(> .badge) {
  /* 直接子元素中有 .badge 的卡片 */
  position: relative;
}

/* 表单验证样式 */
.form-group:has(input:invalid) {
  /* 输入框无效时，整个表单组变红 */
  border-color: red;
}

.form-group:has(input:invalid) .label {
  color: red;
}

/* 选择后面没有特定兄弟的元素 */
h2:not(:has(+ p)) {
  /* 后面不紧跟 p 的 h2 */
  margin-bottom: 2rem;
}

/* 网格布局中根据子项数量调整 */
.grid:has(> :nth-child(4)) {
  /* 超过 3 个子项时 */
  grid-template-columns: repeat(4, 1fr);
}
```

---

### 4. Subgrid

```css
.parent-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 20px;
}

.child-grid {
  grid-column: span 4;
  display: grid;
  /* subgrid — 继承父网格的轨道定义 */
  grid-template-columns: subgrid;
  /* 子元素将对齐到父网格线 */
}

/* 常见场景：卡片列表中的标题/内容/按钮对齐 */
.card-list {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto;
  gap: 20px;
}

.card {
  display: grid;
  grid-row: span 3; /* 每个卡片占 3 行 */
  grid-template-rows: subgrid; /* 继承父级行轨道 */
  /* 所有卡片的标题行、内容行、按钮行自动对齐 */
}
```

---

### 5. 其他现代特性

```css
/* color-mix() — 颜色混合 */
.button {
  background: oklch(0.6 0.2 250);
}
.button:hover {
  /* hover 时混入黑色变暗 */
  background: color-mix(in oklch, oklch(0.6 0.2 250) 80%, black);
}

/* text-wrap: balance — 标题文本均匀分行 */
h1 {
  text-wrap: balance;
  /* 让多行标题每行字数尽可能均匀 */
}

/* scroll-driven animations — 滚动驱动动画 */
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(50px); }
  to   { opacity: 1; transform: translateY(0); }
}

.scroll-reveal {
  animation: fadeIn linear;
  animation-timeline: view();
  animation-range: entry 0% entry 100%;
}

/* anchor positioning — 锚点定位 */
.trigger {
  anchor-name: --my-anchor;
}

.tooltip {
  position: fixed;
  position-anchor: --my-anchor;
  top: anchor(bottom);
  left: anchor(center);
}

/* @scope — 作用域限制 */
@scope (.card) to (.card-footer) {
  p { color: #666; } /* 只在 .card 到 .card-footer 之间生效 */
}
```

---

## CSS 工程化

> CSS 工程化解决大规模项目的样式管理问题

### 1. 预处理器：Sass / Less

```scss
// Sass/SCSS 核心特性

// ① 变量
$primary: #1976d2;
$spacing: 8px;

// ② 嵌套
.card {
  padding: $spacing * 2;
  
  &__title {     // BEM 命名
    font-size: 1.25rem;
  }
  
  &--active {    // BEM 修饰符
    border-color: $primary;
  }
  
  &:hover {
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }
}

// ③ Mixin — 可重用代码块
@mixin flex-center($direction: row) {
  display: flex;
  flex-direction: $direction;
  justify-content: center;
  align-items: center;
}

@mixin responsive($breakpoint) {
  @if $breakpoint == 'sm' { @media (min-width: 640px)  { @content; } }
  @if $breakpoint == 'md' { @media (min-width: 768px)  { @content; } }
  @if $breakpoint == 'lg' { @media (min-width: 1024px) { @content; } }
}

.hero {
  @include flex-center(column);
  @include responsive('md') {
    flex-direction: row;
  }
}

// ④ 继承
%button-base {
  padding: 8px 16px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
}

.btn-primary {
  @extend %button-base;
  background: $primary;
  color: white;
}

// ⑤ 函数
@function rem($px) {
  @return ($px / 16) * 1rem;
}

.text {
  font-size: rem(18); // 1.125rem
}
```

---

### 2. PostCSS 工具链

```javascript
// postcss.config.js
module.exports = {
  plugins: [
    // Autoprefixer — 自动添加浏览器前缀
    require('autoprefixer'),
    
    // postcss-preset-env — 使用未来 CSS 语法
    require('postcss-preset-env')({
      stage: 2,
      features: {
        'nesting-rules': true,
        'custom-properties': false, // 让浏览器原生处理
      },
    }),
    
    // cssnano — 压缩 CSS
    require('cssnano')({
      preset: 'default',
    }),
    
    // PurgeCSS / Tailwind 内置 — 移除未使用的 CSS
    require('@fullhuman/postcss-purgecss')({
      content: ['./src/**/*.{html,js,jsx,tsx}'],
    }),
  ],
};
```

---

### 3. 原子化 CSS 与 Tailwind

```css
/* 原子化 CSS 核心理念：每个类只做一件事 */
/* 类似于函数式编程的单一职责 */

/* 手写原子类 */
.flex     { display: flex; }
.items-center { align-items: center; }
.gap-4    { gap: 1rem; }
.p-4      { padding: 1rem; }
.rounded  { border-radius: 0.25rem; }
.bg-blue  { background-color: #1976d2; }
.text-white { color: white; }
```

```javascript
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{html,js,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        brand: {
          50:  '#e3f2fd',
          500: '#1976d2',
          900: '#0d47a1',
        },
      },
      spacing: {
        '18': '4.5rem',
      },
    },
  },
  plugins: [],
};
```

**CSS 工程化方案选型**：

| 项目规模 | 推荐方案 |
|---------|---------|
| 小型/个人 | Tailwind / 原生 CSS |
| 中型 SPA | CSS Modules + PostCSS |
| 大型团队 | Tailwind / CSS Modules + 设计系统 |
| 组件库 | CSS-in-JS / CSS Modules + CSS Variables |

---

## 实战案例

> 实现常见 UI 组件的 CSS 方案

### 1. 水平垂直居中（8 种方式）

```css
/* 所有方案的父容器 */
.parent {
  width: 400px;
  height: 300px;
  border: 2px solid #333;
}

/* ① Flex（推荐） */
.center-flex {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* ② Grid */
.center-grid {
  display: grid;
  place-items: center;
}

/* ③ Grid + margin */
.center-grid-margin {
  display: grid;
}
.center-grid-margin > .child {
  margin: auto;
}

/* ④ absolute + transform（不定宽高） */
.center-abs {
  position: relative;
}
.center-abs > .child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* ⑤ absolute + margin: auto（定宽高） */
.center-abs-margin {
  position: relative;
}
.center-abs-margin > .child {
  position: absolute;
  top: 0; right: 0; bottom: 0; left: 0;
  margin: auto;
  width: 200px;
  height: 100px;
}

/* ⑥ absolute + 负 margin（定宽高） */
.center-abs-neg {
  position: relative;
}
.center-abs-neg > .child {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 200px;
  height: 100px;
  margin-top: -50px;  /* -height/2 */
  margin-left: -100px; /* -width/2 */
}

/* ⑦ table-cell */
.center-table {
  display: table-cell;
  text-align: center;
  vertical-align: middle;
}
.center-table > .child {
  display: inline-block;
}

/* ⑧ line-height（单行文本） */
.center-line {
  text-align: center;
  line-height: 300px; /* 等于容器高度 */
}
.center-line > .child {
  display: inline-block;
  vertical-align: middle;
  line-height: normal; /* 重置 */
}
```

---

### 2. 三栏布局（5 种实现）

```css
/* 需求：左右固定 200px，中间自适应 */

/* ① Flex（推荐） */
.layout-flex {
  display: flex;
}
.layout-flex .left,
.layout-flex .right {
  flex: 0 0 200px;
}
.layout-flex .center {
  flex: 1;
}

/* ② Grid（推荐） */
.layout-grid {
  display: grid;
  grid-template-columns: 200px 1fr 200px;
  gap: 10px;
}

/* ③ Float + BFC */
.layout-float .left  { float: left; width: 200px; }
.layout-float .right { float: right; width: 200px; }
.layout-float .center { overflow: hidden; /* BFC */ }

/* ④ absolute */
.layout-abs {
  position: relative;
}
.layout-abs .left  { position: absolute; left: 0; width: 200px; }
.layout-abs .right { position: absolute; right: 0; width: 200px; }
.layout-abs .center { margin: 0 210px; }

/* ⑤ calc */
.layout-calc {
  display: flex;
}
.layout-calc .left,
.layout-calc .right { width: 200px; }
.layout-calc .center { width: calc(100% - 420px); }
```

---

### 3. 瀑布流布局

```css
/* CSS Column 方式 */
.masonry-column {
  column-count: 3;
  column-gap: 16px;
}

.masonry-column .item {
  break-inside: avoid;    /* 防止跨列断裂 */
  margin-bottom: 16px;
}

/* Grid + Masonry（实验性） */
.masonry-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: masonry;   /* 实验性 — Firefox 已支持 */
  gap: 16px;
}

/* 实际项目中推荐 JavaScript 方案（如 Masonry.js）配合 Grid 实现 */
```

---

### 4. 文本溢出处理

```css
/* 单行溢出省略 */
.ellipsis {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 多行溢出省略（Webkit） */
.multi-ellipsis {
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 3;        /* 显示 3 行 */
  overflow: hidden;
}

/* 通用多行方案（兼容性更好） */
.multi-ellipsis-fallback {
  position: relative;
  max-height: 4.5em;             /* line-height × 行数 */
  line-height: 1.5em;
  overflow: hidden;
}
.multi-ellipsis-fallback::after {
  content: '...';
  position: absolute;
  bottom: 0;
  right: 0;
  padding-left: 20px;
  background: linear-gradient(to right, transparent, white 50%);
}
```

---

### 5. 自适应正方形

```css
/* 方案一：padding-top 百分比（经典） */
.square-padding {
  width: 50%;
  padding-top: 50%;         /* 百分比相对父元素宽度 */
  height: 0;
  position: relative;
}
.square-padding .content {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
}

/* 方案二：aspect-ratio（推荐） */
.square-aspect {
  width: 50%;
  aspect-ratio: 1 / 1;     /* 宽高比 1:1 */
}

/* 16:9 视频容器 */
.video-container {
  width: 100%;
  aspect-ratio: 16 / 9;
}

/* 方案三：vw 单位 */
.square-vw {
  width: 50vw;
  height: 50vw;
}
```

---

### 6. CSS 三角形与图形

```css
/* 三角形 — 利用 border 原理 */
.triangle-up {
  width: 0;
  height: 0;
  border-left: 50px solid transparent;
  border-right: 50px solid transparent;
  border-bottom: 80px solid #1976d2;
}

.triangle-right {
  width: 0;
  height: 0;
  border-top: 50px solid transparent;
  border-bottom: 50px solid transparent;
  border-left: 80px solid #1976d2;
}

/* 现代方案：clip-path */
.triangle-clip {
  width: 100px;
  height: 80px;
  background: #1976d2;
  clip-path: polygon(50% 0%, 0% 100%, 100% 100%);
}

/* 圆形 */
.circle {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  background: #1976d2;
}

/* 气泡框 */
.bubble {
  position: relative;
  padding: 12px 16px;
  background: #e3f2fd;
  border-radius: 8px;
}
.bubble::after {
  content: '';
  position: absolute;
  bottom: -8px;
  left: 20px;
  width: 0;
  height: 0;
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-top: 8px solid #e3f2fd;
}
```

---

## 面试题自查

### 治理补充（3 条经验）

1. **CSS 命名规范**：大型项目推荐 BEM（Block__Element--Modifier）+ 功能前缀（`l-` 布局 / `c-` 组件 / `u-` 工具），或直接使用 Tailwind 原子化方案。
2. **样式隔离策略**：优先使用 CSS Modules 或 Scoped CSS，避免全局样式污染；第三方组件库的样式覆盖应使用 `@layer` 或特定作用域选择器。
3. **性能预算**：关注 CSS 文件体积（gzip 后建议 < 50KB），利用 PurgeCSS / Tailwind 按需生成；避免深层选择器嵌套（浏览器从右向左匹配）。

---

### 初级高频速刷（5 题）

**Q1: `box-sizing: content-box` 和 `border-box` 的区别？**
A: `content-box`（默认）的 `width/height` 仅指 content 区域；`border-box` 的 `width/height` 包含 content + padding + border。现代项目推荐全局 `border-box`。

**Q2: 如何清除浮动？**
A: ① `display: flow-root`（推荐）；② clearfix 伪元素 `::after { content: ''; display: table; clear: both; }`；③ `overflow: hidden`（有裁剪副作用）；④ 父元素也浮动（不推荐）。

**Q3: Flex 布局中 `flex: 1` 是什么含义？**
A: `flex: 1` = `flex-grow: 1; flex-shrink: 1; flex-basis: 0%`。表示该项目会等分剩余空间，且 basis 为 0 意味着分配前不占初始空间。

**Q4: 如何实现水平垂直居中？**
A: 推荐：① `display: flex; justify-content: center; align-items: center;`；② `display: grid; place-items: center;`。兼容方案：`position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);`。

**Q5: CSS 选择器优先级怎么算？**
A: `!important > inline style > #id > .class/:pseudo/[attr] > element/::pseudo > *`。同权重按出现顺序，后面的覆盖前面的。

---

### 中级高频进阶（5 题）

**Q1: BFC 是什么？列举触发条件和应用场景。**
A: BFC（块级格式化上下文）是一个独立渲染区域，内部布局不影响外部。触发：`overflow: hidden/auto`、`display: flow-root/flex/grid/inline-block`、`position: absolute/fixed`、`float` 不为 none。应用：清除浮动、阻止 margin 折叠、自适应两栏布局。

**Q2: `display: none` / `visibility: hidden` / `opacity: 0` 的区别？**
A: `display: none` — 不占空间、不响应事件、触发重排；`visibility: hidden` — 占空间、不响应事件、触发重绘；`opacity: 0` — 占空间、响应事件、可 GPU 加速。子元素继承行为也不同：`visibility` 的子元素可设为 `visible` 显示。

**Q3: 什么属性可以被继承？**
A: 可继承：`color`、`font-*`、`text-*`、`line-height`、`visibility`、`cursor`、`list-style`。不可继承：`width/height`、`margin/padding`、`border`、`display`、`position`、`background`。可以用 `inherit` 关键字强制继承。

**Q4: rem / em / vw 分别在什么场景使用？**
A: `rem` — 全局尺寸、间距、字体基础大小（方便整体缩放）；`em` — 组件内部的相对尺寸（如按钮 padding 跟随自身 font-size）；`vw` — 全屏布局、流式排版（配合 clamp()）。

**Q5: 如何实现 CSS 暗色模式？**
A: ① 使用 CSS Variables 定义颜色；② `@media (prefers-color-scheme: dark)` 自动跟随系统；③ `[data-theme="dark"]` 允许用户手动切换；④ 结合 `localStorage` 持久化主题选择。

---

### 深入问答（25 题 Q&A）

**Q1: margin 折叠的触发条件和计算规则是什么？** ⭐⭐⭐

A: Margin 折叠仅发生在 **块级元素** 的 **垂直方向**。三种场景：
1. 相邻兄弟：取较大值（正正 → max；正负 → 相加；负负 → 取绝对值大的负值）
2. 父子穿透：子元素的 margin-top 穿透到父元素外
3. 空元素自身：上下 margin 折叠

阻止方式：创建 BFC（`display: flow-root`）、添加 border/padding、使用 Flex/Grid 布局。

```css
/* 父子穿透示例 */
.parent { background: #eee; } /* margin-top 会穿透 */
.child  { margin-top: 20px; }

/* 修复 */
.parent { display: flow-root; }
```

---

**Q2: flex-grow 和 flex-shrink 的计算过程是什么？** ⭐⭐⭐

A:
**flex-grow**：剩余空间 = 容器宽度 - Σ(flex-basis)。每个 item 获得：剩余空间 × (自身 grow / Σgrow)。

**flex-shrink**：溢出量 = Σ(flex-basis) - 容器宽度。加权缩小值 = 溢出量 × (自身 shrink × basis / Σ(shrink × basis))。

```css
/* flex-shrink 计算 */
.container { display: flex; width: 500px; }
.a { flex: 0 1 200px; }
.b { flex: 0 3 200px; }
.c { flex: 0 1 200px; }
/* 溢出 = 600 - 500 = 100 */
/* 加权: 1*200=200, 3*200=600, 1*200=200，总=1000 */
/* a 缩小 100*(200/1000) = 20px → 180px */
/* b 缩小 100*(600/1000) = 60px → 140px */
/* c 缩小 100*(200/1000) = 20px → 180px */
```

---

**Q3: Grid 的 auto-fit 和 auto-fill 有什么区别？** ⭐⭐

A: 两者都在 `repeat()` 中自动计算列数。区别：当项目不够填满一行时——
- `auto-fill`：保留空轨道，它们占据空间
- `auto-fit`：折叠空轨道为 0，已有项目拉伸填满

```css
/* 2 个 item，容器 900px */
.auto-fill { grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); }
/* 创建 4 列：item item (空) (空) — 每列 225px */

.auto-fit { grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); }
/* 折叠空列：item item — 每列 450px */
```

---

**Q4: position: sticky 的工作原理和限制？** ⭐⭐

A: sticky 定位是 relative 和 fixed 的混合：滚动到阈值前表现为 relative，到达阈值后表现为 fixed。

限制：
1. 必须设置 `top/bottom/left/right` 之一作为阈值
2. 父容器不能 `overflow: hidden/auto/scroll`
3. sticky 元素的活动范围仅限于父容器内
4. 父容器高度必须大于 sticky 元素

```css
.sticky-nav {
  position: sticky;
  top: 0;
  z-index: 10;
  /* 滚动到页面顶部时固定 */
}
```

---

**Q5: transform 为什么不触发重排？动画性能优化原则是什么？** ⭐⭐⭐

A: `transform` 和 `opacity` 在 **合成层（Compositor Layer）** 上操作，不经过 Layout 和 Paint 阶段，直接在 GPU 上变换像素。

优化原则：
1. 只动画 `transform` 和 `opacity`
2. 使用 `will-change` 提前提升合成层（但不滥用）
3. 避免动画 `width/height/top/left`（触发 Layout）
4. 用 `translate` 替代 `top/left`
5. 尊重 `prefers-reduced-motion`

```css
/* ❌ */
.bad  { transition: left 300ms; left: 100px; }
/* ✅ */
.good { transition: transform 300ms; transform: translateX(100px); }
```

---

**Q6: :is() 和 :where() 的区别是什么？各自的使用场景？** ⭐⭐

A: 语法一样，都接受选择器列表。核心区别：**权重不同**。
- `:is()` — 取括号内权重最高的选择器权重
- `:where()` — 权重始终为 0

```css
:is(#id, .class) p  { }  /* 权重 (0,1,0,1) — 取 #id */
:where(#id, .class) p { } /* 权重 (0,0,0,1) — :where 贡献 0 */
```

场景：`:is()` 用于简化复合选择器；`:where()` 用于低权重的重置/基础样式，方便被覆盖。

---

**Q7: CSS Variables 和 Sass 变量的区别？** ⭐⭐

A:

| | CSS Variables | Sass 变量 |
|---|---|---|
| 运行时 | ✅ 浏览器运行时 | ❌ 编译时转为值 |
| JS 操作 | ✅ `getComputedStyle` / `setProperty` | ❌ |
| 继承 | ✅ 参与级联和继承 | ❌ 只是字符串替换 |
| 媒体查询内改变 | ✅ | ❌ |
| 兜底值 | ✅ `var(--x, fallback)` | 有 `!default` |
| 可用范围 | 属性值部分 | 任何地方（选择器/属性名） |

```css
/* CSS Variable 在媒体查询中动态变化 */
:root { --columns: 1; }
@media (min-width: 768px) { :root { --columns: 2; } }
@media (min-width: 1024px) { :root { --columns: 3; } }
.grid { grid-template-columns: repeat(var(--columns), 1fr); }
```

---

**Q8: 如何解决移动端 1px 边框问题？** ⭐⭐

A: 高 DPR 屏幕上 `1px` CSS 像素对应多个物理像素，看起来偏粗。

```css
/* 方案一：transform scale（推荐） */
.border-1px {
  position: relative;
}
.border-1px::after {
  content: '';
  position: absolute;
  left: 0; top: 0;
  width: 200%;
  height: 200%;
  border: 1px solid #e0e0e0;
  transform: scale(0.5);
  transform-origin: 0 0;
  pointer-events: none;
  border-radius: inherit;
}

/* 方案二：使用 0.5px（部分浏览器支持） */
.hairline {
  border: 0.5px solid #e0e0e0;
}

/* 方案三：viewport 缩放 */
/* <meta name="viewport" content="initial-scale=0.5"> */
```

---

**Q9: Cascade Layers (@layer) 解决了什么问题？** ⭐⭐

A: 解决了传统 CSS 中 **选择器权重战争** 的问题。在大型项目中，覆盖第三方库样式常常需要写更高权重的选择器甚至 `!important`。

`@layer` 让开发者显式控制层叠顺序，低层的样式无论权重多高都不会覆盖高层的样式。

```css
@layer third-party, base, components, utilities;

@layer third-party {
  .btn { color: red; }  /* 即使有 ID 选择器也不会覆盖上层 */
}

@layer utilities {
  .text-blue { color: blue; } /* 优先级高于 third-party 层 */
}
```

---

**Q10: Container Queries 和 Media Queries 的区别和使用场景？** ⭐⭐

A:
- **Media Queries**：基于 **视口** 尺寸，全局响应
- **Container Queries**：基于 **容器** 尺寸，组件级响应

Container Queries 的优势：同一个组件放在不同宽度的容器（侧边栏 vs 主内容区）中可以自动切换布局，无需 JavaScript 或多套媒体查询。

```css
.card-container {
  container: card / inline-size;
}

/* 容器宽度小于 400px — 垂直布局 */
@container card (max-width: 399px) {
  .card { flex-direction: column; }
}

/* 容器宽度 ≥ 400px — 水平布局 */
@container card (min-width: 400px) {
  .card { flex-direction: row; }
}
```

---

**Q11: 如何实现一个 CSS-only 的 Tooltip？** ⭐

A:

```css
.tooltip-trigger {
  position: relative;
  cursor: pointer;
}

.tooltip-trigger::after {
  content: attr(data-tooltip);
  position: absolute;
  bottom: calc(100% + 8px);
  left: 50%;
  transform: translateX(-50%);
  padding: 6px 12px;
  background: #333;
  color: white;
  font-size: 14px;
  border-radius: 4px;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: opacity 200ms ease, transform 200ms ease;
  transform: translateX(-50%) translateY(4px);
}

.tooltip-trigger::before {
  content: '';
  position: absolute;
  bottom: calc(100% + 2px);
  left: 50%;
  transform: translateX(-50%);
  border: 6px solid transparent;
  border-top-color: #333;
  opacity: 0;
  transition: opacity 200ms ease;
}

.tooltip-trigger:hover::after,
.tooltip-trigger:hover::before {
  opacity: 1;
  transform: translateX(-50%) translateY(0);
}
```

```html
<span class="tooltip-trigger" data-tooltip="这是提示文本">悬停查看</span>
```

---

**Q12: `display: inline-block` 元素之间的空白间隙如何消除？** ⭐⭐

A:

```css
/* 方案一：父元素 font-size: 0（推荐） */
.parent {
  font-size: 0;
}
.parent .child {
  font-size: 16px; /* 恢复 */
  display: inline-block;
}

/* 方案二：负 margin */
.child {
  display: inline-block;
  margin-right: -4px;
}

/* 方案三：直接使用 Flex（根本解决） */
.parent {
  display: flex;
}

/* 方案四：HTML 中消除空白 */
/* <div class="child">A</div><div class="child">B</div> */
```

---

**Q13: 浏览器是如何解析 CSS 选择器的？** ⭐⭐

A: 浏览器 **从右向左** 解析选择器。例如 `.nav ul li a`，先找到所有 `a`，然后逐级向上匹配祖先是否符合 `li > ul > .nav`。

这意味着：
- 最右侧（关键选择器）应该尽可能具体
- 避免使用 `*` 作为关键选择器
- 减少选择器嵌套深度

```css
/* ❌ 低效 — 先匹配所有 a，再逐级向上 */
.page .content .sidebar .nav ul li a { }

/* ✅ 高效 */
.nav-link { }
```

---

**Q14: 如何用纯 CSS 实现一个 Sticky Footer？三种方式。** ⭐⭐

A:

```css
/* 方式一：Flex（推荐） */
.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}
.app main { flex: 1; }

/* 方式二：Grid */
.app {
  display: grid;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
}

/* 方式三：calc */
main {
  min-height: calc(100vh - 60px - 80px); /* 减去 header/footer 高度 */
}
```

---

**Q15: `will-change` 的正确使用方式和陷阱？** ⭐⭐

A: `will-change` 提示浏览器元素即将发生变化，可以提前优化（如创建合成层）。

```css
/* ✅ 正确：动态添加 */
.card { transition: transform 300ms; }
.card:hover { will-change: transform; }

/* ✅ 正确：JavaScript 控制 */
/* el.style.willChange = 'transform'; → 动画 → el.style.willChange = 'auto'; */

/* ❌ 错误：全局/永久使用 */
* { will-change: transform, opacity; }
/* 问题：大量合成层消耗内存，反而降低性能 */
```

---

**Q16: clamp() 函数的原理和实用场景？** ⭐⭐

A: `clamp(min, preferred, max)` = `max(min, min(preferred, max))`。

```css
/* 流体排版 — 字体随视口流畅缩放 */
h1 { font-size: clamp(1.5rem, 4vw + 0.5rem, 3rem); }
/* 视口小时最小 1.5rem，大时最大 3rem，中间流式变化 */

/* 响应式间距 */
.section { padding: clamp(1rem, 5vw, 4rem); }

/* 响应式宽度 */
.container { width: clamp(300px, 80%, 1200px); }
```

---

**Q17: `aspect-ratio` 和传统 padding-top hack 的区别？** ⭐

A:

```css
/* padding-top hack — 内容需要绝对定位 */
.old-ratio {
  width: 100%;
  padding-top: 56.25%; /* 16:9 */
  height: 0;
  position: relative;
}
.old-ratio > * {
  position: absolute;
  top: 0; left: 0; width: 100%; height: 100%;
}

/* aspect-ratio — 现代方案，内容正常流动 */
.new-ratio {
  width: 100%;
  aspect-ratio: 16 / 9;
  /* 内容直接放在里面，无需绝对定位 */
}
```

区别：`aspect-ratio` 更简洁、内容参与正常文档流、不需要额外子元素包裹。

---

**Q18: 如何实现一个纯 CSS 的暗色模式切换动画？** ⭐⭐

A:

```css
:root {
  --bg: #ffffff;
  --text: #212121;
  --surface: #f5f5f5;
  --transition-theme: background-color 0.4s ease, color 0.4s ease, 
                      border-color 0.4s ease, box-shadow 0.4s ease;
}

[data-theme="dark"] {
  --bg: #121212;
  --text: #e0e0e0;
  --surface: #1e1e1e;
}

body {
  background: var(--bg);
  color: var(--text);
  transition: var(--transition-theme);
}

.card {
  background: var(--surface);
  transition: var(--transition-theme);
}

/* View Transition API（更高级） */
/* 配合 document.startViewTransition() 实现圆形展开效果 */
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 0.5s;
}
```

---

**Q19: 什么是层叠上下文（Stacking Context）？z-index 失效的原因？** ⭐⭐⭐

A: 层叠上下文是 HTML 元素的三维概念，决定了元素在 Z 轴上的堆叠顺序。

创建层叠上下文的条件：
- `position: relative/absolute/fixed/sticky` + `z-index` 不为 auto
- `opacity` < 1
- `transform` 不为 none
- `filter` 不为 none
- `will-change` 为 opacity/transform 等
- `isolation: isolate`
- `display: flex/grid` 的子元素 + `z-index` 不为 auto

**z-index 失效原因**：元素不在同一个层叠上下文中比较，z-index 只在同级层叠上下文内有效。

```css
/* 示例：z-index 失效 */
.parent-a { position: relative; z-index: 1; }
.parent-b { position: relative; z-index: 2; }

.parent-a .child { position: relative; z-index: 9999; }
/* child 的 z-index 9999 无效 — 因为 parent-a 的 z-index (1) < parent-b (2) */
/* child 永远在 parent-b 下面 */
```

---

**Q20: CSS 中如何实现主题色的自动明暗变体？** ⭐

A:

```css
:root {
  --hue: 210;
  --primary: hsl(var(--hue), 70%, 50%);
  --primary-light: hsl(var(--hue), 70%, 65%);
  --primary-dark: hsl(var(--hue), 70%, 35%);
  --primary-bg: hsl(var(--hue), 70%, 95%);
}

/* 切换主题色只需改 --hue */
[data-color="red"]    { --hue: 0; }
[data-color="green"]  { --hue: 120; }
[data-color="purple"] { --hue: 270; }

/* 现代方案：oklch + color-mix */
:root {
  --primary: oklch(0.6 0.2 250);
  --primary-light: color-mix(in oklch, var(--primary) 60%, white);
  --primary-dark: color-mix(in oklch, var(--primary) 70%, black);
}
```

---

**Q21: `contain` 属性有什么作用？对性能有什么帮助？** ⭐

A: `contain` 限制元素的渲染影响范围，让浏览器可以做更精准的优化。

```css
.widget {
  contain: layout;  /* 内部布局不影响外部 */
  contain: paint;   /* 内部绘制不会溢出 */
  contain: size;    /* 自身尺寸不依赖子元素 */
  contain: style;   /* 内部计数器不影响外部 */
  contain: content; /* = layout + paint + style */
  contain: strict;  /* = layout + paint + style + size */
}
```

性能帮助：修改 `.widget` 内部时，浏览器知道不需要重新计算外部布局，可以局部重排/重绘。对虚拟列表中的每个项目使用 `contain: content` 可显著提升滚动性能。

---

**Q22: 如何用 CSS Grid 实现一个响应式卡片布局（自动列数）？** ⭐⭐

A:

```css
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 24px;
  padding: 24px;
}

.card {
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  overflow: hidden;
}

.card img {
  width: 100%;
  aspect-ratio: 16 / 9;
  object-fit: cover;
}

.card-body {
  padding: 16px;
}

/* 无需媒体查询 — Grid 自动计算列数 */
/* 容器 < 280px: 1 列 */
/* 容器 560px-839px: 2 列 */
/* 容器 840px+: 3+ 列 */
```

---

**Q23: CSS Modules 的原理是什么？如何解决样式冲突？** ⭐⭐

A: CSS Modules 在构建时对类名进行 **hash 转换**，将 `.button` 变为 `.Button_button__x7f2a`，保证全局唯一。

原理：
1. 编写 `Button.module.css`，使用普通类名
2. 构建工具（Webpack/Vite）处理导入，生成类名映射对象
3. 导出 `{ button: 'Button_button__x7f2a' }` 供 JS 使用
4. CSS 中的类名也被替换

```css
/* Button.module.css */
.button { background: blue; }
.button:hover { opacity: 0.9; }
/* 编译后 → .Button_button__x7f2a { background: blue; } */
```

```jsx
import styles from './Button.module.css';
// styles = { button: 'Button_button__x7f2a' }
<button className={styles.button}>Click</button>
```

组合（composes）允许复用：
```css
.base { padding: 8px 16px; border-radius: 4px; }
.primary { composes: base; background: blue; color: white; }
```

---

**Q24: 如何用 CSS 实现一个自适应的 16:9 视频容器，并在移动端变为 4:3？** ⭐

A:

```css
.video-wrapper {
  width: 100%;
  aspect-ratio: 16 / 9;
  background: black;
}

.video-wrapper iframe,
.video-wrapper video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

@media (max-width: 768px) {
  .video-wrapper {
    aspect-ratio: 4 / 3;
  }
}

/* 或使用 Container Query */
.video-container {
  container-type: inline-size;
}

@container (max-width: 500px) {
  .video-wrapper {
    aspect-ratio: 4 / 3;
  }
}
```

---

**Q25: CSS-in-JS 的运行时开销从何而来？有哪些 zero-runtime 替代方案？** ⭐⭐

A: 运行时开销：
1. **序列化**：将 JS 对象/模板字符串转为 CSS 字符串
2. **Hash 计算**：生成唯一 className
3. **DOM 操作**：动态插入/更新 `<style>` 标签
4. **上下文注入**：通过 React Context 传递 theme

Zero-runtime 替代：
- **Vanilla Extract**：编译时生成 CSS 文件，TypeScript 类型安全
- **Linaria**：编译时提取 CSS，语法类似 styled-components
- **Panda CSS**：编译时原子化 CSS
- **Tailwind CSS**：编译时按需生成原子类
- **CSS Modules**：编译时 hash，无运行时

```typescript
// Vanilla Extract 示例
import { style } from '@vanilla-extract/css';

export const button = style({
  background: 'blue',
  padding: '8px 16px',
  borderRadius: '4px',
  ':hover': { opacity: 0.9 },
});
// 编译后生成 .css 文件，零运行时
```

---

> **复习建议**：盒模型、BFC、Flex 是最高频考点，建议能手写代码。Grid 和现代特性是加分项。响应式和动画性能在实际项目中非常重要，需要结合实践理解。

### 开放式设计题

**D1：设计一个支持主题切换（亮/暗/自定义品牌色）的CSS架构，如何做到零运行时成本？**

**参考思路**：
- CSS变量体系：全局定义--color-primary/--color-bg等语义化变量，组件只引用变量不写死色值
- 主题切换：html[data-theme="dark"]下覆盖变量值，切换只需修改属性，浏览器自动重绘
- 零运行时：纯CSS实现（prefers-color-scheme媒体查询+属性选择器），无需JS运行时计算
- 品牌定制：CSS变量动态注入（style.setProperty）、构建时生成多份主题CSS、按需加载
- 关键注意：第三方组件库（antd/MUI）的主题穿透、图片/图标的暗色适配

**D2：一个复杂布局在iOS Safari上表现异常（底部被遮挡/弹窗无法滚动），如何排查？**

**参考思路**：
- 常见iOS坑：100vh包含地址栏（用dvh/svh代替）、position:fixed在软键盘弹起时失效、-webkit-overflow-scrolling:touch的副作用
- 安全区域：env(safe-area-inset-bottom)处理刘海屏底部
- 弹窗滚动穿透：body加overflow:hidden + position:fixed、记录并恢复scrollTop
- 调试：iOS Safari远程调试（Mac Safari→开发→iPhone）、BrowserStack远程真机
