# 3.1 Zustand：create 与 store 即 hook + selector 粒度

## 讲解原文

### 一、三问的 Zustand 答案

| 三问 | Zustand 的答案 |
|------|-------------|
| 状态放哪？ | `create()` 造一个 store——**store 本身就是 Hook**（不像 Redux 要 Provider、不像 Mobx 要 Context） |
| 谁改？ | store 内部的 `set()`——可变心智、不可变操作（写法和 React setState 同款） |
| 怎么通知？ | **selector 决定粒度**——`useStore(s => s.count)` 只在选中的片段变化时才重渲染 |

**总量级认知**：Zustand 全部核心 API 就是 `create` 一个函数 + `set/get` 两个参数。第 4 模块 Redux 全家的模板代码，在这里被压缩成一个对象字面量——这就是它爆火的直接原因（3.2 细讲）。

### 二、核心语法（全部家当）

```ts
import { create } from 'zustand'

// ── 定义：一个函数造 store，返回值直接当 hook 用 ──
// ⚠️ 注意 create<State>()(...) 的"中间括号"——这是 TS 中间件类型推断的
//    柯里化写法（加中间件时类型不丢），背下来这个形状
type CounterState = {
  count: number
  filterType: 'all' | 'expense' | 'income'
  increment: () => void
  setFilter: (t: CounterState['filterType']) => void
}

export const useCounterStore = create<CounterState>()((set, get) => ({
  // state（普通字段）
  count: 0,
  filterType: 'all',

  // action：set 只有它一个写入口
  increment: () => set((state) => ({ count: state.count + 1 })),   // 函数式：拿旧值算新值

  setFilter: (t) => set({ filterType: t }),          // 对象式：直接给增量

  resetAll: () => set({ count: 0, filterType: 'all' }, true), // 第二参 true = 替换而非合并
}))

// ── 消费：没有 Provider，组件里直接调 hook ──
function Counter() {
  const count = useCounterStore((s) => s.count)      // ← selector：订阅哪片自己定
  const increment = useCounterStore((s) => s.increment)
  return <button onClick={increment}>{count}</button>
}

// action 里读最新值用 get()（不依赖闭包快照）
// const action = () => { if (get().count > 10) set({ count: 0 }) }
```

**set 的两个规则**（React setState 用户零学习成本）：

1. **对象式** `set({...})` **是浅合并**——只改提到的字段，其余保留（不是替换！替换要第二参数 `true`）
2. **函数式** `set(state => ...)`——需要旧值计算时用，和 React 函数式更新同款语义（你 2.1 的快照知识直接迁移，但 Zustand 的 set 读的是 store 最新值，没有"快照过期"问题）

### 三、selector 粒度——Zustand 的订阅模型（本节核心）

```tsx
// ✅ 粒度 1：订阅单个字段——count 变才渲染
const count = useStore((s) => s.count)

// ✅ 粒度 2：订阅多个字段——拆成多个 selector（推荐）
const records = useStore((s) => s.records)
const filterType = useStore((s) => s.filterType)

// ❌ 反模式：selector 返回"新对象"
const bad = useStore((s) => ({ records: s.records, filterType: s.filterType }))
// 每次任何 state 变化，这个 selector 都生成【新对象】，
// 引用比较（Object.is）永远不等 → 组件全量重渲染，selector 白写！
```

**为什么反模式错**：Zustand 的订阅判断 = **每次 state 变化 → 重跑你的 selector → Object.is 比较这次和上次的返回值**。返回原始值/稳定引用 → 比较有效；返回新对象/新数组 → 永远"变了"。

两个修法：

```tsx
// 修法 1：拆 selector（首选）
const records = useStore((s) => s.records)
const filterType = useStore((s) => s.filterType)

// 修法 2：useShallow——浅比较代替 Object.is
import { useShallow } from 'zustand/react/shallow'
const { records, filterType } = useStore(
  useShallow((s) => ({ records: s.records, filterType: s.filterType }))
)
```

