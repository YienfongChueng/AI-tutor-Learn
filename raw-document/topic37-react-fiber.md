# Fiber 架构——React 凭什么敢"暂停渲染"

## 讲解原文

### 子节 A1：Stack Reconciler 的死穴

#### 第 0 段：直觉锚定——抄书的人不肯放笔

想象你要**手工抄写一本书**。React 15 的做法是：拿到书就必须**一口气抄完**才能放下笔——中途不喝水、不接电话、门口有人喊你也不理。

这 500ms 里，"门口有人喊你"（用户点击按钮、输入框打字）全被无视了。**不是浏览器不想响应，是那个抄书的人不肯放笔**——因为 JS 是单线程的，抄书（渲染更新）和接电话（处理输入事件）是同一个人。

Fiber 要做的事：**把"一口气抄完"改造成"每抄完一段，先看看有没有更急的事，有就先去处理，回来接着抄"**。这个类比里的每个要素后面都有真实对应物：

| 类比 | 真实对应物 |
|------|-----------|
| 抄书 | 渲染：把组件树算成 DOM 该长什么样 |
| 抄到哪了 | workInProgress Fiber 节点 |
| 每段抄完检查一次 | workLoop 循环里的 shouldYield() |
| 回来接着抄的能力 | 链表指针（A2 讲） |
| 书 | 组件树 |

#### 第 1 段：问题背景——递归是不可中断的

React 15 的调和器叫 **Stack Reconciler**（栈调和器）。它 diff 虚拟 DOM 的方式是**递归**：

```
render(element)
  └─ 递归处理 children
       └─ 递归处理 children 的 children
            └─ ……直到叶子节点
```

递归的本质是**调用栈**：每层"children 处理到哪了"的信息存在 JS 调用栈里。而调用栈是语言引擎管的，**JS 代码没有能力暂停它再恢复它**——要么跑完整个递归，要么函数 return 时整个栈被销毁。

所以那次 500ms 的 setState：主线程被同步递归占死 500ms，浏览器连"把你点的按钮标记成按下状态"的机会都没有（浏览器处理输入也要主线程），输入延迟、动画掉帧。

**目标**：把"不可打断的长任务"拆成"可暂停、可恢复、可插队的小任务"，且**总工作量不变**。

> ⚠️ **常见先入为主的误解**：很多人以为 React 18+ "天生"就能中断渲染，中断能力是浏览器给的。**错。** 浏览器只给了单线程，什么都没给——可中断是 React **自己把递归改写成"循环 + 链表"** 换来的。这也是为什么 Fiber 重构花了团队好几年：改的不是某个函数，是整个调和器的骨架。

#### 第 2 段：核心数据结构——把"遍历到哪了"从栈搬到堆

对比一下两代方案的信息存放位置：

```
Stack Reconciler（旧）
  "处理到哪了"存在调用栈里
  暂停 = 栈被销毁 = 永远不知道刚才到哪 = 无法恢复

Fiber（新）
  把树的结构 + 遍历进度，做成堆上的对象图
  每个节点是一个 Fiber 对象，互相用指针连着
  暂停 = 只是被调度器"暂时不再调用"
  恢复 = 从手里这个对象的指针继续走
```

FiberNode 和本节相关的字段（其余先省略）：

```
FiberNode
├── type        # 组件类型（函数/类/字符串标签）
├── return      # → 指向父节点（"回到上一级抄书进度"）
├── sibling     # → 指向右边的兄弟节点
├── child       # → 指向第一个子节点
├── flags       # 副作用标记（这棵子树哪些地方要动 DOM）
└── ...         # alternate、lanes 等，后面子节讲
```

三个指针搭出的链表树（实例，3 个节点）：

```
   <App> fiber
    │ child
    ▼ 指向第一个孩子
   <List> fiber ──sibling──▶ <Header> fiber
    │ child                    （return 指回 <App>）
    ▼
   <ListItem/> fiber ──sibling──▶ ...
                  （return 指回 <List>）
```

关键认知：**链表化的树可以在"任意一个节点"停下**——因为你手里的引用本身就是进度。递归做不到这一点，递归的进度在栈里。

#### 第 3 段：运行流程——递归改循环

概念级源码坐标（行号随版本浮动，看函数名）：

1. **定位**：`React@19` · `ReactFiberWorkLoop.js` · `workLoopConcurrent` / `shouldYieldToHost`
2. **阅读入口**：`workLoopConcurrent` 就几行，整个函数就是骨架
3. **关键行**：循环条件里的 `shouldYield()` ——"要不要让出主线程"的判断点

