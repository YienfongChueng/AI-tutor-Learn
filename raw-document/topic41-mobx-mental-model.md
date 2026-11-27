# 2.1 Mobx 心智模型（observable/action/observer，对照 Vue3 proxy 原理）

## 讲解原文

### 一、三问的 Mobx 答案

| 三问 | Mobx 的答案 |
|------|-------------|
| 状态放哪？ | **类 store**——一个普通 class（或对象），字段用 observable 包装 |
| 谁改？ | **action 里可变地直接改**（`user.name = 'Tom'`，不生成新对象！） |
| 怎么通知？ | **proxy 拦截 + 依赖收集**——和 Vue3 响应式同款哲学 |

### 二、心智模型：给 React 外挂一个"响应式大脑"

React 组件默认是"盲目重渲染"的（父渲染子就渲染、state 变了全量对比）。Mobx 的做法是：**把 Vue3 那套响应式系统装到 React 外面**——

```
observable(数据)     ← proxy 拦截读/写（Vue3 的 reactive 同款思路）
     │ 读的时候
     ▼
自动收集依赖          ← "谁在读我？"（Vue3 的 track 同款）
     │ 写的时候
     ▼
精确通知依赖          ← "刚才谁读过我，我变了喊谁"（Vue3 的 trigger 同款）
```

**对照表（你的 Vue3 原理知识直接迁移）**：

| Vue3 | Mobx | 职责 |
|------|------|------|
| `reactive(obj)` / `ref(v)` | `observable(obj)` / `makeAutoObservable` | 数据包装 + 拦截 |
| `computed` | `computed` | 派生值，带缓存 |
| `watchEffect` | `autorun` | 副作用：跑一遍收集依赖，依赖变自动重跑 |
| 组件渲染时读响应式数据 | `observer(Comp)` 的 render 执行 | 渲染即订阅 |
| （框架内建） | `runInAction` / configure({enforceActions}) | 写入必须走 action 的纪律 |

### 三、核心三角：state → derivations → actions

Mobx 官方心智模型是一个三角，和 Vue 的"数据→视图→方法"本质相同：

```
        actions（唯一写入入口，事件驱动）
            │ 改
            ▼
        state（observable，可变）
            │ 读
            ▼
        derivations（computed 派生值）
            │ 被
            ▼
        views（React 组件 render = observer 包着的 reaction）
```

**与 Redux 的根本区别一句话**：Redux 是"事件账本"（action 是数据，reducer 是纯函数，一切不可变）；Mobx 是"活体追踪"（action 是普通方法，state 可变，追踪全自动）。前者换来时间旅行与可预测，后者换来零样板与精确更新。

### 四、observer 的订阅粒度（面试高频）

`observer(Comp)` 的原理：把组件的 render 包成一个 Mobx reaction——

1. render 执行，读到 `store.count` → 依赖收集：**这个组件订阅了 count**
2. `store.count++` → 只通知读过 count 的组件 forceRender
3. **读了 count 但没读 name 的组件不会更新**——粒度精确到"字段级"（这比 Redux 的 selector 级还细，也更接近你熟悉的 Vue 组件更新模型）

对比 React 默认模型的代价：observer 组件**父渲染子必渲染的默认规则被打破**——只有依赖的 observable 变了才渲染。这也带来新纪律：**不要在 observer 里解构 store**（`const {count} = store` 在 render 顶部读取——其实这没问题，Mobx 追踪的是 render 期间读的属性；真正要防的是"读了但没被 observable 追踪的中间快照"——用 `toJS` 才拿快照）。

### 五、严格模式：action 纪律

`configure({ enforceActions: 'observed' })`（现代默认）：**在 action 之外改 observable 会报错**。为什么？

- 一次事件处理往往连续改多个字段——Mobx 会把 action 内的改动**事务化批量**（一次 tick 一批通知），不散装触发
- 这是对"随便改"的工程约束：改可以，但要报备（action）——Pinia 没这条纪律，Mobx 选择了可预测性

### 六、一句话总结

- Mobx = Vue3 响应式哲学在 React 的实现：observable/computed/autorun ↔ reactive/computed/watchEffect
- observer = 组件级 reaction，字段级精确订阅，打破"父渲染子必渲染"
- 写入纪律：enforceActions 让"随便改"变成"报备着改"，action 内事务化批量通知
- 和 Redux 对比：事件账本 vs 活体追踪——下一个模块的 Zustand 会给出第三条路

### 补充：常用 API 速查 + 项目完整案例

#### 常用 API 速查（按使用频率）

