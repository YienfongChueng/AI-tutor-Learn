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
    status: mastered
    completed_at: 2026-09-01
  s3:
    name: 工程化
    mode: knowledge
    status: mastered
    completed_at: 2026-09-03
  s4:
    name: 实战项目（记账本）
    mode: project
    status: mastered
    started_at: 2026-09-03
    completed_at: 2026-11-19
    project: 记账本（Vite+React+TS / json-server / TanStack Query / Context / React Router）
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
    status: mastered
    attempts: 1
    last_tested: 2026-08-24
    mastery_level: 1
    failure_type: detail
    core_question: state 是快照不是引用，多次 set 如何合并
  "2.2":
    name: useEffect：副作用模型与依赖数组
    module: Hooks 深入
    status: mastered
    attempts: 2
    last_tested: 2026-08-24
    mastery_level: 1
    failure_type: conceptual
    core_question: effect 的执行时机模型，依赖数组为何存在
  "2.3":
    name: useEffect 闭包陷阱与 cleanup
    module: Hooks 深入
    status: mastered
    attempts: 2
    last_tested: 2026-08-27
    mastery_level: 1
    failure_type: detail
    core_question: 过期闭包如何产生，cleanup 解决什么
  "2.4":
    name: useRef：可变盒子与 DOM 引用
    module: Hooks 深入
    status: mastered
    attempts: 2
    last_tested: 2026-08-27
    mastery_level: 1
    failure_type: conceptual
    core_question: ref 为什么不触发重渲染，两种用途
  "2.5":
    name: useContext 与状态提升的取舍
    module: Hooks 深入
    status: mastered
    attempts: 1
    last_tested: 2026-08-27
    mastery_level: 1
    failure_type: null
    core_question: context 的重渲染粒度，何时该用/不该用
  "2.6":
    name: useMemo/useCallback 与重渲染优化
    module: Hooks 深入
    status: mastered
    attempts: 2
    last_tested: 2026-08-27
    mastery_level: 1
    failure_type: conceptual
    core_question: 重渲染何时发生，memo 三件套的正确用法
  "2.7":
    name: 自定义 Hook 实战
    module: Hooks 深入
    status: mastered
    attempts: 3
    last_tested: 2026-09-01
    mastery_level: 1
    failure_type: detail
    core_question: 自定义 hook 的本质是逻辑复用而非生命周期
  "3.1":
    name: Vite + React 项目结构与脚手架
    module: 工程化
    status: mastered
    attempts: 2
    last_tested: 2026-09-01
    mastery_level: 1
    failure_type: detail
    core_question: 标准项目结构、StrictMode 的双调用
  "3.2":
    name: React Router（路由与嵌套/守卫）
    module: 工程化
    status: mastered
    attempts: 2
    last_tested: 2026-09-01
    mastery_level: 1
    failure_type: conceptual
    core_question: 声明式路由对照 vue-router 的映射
  "3.3":
    name: 数据请求与服务端状态（async/TanStack Query 选讲）
    module: 工程化
    status: mastered
    attempts: 2
    last_tested: 2026-09-03
    mastery_level: 1
    failure_type: detail
    core_question: 请求放哪、缓存与重复请求谁管
  "3.4":
    name: 性能优化清单（memo/lazy/Suspense）
    module: 工程化
    status: mastered
    attempts: 2
    last_tested: 2026-09-03
    mastery_level: 1
    failure_type: detail
    core_question: 什么时候优化、先测后优化的路径
  "4.1":
    name: 项目初始化（Vite 脚手架/目录/json-server/API 封装）
    module: 里程碑1 MVP
    status: mastered
    attempts: 1
    last_tested: 2026-09-04
    mastery_level: 1
    failure_type: null
    core_question: 脚手架产物怎么组织成可维护结构，api 层怎么封装
  "4.2":
    name: 账目列表页（useQuery 拉取与展示）
    module: 里程碑1 MVP
    status: mastered
    attempts: 1
    last_tested: 2026-09-04
    mastery_level: 1
    failure_type: null
    core_question: queryKey 设计与 loading/error UI
  "4.3":
    name: 记一笔（表单 + useMutation + invalidate）
    module: 里程碑1 MVP
    status: mastered
    attempts: 3
    last_tested: 2026-09-04
    mastery_level: 1
    failure_type: detail
    core_question: 受控表单、mutation 成功后失效哪个 key
  "4.4":
    name: 布局路由与导航（Layout + Outlet + NavLink）
    module: 里程碑2 路由与详情
    status: mastered
    attempts: 1
    last_tested: 2026-11-17
    mastery_level: 2
    failure_type: null
    core_question: 布局路由结构、激活态导航
  "4.5":
    name: 详情页（动态路由 useParams + 单条查询）
    module: 里程碑2 路由与详情
    status: mastered
    attempts: 1
    last_tested: 2026-11-17
    mastery_level: 2
    failure_type: null
    core_question: 动态参数、详情 key 设计（列表数据可否直接用）
  "4.6":
    name: 编辑与删除（mutation + 确认交互）
    module: 里程碑2 路由与详情
    status: mastered
    attempts: 1
    last_tested: 2026-09-21
    mastery_level: 1
    failure_type: null
    core_question: PUT/DELETE 的 mutation 与失效范围
  "4.7":
    name: 筛选（类型/月份，URL query 持久化）
    module: 里程碑3 筛选与统计
    status: mastered
    attempts: 1
    last_tested: 2026-09-22
    mastery_level: 1
    failure_type: null
    core_question: useSearchParams 做可分享的筛选状态
  "4.8":
    name: 搜索防抖（自定义 Hook useDebounce 实战）
    module: 里程碑3 筛选与统计
    status: mastered
    attempts: 1
    last_tested: 2026-11-18
    mastery_level: 1
    failure_type: null
    core_question: 2.7 的 useDebounce 接入 Query 的 key
  "4.9":
    name: 统计页（派生计算 + 分类汇总）
    module: 里程碑3 筛选与统计
    status: mastered
    attempts: 1
    last_tested: 2026-11-18
    mastery_level: 1
    failure_type: null
    core_question: 派生不存储、useMemo 的正确时机
  "4.10":
    name: mock 登录（Context + localStorage）+ RequireAuth 守卫
    module: 里程碑4 登录与收尾
    status: mastered
    attempts: 1
    last_tested: 2026-11-19
    mastery_level: 1
    failure_type: null
    core_question: 登录态的 Context 化与路由保护
  "4.11":
    name: 性能收尾（路由 lazy + Suspense + Profiler 巡检）
    module: 里程碑4 登录与收尾
    status: mastered
    attempts: 1
    last_tested: 2026-11-19
    mastery_level: 1
    failure_type: null
    core_question: 3.4 优化清单落地
  "4.12":
    name: 项目收尾（README + 架构图 + 复盘）
    module: 里程碑4 登录与收尾
    status: mastered
    attempts: 1
    last_tested: 2026-11-19
    mastery_level: 1
    failure_type: null
    core_question: 完整应用的技术串联与自我表达
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
| 2026-08-24 | 2.1 | 通过 | attempts=1（一轮微调后通过）；三问预测题：快照合并（显示1）✅、alert弹0✅ 一次答对；细节错误：setUser({...user})重渲染结论对但说"Object.is判断为true"（自相矛盾，应为false=引用不同=触发渲染），追问一轮后自我纠正。当日跳过了 Vue3 到期复习（1.1-1.3，见 vue3.md） |
| 2026-08-24 | 2.2 | 通过 | attempts=2（多轮微调后通过）；B竞态修复题一次全对（ignore旗标+[id]），但主动提问暴露 ignore 变量覆盖困惑（以为新一轮函数体的 ignore=false 会被旧 cleanup 的 true 覆盖）--补讲"每轮 effect 独立闭包独立变量"后理解。A预测题：[] 场景 effect:0 对，但误认为 cleanup 会打印一次（漏"仅 deps 重跑/卸载才触发"）；[count]+3次点击+卸载的确认题首答"1条 effect:1"（概念性错误：以为 deps 只在最后一次变化时重跑），画三段式时间线后自己补全全部序列。卸载打印 cleanup:0/3（过期闭包）已埋点给 2.3 |
| 2026-08-27 | 2.3 | 通过 | attempts=2（一轮未过+逃生舱+确认题）；三问：alert旧值✅但说"0次渲染"（混淆重渲染vs effect重跑）、deps修法✅但ref修法漏、cleanup后果套错代码（deps[]不堆积是僵尸handler；deps[name]才堆积弹N个alert）。逃生舱补讲后10秒确认题（重新发请求选deps/WebSocket读最新值选ref）全对。课前完成vue3复习：1.1✅ 1.2✅ 1.3❌（见vue3.md） |
| 2026-08-27 | 2.4 | 通过 | attempts=2（重测+微调确认）；首轮两处概念性错误：①以为渲染时读不到.current新值（正确：setLog引发的重渲染会读活值，p显示3不是0，"永远显示0"前提是无任何state变化）②视频快进写成setCurrentTime函数式更新（应videoRef.current.currentTime+=10直拨DOM）。派生状态题（log.length）一次答对。重测Timer暂停题诊断对（应改useState）但预测错（界面冻结在"5s运行中"永不更新=撒谎，暂停期setSeconds被跳过无渲染）。微调确认（恢复运行后显示6s运行中）答对。核心教训已固化："渲染读活值，但只有state/props变化才触发渲染" |
| 2026-08-27 | 2.5 | 通过 | attempts=1 一次全对：两层重渲染原因（父渲染子必渲染+Context广播）、useMemo包value+[user]依赖（setUser稳定不进deps）、四场景判断（用户/主题✅，高频关键词/表单联动->状态库）。仅补精确化：广播扳机是value每次新对象Object.is判变 |
| 2026-08-27 | 2.6 | 通过 | attempts=2（首轮未过+重测全对）；首轮错误：渲染次数算错（2个Row打2次）、找错memo击穿者（以为visible新数组是prop，真凶是onRemove新函数）、useMemo大括号不return+useCallback多包一层箭头、deps误写[todos]（体内没读todos）、微妙预测与优化纪律两问答不知道。重讲"逐prop Object.is+数组换新壳元素稳定即可+deps=体内外部变量+四反对理由"后重测三问全对（Price逐prop定位/[] deps/拆memo仍渲染=三件套咬合）。同款老毛病再现：把方法/函数当独立语法乱包（见1.3/1.5） |
| 2026-09-01 | 2.7 | 通过 | attempts=3（未尝试+积木引导+修正）；useCounter预测结果对但fiber归属错（a/b不是组件，两个state都挂App的fiber，靠调用顺序分槽位--顶层调用铁律的根因）；两句辨析（非单例/非生命周期）一次全对；useDebounce首版结构对但致命bug：return clearTimeout(timer)当场执行掐死定时器（"把调用当传递"第4次，口诀：括号=现在就干，不加括号=到时候再干）+函数式更新滥用；修正后代码与防抖解释全对 | 模块2 Hooks 深入完成，阶段总结见 summaries/react_hooks.md |
| 2026-09-01 | 3.1 | 通过 | attempts=2（预测错+两轮确认）；辨析/输出序列/Vue对照全对；渲染期写ref的预测错（把JSX显示值当成++表达式的返回值，正解开发2生产1）；确认题"移进useEffect后显示几"答1（❌应为0：effect在渲染后执行+改ref不触发重渲染，Vue直觉翻车），最终确认（重渲染后显示2+活值理由）答对并主动提问StrictMode时序--补讲"双渲染是同一提交内连续两次函数调用，中间无effect" |
| 2026-09-01 | 3.2 | 通过 | attempts=2（补答通过）；RequireAuth编码逻辑全对（Navigate+state.from+replace+Login跳转行一字不差）但((children))双括号语法错（第5次手滑）；硬伤：①漏审题"未登录"三连预测首问答成Outlet换Orders（实际被守卫踢到/login）②replace理由答"更新state数据"（实为浏览器历史栈操作，登录后后退不该回/login）③"state被覆盖"表述不准（应为卸载销毁）。补答两问全对（已登录全程不出布局路由state保留/详情push+404返回replace） |
| 2026-09-03 | 3.3 | 通过 | attempts=2（补答通过）；用户主动要求追加TanStack Query零基础详解（补1-8）后考核。主体对：缓存时间线（旧数据秒出/骨架屏）、前缀匹配命中判断、并发去重结论；修正：①queryKey修法写['user','userId']字符串字面量（1.6 key="item.id"同款错误第2次）②失效流vs直写流不懂（补讲三理由：服务器真相/不拼形状/只需知什么变了）③去重理由误说"缓存数据"（首挂无缓存，是并发合并）④补问题答串一次。userId 1→2补答通过（新key新条目发请求，旧key转inactive等gc） |
| 2026-09-03 | 3.4 | 通过 | attempts=2（微调确认）；优化排序（测量→下沉→分割→虚拟化→memo）一次全对、四场景配对全对、keep-alive迁移题答对（无内置/卸载丢状态/路由外+store，补URL query最佳实践）；唯一错点：以为第二次访问仍见fallback（fallback只跟chunk在不在本地有关，已缓存则Promise立即resolve）；微调确认（发版后chunk名变）答对并顺势理解hash缓存击穿设计 | 模块3工程化完成，阶段总结见 summaries/react_engineering.md |

