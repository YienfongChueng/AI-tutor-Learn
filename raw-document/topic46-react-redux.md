# 4.2 react-redux：Provider + useSelector/useDispatch vs 旧 connect

## 讲解原文

### 一、react-redux 解决什么问题

4.1 的原版 Redux 有两个悬而未决的问题：

1. store 是个独立对象，**React 组件怎么拿到它？** → Provider（Context 注入，你 4.10/3.2 熟悉的接线）
2. state 变了之后，**谁重渲染、重渲染到什么粒度？** → useSelector（订阅 + 比较，Zustand selector 的同款机制）

react-redux = **把 Redux 接上 React 的神经**，只干这两件事。

### 二、Provider（接线，复用你的 Provider 模式）

```tsx
import { Provider } from 'react-redux'
import { configureStore } from '@reduxjs/toolkit'   // RTK 的 createStore，4.3 正式讲

const store = configureStore({ reducer: { counter: counterReducer } })

// main.tsx——和你 4.10 的 AuthProvider / 3.2 的 StoreProvider 同款三层三明治
<Provider store={store}>
  <App />
</Provider>
```

注意 RTK 的 `configureStore({ reducer: {...} })` 把多 reducer 按 key **合并成树状 state**（`s.counter.count`）——单一数据源表现为一棵树，每个 reducer 守一棵子树。

### 三、useSelector——订阅 + 粒度（本节核心）

```tsx
import { useSelector, useDispatch } from 'react-redux'
import { RootState } from '@/store'

function Counter() {
  // selector：从树状 state 里"选"出本组件关心的切片
  const count = useSelector((s: RootState) => s.counter.count)
  const dispatch = useDispatch()

  return (
    <>
      <span>{count}</span>
      <button onClick={() => dispatch(increment())}>+1</button>  {/* actionCreator 在这里回收（4.1 优化点②） */}
    </>
  )
}
```

**机制（你在 3.1 已经学过一遍，逐字迁移）**：

```
dispatch(action) → reducer 算新 state → store 通知订阅者
   → useSelector 重跑你的 selector → Object.is 比较前后返回值
   → 相等：不渲染；不等：重渲染本组件
```

**同样的反模式同样成立**（3.1 的知识原样适用）：

```tsx
// ❌ 返回新对象：每次比较都不等 → 全量重渲染
const { count } = useSelector((s: RootState) => ({ count: s.counter.count }))

// ✅ 修法 1：拆多个 selector
const count = useSelector((s) => s.counter.count)
const name = useSelector((s) => s.counter.name)
// ✅ 修法 2：相等函数第二参
import { shallowEqual } from 'react-redux'
const { count, name } = useSelector(
  (s) => ({ count: s.counter.count, name: s.counter.name }),
  shallowEqual,                       // ← 浅比较代替 Object.is（= Zustand 的 useShallow）
)
```

> **跨库闭环**：Object.is 心法贯穿数据流全程——2.6（memo 逐 prop 比较）、3.1（Zustand selector）、4.2（useSelector）、1.1（Redux 不可变的根因）。**React 生态一切"变没变"的判断都是引用比较**，这四处的坑是同一个坑。

### 四、connect vs hooks（面试常问，老代码会见到）

```tsx
// 旧写法：connect 是 HOC，两个 map 函数当"翻译官"
const enhance = connect(
  (state) => ({ count: state.counter.count }),       // mapStateToProps
  { increment, decrement },                          // mapDispatchToProps（对象简写）
)
const Counter = enhance(({ count, increment }) => (
  <button onClick={increment}>{count}</button>
))

// 现代写法：hooks 直调
const count = useSelector((s) => s.counter.count)
const dispatch = useDispatch()
dispatch(increment())
```

| 维度 | connect（HOC） | useSelector/useDispatch |
|------|---------------|------------------------|
| 包装方式 | HOC 层层嵌套（组件树里多一层看不见的包装组件） | 函数内直调，无包装 |
| 数据来源 | mapState 结果作为 **props 传入**（隐式依赖，TS 推导差） | 显式调用（读代码即知依赖） |
| 比较行为 | mapState 返回**对象时默认做一层浅比较**（自带 shallowEqual） | 严格 Object.is，返回对象要自己加 shallowEqual |
| 现状 | 老项目遗留 | 现代项目唯一写法 |

