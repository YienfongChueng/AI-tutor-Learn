# 调度与时间切片——shouldYield 背后的真身

## 讲解原文

### 子节 C1：Scheduler 怎么让出主线程

#### 第 0 段：直觉锚定——咖啡店的节奏

咖啡师（主线程）做单（渲染任务），手边有个**订单架**（Scheduler 优先级队列）。规矩：**每做完一杯，看一眼表**——这杯超时了吗？门口有加急单吗？超了就先把手上这杯递出去，去处理架子上最急的那单。**"每做完一杯看一眼表"** 就是 `shouldYield()`；**订单架按加急程度自动排最上面**就是优先级队列（堆）。

| 类比 | 对应物 |
|------|--------|
| 咖啡师 | 主线程 |
| 做一杯咖啡 | 处理一个 fiber（performUnitOfWork） |
| 看一眼表 | shouldYieldToHost() |
| 订单架 | Scheduler 的任务堆（小顶堆） |
| 新一轮"开始做单" | MessageChannel 宏任务回调 |

#### 第 1 段：问题背景——A1 留下的悬念

A1 说 workLoop 每轮问 `shouldYield()`——**判断依据是什么？** 答案藏在浏览器的帧节奏里：一帧约 16.7ms（60fps），浏览器要在帧内完成 输入事件 → JS → 样式计算 → 布局 → 绘制。React 的策略：**每次只连续干约 5ms 就把主线程让回去**，剩下的活拆成下一个宏任务继续。

> ⚠️ **误解拦截**：很多人以为 Scheduler 用 `requestAnimationFrame` 或 `setTimeout` 排下一片。都不是，用 **MessageChannel**（下面第 4 段说为什么）。

#### 第 2 段：核心数据结构——两个小顶堆

```
Scheduler（Scheduler.js）
├── taskQueue    # 小顶堆：立刻要跑的任务，堆顶 = 最紧急
│                 （按 expirationTime 排，越紧急越先出堆）
├── timerQueue   # 小顶堆：延迟任务（setTimeout 式），到点后搬进 taskQueue
└── 优先级 6 档（PriorityLevel）
    Immediate > UserBlocking > Normal > Low > Idle
    （每档对应不同的 delay/timeout，算出 expirationTime）
```

堆 = "订单架自动把加急单顶到最上面"——`heap.pop()` 永远拿走最紧急的，O(log n)。

#### 第 3 段：运行流程——一条宏任务接一条宏任务

概念坐标：`React@19` · `scheduler/src/Scheduler.js` · `scheduleCallback` / `performWorkUntilDeadline` / `flushWork`

```js
function scheduleCallback(priorityLevel, callback) {
  const task = { priorityLevel, expirationTime, callback }
  入堆（taskQueue 或 timerQueue）
  if (!正在执行循环) {
    isHostCallbackScheduled = true
    requestHostCallback()      // → port.postMessage(null) ★ 启动引擎
  }
}

// MessageChannel 收到消息（一个宏任务）：
function performWorkUntilDeadline() {
  flushWork()                  // 取堆顶执行
  // flushWork 内部 → workLoop（A1 的那个）→ 循环里 shouldYield() 到点就 break
  if (还有任务没跑完) {
    port.postMessage(null)     // ★ 再发一条消息 = 排下一个宏任务
    // 浏览器在两个宏任务之间：处理输入、样式、布局、绘制——主线程让出来了
  }
}
```

**关键认知：时间切片不是"暂停 JS"，是"把长任务切成 N 个宏任务"**——每个宏任务之间浏览器获得完整的喘息窗口。

#### 第 4 段：设计动机与权衡——为什么是 MessageChannel

| 候选 | 为什么不用 |
|------|-----------|
| `setTimeout(fn, 0)` | 最小 4ms 延迟真实存在；嵌套超 5 层会被浏览器 clamp 到 4ms+，切片节奏失控 |
| `requestAnimationFrame` | rAF 在**绘制前**触发——用它续任务，浏览器永远没机会先绘制，等于没让 |
| `MessageChannel` | 宏任务，零延迟排队；执行点在当前微任务之后、浏览器可自由选择绘制/处理输入的间隙——**切片节奏干净** |

代价：比微任务"慢半拍"（微任务跑完才轮到），但对渲染调度无所谓——反正目的就是让出主线程。

#### 第 5 段：次级误解和边界

1. **"浏览器空闲时通知 React"** ——反了。不是浏览器喊 React 干活，是 **React 自己切**（5ms 一片主动 postMessage 续单）；浏览器只负责在宏任务间隙处理自己的事
2. **Immediate 优先级不分片**——最急的事（离散输入如点击）走同步通道一口气跑完（lane 模型里对应 SyncLane，C2）
3. **时间片 ≈ 5ms 不是 16.7ms**——给浏览器留出绘制余量；且"帧内有绘制需求"时片更短

现在我们知道了"什么时候让、怎么续"。但"谁先谁后"的裁决还没讲：为什么点击比列表更新先渲染？`useTransition` 凭什么让更新"可以被输入打断"？——这是 **C2（Lane 优先级模型）**。