新旧伪代码对比（控制流完整保留）：

```js
// ── 旧：Stack Reconciler（递归，无法暂停）──
function render(element) {
  mountComponent(element)             // 本节点的处理
  element.children.forEach(child => {
    render(child)                     // 递归子节点——想停也停不了
  })
}

// ── 新：Fiber workLoop（循环，随时可停可续）──
let nextUnitOfWork = fiberRoot        // 当前要处理的节点（= 抄到哪了）

function workLoopConcurrent() {
  while (nextUnitOfWork !== null && !shouldYield()) {
    // shouldYield()：时间片用完了吗？有更高优先级的事吗？
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
    // performUnitOfWork：处理当前节点，返回"下一个要处理的节点"
    // ↑ 注意：进度不在栈里，在 nextUnitOfWork 这个变量里！
  }
  // 循环退出两种情况：
  // 1. nextUnitOfWork === null → 整棵树走完 → 进入 commit
  // 2. shouldYield() 为 true → 让出主线程，浏览器去处理输入/渲染
  //    恢复时：调度器再次调用 workLoop，从 nextUnitOfWork 继续
}
```

`performUnitOfWork`（在 `ReactFiberBeginWork.js`）内部就是"处理当前 fiber，决定下一个是谁"——它不递归，只**返回**下一个指针。这就是"递归改循环"的全部秘密：**栈上的控制流，被翻译成了堆上的指针**。

#### 第 4 段：设计动机与权衡

- **核心约束**：JS 单线程，且浏览器每帧只给约 16.7ms 预算（60fps）。同步递归一个长任务就爆预算
- **换来了什么**：可中断、可恢复、可插队（优先级，C 块讲）——长更新不再饿死用户输入
- **牺牲了什么**：
  1. **内存**：每个组件节点都是堆上的 Fiber 对象，还常驻两棵树（A3 讲双缓存）——Vue 没有这份开销
  2. **复杂度**：递归 100 行能干的事，链表 + 循环 + 恢复逻辑要写几千行。Fiber 重构搞了两年多才上生产
  3. **单次更新的速度没变快**：diff 总工作量差不多（见第 5 段）

**Vue3 对照（面试金句）**：为什么 Vue 不需要 Fiber 式重构？因为 Vue 把优化放在**编译时**——静态提升、patchFlag 把"哪些节点会变"提前标好，运行时 diff 又快又短，短到**不值得引入可中断的复杂度**。React 的 JSX 是完全动态的 JS，编译期几乎没信息可用，运行时全量 diff 又贵又长，**长到必须可中断**。一句话：*Vue 的聪明在编译期，React 的聪明在运行期；渲染时长决定了谁需要 Fiber。*

#### 第 5 段：次级误解和边界

1. **"可中断 = 任意时刻都能停"** ——错。中断粒度是 **fiber 节点**（处理完一个才检查 shouldYield），且 **commit 阶段不可中断**（动 DOM 必须一口气完成，否则用户看到半渲染的界面——这个在 A4 展开）
2. **"Fiber 让更新变快了"** ——错。单次 diff 总工作量基本不变，Fiber 买到的是**响应性**（不饿死输入），不是更快的 diff。两者别混
3. **中断期间 DOM 不会"半渲染"** ——render 阶段（循环跑的那个）全程**不碰 DOM**，只算"DOM 应该长什么样"；DOM 只在 commit 阶段被一次性修改。所以中断一万次，用户看到的界面也始终是完整的

---

**本节结论**：Fiber 解决的是"同步递归不可中断"——办法是把遍历进度从调用栈搬到堆上的链表对象图，递归改循环，让出主线程的能力由此而来。

现在我们知道了"可暂停"靠的是指针连成的对象图。但一个新问题：这棵 fiber 树上的对象**从哪来**？每次更新是全量重建吗？新旧两棵树什么关系？——这就是 **A2/A3（FiberNode 从哪来 + 双缓存树）** 要回答的事情。

### 子节 A2：双缓存树——diff 的本质是"对照两棵树"

#### 第 0 段：直觉锚定——草稿纸与展台

博物馆要更新一件拼装展品，笨办法是把展台上的**当场拆了重拼**——观众全程看到残骸。聪明办法是：旁边放一张**草稿桌**，在新桌上拼好新版，趁闭馆一刻钟**整桌对换**——观众永远只看到完整的展品。

| 类比 | 对应物 |
|------|--------|
| 展台上的展品 | current 树（当前 DOM 对应的 fiber 树） |
| 草稿桌 | workInProgress 树（正在算的 fiber 树） |
| 趁闭馆换展品 | commit 阶段（同步、不中断） |
| 两张桌子的对等关系 | fiber 的 alternate 字段互相指 |

