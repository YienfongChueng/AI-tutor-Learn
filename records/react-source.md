---
mode: deep-dive
start_date: 2026-11-19
target_depth: conceptual
source_version: "React@19"
background: 面试高频原理为主、不逐行追源码、类比驱动保持有趣；Vue3 原理对照穿插
nodes:
  "A1":
    name: Stack Reconciler 的死穴
    module: Fiber 架构
    status: mastered
    attempts: 2
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: conceptual
    core_question: 为什么递归渲染不可中断，Fiber 要解决什么
  "A2":
    name: 双缓存树（diff 的本质）
    module: Fiber 架构
    status: mastered
    attempts: 1
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: null
    core_question: current 与 workInProgress 的关系，diff 的本质
  "A3":
    name: render 阶段遍历（beginWork/completeWork）
    module: Fiber 架构
    status: mastered
    attempts: 2
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: detail
    core_question: 三指针如何走完整棵树，flags 如何收集
  "A4":
    name: commit 三子阶段（mutation/layout/passive）
    module: Fiber 架构
    status: mastered
    attempts: 2
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: conceptual
    core_question: 三种执行时机的语义分工，commit 为何不可中断
  "B1":
    name: Hooks 链表挂载在哪
    module: Hooks 实现
    status: mastered
    attempts: 1
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: close
    core_question: fiber.memoizedState 链表结构
  "B2":
    name: useState 更新队列
    module: Hooks 实现
    status: mastered
    attempts: 1
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: close
    core_question: 快照语义与函数式更新的源码级解释
  "B3":
    name: 为什么条件调用会崩
    module: Hooks 实现
    status: mastered
    attempts: 1
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: null
    core_question: 依赖调用顺序分槽位的根因（衔接 2.7）
  "C1":
    name: Scheduler 让出主线程
    module: 调度与时间切片
    status: mastered
    attempts: 1
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: null
    core_question: 时间切片如何实现，消息循环/优先级队列
  "C2":
    name: Lane 优先级模型
    module: 调度与时间切片
    status: mastered
    attempts: 1
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: null
    core_question: 为什么用位运算表示优先级
  "C3":
    name: 批量更新与并发特性
    module: 调度与时间切片
    status: mastered
    attempts: 1
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: null
    core_question: 自动批处理、useTransition 是什么
---

# React 源码原理 学习记录

> deep-dive 模式 · 面试高频导向 · conceptual 深度 · 类比驱动
> 衔接：vue3.md（响应式系统对照）、react.md（s1-s4 使用层已全部掌握）

## 学习日志

| 日期 | 节点 | 结果 | 备注 |
|------|------|------|------|
| 2026-11-24 | A1-A4 | 通过 | 诊断题（长任务卡输入）现象答对、原因埋给 A1 揭晓。T2（双缓存后果：脏状态+丢对照物）T3（Vue 对照金句）一次过；T1 首轮：🔴render 中断性标反 + renderWithHooks 执行点未出 + mutation 未点出 → 补讲"render=草稿桌/commit=闪电解"锚；变式重测用户请求逃生舱直给答案（记参考通过）；10 秒确认题"函数体执行几次"首轮答 3 次（混淆中断=重跑）→ 补讲"续写不是重写"后答对（1 次；插队作废/StrictMode 才重跑，3.1"开发2生产1"源码级出处）。复习重点：render 可中断 vs commit 不可中断、函数体执行次数语义、renderWithHooks 位置 |
| 2026-11-24 | B1-B3 | 通过 | T2（if 调 hook 条件成立的风险：未来条件变引爆+setX 贴错单箱）T3（key 冲突+API 啰嗦）一次过；T1 首轮接近正确（格子定位/setAge 格无单 ✅，缺"结算"环节）→ 提示后补答"贴单→入队→render 遍历 queue 结算→字面量 action 覆盖"机制全对（口误 setName/setAge 已纠正）。复习重点：结算环节、顺序=唯一身份、自定义 hook 展开占调用者节点 |
| 2026-11-24 | C1-C3 | 通过 | 三题整体逃生舱（参考通过，复习重点）：T1 输入打断 Transition 链条（橙>绿→wip 作废→输入先渲染→绿灯重放→isPending 窗口语义）；T2"shouldYield 是响应性开关不是正确性开关"（批量更新/bailout 不依赖）；T3 MessageChannel vs setTimeout（4ms clamp）/rAF（绘制前触发=不让出）。s5 三块全部收官——React 并发=Fiber 可中断+队列可重放+lane 优先级三件套组装 |
