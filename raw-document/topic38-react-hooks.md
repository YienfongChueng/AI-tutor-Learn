# Hooks 实现——状态存哪，为什么不能写在 if 里

## 讲解原文

### 子节 B1：Hooks 链表挂载在哪

#### 第 0 段：直觉锚定——临时工的固定工位

函数组件每次渲染，函数体**从头到尾重新执行**——局部变量全部阵亡。那 `useState` 返回的 state 凭什么活过渲染？

类比：函数组件是个**每天换班的临时工**（函数体重跑），但公司给他留了**固定工位**（fiber 对象，堆上跨渲染存活），工位抽屉里有**一排文件袋**（Hooks 链表）。临时工每天上班第一件事：到工位，**按昨天登记的顺序**打开文件袋。文件袋不随临时工消失——所以 state 活了下来。

| 类比 | 对应物 |
|------|--------|
| 临时工 | 组件函数体的一次执行 |
| 固定工位 | fiber 对象 |
| 抽屉里的文件袋串 | fiber.memoizedState 指向的 Hook 链表 |
| 按顺序开袋 | renderWithHooks 按调用顺序推进链表 |

#### 第 1 段：问题背景——函数组件的"实例"在哪

1.2 学过：组件就是函数，调用它的是 React（`renderWithHooks`）。类组件的状态挂在 `this` 上——类实例是跨渲染存活的。函数组件没有 `this`，但 state 也要跨渲染存活，必须找一块**堆上、被 fiber 持有**的内存。

**Fiber 就是函数组件的"实例"**（官方原话：fiber 是 virtual state / instance）。Hook 链表挂在这份"实例"上，这就是 2.7 考过的 fiber 归属问题的正主。

> ⚠️ **误解拦截**：Hooks 数据不是存在某个全局变量里（2.7 考过"非单例"）。每个组件实例（每个 fiber）**各自一份链表**——两个 `<Counter/>` 互不串号，因为它们是两个 fiber、两个工位。

#### 第 2 段：核心数据结构——Hook 对象串成单链表

```
Hook 对象（ReactFiberHooks.js）
├── memoizedState   # 本 hook 的当前值（useState 存 state，useEffect 存 effect 对象）
├── next            # → 下一个 Hook（调用顺序串联）
└── queue           # 更新队列（B2 主角，useState 专属）

fiber.memoizedState ──指向──▶ Hook1 ──next──▶ Hook2 ──next──▶ Hook3
（组件里第 1 个调用的 hook）   （第 2 个）        （第 3 个）
```

实例（一个组件里 `useState(a)` → `useEffect(b)` → `useState(c)`）：

```
fiber.memoizedState ──指向──▶ [memoizedState: a] ──next──▶ [memoizedState: effect对象] ──next──▶ [memoizedState: c]
                               第1个调用 = useState(a)        第2个 = useEffect(b)                第3个 = useState(c)
```

**关键认知：链上节点的身份 = 调用顺序**。没有任何名字或 key 标识"这是 count 的 hook"——第 1 个被调用的就是 Hook1。这半句话是 B3 全部剧情的伏笔。

#### 第 3 段：运行流程——链表怎么被推进

概念坐标：`React@19` · `ReactFiberHooks.js` · `renderWithHooks` / `mountState` / `updateState`

```js
function renderWithHooks(fiber) {
  // mount：fiber.memoizedState = null → 创建链表
  // update：把 current fiber 的链表"接"过来作为起点
  ReactCurrentDispatcher.current = mount ? HooksDispatcherOnMount
                                         : HooksDispatcherOnUpdate
  // ↑ 关键机关：useState 在 mount/update 是两个不同实现
  const children = Component(props)   // ← 你的函数体执行
  //   执行期间每碰到一个 hook 调用：
  //   mount → createWorkInProgressHook()：新建节点接上链尾
  //   update → 把 workInProgressHook 指针拨到 next：读对应节点的 memoizedState
  return children
}
```

