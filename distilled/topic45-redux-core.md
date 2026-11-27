# 4.1 redux 核心：store/action/reducer，为什么 reducer 必须纯

**它解决什么问题**
状态可被随处改动 = 变更不可追溯。Redux 用"谁都别改，交申请单"的纪律，换可预测——模板代码是纪律的成本。

**核心机制**
- 三件套：action（纯数据描述"发生了什么"）→ reducer（纯函数，旧state+action→新state）→ store（getState/dispatch/subscribe）
- 三原则：单一数据源 / state 只读（唯一写入口 dispatch）/ 纯 reducer
- 单向数据流铁环：view → dispatch → reducer → 新 state → 通知 → view，方向永不变
- 纯的三重回报：可测试（无 mock）、可重放（action 序列走到任何时刻）、可追溯（变更集中一处）
- 不可变根因：React 引用比较，变化必须体现在新对象上

**最容易被追问的点**
- "reducer 里为什么不能发请求？" → 副作用破坏可测试 + 异步时序破坏可重放；副作用推到中间件（thunk/saga）
- "payload 该放什么？" → 服务 action 语义的形状：INCREMENT 就放增量数字或无 payload，别放设值对象
- "default 分支干什么？" → 原样返回 state——且必须有 return（漏了返回 undefined 全局崩）
- "为什么 dispatch 前先 subscribe？" → 后订阅收不到已发生的变化
- "模板代码值得吗？" → 小项目纯负担（Zustand 完胜），大团队是保险——RTK = 保留纯度消灭模板

**一句话类比**
Redux 是财务制度：谁也不能直接涂改账本（state 只读），只能提交报销单（action）；会计（reducer）只按单子记账、记完出**新账页**不涂旧页；老账页永远可查——这就是时间旅行，而会计私改账本（副作用）是开除级事故。
