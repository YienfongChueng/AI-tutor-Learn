# 2.3 useEffect 闭包陷阱与 cleanup

## 讲解原文

> 承接 2.2：你已经在确认题里亲手推过 `cleanup: 0`（闭包停在第一次渲染）和 `ignore` 旗标（旧闭包发现自己过气）。本节把"过期闭包"系统化：它为什么会发生、四种典型症状、每种的正解。

### 一、过期闭包：一句话定义

**每个 effect 闭包捕获的是"它那一轮渲染"的 state/props 快照。只要这个闭包活得的比那轮渲染久，它读到的就是旧值。**

```
渲染 #1（count=0）──> effect闭包A 里面 count === 0，永远是 0
渲染 #2（count=1）──> effect闭包B 里面 count === 1，永远是 1
渲染 #3（count=2）──> effect闭包C 里面 count === 2，永远是 2
```

过期闭包不是 React 的 bug，是 2.1 快照语义的直接推论。它只在**异步/持久性副作用**里发作：定时器、事件监听、fetch 回调--这些代码"出生"在一轮渲染里，却在很久之后才执行。

### 二、症状 1：定时器数到 1 就停

```tsx
function Counter() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const t = setInterval(() => {
      setCount(count + 1)   // 这里的 count 是哪一轮的？
    }, 1000)
    return () => clearInterval(t)
  }, [])

  return <span>{count}</span>
}
```

现象：0 -> 1 -> 1 -> 1 ...卡死。

推因：deps `[]`，effect 只在挂载时跑过一次，**闭包 A 是渲染 #1 创建的，里面的 count 永远是 0**。所以每秒执行的都是 `setCount(0 + 1)`，也就是永远设成 1。

**两种修法，代价完全不同：**

```tsx
// 修法①：函数式更新（推荐）
useEffect(() => {
  const t = setInterval(() => {
    setCount(c => c + 1)   // 不读闭包，把"加一"作为指令递给最新状态
  }, 1000)
  return () => clearInterval(t)
}, [])                       // 定时器只开一次，零重开开销

// 修法②：把 count 放进 deps
useEffect(() => {
  const t = setInterval(() => setCount(count + 1), 1000)
  return () => clearInterval(t)
}, [count])                  // 每秒：清旧定时器 -> 开新定时器（闭包拿到新 count）
```

修法②能跑，但每秒都在"销毁再重建"定时器；修法①让闭包根本不需要读 count。**心法：能让闭包"不读旧值"的，就不要让它"读新值"。** 函数式更新、useRef（2.4）都是这个思路。

### 三、症状 2：事件监听器拿到旧 props

```tsx
function Search({ keyword }) {
  useEffect(() => {
    const handler = (e) => {
      console.log('当前关键词:', keyword)  // 挂载时的旧值
    }
    window.addEventListener('keydown', handler)
    return () => window.removeEventListener('keydown', handler)
  }, [])  // ❌ 漏写 keyword
  return <input ... />
}
```

监听器是挂载时注册的"长寿闭包"，`keyword` 停在第一轮。修法：`[keyword]` 进 deps（每次变化重挂监听器，cleanup 保证不叠加）。

### 四、症状 3：fetch 回调覆盖新数据（2.2 已拆）

`ignore` 旗标就是过期闭包的**反向利用**：与其修复旧闭包，不如给旧闭包发"死亡通知书"。同族方案还有 `AbortController`（直接取消请求，更彻底）。选择标准：能取消就取消（AbortController），取消不了就作废（ignore 旗标）。

### 五、症状 4：定时器/轮询里 setState 导致的无限循环（反向症状）

```tsx
useEffect(() => {
  setCount(count + 1)      // 改了 count
}, [count])                 // count 是自己的依赖
```

count 变 -> effect 重跑 -> 又改 count -> 无限循环。这是**依赖闭环**：副作用修改了自己的依赖。解法：审视这个更新到底该不该放 effect 里（大多数时候该用事件处理函数或函数式更新）。

### 六、诊断流程（背下来）

遇到 effect 行为诡异，按顺序自问三句：

1. **这个闭包是哪轮渲染创建的？** 它读的每个值都是那轮的快照
2. **它执行的时候，那个值变了吗？** 变了 -> 过期闭包实锤
3. **我要的是"更新时重跑"（加 deps）、"不读旧值"（函数式更新/useRef），还是"旧闭包作废"（cleanup + 旗标）？**

### 七、Vue 对照：为什么 Vue 里没这病

`watch(count, cb)` 的回调执行时，读 `count.value` 走的是 Proxy getter，**永远拿到最新值**--响应式系统是"活引用"。React 的闭包是"死快照"。所以：

| | Vue | React |
|---|---|---|
| 回调里读状态 | `.value` 永远最新 | 闭包快照，可能过期 |
| 依赖 | 自动收集 | 手动声明 deps |
| 应对 | 几乎不用想 | 心里常驻"这闭包几岁了" |

### 八、本节心智模型

1. **effect 闭包几岁，它读到的值就停在几岁**--定时器/监听器/fetch 回调都是"长寿闭包"高危区
2. **优先让闭包不读旧值**（函数式更新），其次"重跑拿新值"（加 deps，有重建开销）
3. **cleanup + 旗标 = 给旧闭包发死亡通知书**；能取消就用 AbortController
4. **副作用修改自己的依赖 = 无限循环**，先怀疑设计而不是语法
5. Vue 活引用无此病，React 死快照是常态--迁移期最危险

## 考核过程

### 题目 1：Greeter 三问（deps [] 的全局监听器）

**用户回答**：
1. 触发 0 次渲染；alert 显示"你好，张三"，因为读的是第一轮渲染的初始值
2. 修法：deps 加入 name，name 变化时重新执行 useEffect
3. cleanup 解决每轮 effect 前先清除上一轮监听；删掉 return 会每轮都开新监听，多次触发 online 导致内存溢出

**点评**：未通过（细节错误为主）。
- 第 1 问：alert 内容✅、闭包归因✅，但"0 次渲染"❌--受控输入每键 setName 都触发重渲染，混淆了"重渲染"与"effect 重跑"（deps [] 挡的是后者）
- 第 2 问：deps 修法✅，但 useRef 活引用修法未答出
- 第 3 问：方向对但套错代码--本题 deps [] effect 只跑一次，不存在堆积；"内存溢出"结论不准确

### 补充对话

- 指出三处问题后用户请求直接给答案（逃生舱，同 1.6 模式）
- 给出 ref 盒子完整代码 + deps/ref 对比表（行为依赖值用 deps；仅用最新值用 ref）+ 情形 A（deps [] 无 cleanup -> 僵尸 handler）/ 情形 B（deps [name] 无 cleanup -> 监听器堆积、一次 online 弹 N 个各含旧值的 alert）的真实后果
- cleanup 本质：本轮 effect 的遗嘱

### 题目 2：10 秒确认题（场景选 deps 还是 ref）

**用户回答**：① query 变化重新发请求 -> deps；② WebSocket 监听要读最新用户名 -> ref

**点评**：✅ 全对，通过。

---
通过时间：2026-08-27（逃生舱模式：一轮未过 + 补答确认通过）
