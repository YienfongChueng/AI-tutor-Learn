# 响应式数据：Proxy + reactive 创建 + get/set handler

**它解决什么问题**
Vue2 defineProperty 检测不到属性增删、数组受限、初始化全量递归；Vue3 用 Proxy 整体代理拦截 13 种操作。

**核心机制**
- Proxy 包整个对象，get 拦读取、set 拦赋值
- createReactiveObject 五步：isObject→IS_REACTIVE→缓存命中→targetType非INVALID→new Proxy
- WeakMap 缓存原始对象→代理，同对象不重复包
- get trap：Reflect.get + track 收集依赖
- set trap：Reflect.set + 判断 ADD/SET + hasChanged 守卫 + trigger
- target===toRaw(receiver) 仅在属主上 trigger 一次

**最容易被追问的点**
- "reactive 会递归代理所有子对象？" -> 不会，只包一层，访问时才惰性代理
- "target===toRaw(receiver) 防什么？" -> 防原型链 set 重复触发（不是防不代理对象）
- "多层 a.b.c 几次 get trap？" -> 3 次，每次一次 track（c 原始值终止递归不代理）
- "createReactiveObject 有 isRef？" -> 没有，isRef 属于 ref.ts

**一句话类比**
Proxy 是办公楼总门禁（只装入口），非逐房间传感器；门禁登记谁进过哪房间=track，变更广播=trigger。