**高频追问**："为什么 connect 返回对象没事，useSelector 返回对象是坑？"——因为 connect 对 mapState 的对象结果**默认做了浅比较**（react-redux 内置），而 useSelector 默认严格 Object.is，得手动传 shallowEqual。记住这句，两边坑都接得住。

### 五、一句话总结

- react-redux = Provider（Context 注入 store）+ hooks（订阅接上重渲染）
- useSelector 机制与 Zustand selector 逐字相同：重跑 selector → Object.is → 决定渲染；反模式与修法同款（拆 selector / shallowEqual）
- connect 是历史：HOC + props 隐式注入 + 默认浅比较；现代代码一律 hooks

## 考核过程
🎯 考核（编码 + 口头混合）

  任务 1（编码）：给 4.1 的 counter 接上 React——写出：configureStore 接线（树状） + 一个展示 count
  并含加/减按钮的组件（用 actionCreator + useSelector/useDispatch）。要求：selector 
  返回原始值（你自己解释为什么这么写）。

```ts
const store = configureStore({
  reducer:{
    counter:counterReducer
  }
})

<Provider store={store}>
  <App/>
</Provider>


import { increment, decrement } from './counterSlice' 

function Counter() {
  const count = useSelector(s => s.counter.count) // 返回原始值，每次Object.is对比新旧值
  const dispatch = useDispatch()
  return (
    <>
      <span>{count}</span>
      <button onClick={()=> dispatch(increment())}>➕1</button>
      <button onClick={()=> dispatch(decrement())}>➖1</button>
    </>
  )
}
```

  任务 2（口头两问）：

  ① 两个组件 A、B 都 useSelector((s) => s.counter.count)，A 只读 count 不读别的——dispatch(increment())
  时 B（只读 s.counter.name）会重渲染吗？说出完整的判断链条（从 dispatch 到“不渲染”的每一步）。
  
  回答：不会。
   dispatch → reducer 算新 state → store 通知【所有】订阅者
    → A 和 B 的 selector 【都重跑】（B 的也跑！）
    → A：count 变了 → Object.is 不等 → 渲染
    → B：name 的引用没变 → Object.is 相等 → 不渲染 ✅

  ② 追问面试官最爱的一句：“你说的这些都是组件订阅粒度。那 dispatch 一个 action 后，所有 useSelector 的
  selector 是不是都会重跑一遍？这算不算性能问题？什么时候需要担心？”

  回答：
    1. selector 里有昂贵计算：useSelector(s => s.records.filter(...).sort(...)) ——每次 dispatch 都重算大数组。修法：reselect 的createSelector（记忆化：依赖没变直接返回缓存结果）或算好放 store
    2. 返回新对象（3.1 的反模式）→ 比较失效 → 全量渲染风暴

    selector 本身“读对象返回字段”是纳秒级，重跑不可怕，重跑里藏了重活才可怕。

（此段在考核通过后追加填充）

## 考核过程

### 编码 + 口头混合题（两轮）

**首版**：任务 1 一处概念错 🔴——`useSelector(s => s.counter.increment)` 把 actionCreator 当 store 字段选（store 树里只有 state，actionCreator 是模块普通函数，import 直用）；任务 2 ① 结论对但表述错位（"B 没订阅所以不跑"——实际所有 selector 都重跑，跑和渲染是两步）；② 后半不准（"state 大有问题"——真凶是 selector 里的昂贵计算与新对象反模式）。

**点评**：未通过，按提示修订。

**修订版**：三处全修 ✅（import actionCreator 直用、① 精确链条全抄对、② 昂贵计算→reselect createSelector 记忆化 + 新对象反模式 + 收束句"重跑不可怕，重跑里藏了重活才可怕"）。

**点评**：通过。顺带微提醒：selector 的 s 参数实战要标 RootState 类型（演示可省）。跨库闭环已成型：Object.is 心法贯穿 2.6/3.1/4.2——"React 生态一切变没变的判断都是引用比较"。

---
通过时间：2026-11-26
