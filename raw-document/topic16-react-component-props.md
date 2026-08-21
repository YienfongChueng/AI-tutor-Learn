# 函数组件与 Props（vs SFC）

## 讲解原文

### 一句话锚定

**React 组件就是一个函数：接收 props 对象，返回 element 树。调用它的人不是你，是 React。**

这句话里藏着你在 Vue 里从未遇到过的执行模型，也是后面所有 hooks 行为的根源。

---

### 1. 组件即函数：执行模型的关键差异

```jsx
function Badge({ color, children }) {
  return <span className={color}>{children}</span>
}
// <Badge color="red">Hi</Badge>
// 编译为：_jsx(Badge, { color: 'red', children: 'Hi' })
//                              ↑ Badge 作为 type 传进去，此刻还没执行
```

React 拿到 `type: Badge` 后，在渲染时**调用它**：`Badge(props)`，拿到返回的 element 树，再递归处理。

对照 Vue，注意执行次数的差异（这是本节最重要的一张表）：

| | Vue SFC (setup 语法) | React 函数组件 |
|---|---|---|
| `setup()` / 组件函数体 | **只执行 1 次**（挂载时） | **每次渲染都重新执行** |
| 模板 render / JSX | 每次更新执行 | 同上（就是函数体本身） |
| `let count = 0` 定义在 setup 里 | 闭包住整个生命周期，一直活着 | 每次渲染重新创建、值重置 |
| 状态存哪 | setup 的闭包 + 响应式代理 | 不存在于函数里，由 React 在外部代管（hooks，1.3） |

推论（现在只需要直觉，1.3/2.1 兑现）：

- **React 函数组件里，函数体每次渲染都从零跑一遍**。里面定义的变量、内部函数，每轮渲染都是**新的一份**--这正是"state 是快照"的来源
- **hooks 必须在顶层无条件调用**（函数体每次都执行，调用量必须稳定，React 靠调用顺序对号入座--s5 讲链表时兑现）
- Vue 里"`setup 执行一次` + `模板依赖收集`"的组合，被 React 换成了"`反复执行整个函数` + `diff 新旧两棵树`"的暴力方案

---

### 2. Props：只读快照，单向数据流

和 Vue 概念一致的部分不重复（单向、父传子、不可子改父）。差异在**形式和细节**：

```jsx
// 解构接收（主流写法），默认值直接给
function Avatar({ src, size = 48, ...rest }) {
  return <img src={src} width={size} {...rest} />
}
```

- **解构 vs `props.x`**：React 没有 Vue 的 `defineProps` 声明层，函数签名就是 props 的"声明"
- **默认值**：解构默认值（`size = 48`）取代 Vue 的 `default:`
- **透传**：`{...rest}` 取代 Vue 的 `$attrs` 继承（Vue 的非 props 属性自动落到根元素，React 默认**什么都不做**，要透传得自己写）
- **props 是快照**：每轮渲染拿到的是当次的新对象。父组件重渲染 -> 子组件拿到新 props 对象重新执行（不管值变没变，2.6 优化话题）

三个**不进 props** 的特殊 prop：

| prop | 去向 | 用途 |
|------|------|------|
| `key` | 被 createElement 摘走 | diff 时的身份标识（1.4 详讲） |
| `ref` | 被摘走 | DOM/实例引用（2.4 详讲） |
| `children` | **在 props 里**，但位置特殊 | 内容分发 |

---

### 3. children = 默认插槽；具名插槽 = 传函数

Vue 的插槽体系在 React 里的对应物非常朴素：

```jsx
// 默认插槽
<Card><p>内容</p></Card>
// Card 内部：function Card({ children }) { return <div>{children}</div> }

// 具名插槽：传 JSX 当属性（本质：props 的值是 element）
<Layout header={<h1>标题</h1>} body={<Table/>} />
// 对应 Vue: <template #header> / <template #body>

// 作用域插槽：传函数当属性（"render props"模式）
<Foo render={(data) => <Item {...data}/>} />
// 对应 Vue: <template #default="{ data }">
```

结构上完全同构：**插槽就是"以 prop 为载体传 JSX/函数"**，Vue 只是用 `<template #>` 给它做了语法糖。

---

### 4. 纯函数约定：渲染期间不许干别的

React 要求组件函数是**近似纯函数**：同样的 props -> 同样的 JSX，渲染期间不修改任何外部状态、不发请求。为什么？因为**函数体会被反复调用**（父组件渲染、自身 state 变化都会触发），副作用会以不可预期的次数执行。请求、订阅、定时器去 `useEffect`（2.2），变更状态去事件回调（1.3）。

对照：Vue 的 setup 只跑一次，你把请求写进 setup 顶层"碰巧"没出过事（其实 Vue 也建议放 onMounted）；React 里这个习惯会直接出 bug--**组件渲染几次，请求就发几次**。

## 考核过程

### 题目 1：情景分析（UserList 顶层 fetch + `let count = 0` 为何不能自增）

**用户回答（第 1 轮）**：
- ① fetch 会重复请求，"因为 React 的 state 是快照，父组件重新渲染会从头到尾重新执行，包括没变更的子组件"
- ② state 存在 React 外部，不能直接修改

**点评**：
- ① 根因（函数体反复执行）碰到了，但把"state 是快照"错当原因--快照是反复执行的**结果**；漏答连锁反应（提示：fetch 成功后 setState 会怎样）
- ② "存在 React 外部由 React 代管"正确；但漏了两个失败原因：A. `let count = 0` 每轮渲染重新初始化；B. 局部变量变更不会通知 React
- 错误类型：接近正确，给提示后微调

**用户回答（第 2 轮微调）**：
- ① "会触发重新渲染，一直循环触发重新渲染"（fetch -> setState -> rerender -> fetch... 死循环）✅
- ② "重新定义 count=0"（原因 A）✅；"count++ 改变了局部变量值，但由父组件决定重新渲染，没有（通知到）"

**点评**：
- 原因 A/B 机制都到位。归因小错：**触发重渲染的还有组件自身的 state 变化（setter）**，不只父组件--这正是 useState 的意义（1.3）
- 通过 ✅

---
通过时间：2026-08-17