**"拨指针"是理解一切的核心**：函数体里每调用一个 Hook，`workInProgressHook` 沿链表前进一格；`useState` 返回的就是**当前这格**的 `memoizedState`。第 N 次调用读到第 N 格——顺序就是身份。

#### 第 4 段：设计动机与权衡

- 为什么用**调用顺序**当身份而不让每个 hook 报 key？——**约定优于配置**：顺序自动、零参数、零出错面（不存在的 key 冲突）；代价是"顶层调用"铁律（B3）
- 为什么是链表不是数组？——与 fiber 的对象图风格一致；mount 时逐个追加节点天然方便。数组 + 索引其实也能干，链表只是实现品味
- 牺牲了什么：写法自由度（不能条件调用）、lint 依赖（rules-of-hooks 靠静态分析调用顺序）

#### 第 5 段：次级误解和边界

1. **自定义 Hook 没有自己的存储**——它只是函数，展开后占用**调用者**链表上的节点。2.7 的 useCounter 两实例：两个 fiber 各自的链表，各挂各的 state
2. **同一个 Hook 不同渲染类型不同实现**——mount 用一套 dispatcher、update 用另一套（ReactCurrentDispatcher 这个"换实现"机关是源码里很漂亮的开关）

现在我们知道了 Hook 的值挂在 fiber 链表上、按调用顺序取用。但一个新问题：`setCount(3)` 调用那一刻发生了什么？它怎么找到"自己对应的那个 hook"？新值什么时候真的算出来？——这是 **B2（useState 更新队列）**。

### 子节 B2：useState 更新队列——"贴单"与"结算"

#### 第 0 段：直觉锚定——快递前台的两张单

1.3 考过快递模型：**传值 = 寄定格成品，传函数 = 寄指令单**。现在补全后台：前台有个**排队箱**（queue），不管你寄成品还是指令单，都先**贴单进箱**——setCount 这一刻**什么都还没结算**。真正拆包裹算钱（算新 state）发生在**下一次 render**，按单号顺序逐单处理。

| 类比 | 对应物 |
|------|--------|
| 贴单进箱 | setCount 生成 update 对象挂上 hook.queue |
| 排队箱 | queue.pending（环形链表） |
| 结算 | 下次 render 时遍历 queue 算新 memoizedState |
| 指令单 | 函数式更新 `setCount(c => c + 1)` 的 action |
| 定格成品 | `setCount(3)` 的字面量 action |

#### 第 1 段：问题背景——set 到底干了什么

> ⚠️ **误解拦截**：很多人以为 `setCount(3)` 把 state 变量改成了 3。**不可能**——`const [count] = useState()`，count 是 const，赋值都赋不进去。setCount 只干两件事：①生成 update 对象**入队**；②**调度一次 render**（通知 React"有单要结了"）。新值在下一次 render 的结算里才诞生。

这直接解释了 2.1 的考题：函数体里读到的 `count` 是**本轮结算完的快照**，不是"点击瞬间的最新值"。

#### 第 2 段：核心数据结构——环形链表的 queue

```
hook
├── memoizedState: 0        # 当前值（上轮结算结果）
└── queue
    └── pending ──▶ [update3] ──next──▶ [update1] ──next──▶ [update2] ──┐
                        ▲_______________________________________________|
                        （循环链表：pending 指向最后入队的节点，
                          pending.next 是最早的单——方便 O(1) 追加且知道队头）

update = { action: 3 | (prev) => prev + 1, next: ... }
```

为什么用**循环**链表？单链表尾插要遍历 O(n)；循环链表 `pending.next` 即队头、`pending` 即队尾，追加 O(1) 还能立刻拿到插入位置。

#### 第 3 段：运行流程——结算

概念坐标：`ReactFiberHooks.js` · `updateReducer`（useState 底层和 useReducer 共用一套）· `processUpdateQueue`

