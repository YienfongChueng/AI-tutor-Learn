# 2.1 Mobx 心智模型 + API 与完整接线

**它解决什么问题**
React 组件默认盲目重渲染——Mobx 把 Vue3 那套"proxy 拦截 + 依赖收集"外挂给 React，实现字段级精确订阅。

**核心机制**
- observable 包装数据，读时收集依赖、写时精确通知（Vue3 reactive 同款哲学）
- `makeAutoObservable(this)`：一行启动——字段标 observable、getter 标 computed、方法标 action（**没有它整个 store 是死的**）
- observer(Comp)：把 render 包成 reaction，渲染即订阅；打破"父渲染子必渲染"
- 写入纪律：enforceActions 下非 action 改 observable 直接 throw；异步回调改 state 必须包 `runInAction`
- 三角：action（写入）→ state（可变）→ computed（派生缓存）→ views

**最容易被追问的点**
- "computed 链上游变了会通知吗？" → computed 重算后**值相等不发通知**；订阅 number 派生值稳，订阅 filter 新数组（引用变化）会渲染——读什么派生值是性能决策
- "为什么异步里要 runInAction？" → await 后脱离 action 作用域，直接改会被 enforceActions 拦；runInAction 是报备条
- "不更新排查第一步？" → 查 makeAutoObservable 存不存在；再查组件包没包 observer
- "发给 API 为什么要 toJS？" → observable 是 Proxy 壳，序列化/结构克隆会出问题，toJS 摘壳
- "和 Redux 的根本区别？" → 活体追踪（可变+自动）vs 事件账本（不可变+手动）

**一句话类比**
Mobx 给 React 装了 Vue 的神经系统：observable 是神经末梢（拦截），依赖收集是建突触，observer 组件是只连了相关神经的反射弧——不是全身抽搐（父渲染子），而是精确反射。
