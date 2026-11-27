# Fiber 架构——React 凭什么敢"暂停渲染"

**它解决什么问题**
同步递归的渲染进度在调用栈里，JS 无法暂停恢复——长更新占死主线程 500ms，用户输入全被饿死。

**核心机制**
- 进度从调用栈搬到堆上链表：child/sibling/return 三指针，任意节点可停
- 递归改循环：workLoop 每轮问 shouldYield，让出主线程后从 nextUnitOfWork **续写**
- 双缓存：diff = 新 element 对照 current fiber 产出 wip，DOM 从不参与
- beginWork 下楼决策 / completeWork 上楼登记 flags；renderWithHooks 执行组件函数体
- commit 三子阶段不可中断：mutation 改 DOM → layout（绘制前同步）→ 绘制 → passive（useEffect 异步）

**最容易被追问的点**
- "哪段可中断？" → 只有 render；commit 必须一口气（否则半渲染界面）
- "中断后函数体重跑吗？" → 续写不是重写，跑 1 次；仅插队作废 wip / StrictMode 才重跑（"开发 2 生产 1"的源码出处）
- "diff 拿什么比？" → 新 element vs current fiber，按位置+key 复用（state 跟位置走的根因）
- "Vue 为什么不用 Fiber？" → 编译期优化让 diff 短到不值得中断；React 全动态 JSX 长到必须可中断
- "Fiber 让更新变快了吗？" → 没有，总工作量不变，买到的是响应性不是速度

**一句话类比**
render 是草稿桌随便停，commit 是闭馆换展闪电解；抄书人每抄一段看一眼门口，有人喊就放笔去处理，回来**接着抄**——不是从头再抄。
