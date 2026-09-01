# 2.6 useMemo/useCallback 与重渲染优化

**它解决什么问题**
父组件一重渲染全子树陪跑（重渲染三条路之②），React.memo 想拦却被"每次渲染都新鲜的引用"击穿；需要一套咬合的关卡让无关子树跳过渲染。

**核心机制**
- 重渲染三条路：自身 setState / 父渲染 / Context 广播
- memo 逐 prop 做 Object.is 浅比较，全没变才跳过
- useMemo 稳定值、useCallback 稳定函数（= useMemo(()=>fn) 的糖）
- deps 判定：函数体内用到的外部变量；setState 引用恒定不进
- 三件套咬合：没有 memo，稳定引用毫无意义

**最容易被追问的点**
- "父组件的新数组会导致 memo 子组件重渲染吗？" -> 不会，子组件收的是数组元素，元素引用稳定就过关；击穿者是新函数/新对象直接当 prop
- "useCallback 的 deps 为什么常是 []？" -> 体内只用了恒定的 setState + 函数式更新的参数
- "全组件包 memo 保险吗？" -> 反模式：比较开销/藏旧闭包/噪音/先测量先状态下沉
- "和 computed 什么区别？" -> computed 依赖追踪强缓存一等公民；useMemo 是性能提示，默认不写

**一句话类比**
memo 是小区门禁（逐个查证件），useCallback/useMemo 是给常客办长期通行证--门禁都没装，办证纯属浪费。