#### 第 1 段：问题背景——为什么要有两棵树

A1 说过 render 阶段可以中断一万次，中断期间用户看到的还是旧界面。如果 React 只有**一棵** fiber 树，边算边改这棵树，中断时树就是"半新半旧"的脏状态，恢复都无从恢复——更别说要对照 diff 了。

所以必须**新算的放在另一棵树上**：current 树永远代表屏幕上的真实 DOM；workInProgress 树随便中断，坏了就扔，current 纹丝不动。

> ⚠️ **误解拦截**：很多人以为"diff 是拿新 JSX 和旧 DOM 比"。实际是拿**新的 ReactElement（你 return 的 JSX 对象）和旧的 current fiber 树**对照，产出 workInProgress 树。DOM 从头到尾不参与 diff。

#### 第 2 段：核心数据结构——alternate 互指

```
         current 树                      workInProgress 树
   <App> current fiber ◄──alternate──► <App> wip fiber
      │ child                              │ child
      ▼                                    ▼
   <List> current      ◄──alternate──► <List> wip
```

每对节点通过 `alternate` 互指。**首次挂载（mount）时没有 current**，React 从你的 JSX element 树开始边递归边生成 fiber 节点，直接建 workInProgress 树，建完 commit 后它转正当 current。

**更新时**：workLoop 从 current 树的对应节点出发，逐节点对照新的 element：
- type 没变 → **复用** current fiber（`alternate` 那个对象），只更新 props——这就是 React "state 跟位置走"（1.4 考过）的底层原因：**fiber 复用按位置判定**
- type 变了 → **丢弃**旧 fiber，新建节点，旧子树全部重置

#### 第 3 段：运行流程——树的交换

```js
// commit 完成后的一次"换桌"（概念伪代码）
root.current = workInProgressRoot      // wip 转正当 current
// 下一轮更新再建新的 wip（旧 current 的 alternate 又被拿来当 wip 用）
// 两棵树的角色每轮互换——这就是"双缓存"名字的由来
```

概念坐标：`ReactFiberWorkLoop.js` · `prepareFreshStack` / `commitRoot`；alternate 的复用逻辑在 `ReactFiberBeginWork.js` 的 `reconcileChildFibers` 附近。

#### 第 4 段：设计动机与权衡

- **约束**：render 可中断，中断期间不能污染"屏幕上正在展示的树"
- **换来**：中断安全性 + diff 天然有"旧版"可对照（reconcile 的前提就是有 current）
- **代价**：双倍 fiber 内存。Vue3 没有 fiber 双缓存——它 diff 时直接在旧 VNode 上打补丁，因为**不可中断所以不需要两份**（对照 A1 的 Vue 金句，闭环）

#### 第 5 段：次级误解和边界

1. **"key 重置子树"的源码级原因**：key 变 → reconcile 判定"不是同一个节点" → 不复用 fiber → 全新 fiber 对象 → 挂其上的 state 全新。1.4 那句"state 跟位置走"的完整版是：**state 挂在 fiber 对象上，fiber 按位置+key 复用**
2. **flags 在哪个阶段标**：diff 时只在 workInProgress 上打 `flags`（Placement/Update/Deletion），**不会**碰 DOM；flags 是 A4 commit 的施工清单

现在我们知道了 diff 的全貌：拿新 element 对照 current 树，在 workInProgress 上产出打了 flags 的新树。但"逐节点对照"这个动作具体怎么走？flags 怎么被收集起来？——这是 **A3（render 阶段的 beginWork / completeWork）**。

### 子节 A3：render 阶段——beginWork 与 completeWork 的散步

#### 第 0 段：直觉锚定——验货员下楼

处理一栋楼的报修单：验货员先进一间房（**beginWork**：检查这间房要修什么，决定下一间去哪）；一个房间彻底查完，退出来时在**楼道汇总单**上登记"这间要换窗"（**completeWork**：收尾登记副作用）。走完整栋楼 = 一轮 render。随时可以中途停——手里那张"当前房间"就是进度。

#### 第 1 段：问题背景

A1 的 `performUnitOfWork` 内部具体做什么？它其实是两个函数的交替：`beginWork`（进入节点：处理它、产出子节点指针）和 `completeWork`（离开节点：收尾、收集 DOM 相关信息）。理解它们的分工，"React 怎么走一棵链表树"就完整了。

#### 第 2 段：核心结构——遍历的"下楼算法"

