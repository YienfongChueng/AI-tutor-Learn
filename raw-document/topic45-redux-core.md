# 4.1 redux 核心：store/action/reducer，为什么 reducer 必须纯

## 讲解原文

### 一、三问的 Redux 答案

| 三问 | Redux 的答案 |
|------|-------------|
| 状态放哪？ | **单一数据源**——全应用只有一个 store，树状 state |
| 谁改？ | **没人能直接改**——只能 `dispatch(action)` 描述"发生了什么"，reducer 纯函数算出新 state |
| 怎么通知？ | `store.subscribe()` 订阅（react-redux 封装成 useSelector，4.2 讲） |

一句话：**Zustand 回答"谁都能改，改就完了"；Redux 回答"谁都别改，提交申请单"**——严得多，换来的是可预测。

### 二、核心三件套（原版 Redux——理解哲学必须脱掉 RTK 糖衣）

```js
import { createStore } from 'redux'   // RTK 时代用 configureStore，这里看原教旨版

// ① action：描述"发生了什么"的普通对象，type 是唯一身份
const ADD_RECORD = 'ADD_RECORD'
const DELETE_RECORD = 'DELETE_RECORD'
// { type: ADD_RECORD, payload: {...} }  ← action 长这样，纯数据，无逻辑

// ② reducer：(旧state, action) => 新state 的【纯函数】
function recordsReducer(state = [], action) {
  switch (action.type) {
    case ADD_RECORD:
      return [...state, action.payload]        // 不可变：新数组，不 push
    case DELETE_RECORD:
      return state.filter((r) => r.id !== action.payload)  // filter 产生新数组
    default:
      return state                             // 认不出的 action 原样返回
  }
}

// ③ store：三方法
const store = createStore(recordsReducer)

store.getState()                    // 读
store.dispatch({ type: ADD_RECORD, payload: { id: '1', amount: 25 } })  // 唯一写入口
store.subscribe(() => console.log(store.getState()))    // 订阅变化
```

**单向数据流铁环**（数据只能顺这一个方向流动）：

```
view（事件）
   │ dispatch(action)
   ▼
reducer(旧state, action) ──纯计算──▶ 新 state
   │
   ▼
store 通知所有订阅者 ──▶ view 重渲染
   └────── 循环，方向永不变 ──────┘
```

**三大原则**（面试背得出）：单一数据源 / state 只读（唯一改变方式是 dispatch）/ 变更由纯函数 reducer 完成。

### 三、本节核心：为什么 reducer 必须纯（面试最高频）

**纯函数 = 同样输入永远同样输出，且不碰外部世界**（不改参数、不发请求、不用 Math.random/Date.now、不跳路由）。

Redux 拿"纯"换了**三样东西**——这是"忍受模板代码"的真正回报：

1. **可测试**：`recordsReducer(旧state, action)` 不依赖任何外部环境 → 测试不用 mock 一切，喂两个参数断言返回值即可
2. **可重放（时间旅行）**：devtools 只记录 **action 序列**就能完整重现任意时刻的 state——因为 `f(s0, a1) = s1`，从初始 state 逐个 dispatch 历史 action 就能走到任何一页。**reducer 不纯，重放就失真**（Math.random 每次不同）
3. **可追溯调试**："这个 state 为什么变成这样" → 看/action 日志流即可，所有变更集中在一处（reducer），不散落在组件里——对比 Zustand/Mobx 的"改就完了"（自由但变更分散）

**不可变的根因**（1.1 已学，Redux 场景复述）：React 判断"变没变"只能靠引用比较，reducer 必须返回新对象，变化才体现在引用上。

**reducer 禁忌清单**（写 reducer 时自查）：

```
✗ state.push(...) / state.xxx = ...      改了参数（要返回新对象）
✗ axios.get(...) / setTimeout            副作用（推到中间件/外层，4.4 讲 thunk）
✗ Date.now() / Math.random()             非确定性（破坏重放）
✗ 依赖全局变量                            隐式输入（破坏"同样输入同样输出"）
```

### 四、财务制度类比（结构对应）

| 类比 | 对应物 |
|------|--------|
| 账本 | store state |
| 报销单（描述"发生了什么"，不含怎么改） | action |
| 会计：只按单子记账，不问单子哪来，记完出**新账页**不涂改旧页 | reducer 纯函数 + 不可变 |
| 老账页永远可查（审计追溯） | 时间旅行 |
| 会计下班后私改账本 | reducer 里的副作用 = 开除级事故 |

### 五、为什么忍受模板代码（引 4.3）

模板代码（actionType 常量、actionCreator、reducer switch）是**纪律的成本**。小项目是纯负担（Zustand 完胜），大团队是保险：变更集中、可追溯、新人上手看目录就知道"改 state 的路只有一条"。**RTK 的历史使命 = 保留纯度和纪律，消灭模板**（4.3 讲它怎么做到）。