> **知识回响**：这个坑你其实全见过——2.6 的"逐 prop Object.is 比较"、4.8 的"防抖值进 key"、1.1 的"Redux 必须不可变"——**全部是同一件事：React 生态判断"变没变"只能靠引用比较**。Zustand 只是把比较点交给了你的 selector。

### 四、底层一句话：useSyncExternalStore + 闭包 store

Zustand 的 store 是**模块级闭包对象**，`create` 返回的 hook 底层用的是 **React 18 官方订阅 API** `useSyncExternalStore`：

```
store（闭包，模块单例）
   ├── getState() / setState() / 订阅回调列表
   └── create 返回的 hook = useSyncExternalStore(
         store.subscribe,        // 订阅
         () => selector(store.getState())  // 快照 = 你的 selector 结果
       )
```

所以"store 即 hook、无需 Provider"不是魔法——**store 不住 React 树里，住模块作用域里**；hook 只是从外面"探"进来。这同时解释了：为什么能在组件外调用 `useStore.getState()`（它就是个普通对象方法）。

### 五、四库订阅模型对照（阶段小结，5.1 会收敛成决策树）

| 库 | 订阅入口 | 粒度由谁定 | Provider |
|----|---------|-----------|----------|
| Redux | useSelector（Context 广播） | selector + 引用比较（要 memo 化） | 要 |
| Mobx | observer HOC | **自动**（依赖收集，字段级） | 可选（RootStore 常配 Context） |
| Pinia | 响应式追踪 | **自动**（proxy 依赖收集） | 不要（app.use 注入） |
| Zustand | selector | **手动**（selector 写法决定） | 不要 |

类比：Redux = 全楼广播（人人听到，自己筛）；Mobx/Pinia = 智能门铃（谁关联响谁）；**Zustand = 自己订阅公众号**（你选关注谁，号更新才推给你——选错关注（反模式 selector）就天天被推垃圾）。

### 六、一句话总结

- 全部 API = `create` + `set`/`get`；`create<T>()(...)` 中间括号是 TS 柯里化
- set 对象式**浅合并**（第二参 true 才替换）；函数式同 React 语义
- **selector = 手动定订阅粒度**；返回新对象的反模式 = Object.is 失效——两个修法：拆 selector / useShallow
- 底层 = 模块级闭包 store + useSyncExternalStore——"无需 Provider"的根因

### 补充：create 的柯里化写法——"加中间件时类型不丢"什么意思？

#### 柯里化本身：一次传参拆成两次

`create<State>()(setup)` 拆开看是**两次调用**：

```ts
create<State>()    // 第一次：只传类型 → 返回一个"锁定好类型的 create"
  (setup)          // 第二次：传 setup 函数 → 返回 store hook
```

"柯里化"就是把"一次传两个参数"改成"每次传一个、分两步传"。这样**类型在第一步就被锁死**，第二步不用再猜。

#### 为什么需要它：中间件会"挡住"类型推断

**不加中间件**时，两种写法等价且都没问题：

```ts
const useStore = create<CounterState>((set) => ({ ... }))        // 直接版 ✅
const useStore = create<CounterState>()((set) => ({ ... }))      // 柯里化版 ✅
```

**加中间件**（如 devtools / persist）时，setup 被中间件函数**包了一层**：

```ts
// ❌ 直接版：类型推断被中间件挡住
const useStore = create<CounterState>(devtools((set) => ({ count: 0 })))
//                                        ↑ 中间件包住 setup
// TS 要"穿过" devtools 的函数签名去推断 state 类型——
// 类型复杂的中间件组合会推断失败：set 的参数变成 never/unknown，报一串错

// ✅ 柯里化版：类型在第一步就锁死，中间件只管行为不管类型
const useStore = create<CounterState>()(devtools((set) => ({ count: 0 })))
//              ↑ 第一步：State 已确定     ↑ 第二步：setup 随便包几层，类型都从第一步来
```

