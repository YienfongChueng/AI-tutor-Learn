# State 与事件处理（useState 基础）

## 讲解原文

### 一句话锚定

**Vue 是"你改数据，它自己发现"；React 是"你喊一嗓子（setter），它替你改并重跑函数"。** useState 就是"把状态托管到 React 外部 + 拿到喊嗓子的喇叭"这对组合。

---

### 1. useState 基本形态

```jsx
import { useState } from 'react'

function Counter() {
  const [count, setCount] = useState(0)   // [当前值, 触发器] = useState(初始值)
  return <button onClick={() => setCount(count + 1)}>{count}</button>
}
```

执行模型（接 1.2）：

```
点击按钮
  └─ setCount(1)
       └─ React 在外部仓库里把这个组件槽位的 state 更新为 1
            └─ 安排一次重渲染（异步合批，2.1 详讲）
                 └─ 重新调用 Counter()
                      └─ 这次执行的 useState(0) 不再取初始值，
                         而是从仓库取回 1 -> count = 1
                           └─ 新 JSX -> diff -> 更新 DOM
```

注意一个细节：**`useState(0)` 的参数只在首次渲染生效**。之后每次渲染这个调用变成"按调用顺序取回自己的那份 state"（为什么靠顺序、hooks 链表，s5）。

---

### 2. 快照语义：你调用的 set 用的是"旧照片"

还记得前置诊断那道 `var/let` 闭包题吗？正式兑现：

```jsx
function handleClick() {
  setCount(count + 1)   // count 是本轮渲染的快照，此刻是 0
  setCount(count + 1)   // 还是 0！不是"还没执行"而是"读的还是旧照片"
}
// count=0 时点击 -> 界面显示 1，不是 2
```

两次 `setCount(count + 1)` 都基于**同一轮渲染的 count（0）**计算，React 各自执行 `setCount(0 + 1)`。这就像诊断题里 `let` 的三次迭代：**每轮渲染的 `count` 是独立绑定，函数体里的代码读到的永远是本轮那份**。

修复：**函数式更新**，让 React 把"最新值"递给你：

```jsx
setCount(c => c + 1)   // 接收最新 state，返回新 state
setCount(c => c + 1)   // 基于上一步的结果，最终 0 -> 2 ✅
```

> ⚠️ **Vue 人的条件反射陷阱**：Vue 里 `count++` 连写两次天经地义（响应式代理同步更新）；React 里 `count` 只是个局部 const，**改它没用，读它读到旧值**。一切更新必须经过 setter。

---

### 3. 事件处理对照

| Vue | React |
|-----|-------|
| `@click="handle"` / `@click="add(n)"` | `onClick={handle}` / `onClick={() => add(n)}` |
| `@click.prevent` 修饰符 | 没有修饰符系统，函数体第一行 `e.preventDefault()` |
| `@input` 直接拿 `$event.target.value` | `e.target.value`，事件对象是合成事件 SyntheticEvent（池化已废弃，当普通对象用） |
| methods 定义一次 | 事件函数**每轮渲染重新创建**（2.6 优化话题） |
| 传参写 `@click="add(n)"` 字符串 | 必须传函数引用：`onClick={add}` 是注册，`onClick={add(n)}` 是**立即调用**（经典 bug） |

```jsx
// 经典错误：渲染时立即执行了 add，并把返回值 undefined 当事件处理器注册
<button onClick={add(n)}>+</button>
// 正确
<button onClick={() => add(n)}>+</button>
```

---

### 4. 对象/数组 state：不可变更新（Vue 人最痛的一刀）

React **没有响应式代理**（对照 Vue3 主题块 1：Proxy 拦截 get/set）。React 判断"变没变"的唯一依据是 **`Object.is`（引用比较）**。推论：

```jsx
const [list, setList] = useState([1, 2, 3])

function addBad(n) {
  list.push(n)          // 原地修改：引用没变
  setList(list)         // Object.is(旧引用, 新引用) === true -> React 认为没变，跳过渲染 ❌
}

function addGood(n) {
  setList([...list, n]) // 新数组、新引用 -> React 检测到变化 ✅
}
```

对象同理：`setUser({...user, age: 18})`。心法一句话：**永远创建新引用，而不是修改旧引用**。代价是写法啰嗦 + 大对象深拷贝浪费，React 用"浅比较换实现简单"；Vue 用 Proxy 精确追踪换运行时开销（你在主题块 1 读过的那套机制，正好是 React 没有的东西）。

嵌套结构的套路：

```jsx
setForm(prev => ({ ...prev, address: { ...prev.address, city: 'SH' } }))  // 改深层
setList(prev => prev.map(item => item.id === id ? { ...item, done: true } : item))  // 改某项
setList(prev => prev.filter(item => item.id !== id))  // 删某项
```

---

### 5. 本节心智图

```
state 快照模型（1.2 执行模型 + 本节 setter）
├─ 读：本轮渲染的绑定，永远是"当轮照片"
├─ 写：只能 setXxx(新值) 或 setXxx(前值 => 新值)
├─ 更新判定：Object.is 引用比较（无代理）
└─ 多次 set：同轮快照算多遍 -> 函数式更新兜底
```

## 考核过程

### 题目 1：TodoList 调试题（A/B/C + 一处隐藏 bug）

**用户回答（第 1 轮）**：
- 隐藏 bug：`onClick={handleAdd()}` 渲染时立即执行，返回 undefined 给 onClick ✅（一次答对）
- A：push 原地修改，引用不变，`Object.is(旧,新)=true` 不触发渲染 ✅（机制对）
- B：改深层属性，todos 引用仍不变，不触发渲染 ✅（机制对）
- C："快速连点会一直往队列里添加执行函数"（未答到点）

**点评**：A/B 机制全对；C 传的不是函数是立刻求值的"寄成品"，连续删除时第二次 filter 基于旧照片，覆盖第一次结果（**更新丢失**）。用快递模型演示了两次删除的覆盖过程。隐藏 bug 一次通过。

**修法（第 2 轮）**：
- A：`todos = (...todos, {...})` -- 意图对但代码错：`(...)` 非法表达式、给 const 赋值（必须走 setter）
- B：`todo.done = !todo.done; todos = {...todos, todo}` -- 数组 spread 进对象变伪数组；且仍在原地改旧对象
- C：原样抄写未修

**点评**：已示范 A（`setTodos([...todos, item])`）、B（`setTodos(todos.map(...))`，"换人不是整容"）；C 留空让用户补函数式更新。

**修法（第 3 轮）**：
- C：`setTodos(todos => todos.filter(t => t.id !== id))` ✅

### 补充对话
- 用户主动提问"为什么 setCount(count+1) 不行而 setCount(c=>c+1) 可以"（考核前）：补讲**快递模型**--传值=寄出瞬间定格的成品，传函数=寄指令单，React 处理队列时才执行且把流水线最新值作参数。这个模型随后被用于解释 C 的更新丢失，形成"讲->问->用"闭环
- 顺带纠正 `setCount(count++)`：count 是 const，`++` 直接 TypeError

---
通过时间：2026-08-17