```
        App
       ↙ child ↘ sibling
     List      Header
      │ child
      ▼
   ListItem ──sibling──▶ ListItem2

遍历顺序：App.begin → List.begin → ListItem.begin
        → ListItem.complete → ListItem2.begin → ListItem2.complete
        → List.complete → Header.begin → Header.complete → App.complete
（深度优先，begin 下楼 / complete 上楼）
```

没有递归怎么"下楼"？靠三个指针就能走完整棵树：
- beginWork 返回 child → 有孩子就下楼
- 没孩子 → completeWork 自己，然后看 sibling → 有兄弟就横移
- 没 sibling → return 回父节点，父节点做 completeWork，再找父的 sibling

#### 第 3 段：运行流程

```js
// 概念伪代码：performUnitOfWork 的真身
function performUnitOfWork(fiber) {
  const next = beginWork(fiber)     // 进入：算这棵子树的 wip，返回第一个 child
  if (next) return next             // 有孩子 → 下楼
  // 没孩子：一路向上 completeWork，顺带找兄弟
  let current = fiber
  while (current) {
    completeWork(current)           // 离开：收尾登记（flags → effect 收集）
    if (current.sibling) return current.sibling   // 横移
    current = current.return        // 上楼
  }
  return null                       // 整棵树走完
}
```

坐标：`ReactFiberBeginWork.js` · `beginWork`（内部按 fiber.tag 分流到 updateFunctionComponent 等——**函数组件就是在这里被真正调用的**，`renderWithHooks` 一行触发你的组件函数体，也是 B 块 Hooks 链表被读写的地方）；`ReactFiberCompleteWork.js` · `completeWork`。

**中断恢复的真实含义**：`workLoop` 被调度器再次唤醒时，从 `nextUnitOfWork`（某棵树上正在处理的那间"房"）直接继续——不需要任何"重建上下文"，因为上下文就是 fiber 对象本身。

#### 第 4 段：设计动机与权衡

- begin/complete 分离的动机：**向下是"决策"（要不要复用、生成什么 child），向上是"登记"（真正动 DOM 的活儿清单）**。决策可中断，登记不可撤销——职责分开才敢中断
- 牺牲了递归的简洁性（一个函数嵌套搞定 vs 三指针循环）

#### 第 5 段：次级误解和边界

1. **"beginWork 里执行组件函数，所以组件函数是每棵子树跑一次"** ——对，一次 render 里每个函数组件体恰好执行一次，它 return 的 element 就是"新对照物"
2. **render 阶段可能被丢弃重跑**：中断后若来了更高优先级更新（C 块），整棵 wip 可能作废重算——所以 render 阶段必须是**纯计算**，这就是"组件函数要纯"的架构级原因：你的组件体可能被执行两次（StrictMode 双调用正是模拟这个）

现在我们知道了 render 阶段产出一棵打了 flags 的 workInProgress 树。但 DOM 到底什么时候被真正修改？**useEffect 和 useLayoutEffect 的执行时机差异**为什么存在？——这是 **A4（commit 三子阶段）**。

### 子节 A4：commit 阶段——mutation / layout / passive

#### 第 0 段：直觉锚定——装修完工交房

草稿桌上的新展品拼完了（render 完），现在**闭馆交货**。交货分三步走且一步不回退：**拆卸旧件**（mutation：动 DOM）→ **通电验收**（layout：读布局、同步跑挂载后的钩子）→ **客人回访**（passive：异步跑 useEffect）。

#### 第 1 段：问题背景

render 阶段算出了"DOM 应该长什么样"（flags 施工清单）。commit 负责真正施工。为什么拆三步？因为**三种事的时机语义不同**：DOM 修改必须同步一口气；`useLayoutEffect` 必须在浏览器绘制前同步跑完（否则测量布局会闪）；`useEffect` 不阻塞绘制，可以异步慢慢跑。一种时机装不下三种语义，所以 commit 拆成三个子阶段。

> ⚠️ **误解拦截**：commit **不可中断**。render 可中断是 A1 的结论，但一旦进入 commit，DOM 操作必须一口气完成——中断在 commit 里会让用户看到半渲染界面。这就是"render 慢慢算、commit 闪电解"的设计。

#### 第 2 段：核心结构——三个子阶段各干什么

| 子阶段 | 干什么 | 同步性 | 你认识的钩子 |
|--------|--------|--------|-------------|
| mutation | 按 flags 改 DOM（插/改/删） | 同步一口气 | ref 卸载（旧 ref置 null） |
| layout | 绘制前同步执行 | 同步 | **useLayoutEffect**、ref 赋值 |
| passive | 绘制后异步执行 | 异步（调度器安排） | **useEffect** |

