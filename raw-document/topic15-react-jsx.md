# JSX 本质与 Vue 模板对照

## 讲解原文

### 一句话锚定

**JSX 不是模板，是 `createElement` 函数调用的语法糖；它的值是一个普通 JS 对象（ReactElement）。**

你已经会 Vue，所以本节不回答"JSX 怎么写"，只回答"JSX 是什么"--这决定了后面所有 React 心智模型的起点。

---

### 1. JSX 编译成什么

```jsx
const el = <h1 className="title">Hello {name}</h1>
```

构建时（Vite 用 esbuild/SWC，Babel 同理）被转译为：

```js
// React 17+ 的自动 runtime（不需要 import React）
import { jsx as _jsx } from 'react/jsx-runtime'
const el = _jsx('h1', { className: 'title', children: 'Hello ' + name })
```

多个 children 时用 `_jsxs`（注意结尾多了个 s，仅是打包 children 数组的细节）：

```jsx
const el = (
  <ul>
    <li>a</li>
    <li>b</li>
  </ul>
)
// 编译为
const el = _jsxs('ul', { children: [_jsx('li', {children:'a'}), _jsx('li', {children:'b'})] })
```

运行时 `_jsx(...)` 执行后返回的对象大致是：

```js
{
  $$typeof: Symbol(react.element),   // 防 XSS 注入的安全标记
  type: 'h1',                        // 字符串=DOM元素；函数=组件；这是分流依据
  key: null,
  ref: null,
  props: { className: 'title', children: 'Hello ' + name },  // children 只是普通 prop
}
```

三个立即可用的推论：

1. **JSX 是表达式**：可以存变量、放数组、当参数传、条件返回。所以 React 里"指令"不需要存在--`v-if` 是三元表达式，`v-for` 是 `.map()`，`v-slot` 是 `props.children`（函数）
2. **element 是不可变描述**：它只是"想要的 UI 长什么样"的快照对象，不是 DOM。更新 = 生成**新的** element 对象交给 React，React 去算 diff
3. **组件就是 `type` 字段里的函数引用**：`<Comp />` 编译成 `_jsx(Comp, {...})`--首字母必须大写，因为小写会被当成字符串 `'comp'`（DOM 标签）处理

---

### 2. 和 Vue 模板编译的对照

```html
<!-- Vue SFC -->
<template>
  <h1 class="title">Hello {{ name }}</h1>
</template>
```

编译产物本质相同：

```js
// 简化的 Vue 编译产物（setup 语法）
function render(_ctx) {
  return h('h1', { class: 'title' }, 'Hello ' + _ctx.name)
}
```

`_jsx()` 和 `h()` 干的是同一件事：**接收 (type, props, children)，返回描述 UI 的对象**。所以 JSX 和 Vue 模板殊途同归：

```
Vue:  template --编译--> render() 内 h() 调用 --执行--> VNode 树
React: JSX     --编译--> 组件体内 _jsx() 调用 --执行--> Element 树
```

关键差异在**编译器掌握了多少信息**：

| 维度 | Vue 模板 | React JSX |
|------|---------|-----------|
| 语法 | 约束的 DSL（指令/插槽语法受限） | JS 起子（任意表达式，图灵完备） |
| 编译优化 | 静态提升、patchFlag（标记哪个节点动态）、block tree--编译器知道哪里是静态的 | 基本没有：编译器不知道 `obj.x` 和 `obj.y` 谁会变，运行时全量 diff |
| 优化路线 | 把聪明放在编译时 | 把聪明放在运行时（Fiber/并发），React Compiler 才开始补编译时优化（s5 讲） |
| 手写渲染函数 | `h()`（体验差，一般不写） | JSX 本身就是"手写渲染函数"，写组件=直接写 render |

一句话：**Vue 用语法约束换编译优化，React 用语法自由换表达力、把性能问题留给运行时解决。**这也是你从 Vue 迁 React 时最容易带着走的误解--Vue 里"模板里写复杂 JS 表达式会有性能警告"，React 里不存在这个概念，JSX 里的表达式就是普通 JS。

---

### 3. 你需要改写的 Vue 习惯映射表

| Vue 写法 | React 等价 |
|---------|-----------|
| `<div :class="{active: isActive}">` | `<div className={isActive ? 'active' : ''}>`（自己拼，或 `clsx` 库） |
| `v-if` / `v-else` | `{cond ? <A/> : <B/>}` 或 `{cond && <A/>}` |
| `v-for="x in list" :key="x.id"` | `{list.map(x => <X key={x.id} />)}` |
| `{{ text }}` | `{text}` |
| `v-html="raw"` | `dangerouslySetInnerHTML={{ __html: raw }}` |
| `<Comp v-if="ok"/>`（惰性求值） | `{ok && <Comp/>}`（同语义：不渲染就不执行组件函数） |

注意属性名是 **camelCase**：`class -> className`、`for -> htmlFor`、`onclick -> onClick`（因为 props 是 JS 对象的键，DOM 保留字冲突）。

---

### 4. 结构总览：从 JSX 到屏幕（1.1 版，粗粒度）

```
JSX 源码
  │ 构建时编译（esbuild/SWC）
  ▼
_jsx(...) 调用（写在组件函数体内）
  │ 组件函数被 React 调用时执行
  ▼
ReactElement 树（{type, props} 的嵌套对象，纯描述）
  │ React 运行时（Fiber，s5 讲）拿新旧 element 树对比
  ▼
DOM 操作
```

记住分层：**JSX（语法）-> Element（数据）-> Fiber（运行时）-> DOM（结果）**。本节只需要到第二层。

## 考核过程

### 题目 1：代码阅读题（Greeting 组件，问 console 输出 + 手写编译产物）

**用户回答（第 1 轮）**：
- ① 输出："React Element, 'p', Hi Tom"，children 是数组
- ② `_jsxs('p',{className:'msg',children:[_jsx('b',{children:'!'})]})`

**点评**：
- `typeof el` 错："React Element"是身份不是 `typeof` 返回值，应为 `'object'`（细节错误）
- `children[1]` 错：把文本段和表达式段合并成 `'Hi Tom'`。实际逐段切分：`['Hi ', 'Tom', <b/>]`，`[1]` 是 `'Tom'`（细节错误，本节核心盲区）
- ② 编译产物丢失了两个文本 children，且选对 `_jsxs`（加分）
- 错误类型：细节错误。给了两个提示（typeof 的可能返回值、children 三坑位切分图）

**用户回答（第 2 轮微调）**：
- ① `object, 'p', Tom` -- 全对 ✅
- ② `_jsxs('p',{className:'msg',children:[_jsx({children:'Hi' + name}),_jsx('b',{children:'!'})]})` -- 文本段仍被包进 `_jsx` 且仍合并文本/表达式

**点评**：
- 规则重申：只有"元素"（带标签）才编译成 `_jsx()` 调用，文本段/表达式段是原始值直接进数组

**用户回答（第 3 轮微调）**：
- ② `_jsxs('p',{className:'msg',children:['Hi','Tom',_jsx('b',{children:'!'})]})` -- 通过 ✅

**点评**：
- 唯一残留：`'Hi '` 的尾随空格被 JSX 保留（同行文本与表达式之间的空格不丢弃），无害细节，记入蒸馏

### 补充对话
- 前置诊断（var/let 闭包题）：`var` 版答 2,2,2（应为 3,3,3，循环退出时 i=3），"共享绑定"方向正确；`let` 版 0,1,2 正确。已预告与 state 快照（2.1）、useEffect 闭包陷阱（2.3）的关联

---
通过时间：2026-08-17