### 子节 C2：Lane 优先级模型——一排灯的开关面板

#### 第 0 段：直觉锚定——一排灯的开关面板

想象一块**灯控面板**，一排灯从左到右代表从"最急"到"最闲"：红色（同步点击）、橙色（连续输入）、黄色（默认更新）、绿色（transition）、灰色（空闲任务）。**"现在有哪些活要干"= 哪些灯亮着**。看一眼整排灯就知道全部状态（不用逐个查）；要加一个急活 = 开一盏灯；干完一档 = 一整排一起关。

| 类比 | 对应物 |
|------|--------|
| 一盏灯 | 一个 lane（一个二进制位） |
| 整排灯的开关状态 | lanes（位掩码，一个 31 位整数） |
| 开灯 | 按位或 `|=` 合并新优先级 |
| 关一排 | 按位与清除 |
| 看一眼面板 | 一次位比较判断"要不要处理这棵子树" |

#### 第 1 段：问题背景——为什么不用"数字优先级"排个序就完事

React 需要的不是"给任务排序"，而是**一次表达一组混合紧急度的活**：同一轮里可能同时有"输入要立即响应 + 列表更新慢慢来"。数字模型（一个数）只能表达单一优先级；lane 是**集合语义**——一个 31 位整数同时亮着几盏灯。还支持子优先级（一次更新内部分片：急的部分先渲染，慢的挂着）——旧 expirationTime 模型做不到，这正是 React 重写优先级系统的原因（2019 年，expirationTime → lanes）。

#### 第 2 段：核心结构

```
常用 lane（位从高位到低位 = 优先级从高到低，共 31 位）
├── SyncLane           # 同步：离散输入（click），不分片，一口气
├── InputContinuousLane # 连续输入（typing/scroll/over），可分片
├── DefaultLane        # 普通更新（setState 默认档）
├── TransitionLane     # useTransition 的更新档
└── IdleLane           # 空闲档（离屏渲染等）

fiber.lanes        # 这个 fiber 自己有更新（哪些灯亮）
fiber.childLanes   # 子树里有更新（冒泡汇总——"楼下有亮灯的"）
root.pendingLanes  # 全树的待办汇总（调度器就看它）
```

#### 第 3 段：运行流程——从 setState 到"这棵子树跳不跳过"

概念坐标：`React@19` · `ReactFiberLanes.js`（lane 常量）/ `ReactFiberWorkLoop.js` · `markRootUpdated` / `getNextLanes`

```js
// setState 时刻
const lane = 按更新类型选一盏灯           // 默认 setState = DefaultLane
markRootUpdated(root, lane)
//   1. root.pendingLanes |= lane          ← 面板上开灯（位运算 O(1)）
//   2. 沿 fiber 向上冒泡，沿途 fiber.childLanes |= lane
//      ↑ "楼下有活"的路标——让父层不用下潜就知道要往下走

// render 时（beginWork 内）
if ((fiber.lanes | fiber.childLanes) & renderLanes === 0) {
  return 复用现成的 children，bailout        ← 这棵子树灯全灭，跳过 diff！
}
// renderLanes = 本轮要处理的灯 = root.pendingLanes 的某个子集
```

**高优先级插队（A 块"重写"的触发器在这里）**：

```js
// render 进行中（wip 算了一半），用户点击 → SyncLane 入队
getNextLanes() 发现新灯优先级高于当前正在 render 的灯
   → 当前 render 作废：wip 丢弃
   → 用新 lane 从 root 重新 render
   → 高优先级跑完后，被作废的更新重新入队重跑（B2 的"队列可重放"）
```

#### 第 4 段：设计动机与权衡

- **位运算 O(1)**：合并 `|=`、比较 `&`、清除 `&= ~`，全部一条指令——调度器每帧要做海量这类判断
- **集合语义**：一个整数表达"待办的完整混合"，批量判断
- **代价**：31 位上限（React 团队在 31 种 lane 里精打细算）；位运算可读性差（源码里满屏 `|` `&`）

#### 第 5 段：次级误解和边界

1. **"lane 越大优先级越高"** ——反了，**位越靠左（数值的低位）越急**；SyncLane 是个很小的数
2. **bailout ≠ 永不更新**——灯灭只是本轮跳过 diff；灯亮时照常处理。4.8 搜索时大量 fiber 灯灭不参与 diff，这是"输入流畅"的一环
3. **lane 不直接等于 Scheduler 的 6 档**——ReactFiber 的 lane 和 Scheduler 的 priorityLevel 是两套体系，中间有映射（面试别混为一谈，说"映射"即可）

现在我们知道了优先级的裁决机制。最后一块拼图：把 lane、可中断 render、可重放队列**组装成用户看得见的特性**——为什么 React 18 后 `setTimeout` 里的 set 也自动合并了？`useTransition` 到底改了哪个灯？——这是 **C3**。

### 子节 C3：批量更新与并发特性——三件套的组装

#### 第 0 段：直觉锚定——电梯

**批量更新 = 电梯**：大家陆续按按钮（多次 setState），电梯**关门才发车**（一次 render 结算）——没人按一次就开一次门。React 17 前：只有"React 管理的场合"（合成事件）才等关门；setTimeout 里每次 set 都单独发车。React 18 起：**全宇宙统一等关门**（自动批处理）。

