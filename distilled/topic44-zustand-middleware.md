# 3.2 Zustand 中间件与爆火原因

**它解决什么问题**
store 有了但要持久化/时间旅行/可变写法——中间件让"改 state 这条路"可以插层加工，而 API 依然一页纸。

**核心机制**
- 中间件 = 包装 setup 的高阶函数，洋葱模型：外层先拦 set
- 三件套：persist（订阅 set 写 storage+初始化读回）、devtools（接 Redux DevTools）、immer（Proxy 记录改动产出不可变新对象）
- 柯里化承接类型：`create<T>()(persist(devtools(setup, devOpts), persistOpts))`——**谁的 options 跟在谁的 setup 后**
- persist 参数：name（key）/ partialize（只存部分字段，对照 Pinia pick）/ storage

**最容易被追问的点**
- "partialize 写在哪？" → persist 的 options 里（持久化概念找 persist）；挂到 devtools 上等于裸奔
- "Zustand 能替代 TanStack Query 吗？" → 不能——客户端状态 vs 服务端状态；接口数据要缓存/失效/去重/后台刷新，Zustand 一个都没有
- "为什么爆火？" → 零仪式（无 Provider 无模板）/ selector 粒度可控 / 模块级单例组件外可用（getState 脱离 React 树）
- "什么时候 RTK 反而更合适？" → 大团队强规范 / 完整时间旅行生态 / RTK Query 一体化——防 religious war 收尾句式
- "服务端数据要不要 persist？" → 不要，数据主权在服务端，存本地只会造成陈旧镜像

**一句话类比**
中间件是净水器串联滤芯：水流（set）先过外层滤芯（devtools 记账）再到内层（persist 存样），每层只干自己的活；装错了位置（partialize 挂 devtools）等于滤芯空转。