```js
// 下次 render，updateState 被调用（链表指针已拨到 count 这一格）
function updateReducer(hook) {
  let newState = hook.memoizedState          // 从旧值出发
  const updates = 从 queue.pending 摘出全部按序排列的 update
  for (update of updates) {
    newState = typeof update.action === 'function'
      ? update.action(newState)              // 指令单：结算时才执行，吃到"上一单的结果"
      : update.action                        // 成品单：直接覆盖
  }
  hook.memoizedState = newState              // 结算落账
  return [newState, dispatchSetState]        // dispatchSetState 就是 setCount 的真身
}
```

**考过的题源码级对号**：

- `setCount(1); setCount(2); setCount(3)` 三连 → 三单入队 → **一次 render 一次结算** → 这就是 2.1 的"批量更新合并"
- 函数体里 `alert(count)` 弹旧值 → 函数体执行于本轮结算**后**，count 是快照；下一次结算要等下一轮 render
- `setCount(c => c + 1)` 三连 → 三张指令单按序执行，每张吃上一张的输出 → 0→1→2→3
- 1.3 的"更新丢失"（`setX({...x, ...})` 连发）→ 三张**成品单**都基于**同一旧值**结算，后两张覆盖前一张——指令单才是解法

#### 第 4 段：设计动机与权衡

- 为什么入队不直接覆盖？①**合并**：一轮里 N 个 set 只渲染一次；②**延迟求值**：函数式更新能吃到最新值；③**可重放**：并发模式下（C 块）render 可能丢弃重跑，队列可以重新结算——直接覆盖就没法重放了
- 牺牲：心智负担（快照语义要学）；`Object.is` 判等的跳过渲染逻辑也藏在这套结算里

#### 第 5 段：次级误解和边界

1. **queue 在 hook 对象上，不在全局**——两个组件的 useState 各结各的账；这也是"状态隔离"的源码依据
2. **dispatch（setX 函数）身份稳定**——返回的 setCount 每轮是同一个引用，所以 2.6 的 useCallback 对 setState 的讨论里"setState 天然稳定"
3. **结算发生在 render 阶段**（可中断的那段）——所以高优先级插队作废重跑时，队列能支撑重放（C 块回收）

现在我们知道了 setX 只是贴单、结算在 render。把 B1 和 B2 合起来，那个面试必考的问题已经可以自己推了：**如果函数体里把某个 hook 写进了 if，链表会发生什么？**——先自己想十秒，再看 **B3** 的"取错柜子"故事。

### 子节 B3：为什么条件调用会崩——取错柜子

#### 第 0 段：直觉锚定——快递柜只认单号顺序

快递柜（Hooks 链表）**不看你取什么，只看你报的单号顺序**。今天你跳过 2 号柜直接说"取我的第三件"——柜子给你开的是 2 号柜，里面是别人（其实是你的另一个 hook）的东西。从这一格起，**后面每格全部错位**。

| 类比 | 对应物 |
|------|--------|
| 柜子 | Hook 链表节点 |
| 单号顺序 | hook 的调用顺序（唯一身份） |
| 跳过一格 | if 里少调一个 hook |
| 后面全错位 | 后续 hook 读到前一个的 memoizedState/queue |

#### 第 1 段：问题背景——B1 的自然推论

renderWithHooks 推进链表只看"第 N 次调用"。mount 时 3 次 hook 调用建了 3 格；update 时函数体如果因为 if 只调了 2 次——第 2 次调用会读**第 2 格**（原本属于 useEffect 的格子），第 3 格永远没人认领。**结构错位，不是值错位**：读到错的 memoizedState 还是小事，读到**错的 queue** 才是大祸——后续 setX 会把更新贴到别人的单箱上。

#### 第 2 段：核心结构——错位演示