**useTransition = 把你的更新贴上"绿色标签"**（TransitionLane）：绿色单可以随时被红色单（点击）插队——**宁可列表晚到，不让输入卡顿**。

#### 第 1 段：问题背景

2.1 考过"多次 set 合并"。旧版合并的实现是**包装层开关**（React 事件处理函数包一层 isBatchingEventUpdates）——所以 setTimeout 里不合并。React 18 的自动批处理换了个**架构级位置**：合并不再靠事件包装，而是**所有 setState 都进同一个调度器**（C1 的队列）——同一轮宏任务里贴的所有单，下次 render 一次结算。合并从"事件层的约定"变成了"调度层的必然"。

> ⚠️ **误解拦截**：自动批处理 ≠ "state 同步变了"。贴单-结算模型没变（B2），变的只是"什么时候发车"——18 把发车时机统一成"本轮宏任务结束后"。语义上比旧版**更**一致，不是更玄。

#### 第 2 段：核心结构——useTransition 改了什么

```
startTransition(fn) 的全部魔法：
  把 fn 里触发的 setState 打上 TransitionLane（绿色灯）标
  其余一切（贴单、结算、commit）原封不动

const [isPending, startTransition] = useTransition()
  isPending：绿色单在结算中（render 已开始未 commit）的标志 → 显示加载态
useDeferredValue(value)
  派生值延迟版：旧值立即渲染，新值走绿色通道
```

**一行面试答案**：`useTransition` 不改变任何更新逻辑，只**降一档灯**（DefaultLane → TransitionLane）——让输入的 `InputContinuousLane` 可以插队打断它。

#### 第 3 段：运行流程——打断与重放的三方联演

```
用户在搜索框打字（受控组件），同时页面有个 useTransition 的大列表更新：

1. 打字 → InputContinuousLane（橙灯）+ Transition 更新（绿灯）同时在列
2. render 开始（绿灯的列表算到一半）
3. 又一个键按下 → 橙灯插队 → getNextLanes 判定优先级更高
      → 当前 render 作废重跑（A 块：重写）
      → 输入的 render 先跑 → 输入框立刻更新（不卡字！）
4. 输入跑完 → 绿灯更新重新入队重跑 → B2 的队列支撑重放
5. 整个过程 isPending = true → 你可以显示旧列表 + 小加载态
```

这就是 React 18/19 并发的全部真相：**A 的可中断 render + B 的可重放队列 + C 的 lane 优先级**——三个主题块拼成一台机器。useActionState / useOptimistic（19 新增）都是这台机器上新的操作面板。

#### 第 4 段：设计动机与权衡

- 并发特性不是独立功能，是**架构红利**：没有 Fiber（可中断）这份数据结构、没有队列重放（B2），lane 只是个数字游戏
- 代价：Transition 里的更新可能被丢弃重跑 → **trans 函数里必须是可重放的**（纯计算 + setState；不能在里面发不可撤销的副作用——发请求请放外面或用 useActionState）

#### 第 5 段：次级误解和边界

1. **"useTransition 能让更新不卡"** ——精确说法：它让更新**可被插队**。如果 CPU 慢到连输入 render 都超预算，谁也救不了（那时要靠虚拟列表/优化，4.11 的地界）
2. **异步批处理的边界**：`await` 之后的 setState 在 18+ 依然合并（同一宏任务内）；但跨 await 的两段不会合成一车——发车时机是宏任务边界
3. **startTransition 包 async**：trans 函数应保持同步；async 里的更新不会带绿标（除非 useActionState 这类封装）

---

**主题块 C 收束 + s5 全境收束**：

```
C1：切片 = 宏任务接力（MessageChannel），shouldYield 每片看表
C2：lane = 灯控面板（位掩码集合），bailout 是"灯灭跳过"，插队 = 亮更急的灯
C3：批量 = 统一发车；并发特性 = 三件套组装（可中断 + 可重放 + 优先级）
```

Fiber（A）给了可暂停的骨架，Hooks 队列（B）给了可重放的账本，Scheduler + lane（C）给了谁先谁后的交通规则——**React 18/19 的"并发"没有魔法，全是这三块的组装**。

## 考核过程

### 题目 1（源码追踪题）：输入打断 Transition 的完整流程

### 题目 2（机制推理题）：去掉 shouldYield 后哪些失效

### 题目 3（设计理解题）：MessageChannel vs setTimeout/rAF

**用户回答**：三题整体请求逃生舱直接给答案。

**点评**：按逃生舱处理——给出三题标准答案并逐段讲透，记为"参考通过，复习重点"。T1 关键链条（橙灯插队 → wip 作废 → 输入先渲染 → 绿灯重放 → isPending 窗口语义）；T2 深层结论"shouldYield 是响应性开关不是正确性开关，批量更新/bailout 不依赖它"；T3 两家致命缺陷（setTimeout 4ms clamp 失控 / rAF 绘制前触发等于不让出）。建议下次复习优先自测 T1。

---
通过时间：2026-11-24
