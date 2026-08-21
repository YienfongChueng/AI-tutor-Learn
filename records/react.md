---
mode: mixed
start_date: 2026-08-17
background: 中级前端，Vue3 熟练，TS 已完成，零 React 基础假设
submodules:
  s1:
    name: 基础复盘（Vue 对照版）
    mode: knowledge
    status: mastered
    completed_at: 2026-08-17
  s2:
    name: Hooks 深入
    mode: knowledge
    status: pending
  s3:
    name: 工程化
    mode: knowledge
    status: pending
  s4:
    name: 实战项目
    mode: project
    status: pending
  s5:
    name: 源码原理（Fiber/Hooks 链表/调度）
    mode: deep-dive
    status: pending
nodes:
  "1.1":
    name: JSX 本质与 Vue 模板对照
    module: 基础复盘
    status: mastered
    attempts: 1
    last_tested: 2026-08-17
    mastery_level: 1
    failure_type: detail
    core_question: JSX 编译成什么，和 Vue 模板编译产物的本质异同
  "1.2":
    name: 函数组件与 Props（vs SFC）
    module: 基础复盘
    status: mastered
    attempts: 1
    last_tested: 2026-08-17
    mastery_level: 1
    failure_type: close
    core_question: 组件即函数意味着什么，props 单向数据流对照
  "1.3":
    name: State 与事件处理（useState 基础）
    module: 基础复盘
    status: mastered
    attempts: 1
    last_tested: 2026-08-17
    mastery_level: 1
    failure_type: detail
    core_question: setState 语义 vs Vue 自动响应式，事件命名/this 对照
  "1.4":
    name: 条件与列表渲染（key 的真实作用）
    module: 基础复盘
    status: mastered
    attempts: 1
    last_tested: 2026-08-17
    mastery_level: 1
    failure_type: close
    core_question: key 在 diff 中的角色，为什么 index 作 key 有坑
  "1.5":
    name: 受控组件与表单（vs v-model）
    module: 基础复盘
    status: mastered
    attempts: 1
    last_tested: 2026-08-17
    mastery_level: 1
    failure_type: detail
    core_question: 受控/非受控区别，v-model 在 React 里怎么拆
  "1.6":
    name: 组件通信方式全景（vs emit/expose/v-model）
    module: 基础复盘
    status: mastered
    attempts: 1
    last_tested: 2026-08-17
    mastery_level: 1
    failure_type: detail
    core_question: 回调 props/组合组件如何替代 Vue 的双向通道
  "2.1":
    name: useState 深入：快照语义与批量更新
    module: Hooks 深入
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: state 是快照不是引用，多次 set 如何合并
  "2.2":
    name: useEffect：副作用模型与依赖数组
    module: Hooks 深入
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: effect 的执行时机模型，依赖数组为何存在
  "2.3":
    name: useEffect 闭包陷阱与 cleanup
    module: Hooks 深入
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 过期闭包如何产生，cleanup 解决什么
  "2.4":
    name: useRef：可变盒子与 DOM 引用
    module: Hooks 深入
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: ref 为什么不触发重渲染，两种用途
  "2.5":
    name: useContext 与状态提升的取舍
    module: Hooks 深入
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: context 的重渲染粒度，何时该用/不该用
  "2.6":
    name: useMemo/useCallback 与重渲染优化
    module: Hooks 深入
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 重渲染何时发生，memo 三件套的正确用法
  "2.7":
    name: 自定义 Hook 实战
    module: Hooks 深入
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 自定义 hook 的本质是逻辑复用而非生命周期
  "3.1":
    name: Vite + React 项目结构与脚手架
    module: 工程化
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 标准项目结构、StrictMode 的双调用
  "3.2":
    name: React Router（路由与嵌套/守卫）
    module: 工程化
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 声明式路由对照 vue-router 的映射
  "3.3":
    name: 数据请求与服务端状态（async/TanStack Query 选讲）
    module: 工程化
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 请求放哪、缓存与重复请求谁管
  "3.4":
    name: 性能优化清单（memo/lazy/Suspense）
    module: 工程化
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 什么时候优化、先测后优化的路径
  "4.1":
    name: 实战项目（project 模式，启动时再拆节点）
    module: 实战项目
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 用 React 复刻一个带路由/请求/状态管理的完整应用
  "5.1":
    name: 源码原理（deep-dive 模式，启动时再拆主题块）
    module: 源码原理
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: Fiber 架构/Hooks 链表/调度（衔接 Vue3 主题块的对照）
---

# React 学习记录

> 混合模式：s1-s3 知识模式（Vue 对照视角）· s4 项目模式 · s5 深度讲解模式。
> 用户背景：Vue3 熟练 -> 基础复盘走"对照迁移"路线，不从零讲。

## 学习日志

| 日期 | 节点 | 结果 | 备注 |
|------|------|------|------|
| 2026-08-17 | 1.1 | 通过 | attempts=1（两轮微调后通过）；细节错误：typeof el 答"React Element"（应为'object'）、children 文本段与表达式段合并（实际逐段切分['Hi ',name,<b/>]）、文本段误包 _jsx（只有元素才编译为 _jsx 调用）。前置诊断：var/let 闭包题 var 值答 2,2,2（应为 3,3,3），方向对 |
| 2026-08-17 | 1.2 | 通过 | attempts=1（一轮微调后通过）；接近正确：①根因（函数体反复执行）对但误归因"state快照"（快照是结果非原因）；②"state存React外部"对但漏"let count=0每轮重新初始化"，微调补齐；归因小错：以为只有父组件触发重渲染（自身state变化也触发）。fetch+setState死循环答对 |
| 2026-08-17 | 1.3 | 通过 | attempts=1（多轮微调后通过）；主动提问快照vs函数式更新（补讲快递模型：传值=寄定格成品/传函数=寄指令单接流水线最新值）。TodoList调试：隐藏bug onClick={handleAdd()}立即执行✅、A/B的Object.is引用判定✅但修法代码错（A非法spread语法+给const赋值、B对象spread数组+原地改旧对象，已示范正解）、C更新丢失三轮后答出函数式更新✅ |
| 2026-08-17 | 1.4 | 通过 | attempts=1（一轮微调后通过）；②③一次答对（key=id复用+key重置技巧）；①机制对（state挂位置）但结论推错：❤应在头部新消息上、"你好"的赞丢失显示空白，画位置对照表后修正。核心口诀：props跟数据走，state跟位置走 |
| 2026-08-17 | 1.5 | 通过 | attempts=1（两轮微调后通过）；细节错误：checkbox 用 value 而非 checked、form 挂 onClick 而非 onSubmit、trim 当全局函数调用（同1.3把方法当独立函数的老毛病）；修完后新引入"isNaN 全局判断代替字段语义"（"007"会变7），按 name 分派后通过。加分：函数式更新防更新丢失一次写对 |
| 2026-08-17 | 1.6 | 通过 | attempts=1（三轮+逃生舱）；架构判断一次全对（状态放父级/草稿放子组件/回调props）。硬伤回炉：addFn={setTodoList(...)}渲染期立即调用（同款错误第3次）、filter条件写反（===应为!==）、key="item.id"字符串字面量（1.4回炉）、list&&map漏{}。用户请求直接给答案（逃生舱）：完整代码+概念标准答案+10秒确认题（showToast回调props）答对。概念题术语混乱（"提升为受控"）已纠正：提升=上移到共同父级，受控=props驱动+回调上报，草稿=子组件私有态例外 | 模块1基础复盘完成 |

## 模块完成状态

- **s1 基础复盘：2026-08-17 完成**（6/6 节点 mastered，阶段总结见 summaries/react_basics.md）
