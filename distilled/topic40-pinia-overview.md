# 1.1 Pinia 校准 + 为什么 Vue 生态不需要 reducer

**它解决什么问题**
React 同事抱怨"改个数字要 dispatch+reducer"——不是 React 有毛病，是两个生态响应式能力不同导致的必然设计。

**核心机制**
- 三问框架贯穿所有库：状态放哪 / 谁改 / 怎么通知
- Pinia 三问答案：defineStore 单例 / 谁都能改（action+直接改） / proxy 自动追踪
- Vue3 响应式免费提供精确追踪：读时收集依赖，写时 set 拦截精确通知
- React 没有依赖收集，订阅只能靠引用比较（Object.is）→ 可变数据下失效 → 必须不可变新对象
- Pinia vs Vuex：mutation 之死 = 时间旅行换 state 快照 diff；扁平化 + TS 完整推导

**最容易被追问的点**
- "Pinia 为什么不要 mutation？" → proxy 追踪已保证精确更新；devtools 改用 state 快照
- "直接改 store 合法吗？" → 合法且是流派特色；action 只是工程约定
- "storeToRefs 为什么必须？" → 直接解构丢响应性；action 是函数不能进 storeToRefs
- "React 里想直接改用哪个库？" → Mobx——靠 proxy 拦截 + 依赖收集 + observer 精确订阅（Vue3 同款哲学）
- "persist 插件本质是什么？" → 订阅 $subscribe 写 localStorage + 初始化读回，只是通知链上的副作用钩子

**一句话类比**
响应式流派是"智能家居"——改了什么传感器自动知道（proxy 追踪）；单向流是"人工巡检"——必须换新报告（不可变新对象），巡检员（useSelector）才能看出"东西变了"。