### 六、一句话总结

- 三件套：action（纯数据描述）→ reducer（纯函数算新 state）→ store（getState/dispatch/subscribe）
- 三原则：单一数据源 / state 只读 / 纯 reducer
- **纯的三重回报**：可测试、可重放、可追溯——reducer 禁忌 = 改参数/副作用/非确定性/隐式输入
- 不可变的根因是 React 引用比较（1.1 的推论在 Redux 的落点）

## 考核过程

### 题目 1：原版 Redux counterReducer（用户回答的优化版）

> 优化点对照原回答：①补 TS 类型（state/action 都有形状，reducer 签名可测试性更好）②新增 actionCreator（action 语义集中在工厂函数，dispatch 处不再手写对象字面量——4.3 RTK 的 createSlice 就是把这个自动化）③initialState 提取为具名常量（默认参数引用它，reset 与初始一致不靠魔法数字）④**先订阅后 dispatch**（订阅晚了收不到变更日志）⑤default 返回原引用 `state` 而非 `{...state}`（无变化的 action 不该制造新引用，避免无谓重渲染）⑥补算三连结果。

```ts
import { createStore } from 'redux'

// ── 类型 ──
type CounterState = { count: number }
type CounterAction =
  | { type: 'INCREMENT' }                       // 无 payload：语义约定加 1
  | { type: 'DECREMENT' }
  | { type: 'RESET' }

// ── actionCreator：语义集中在工厂里，调用处不手写字面量 ──
const increment = (): CounterAction => ({ type: 'INCREMENT' })
const decrement = (): CounterAction => ({ type: 'DECREMENT' })
const reset     = (): CounterAction => ({ type: 'RESET' })

// ── reducer：纯函数，旧 state + action → 新 state ──
const initialState: CounterState = { count: 0 }

export function counterReducer(
  state: CounterState = initialState,   // ES6 默认参数 = 初次挂载的种子
  action: CounterAction,
): CounterState {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 }        // 不可变：新对象
    case 'DECREMENT':
      return { ...state, count: state.count - 1 }
    case 'RESET':
      return { ...state, count: initialState.count }     // 复用初始值，不写魔法数字
    default:
      return state   // ⚠️ 原样返回【同一个引用】：未知 action = 没有变化 = 不触发订阅者
  }
}

// ── store：先订阅，后 dispatch ──
const store = createStore(counterReducer)

store.subscribe(() => console.log('state 变了 →', store.getState()))

store.dispatch(increment())   // 0 → 1
store.dispatch(increment())   // 1 → 2
store.dispatch(decrement())   // 2 → 1
// dispatch 三连后：store.getState() === { count: 1 }
```

### 题目 2：reducer 里发 axios 的情景分析（用户原回答，一次通过）

  case 'FETCH_RECORDS':
    axios.get('/records').then((res) => {
      state.records = res.data    // 他觉得"异步拿数据再改 state 很自然"
    })
    return state

  指出这段代码至少 3 个问题，每个问题对应"纯函数禁忌"里的哪一条、破坏了 Redux 的哪样回报（可测试/可重放/可追溯），并说出正确的做法方向（具体中间件4.4 才讲，说方向即可）。

  - 问题1、reducer不纯，依赖了外部环境，不能在reducer里进行接口请求，破坏了可测试的回报；应该接口请求在各自组件里实现，数据进行dispatch去修改
  - 问题2、直接改state，破坏了可追溯调试的回报，变更数据散落在各个组件里，追溯变难，应该变更数据要集中一处，使用dispatch一个reducer去集中修改数据
  - 问题3、返回原来的state，引用地址不变，无法监测到数据变化，应该返回新的state
  - 问题4、axios 是异步的——dispatch 早就 return 了、数据还没回来；就算 .then里改了，重放时网络时序无法复现，破坏的是第三样回报：可重放


## 考核过程

### 编码 + 情景混合题（两轮）

**首版**：任务 2（reducer 里发 axios 的 3 问分析）一次通过——副作用/改参数/返回原 state 对应禁忌与回报，补讲"异步时序破坏可重放"凑齐三回报；任务 1 三处错误：🔴①INCREMENT 的 payload 是 {count:2} 对象与语义错位（number+object=NaN）②default 分支缺 return（2.6 useMemo 不 return 同款第 2 次回潮）③题目要求算三连结果未答。

**点评**：未通过，任务 2 通过、任务 1 按组点评修订。

**修订版**：payload 语义修正（无 payload 约定加/减 1）✅、default 补 return ✅；三连结果仍漏（第 2 次）→ LLM 给出 `getState() === {count: 1}`（0→1→2→1）。顺带时序提醒：subscribe 应放在 dispatch 之前（先订阅后发车）。

**点评**：通过（close，微调确认）。复习重点：action 语义与 payload 形状一致、default 必须有 return、先订阅后 dispatch、三回报与异步时序。

---
通过时间：2026-11-26