#### 第 3 段：运行流程——这解释了你考过的一切

```
render（可中断，不碰 DOM）
   │ 走完
   ▼
commit-mutation：DOM 一次性更新         ← 此时浏览器还没绘制
   │
   ▼
commit-layout：同步跑 useLayoutEffect   ← 你在这里读 ref/改样式，绘制前生效
   │
   ▼
浏览器绘制（用户看到了新界面）
   │
   ▼（异步，调度）
commit-passive：跑 useEffect            ← 3.3 的竞态修复在这里跑
```

坐标：`ReactFiberCommitWork.js` / `ReactFiberCommitEffects.js`（19 中按子阶段拆成 commitMutationEffects / commitLayoutEffects / flushPassiveEffects 等函数）。

**考过的题在这里全部对上号**：
- 3.3 的 `ignore` 旗标修复竞态：cleanup 在 effect 重跑/卸载时执行——effect 的注册在 render，真正 flush 在 passive，所以 cleanup 拿的是**上一轮**的闭包
- StrictMode 双调用：render 双跑（A3 的"可能丢弃重跑"模拟）+ effect 双注册双清理，都在这套流水线上
- 4.8 的 SearchBar 闪失：passive 阶段之后数据回来触发新 render——loading 早退发生在新一轮 render，SearchBar 被"换房"

#### 第 4 段：设计动机与权衡

- mutation 前不可中断 render 已由双缓存兜底；mutation 本身不可中断由"DOM 一次性"兜底——**两道保险合起来才敢把 render 打断**
- 牺牲：useEffect 永远晚一帧（passive 异步），所以要 useLayoutEffect 救"绘制前必须完成"的场景（但同步阻塞，慎用）

#### 第 5 段：次级误解和边界

1. **"setState 后 DOM 立刻变了，但我 useEffect 里读到的是旧值"** ——不是 bug：mutation 已改 DOM，passive 还没跑，effect 闭包是**那次 render 的快照**（B 块讲闭包来源）
2. **同步渲染 vs 并发渲染**：老版本/同步模式下整棵树一轮跑完不分片；并发特性（C 块）只是让 render 阶段**可以**分片，commit 的三段结构从没变过

---

**主题块 A 收束**：C 发许可证（调度）→ A 的 render 阶段在双缓存上散步打 flags（可中断）→ commit 三子阶段闪电解（不可中断）→ DOM 更新。整条流水线的每一站都接触过了。

进入考核：3 道题，见下一条消息。

## 考核过程

### 题目 1（源码追踪题）：删除触发 re-render 的完整调用链

**用户回答**：performUnitOfWork、shouldYield、workLoop、nextUnitOfWork、commit（不可中断）、workInProgress、beginWork、completeWork、render（❌不可中断）——词基本齐但顺序散、三处缺失。

**点评**：未通过。🔴render 中断性标反（本主题块第一结论）；组件函数体执行点（beginWork → updateFunctionComponent → renderWithHooks）未点出；DOM 更新落点（commit-mutation）未点出。错误类型：conceptual。

### 题目 2（机制推理题）：去掉 alternate 会怎样

**用户回答**：中断时树是"半新半旧"脏状态无法恢复；没有对照 diff 的对象；wip 坏了就扔、current 不受影响。

**点评**：通过（一次过）。两个必须项（哪个环节错、为什么）全中，还主动复述了修复逻辑。加分。

### 题目 3（设计理解题）：Vue3 为什么不需要 Fiber

**用户回答**：Vue3 编译时静态提升 + patchFlag 预标变化节点，运行时 diff 又快又短；React JSX 全动态、编译期无信息、运行时全量 diff 贵又长，长到必须可中断。

**点评**：通过（一次过）。核心权衡完整，且给出了两家的优化路线差异（面试金句级别）。

### 题目 1 变式重测（搜索触发 re-render 调用链）

**用户回答**：请求逃生舱直接给答案。

**点评**：按逃生舱处理——给出标准答案并逐段解释（loop → unitOfWork → commit 三连骨架），记为"参考通过，复习重点"。10 秒确认题"中断两次，函数体执行几次"：首轮答 3 次（把"可中断"混淆成"每次中断重跑函数体"，conceptual），不闪烁理由答对；补讲"中断是续写不是重写——Fiber 把进度搬上堆就是为了续写"后答对（1 次；重跑仅两种情况：高优先级插队作废 wip、StrictMode 模拟——3.1"开发 2 生产 1"的源码级出处）。确认通过。

---
通过时间：2026-11-24