| API | 作用 | 一句话记忆 |
|-----|------|-----------|
| `makeAutoObservable(this)` | 构造器里一行：自动把字段标 observable、getter 标 computed、方法标 action | **自动贴标签**（90% 场景只用它） |
| `makeObservable(this, {...})` | 手动指定标注（自动推断不满足时用，如继承类） | 手动贴标签 |
| `computed({ get, set })` | 派生值，依赖不变时读缓存 | 可写型 getter |
| `action` / `runInAction(fn)` | 写入入口；**异步回调里改 state 必须包 runInAction**（异步部分不在 action 作用域内） | 异步改 state 的报备条 |
| `flow` | generator 版异步 action（自带 yield 分步 + 自动 action），`flow(function*(){...})` | 异步 action 的优雅版 |
| `configure({ enforceActions: 'observed' })` | 全局纪律：非 action 改 observable 报错 | 报备制度开关 |
| `autorun(fn)` | 立即跑一遍 fn，读过的 observable 变化自动重跑（副作用用） | watchEffect 同款 |
| `reaction(() => 数据, (新, 旧) => {})` | 盯指定数据，变化才跑（首个不跑） | watch 同款 |
| `when(predicate)` | 条件成立跑一次后自动销毁 | 一次性 reaction |
| `observer(Component)` | HOC：把 render 包成 reaction，渲染即订阅 | 组件级订阅入口 |
| `toJS(obj)` | 把 observable 转普通 JS 对象（发给 API/存 localStorage 前**必用**） | 摘掉代理壳 |
| `configure({ isolatedWarnings })` 等 | 其他全局配置 | 少用 |

#### 项目完整案例：记账本 Mobx 版（定义 → 注册 → 使用）

**第 1 步：定义 store（**`src/stores/recordsStore.ts`**）**

```ts
import { makeAutoObservable, runInAction } from 'mobx'
import { getRecords, createRecord } from '@/api/records'
import type { RecordItem, RecordType } from '@/types/record'
import { toJS } from 'mobx'

class RecordsStore {
  records: RecordItem[] = []                    // state
  loading = false
  filterType: RecordType | 'all' = 'all'        // state

  constructor() {
    makeAutoObservable(this, {}, { autoBind: true })  // autoBind: 方法 this 绑定
  }

  // computed：getter 自动被 makeAutoObservable 标注
  get filteredRecords(): RecordItem[] {
    if (this.filterType === 'all') return this.records
    return this.records.filter(r => r.type === this.filterType)
  }
  get totalBalance(): number {
    return this.filteredRecords.reduce(
      (sum, r) => r.type === 'income' ? sum + r.amount : sum - r.amount, 0
    )
  }

  // action：同步方法直接标
  setFilter(type: RecordType | 'all') {
    this.filterType = type                      // 可变直接改 ✅
  }

  // 异步：await 之后改 state 必须包 runInAction
  async fetchRecords() {
    this.loading = true                         // await 前还在 action 作用域内，可直改
    try {
      const data = await getRecords()
      runInAction(() => {                       // ← 报备条
        this.records = data
        this.loading = false
      })
    } catch (e) {
      runInAction(() => { this.loading = false })
      throw e
    }
  }

  async addRecord(payload: Omit<RecordItem, 'id'>) {
    const created = await createRecord(payload)
    runInAction(() => { this.records.push(created) })  // 可变 push ✅
  }

  // 发给 API 前摘代理壳
  getSnapshotJson() { return toJS(this.records) }
}
export const recordsStore = new RecordsStore()   // 模块级单例（简单场景）
```

**第 2 步：注册——两种模式**

```ts
// 模式 A：模块级单例（上面的 export default）——小项目直用
// 模式 B：RootStore + Context（你 4.10 的 AuthProvider 同款接线，大项目用）
class RootStore { records = new RecordsStore(); auth = new AuthStore() }
const rootStore = new RootStore()
const StoreContext = createContext(rootStore)
export const StoreProvider = ({ children }) => (
  <StoreContext.Provider value={rootStore}>{children}</StoreContext.Provider>
)
export const useStore = () => useContext(StoreContext)
// main.tsx: <StoreProvider><App/></StoreProvider>（包在 AuthProvider 外层即可）
```

**第 3 步：组件使用（**`observer` **是唯一新东西）**

```tsx
import { observer } from 'mobx-react-lite'
import { useStore } from '@/stores'

const RecordsPage = observer(() => {
  const { records } = useStore()
  // 读 store.filteredRecords → render 期间被依赖收集：本组件订阅 filteredRecords 相关字段
  return (
    <List>
      {records.filteredRecords.map(r => (
        <List.Item key={r.id}>{r.category} {r.amount}</List.Item>
      ))}
    </List>
  )
})
// ⚠️ 三条纪律：
// 1. 组件必须包 observer，否则数据变了不更新（没有订阅）
// 2. 不要在 render 外/顶层解构丢失追踪（如把 store.toJS 的快照存 state）
// 3. 派生值直接读 getter，不要在组件里自己 useMemo 包（computed 已带缓存）
```

