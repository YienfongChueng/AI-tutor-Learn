# 3.1 Zustand：create 与 store 即 hook + selector 粒度

**它解决什么问题**
Redux 的仪式感 + Mobx 的类体系都嫌重——"一个函数造 store，store 本身就是 hook"，全部 API 一页纸。

**核心机制**
- `create<T>()(setup)`：柯里化锁型——中间件在里、类型在外（加 devtools/persist 时类型不丢）
- set 两个规则：对象式**浅合并**（第二参 true 才整体替换）；函数式 `set(s => 增量对象)` 拿旧值算新值
- selector = **手动定订阅粒度**：state 变 → 重跑 selector → Object.is 比较返回值
- 底层：模块级闭包 store + React 18 `useSyncExternalStore`——store 不住 React 树，无需 Provider 的根因

**最容易被追问的点**
- "selector 返回新对象会怎样？" → 引用恒变 → 全量重渲染，selector 白写；修法：拆 selector / useShallow 浅比较
- "set({x:1}) 是替换吗？" → 浅合并，只改提到的键；整体替换要 set(obj, true)
- "为什么不需要 Provider？" → store 住模块作用域（闭包），hook 用 useSyncExternalStore 从外面探进来
- "组件外能用吗？" → useStore.getState()/setState() 就是普通对象方法
- "set 的 updater 返回什么？" → 部分 state 对象（增量），返回数组会把整个 state 换成数组

**一句话类比**
Redux 是全楼广播人人自己筛，Mobx/Pinia 是智能门铃自动响，Zustand 是自己订阅公众号——selector 就是选关注，选错关注（返回新对象）天天被推垃圾；useShallow 是把推送规则从"只要发了就算"改成"内容真变了才推"。