```jsx
function Comp({ tab }) {
  const [a, setA] = useState(1)      // mount: 格1
  if (tab === 'b') {
    const [x, setX] = useState(2)    // mount(tab=b): 格2 ⚠️
  }
  useEffect(() => { /* ... */ })     // mount(tab=b): 格3 / mount(tab=a): 格2
  ...
}
// tab=b 挂载：格1=a  格2=x  格3=effect
// 切到 tab=a 更新：只调 useState(a) + useEffect →
//   第2次调用(useEffect)读格2 —— 格2 是 useState 节点！
//   effect 的注册逻辑跑到 useState 的 queue 上 → 结构性崩坏
```

#### 第 3 段：运行流程——React 怎么发现并报错

```
update 轮：renderWithHooks 沿链推进
   │ 第 N 次调用 hook
   ▼
workInProgressHook = workInProgressHook.next   ← 没格子可拨（next === null）
   ▼
if (workInProgressHook === null) → throw "Rendered more hooks than during the previous render"
// 反向：本轮调用数 < mount 时 → "Rendered fewer hooks than expected"
```

坐标：`ReactFiberHooks.js` · mountWorkInProgressHook / updateWorkInProgressHook 的判空 throw。**这是运行时兜底**——很多错位场景（不触发判空的）直接静默错乱，所以 ESLint rules-of-hooks 才要在编译期就拦（4.10 的 use 前缀纪律在这里闭环：**静态分析靠的正是"hook 调用必须能在顶层枚举"这个约定**）。

#### 第 4 段：设计动机与权衡

- 顺序身份的代价就是这条铁律。备选方案（每个 hook 声明 key）会让 API 变啰嗦、还可能 key 冲突——React 选了"纪律换简洁"
- 这也是为什么**自定义 Hook 也必须在顶层调用**：它展开成调用者的节点，条件化自定义 hook = 条件化一批节点

#### 第 5 段：次级误解和边界

1. **"写在 if 里但条件不变就不崩"** ——能跑，但**没有保护**：条件一变就是定时炸弹，lint 拦的就是这个风险
2. **早退 return 后不能再有 hook**：`if (loading) return <Spin/>` 之后写 useState，等于条件调用同款
3. **2.7 那道题的完整答案**：`useCounter()` 在两个组件里各调一次 → 两个 fiber、两条独立链表、两个互不干扰的 state——"非单例"的根源是**存储在 fiber 上而非全局**

---

**主题块 B 收束**：Hooks 的全部行为都由三句话推出——**状态在 fiber 链表上（B1）、setX 只是贴单结算在 render（B2）、调用顺序是唯一身份（B3）**。加上 A 块的 renderWithHooks，"为什么 Hooks 不能写在 if 里"从铁律变成了自然推论。

## 考核过程

### 题目 1（源码追踪题）：setName('Tom') 后第二轮 render 的取值过程

**用户回答**：第一格和第二格，fiber.memoizedState 上挂载的 hook 按顺序调用算出来的；setAge 那个格子没有单子。

**点评**：接近正确（close）。格子定位 ✅、setAge 格 queue 空 ✅；缺"经过什么过程"——即 name 格的**结算环节**。提示后补答："生成 update 对象挂在 hook.queue 上，下次 render 时遍历 queue 算新的 memoizedState，按字面量 action 执行"——机制全对（贴单→入队→updateReducer 遍历结算→字面量 action 直接覆盖），仅口误把 setName 说成 setAge。通过。

### 题目 2（机制推理题）：if 调 hook 且条件恰好都成立、未触发 throw

**用户回答**：这次条件恰好成立能跑，但可能有不成立的时候——错位取 hook，之后的 hook 全取错，还 setX 的话数据全错位。

**点评**：通过（一次过）。命中要害"风险在未来条件变化时引爆"+ setX 贴错单箱的后果。

### 题目 3（设计理解题）：为什么放弃 hook 声明 key 的方案

**用户回答**：key 可能重复出错；API 会变啰嗦。

**点评**：通过（一次过）。补加：顺序法可在编译期静态枚举调用，rules-of-hooks 才能静态拦截——4.10 use 前缀纪律的闭环。

---
通过时间：2026-11-24
