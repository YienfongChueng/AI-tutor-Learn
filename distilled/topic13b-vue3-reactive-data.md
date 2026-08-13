# 响应式数据：ref + 惰性递归 + 变体

**它解决什么问题**
原始值不能被 Proxy（JS 规范）；深层对象要响应式但不能初始化全量递归；需只读/浅层变体控制粒度。

**核心机制**
- ref 用 RefImpl 类包原始值，.value 访问器走 track/trigger（不走 Proxy）
- _value 存响应式值，_rawValue 存原始值供 hasChanged 比较
- 对象值 _value=toReactive(value) 复用 reactive
- 惰性递归：get trap 里 isObject(res)?reactive(res) 才包子对象
- 四变体由 isReadonly×isShallow 参数化 createGetter/createSetter

**最容易被追问的点**
- "ref 是 reactive({value:x}) 语法糖？" -> 不是，RefImpl 不走 Proxy 更轻量，__v_isRef 支持 isRef+模板解包
- "readonly(reactive(o)) 为何能更新？" -> readonly 不 track，依赖收集在底层 reactive 的 get
- "shallowReactive 深层修改触发？" -> 不触发，shallow 直接返回原始对象不递归
- "WeakMap 还是 Map 缓存？" -> WeakMap，弱引用防原始对象无法 GC 致内存泄漏

**一句话类比**
原始值是散装糖果没法贴门禁，装进糖果盒（RefImpl）开 .value 窗口；大楼只装入口门禁，推哪层门才给那层装（惰性递归）。
