---
mode: knowledge
start_date: 2026-11-24
background: Pinia 熟（轻扫校准）；Redux/Mobx demo 级已忘（重点重建）；Zustand 全新；TanStack Query/Context 刚学完可对照
nodes:
  "1.1":
    name: Pinia 全流程校准 + 为什么 Vue 生态不需要 reducer
    module: Pinia 校准
    status: mastered
    attempts: 1
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: null
    core_question: 响应式流派如何回答"放哪/谁改/怎么通知"三问
  "2.1":
    name: Mobx 心智模型（observable/action/observer，对照 Vue3 proxy 原理）
    module: Mobx
    status: mastered
    attempts: 1
    last_tested: 2026-11-24
    mastery_level: 1
    failure_type: null
    core_question: 依赖收集/自动追踪在 React 里如何落地
  "2.2":
    name: Mobx React 接线（makeAutoObservable + observer 实战）
    module: Mobx
    status: mastered
    attempts: 2
    last_tested: 2026-11-25
    mastery_level: 1
    failure_type: detail
    core_question: 类 store 怎么写，observer 粒度
  "3.1":
    name: Zustand create 与 store 即 hook + selector 粒度
    module: Zustand
    status: mastered
    attempts: 2
    last_tested: 2026-11-25
    mastery_level: 1
    failure_type: detail
    core_question: 一行 create 背后的订阅模型
  "3.2":
    name: Zustand 中间件与爆火原因（对照 Redux）
    module: Zustand
    status: mastered
    attempts: 2
    last_tested: 2026-11-25
    mastery_level: 1
    failure_type: detail
    core_question: persist/devtools 生态位，为什么比 Redux 轻
  "4.1":
    name: redux 核心：store/action/reducer，为什么 reducer 必须纯
    module: Redux 全家
    status: mastered
    attempts: 2
    last_tested: 2026-11-26
    mastery_level: 1
    failure_type: detail
    core_question: 单向数据流哲学三问
  "4.2":
    name: react-redux：useSelector/useDispatch vs 旧 connect
    module: Redux 全家
    status: mastered
    attempts: 2
    last_tested: 2026-11-26
    mastery_level: 1
    failure_type: conceptual
    core_question: 订阅与重渲染粒度
  "4.3":
    name: RTK：createSlice + Immer
    module: Redux 全家
    status: learning
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: RTK 消灭了哪些模板代码
  "4.4":
    name: RTK 异步：createAsyncThunk + extraReducers
    module: Redux 全家
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 请求生命周期的 action 化
  "4.5":
    name: RTK Query（对照 TanStack Query）
    module: Redux 全家
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 数据请求缓存选谁，两 Query 的异同
  "5.1":
    name: 四库横向对比 + 决策树 + 面试叙事
    module: 选型收官
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 什么场景选什么，为什么
---

# 数据流（Pinia/Mobx/Zustand/Redux 全家）学习记录

## 学习日志

| 日期 | 节点 | 结果 | 备注 |
|------|------|------|------|
| 2026-11-24 | 1.1 | 通过 | 情景分析题两问：①React 无依赖收集→引用比较失效→必须不可变新对象 / Vue proxy set 拦截精确通知——满分；②直接改的库选 Mobx ✅，机制追问"靠什么"答"响应式系统"（哲学对，面试需具体化：proxy 拦截+依赖收集+observer 精确订阅）。校准完成，起点确认 |
| 2026-11-24 | 2.1 | 通过 | 心智模型：observable/computed/autorun ↔ reactive/computed/watchEffect 对照迁移顺畅。应用户要求补充 API 速查 12 条 + 记账本完整案例（class 定义/RootStore+Context 接线/observer 使用/persist，并入 2.2 大部分内容）。编码题两处 🔴：缺 makeAutoObservable（启动开关）、addRecord 缺 async；两问逃生舱给答案：①computed 值相等不发通知→不渲染（订阅数组引用会渲染的差异是精妙点）②enforceActions 抛错。参考通过，复习重点：makeAutoObservable 必须存在、computed 值相等优化、订阅派生值选型 |
| 2026-11-25 | 2.2 | 通过 | 接线五步带注释讲解（类定义→RootStore→Context→Provider→observer；两条对照：storeToRefs 不需要因属性访问追踪、RootStore=AuthProvider 多 store 版；坑排行 6 条）。实战验收题（分类统计页）两轮：首版 8 处错误——🔴makeObservable/makeAutoObservable 用混（排除表语义只在 Auto 版）、CategoryStore 没传 root、toJS 用在 boolean 上、try() 语法、解构缺括号、export 进类体、**过滤条件反（!== income，1.6/4.6 老坑第 3 次回潮）**、amount/total 字段名；修订版 8 处全修 ✅。残留提醒 2 条：toJs→toJS 拼写、onClick 直传 action 的参数是 MouseEvent（快照从 store 内部取）。亮点：autoBind 解构 action 当回调的组合用对了。Mobx 收官 |
| 2026-11-25 | 3.1 | 通过 | 讲解 create/set/get 全 API + selector 粒度（反模式 Object.is 失效，修法拆 selector/useShallow）+ 底层 useSyncExternalStore+闭包 store + 四库订阅对照（广播/门铃/公众号）。应用户要求补充柯里化 create<T>()() 讲解（中间件挡类型推断，"中间件在里类型在外"）。编码题两轮：首版两问满分但 4 处 🔴（对象字面量缺逗号第 3 次回潮、resetAll 忘包 set 成逗号表达式、set 返回数组应为部分 state、records ?? map 永不渲染）+ onClick 直传带参 action 回潮；修订版 6 处全修 ✅。复习重点：set updater 返回部分 state 对象、反模式与 useShallow、柯里化形状 |
| 2026-11-25 | 3.2 | 通过 | 中间件洋葱模型（persist/devtools/immer 三件）+ 爆火三叙事 + 边界（服务端状态归 TanStack Query）。编码题：首版把 {name, partialize} 挂到 devtools options（persist 裸奔）→ 提示"谁的功能找谁"后自行修对；口头两问缺后半（RTK 对照/机制理由）→ LLM 补参考稿。面试标准答案沉淀："客户端状态 Zustand，服务端状态 TanStack Query，互补不替代"。Zustand 收官 |
| 2026-11-26 | 4.1 | 通过 | 原版 redux 三件套（原教旨脱糖衣）+ 纯的三重回报（可测试/可重放/可追溯）+ 禁忌四条 + 财务制度类比。任务 2（reducer 里发 axios 三问分析）一次过，补讲"异步时序破坏可重放"凑齐三回报；任务 1 两轮：payload 语义错位（{count:2} 对象 vs 增量语义）、default 缺 return（2.6 同款回潮）→ 修订后仍漏算结果 → 给出 {count:1} 确认通过（close）。复习重点：action 语义与 payload 形状一致、default 必 return、先订阅后 dispatch |
| 2026-11-26 | 4.2 | 通过 | Provider+hooks 接线、useSelector 与 Zustand selector 机制逐字相同（Object.is 心法跨库闭环：2.6/3.1/4.2）、connect 对照表（默认浅比较 mapState vs useSelector 严格 Object.is）。两轮：首版 🔴actionCreator 当 store 字段 selector（概念：store 树里只有 state）+①链条"没订阅不跑"错位（所有 selector 都重跑）+②"state 大"不准（真凶=昂贵计算+新对象反模式）→ 修订全过。金句沉淀："重跑不可怕，重跑里藏了重活才可怕"。复习重点：actionCreator 不进 store、跑与渲染两步分离、reselect 记忆化 |