## 模块完成状态

- **s1 基础复盘：2026-08-17 完成**（6/6 节点 mastered，阶段总结见 summaries/react_basics.md）
- **s2 Hooks 深入：2026-09-01 完成**（7/7 节点 mastered，阶段总结见 summaries/react_hooks.md）
- **s3 工程化：2026-09-03 完成**（4/4 节点 mastered，阶段总结见 summaries/react_engineering.md）
- **s4 里程碑2 路由与详情：2026-09-21 完成**（4.4-4.6，项目模式无独立阶段总结，见 summaries/expense-tracker_m2-routing.md）
- **s4 里程碑3 筛选与统计：2026-11-18 完成**（4.7-4.9，见 summaries/expense-tracker_m3-filter-stats.md）
- **s4 实战项目（记账本）：2026-11-19 完成**（里程碑4 4.10-4.12，见 summaries/expense-tracker_m4-login-windup.md；全项目 12/12 节点 mastered）

## 实战项目进度（s4 记账本，node 4.x 见上方 frontmatter）

| 日期 | 节点 | 结果 | 备注 |
|------|------|------|------|
| 2026-09-04 | 4.1 | 通过 | 选型：记账本 + json-server + TanStack Query + axios（未用 fetch 参考版，自主选 axios）+ Context。Vite react-ts 搭好，@ 路径别名 vite+tsconfig 双配。依赖坑：scss 装错包（实为 sass，已换）。json-server v1 beta 行为：--watch 已废弃默认监听、id 返回 string。端口 3001 被占主动改 3002。目录：api/{client,records} + components + pages，组件用 index.tsx+module.scss 文件夹式 |
| 2026-09-04 | 4.2 | 通过 | axios 实例 + CRUD 全 API + RecordItem 类型（主动改名避开内置 Record 泛型冲突，采纳上节建议）。RecordsPage：useQuery ['records']、isLoading/isError + invalidate 重试、key={record.id}（非 index）、income/expense 分色。client.ts 的 request interceptor 预留 4.10 token 位；response interceptor 现为纯透传 |
| 2026-09-04 | 4.3 | 通过 | attempts=3（参考实现+自己改对）。手写版审查揪出：①发明 id（Math.random 传 POST，应 Omit 掉交给 json-server 生成）②reset 时序错（mutate 后同步读 isSuccess 恒 false；且 form.reset 重置 DOM 与受控 state 脱节，应 onSuccess 里 setFormData 初值）③无初始值→受控/非受控切换 warning+type 字段丢失④useRef(null) 缺泛型⑤(e)=>implicit any×2。参考版引入：NewRecord=Omit<RecordItem,'id'>、FormState 草稿类型（amount 表单期 string 提交时 Number()）、EMPTY_FORM 放组件外、FormEvent 类型。自己改回后 tsc 0 错误（额外学：React.ChangeEvent 泛型、SubmitEvent） |
| 2026-09-07 | 4.4 | 开始 | 恢复时跳过全部到期复习（用户选择：TS/Vue/React基础/4.1-4.3 均顺延）。当前：App.tsx 直渲 RecordsPage、main.tsx 仅 QueryClientProvider，react-router-dom v7 未接线。目标：Layout+Outlet+NavLink 布局路由，记一笔拆独立页 |
| 2026-09-08 | 4.4 | 通过 | 用户自主引入 antd-mobile 重构列表/表单（List/ErrorBlock/SpinLoading/Form/DatePicker/Toast）。代码审查+修 2 轮：①TS6133（onClick e 未用）②真 bug：navigate 紧跟 mutate 立即跳转→POST 失败也被踢回列表且无 onError；改放 onSuccess + 加 onError Toast（成功才跳）。自查纠偏：DatePicker trigger='onConfirm'+onClick open 为标准示例；ErrorBlock 会渲染 children（重试按钮有效）；antd 样式自动按需引入（sideEffects 声明）；表单空 scss 无引用。验收 5 条行为通过，tsc 0 错误。考核 3 问用户请求直接给答案（逃生舱，未自答），**下次复习重点测：①布局复用=父路由 element 不卸载只换 Outlet 子组件 ②NavLink 激活=段边界匹配+end，to=/ 天然仅精确命中（源码 chunk:491） ③invalidate 挂 queryClient 因缓存住 queryClient 组件会死缓存不会**。遗留（4.5 用）：详情页 /records/:id 下让"账目"高亮需段根 /records |
| 2026-09-08 | 4.5 | 开始 | 启动时再次跳过全部到期复习（TS 最早批 08-17 到期、React 1.x-4.3 全过期、4.4 逃生舱 3 问未自测——继续顺延，欠账滚动累积）。4.5 目标：useParams 动态路由 + 单条查询 key 设计 + 列表点击进详情 |
| 2026-09-10 | 4.5 | 通过 | 选了方案A（独立请求，B initialData 后补）。方案讲解环节即埋考点。代码审查修 2 轮：🔴①queryFn: getRecordById(params.id||'')把调用当传递（**第5次**，且是全部TS连环错的震中）→ queryFn: ()=>getRecordById(params.id!)；🔴②onClick={()=>handleBack}箭头返回函数不执行（口诀两面性补全：括号=现在就干/不加括号=交出去等人调）。追问"No routes matched /record/"→补讲**路由匹配在组件之前**的流水线（/record/ 匹配不到 :id 需空段 → Routes渲染null白屏；组件内守卫没机会跑）→ 加 catch-all `path="*"` NotFoundPage（放"/" children最后保导航栏）。if(!params.id) 可达性讨论：当前路由配置下是纯防御（路由保证id存在），临时加无参路由/可选段 :id?/单测 MemoryRouter 三种可达路径。验收 5/5 过。考核：Q1层级key前缀失效不会（讲解：key=缓存目录树/前缀=目录，4.6 编辑需同时失效 ['records'] 和 ['record',id] 两棵树）；Q2两层兜底归因全对（路由层 vs 数据层）；Q3接近（误说默认30s、漏stale-while-revalidate——补：staleTime管新不新鲜gcTime管在不在，过期≠删除，先渲旧缓存后台刷新）。**复习重点：Q1前缀失效 + staleTime/gcTime 区分**。遗留：NotFoundPage 还是裸文案（待升级 ErrorBlock+回首页） |
| 2026-09-10 | 4.6 | 开始 | 里程碑2 收官：PUT/DELETE mutation + 失效两棵 key 树 + Dialog.confirm 确认 + 异步数据预填表单的坑（async initialValues） |
| 2026-09-21 | 4.6 | 通过 | 自主重构：抽共享组件 AddOrEditForm（record?: RecordItem|null + isEdit 分支 + recordToFormState 转换器 String(amount)/toDate()），Add/Edit 两页复用。课前答 Q3（编辑失效两棵树/删除不管详情树）Q4（replace 语义）全对。审查修 3 红：①invalidate({queryKey:['records','record']})想一次失效两棵树把 key 并成数组——**前缀语义没吃透（Q1 考过仍错）**，正解两句话各失效一棵；②删除的 invalidate(['records']) 被注释残留；③async initialValues 坑**被详情缓存意外掩护**（从详情进编辑 30s 缓存新鲜→data 首渲染同步可用→侥幸过），硬刷新即露馅——修法 `if(!data) return SpinLoading; return <AddOrEditForm/>`（顺带教学：return 后裸 {} 是对象字面量不是 JSX 占位，TS1005）。小项：Form.Header 动态文案/id 单点添加/as 断言消除/死 import。验收 7/7 过（含硬刷新编辑页补撞坑）。里程碑2 路由与详情完成，总结见 summaries/expense-tracker_m2-routing.md |
| 2026-09-21 | 4.7 | 开始 | 里程碑3 第一节点：服务端筛选（已确认 json-server v1 beta.15 支持 ?type= 与 ?date_gte/_lte，日期字符串字典序可比）。新增概念：useSearchParams、queryKey 含筛选参数（衔接 3.3 userId 换 key）、空态 vs 错误态 |
| 2026-09-22 | 4.7 | 通过 | 课前答 Q1（setSearchParams 整体替换→函数式展开 prev）Q2（invalidate(['records']) 前缀命中所有筛选组合）全对——4.6 前缀坑这次用对了。亮点：自创 date_contains=YYYY-MM 匹配整月（优于 gte/lte 区间方案）、方言转换留 api 层、isSuccess 区分空态/错误态。审查修 3 轮：🟠①selectDate useState 与 URL 双状态源（违反"URL 是唯一真相源"，硬刷新按钮文案与数据脱节）→删 state 从 URL 派生；🟠②resetDate 直接 searchParams.delete() 改在只读快照上不生效（读写分离：searchParams 只读，写必须走 setSearchParams）；🟡③FilterRender 组件定义在组件内部→每次渲染重挂子树→改 JSX 变量消除组件边界。Empty 提取/图标语义等杂项。验收 7/7 过（含刷新保筛选、✕ 清除月份）。遗留：行内注释 icons 块再留一轮（下轮必删）、FilterRender 命名建议 filterBar |
| 2026-11-17 | 4.4/4.5 复习 | 通过 | 用户选测标记重点（非最早批 1.x，其余欠账继续顺延）。Q2 NavLink 段边界匹配 3/3 一次对（/records 高亮、/ 不高亮、end 后不高亮）；Q3 失效两棵树+前缀原理+staleTime/gcTime 一次对；Q1 布局复用首轮只答现象（"只加载对应组件"未触及机制），提示"Layout 还在吗？被换掉的是哪部分"后答出"Layout 保持挂载、Outlet 子组件替换"通过。4.4 逃生舱 3 问与 4.5 标记重点全部清账 |
| 2026-11-17 | 4.8 | 开始 | 目标：useDebounce（2.7 重写）接入 Query key，RecordsPage 搜索框 + 防抖 300ms + ?q= 全文搜索 |
| 2026-11-18 | 4.8 | 通过 | 讲解时间线机制后自写一次通过 useDebounce 核心（未踩"调用当传递"坑）；自发重构 queryKey 工厂（官方写法，加分）。审查修 3 轮：①useDebounce 写死 string→泛型 ②handleKeywordChange 死代码 ③invalidateRecordLists→useInvalidateRecordLists（Hook 命名约定）。实战 bug：搜索触发后 SearchBar 文字清空——根因 isLoading 早退卸载非受控 SearchBar（内部 state 随实例销毁，rawKeyword 未丢），用户自选受控化 value={rawKeyword} 修复（合法）+ 补 placeholderData 借旧数据防闪。🔴顺带纠正：onClick 里调 Hook（改名前就存在的违规）→hook 返回回调顶层调用。思考题（防抖值进 key 的原因）通过。验收 5/5 过。蒸馏见 distilled/topic32-react-search-debounce.md |
| 2026-11-18 | 4.9 | 通过 | "派生不存储"一次写对（无第二份 state，useMemo 一次产出三数+分类汇总，deps=[records] 用对——2.6 误写 deps 未复发）。reduce 累收/Map 分组/降序全对。审查修 2 处：🟡key={index}（1.4 老坑回潮）→key={item.category}；🟡.map 做副作用→forEach（顺带：冗余类型注解）。口头答"为什么不用写失效"✅（同 lists() 前缀）；useMemo 必要性追问两轮：首轮只答成本角度，引导后补出"消费者不在乎引用→用途2不成立→可摘"。验收全过。里程碑3 筛选与统计完成，总结见 summaries/expense-tracker_m3-filter-stats.md，蒸馏见 distilled/topic33-react-stats-derived.md |
| 2026-11-19 | 4.10 | 通过 | auth.tsx 按骨架一次写对（惰性初始化/双写/useAuth 判空 throw）；但首版绕过 Context 走纯 localStorage 路线（api/user.ts + 手动 setItem + RequireAuth 直读 localStorage）——auth.tsx 成死代码，主动提出"连接不上怎么使用"。教学两步接线法（main.tsx 包 Provider + useAuth 取电）后改回 Context 路线，接线全对。审查修：🔴FormState 字面量类型 'string'→string；🟡登录成功丢 state.from→location.state?.from ?? '/'；补 loading 态。用户自问 location.state 哪来的→顺势教学：state 是跳转方 Navigate 塞的随行包裹（RequireAuth 漏塞——3.2 考过仍漏，回炉），不进 URL、刷新即丢。验收 5/5 过。蒸馏见 distilled/topic34-react-auth-context.md |
| 2026-11-19 | 4.11 | 通过 | 先测量后优化落地：术前 637.73kB 单包（gzip 208.68）+500kB 警告 → 术后 299.54kB（gzip 97.12）↓53%、~17 chunk、警告消失。6 页面 lazy + Suspense 包 Layout/Outlet 外（自选进阶方案，导航不闪），LoginPage 保持 static 与之自洽。build 输出教学：rolldown 自动共享提取（AddOrEditForm 4.62kB 公共 chunk、壳页面 0.14kB、date-picker 52kB 独立分片）。思考题（lazy 粒度）通过：用户答 UX 角度，补请求瀑布+缓存碎片化，判断法"只在用户自然等待的边界 lazy"。直访 /stats 拉多小文件判"正常"✅（按需集合 vs 全量单文件）。顺带收掉 m2 遗留 NotFoundPage（ErrorBlock+回首页）。验收过、tsc 0 错。蒸馏见 distilled/topic35-react-lazy-suspense.md |
| 2026-11-19 | 4.12 | 通过 | 项目答辩模式（用户输出+LLM审查）。README：用户填写功能/选型/运行/结构职责（质量不错），LLM 修正 4 处（补 npm install、技术栈逐项给理由、补 pages/ 分层规则、功能介绍去围栏）+ 代写技术亮点三篇（逃生舱：用户"不会表述"）。变式验证：用户口头复述数据失效 story——结构完整术语全对 ✅，但坑被泛化成"写法不一致"（真实坑是 4.6 并数组失效），已教学"复盘讲事故，事故要真"。里程碑4 登录与收尾完成，s4 实战项目收官（12/12）。总结见 summaries/expense-tracker_m4-login-windup.md，蒸馏见 distilled/topic36-react-project-windup.md |