#### 一句话锚定

> **柯里化 = 把"类型"和"实现"分两次传——第一次先声明 state 长什么样（锁型），第二次再让中间件包裹实现（行为）**。中间件想包多少层都挡不住类型，因为类型根本不从它那儿来。

记忆法：**中间件在里，类型在外**——`create<T>()(中间件(setup))`，`T` 永远紧贴 `create`。

（3.2 讲 persist/devtools 中间件时会实际用到这个形状。）

## 考核过程
任务：

  1. useRecordsStore：
   state 有 records: RecordItem[]、filterType: 'all'|'expense'|'income'；
   action 有 setFilter(t)、addRecord(record)（要求函数式set 写法）、resetAll()（清空到初始）
  2. 组件里正确消费：展示 records，点按钮切换 filterType——写出 selector 用法（注意粒度写法）
  3. 口头两问：
     ① 为什么 useStore((s) => ({ records: s.records, filterType: s.filterType })) 是反模式？说出机制 + 两个修法
     - 因为state变化，重跑selector，Object.is比较这次和上次返回值，上面这种反模式返回的是对象，引用值变化了，组件全量重渲染，selector白写。
     - 修法1:拆 selector
     - 修法2:useShallow——浅比较代替 Object.is。 ```useStore(useShallow((s) => ({ records: s.records, filterType: s.filterType }))) ```

     ② set({ count: 0 }) 是替换还是合并？想把整个 state 换成全新对象该怎么做？
     - 合并，替换需要加参数：```set({count:0},true)```
     

  ```ts
  type RecordState = {
    records: RecordItem[]
    filterType: 'all' | 'expense' | 'income'
    setFilter: (t: RecordState['filterType']) => void
    addRecord: (record: Omit<RecordItem,'id'>) => void
    resetAll:()=> void
  }
  export const useRecordsStore = create<RecordState>()((set,get)=>({
    records: [],
    filterType: 'all',

    setFilter:(t)=> set({filterType: t}),

    addRecord:(record) => set(s => ({records:[...s.records,record]})),

    resetAll: ()=> set({records:[],filterType: 'all'},true)
  }))

  // ── 消费：没有 Provider，组件里直接调 hook ──
  function Record() {
    const records = useRecordsStore((s) => s.records)  // ← selector：订阅哪片自己定
    const filterType = useRecordsStore(s => s.filterType)    
    const setFilter = useRecordsStore((s) => s.setFilter)
    return (
      <>
        <Button onClick={()=>setFilter('expense')}>{filterType}</Button>
        <List>
          {records.map(item => (
            <List.Item key={item.id}>{item.category} {item.amount} </List.Item>
          ))}
        </List>
      </>
    )
  }
  ```


## 考核过程

### 编码题：列表筛选 store + 两问（两轮）

**首版**：两问满分（反模式机制 Object.is + 拆 selector/useShallow 两修法；set(obj) 合并 / set(obj, true) 替换）；柯里化形状 `create<RecordState>()(...)` 用对；addRecord 用了函数式 set（方向对）。4 处 🔴：①对象字面量两处缺逗号（第 3 次回潮，1.6 混用/4.3 分号）②resetAll 忘包 set——写成逗号表达式 `({...}, true)` 返回 true、state 不动 ③addRecord 的 set 返回数组（应为部分 state 对象 `{records:[...]}`）④`records ?? records.map` 永不渲染（records 永远是数组，?? 左边恒真）。⚠️ onClick 直传带参 action（2.2 刚讲的 MouseEvent 坑立即回潮）。类型漏登记 resetAll。

**点评**：未通过，按组点评修订。

**修订版**：6 处全部修对（逗号/set 包裹/返回对象/map 直调/onClick 箭头/类型登记）。

**点评**：通过。微小残留不返工：get 解构未用（noUnusedParameters 会唠叨）、setFilter 硬编码 'expense'（演示可接受，真切换用循环按钮组）。

---
通过时间：2026-11-25
