# 2.3 useEffect 闭包陷阱与 cleanup

**它解决什么问题**
effect 闭包捕获的是"它那一轮渲染"的快照，活得比那轮久就读到旧值--Vue 活引用天然无此病，React 死快照是迁移期最危险的常态。

**核心机制**
- 每轮渲染生成独立 effect 闭包，捕获当轮 state/props
- 只在异步/持久副作用发作：定时器、监听器、fetch 回调
- deps 变化 -> 先跑 cleanup 销毁旧 effect，再建新闭包
- ref 盒子：闭包捕获盒子对象（引用恒定），执行时取 .current 活值
- cleanup 是本轮 effect 的遗嘱：重跑前/卸载时替它还资源

**最容易被追问的点**
- "输入后明明重渲染了，effect 为啥不重跑？" -> deps [] 未变；重渲染≠effect 执行
- "只想读最新值但监听器不想重建，用啥？" -> useRef 活引用，不是加 deps
- "deps [name] 忘写 cleanup 会怎样？" -> 监听器堆积，一次事件触发 N 个 handler、弹 N 个各含历史值的 alert
- "deps [] 忘写 cleanup 会堆积吗？" -> 不会（effect 只跑一次），是卸载后僵尸 handler 照常执行
- "deps 和 ref 怎么选？" -> 行为本身依赖该值（重新请求/重连）用 deps；仅要用最新值（展示）用 ref

**一句话类比**
闭包像给快递员看的收件人照片：deps 是每次变化重拍一张并撕掉旧照（cleanup 撕），ref 是干脆给快递员一面实时监控屏。