**第 4 步：持久化（对照你 Pinia 的 persist 插件）**

```ts
// mobx 没有官方一行的 persist 插件，常用二选一：
// ① mobx-persist-store 库
import { makePersistable } from 'mobx-persist-store'
constructor() {
  makeAutoObservable(this)
  makePersistable(this, { name: 'RecordsStore', properties: ['filterType'],
                          storage: window.localStorage })
}
// ② 手动：autorun(() => localStorage.setItem('k', JSON.stringify(toJS(this.filterType))))
//    初始化时构造器里读回——本质就是给"怎么通知"挂副作用，和 Pinia persist 同构
```

**三问复盘（Mobx 完整答案）**：

- 放哪：class 实例（模块单例或 RootStore+Context）
- 谁改：action（同步直改、异步 runInAction/flow），可变直接改
- 怎么通知：observable 拦截 + 依赖收集 → observer 组件字段级精确更新

## 考核过程

  用 Mobx 给记账本项目写一个“余额统计”store（不用管 React 细节接线，2.2 才讲，类写法即可）：
  2. 派生值：filteredRecords（按类型过滤）、totalBalance（过滤后收入减支出）
  3. action：addRecord(record)、setFilter(type)

```ts
class BalanceStore {
  records: RecordItem[] = []                    // state
  loading = false
  filterType: RecordType | 'all' = 'all'        // state

  constructor() {
    makeAutoObservable(this, {}, { autoBind: true })  // autoBind: 方法 this 绑定
  }
  get filteredRecords():RecordItem[] {
     if (this.filterType === 'all') return this.records
     return this.records.filter(r => r.type === this.filterType)
  }

  get totalBalance():number {
    return this.filteredRecords.reduce(
      (sum, r) => r.type === 'income' ? sum + r.amount : sum - r.amount, 0
    )
  }
  // action：同步方法直接标
  setFilter(type: RecordType | 'all') {
    this.filterType = type                      // 可变直接改 ✅
  }
  async addRecord(record:Omit<RecordItem,'id'>) {
    const created = await createRecord(record)
    runInAction(() => { this.records.push(created) })  // 可变 push ✅
  }
}
export const balanceStore = new BalanceStore()
```

  写完回答两问：
  ① 哪一行代码体现了“字段级精确订阅”？具体说：一个组件 render 里只读了 totalBalance，当 setFilter('expense')
  执行但过滤结果恰好没变时，这个组件会重渲染吗？为什么？
  不会重渲染。 链条是这样断的：

  setFilter('expense') → filterType 变
    → filteredRecords 重算（computed 依赖失效重求值）
    → totalBalance 重算 → 新值 === 旧值（比如都是 -25）
    → Mobx computed 有"值相等不发通知"的优化（默认结构比较）
    → 订阅 totalBalance 的 observer 组件：收不到通知，不渲染 ✅

  精妙之处：如果组件订阅的是 filteredRecords（数组），会重渲染——filter 每次都生成新数组引用，引用不相等。订阅 
  number（值比较）稳如老狗，订阅新数组（引用比较）会多渲染。所以“读什么派生值”本身就是性能决策。

  ② addRecord 里如果写成在 action 外直接 store.records.push(...)，开了 enforceActions 会发生什么？

直接抛错，改动不生效。 enforceActions: 'observed' 下，action 外改 observable（push 算改）会 throw：

  ▎ "Since strict-mode is enabled, changing (observed) observable values outside actions is not allowed..."

  目的：强迫所有变更走 action——换来 devtools 可追溯 + 事务化批量（一次 tick 一批通知，不散装触发）。这就是“改可以，但要报备”的强制版。

## 考核过程

### 编码题：StatsStore 类 + 两问

**用户回答**：提交了 BalanceStore 类（records/loading/filterType state、filteredRecords/totalBalance getter 链、setFilter/addRecord action）；两问请求直接给答案（逃生舱）。

**点评**：

- 代码审查两处 🔴：①缺 `makeAutoObservable(this)`（连 constructor 都没写）——Mobx"启动开关"，没有它全 store 不追踪不通知，最高频"为什么不更新"事故；②`addRecord` 用 `await` 缺 `async`——语法报错。其余（getter 链式派生、runInAction 位置、可变 push）正确
- 两问按逃生舱给标准答案：①不重渲染——computed 链重算后 totalBalance 值相等不发通知（值相等优化）；精妙点：订阅 filteredRecords（新数组引用）会渲染，订阅 number 派生值稳——"读什么派生值"是性能决策；②enforceActions 下直接 throw（strict-mode 报错，改动不生效），换来可追溯+事务化批量
- 判为参考通过，复习重点：makeAutoObservable 必须存在、computed 值相等不发通知、订阅数组 vs 订阅 number 的差异

---

通过时间：2026-11-24
