# 2.2 Mobx React 接线：完整流程与语法（带注释）

## 讲解原文

### 接线总流程图（五步）

```
Step1 定义 store 类        Step2 RootStore 组合       Step3 Context 注入
（class + makeAuto-        （多 store 互相引用的       （Provider + useStore
 Observable）               中枢）                     自定义 hook）
      │                          │                          │
      └──────────┬───────────────┘                          │
                 ▼                                          ▼
          Step4 main.tsx 全局包裹  ──────────▶  Step5 observer 组件消费
          （挂 Provider）                      （读 store，自动订阅）
```

---

### Step 1：定义 store 类（逐行注释版）

```ts
// src/stores/recordsStore.ts
import {
  makeAutoObservable,   // 自动标注器：字段→observable，getter→computed，方法→action
  runInAction,          // 异步回调里改 state 的"报备条"
  toJS,                 // 摘 Proxy 壳：发给 API / 存 localStorage 前必用
} from 'mobx'
import { getRecords, createRecord } from '@/api/records'
import type { RecordItem, RecordType } from '@/types/record'

export class RecordsStore {
  // ── state：普通类字段，makeAutoObservable 会自动把它们变成 observable ──
  records: RecordItem[] = []               // 账目全量（服务端数据的镜像）
  loading = false                          // 请求态（UI 直接消费）
  filterType: RecordType | 'all' = 'all'   // 筛选条件

  constructor() {
    // makeAutoObservable(target, overrides?, options?)
    //   target:        this，必传
    //   overrides:     手动覆盖标注（ rarely 用，见下方对照）
    //   options.autoBind: true → 方法自动 bind this
    //     （不设的话，把 action 当 props 传下去会丢 this——和类组件 handler 丢 this 同款问题）
    makeAutoObservable(this, {}, { autoBind: true })
  }

  // ── computed：getter 形式，makeAutoObservable 自动标注 ──
  //   特性：依赖不变读缓存；重算后值相等不发通知（2.1 考过）
  get filteredRecords(): RecordItem[] {
    if (this.filterType === 'all') return this.records
    return this.records.filter((r) => r.type === this.filterType)
  }

  get totalBalance(): number {
    // computed 可以依赖另一个 computed（链式派生），依赖收集自动串联
    return this.filteredRecords.reduce(
      (sum, r) => (r.type === 'income' ? sum + r.amount : sum - r.amount),
      0,
    )
  }

  // ── action：同步方法，makeAutoObservable 自动标注 ──
  setFilter(type: RecordType | 'all') {
    this.filterType = type    // 可变直接改——不用展开旧对象（对比 Redux）
  }

  // ── 异步 action：await 之后脱离了本方法的 action 作用域 ──
  //   所以 await 前可直改（还在报备区内），await 后必须 runInAction 包起来
  async fetchRecords() {
    this.loading = true                     // ✅ await 之前，合法
    try {
      const data = await getRecords()       // await 之后 = 普通 callback 上下文
      runInAction(() => {                   // ✅ 报备后再改
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
    runInAction(() => {
      this.records.push(created)            // 可变 push ✅（不需要 [...spread]）
    })
  }

  // ── toJS：摘壳。给 API / JSON.stringify / 结构化克隆前必须转 ──
  get recordsSnapshot(): RecordItem[] {
    return toJS(this.records)
  }
}
```

**makeObservable 手动版对照**（自动推断不够用时）：

```ts
import { makeObservable, observable, computed, action } from 'mobx'

class Counter {
  count = 0
  get double() { return this.count * 2 }
  increment() { this.count++ }

  constructor() {
    // 手动逐个贴标签——字段/派生/方法的对应关系一目了然（也啰嗦）
    makeObservable(this, {
      count: observable,
      double: computed,
      increment: action,
    })
  }
}
// 经验法则：新代码一律 makeAutoObservable；只有继承类字段推断歧义时才手动
```

---

### Step 2：RootStore 组合（跨 store 交互的中枢）

```ts
// src/stores/rootStore.ts
import { RecordsStore } from './recordsStore'
import { AuthStore } from './authStore'
import { StatsStore } from './statsStore'

export class RootStore {
  records: RecordsStore     // 各子 store 声明在前，构造器里赋值
  auth: AuthStore
  stats: StatsStore

  constructor() {
    this.records = new RecordsStore()
    this.auth = new AuthStore()
    // 跨 store 依赖：把 this（RootStore）传给子 store，
    // 子 store 内部就能 this.root.records.xxx——互相引用不打结
    this.stats = new StatsStore(this)
  }
}

export const rootStore = new RootStore()   // 全局唯一实例（单例）
```

```ts
// src/stores/statsStore.ts —— 跨 store 派生的示例
import { makeAutoObservable } from 'mobx'
import type { RootStore } from './rootStore'

export class StatsStore {
  root: RootStore   // 持有 root 引用 → 能读到所有兄弟 store

  constructor(root: RootStore) {
    this.root = root
    // ⚠️ root 字段会被自动标 observable——引用永不变所以无害，
    //    但干净起见可用 overrides 排除：root: false = 不标注这个字段
    makeAutoObservable(this, { root: false })
  }

  // 派生值直接读兄弟 store 的 computed——跨 store 组合零仪式
  // （4.9 统计页同款逻辑的 Mobx 版）
  get expenseByCategory(): { category: string; total: number }[] {
    const map = new Map<string, number>()
    for (const r of this.root.records.filteredRecords) {  // ← 跨 store 读
      if (r.type !== 'expense') continue
      map.set(r.category, (map.get(r.category) ?? 0) + r.amount)
    }
    return [...map.entries()]
      .map(([category, total]) => ({ category, total }))
      .sort((a, b) => b.total - a.total)     // 降序
  }
}
```

---

### Step 3：Context 注入（复用你 4.10 的 AuthProvider 接线模式）

```tsx
// src/stores/storeContext.tsx
import { createContext, useContext, type ReactNode } from 'react'
import { rootStore, type RootStore } from './rootStore'

// 泛型给实际单例（不是 null）——Store 只会在 Provider 内用，占位用 undefined
const StoreContext = createContext<RootStore | undefined>(undefined)

export function StoreProvider({ children }: { children: ReactNode }) {
  return <StoreContext.Provider value={rootStore}>{children}</StoreContext.Provider>
}

// 自定义 hook：判空 throw（和 4.10 的 useAuth 同款防绕过）
export function useStore(): RootStore {
  const ctx = useContext(StoreContext)
  if (!ctx) throw new Error('useStore 必须在 <StoreProvider> 内使用')
  return ctx
}
```

---

### Step 4：main.tsx 全局包裹

```tsx
// src/main.tsx（只展示 Provider 嵌套，其他省略）
<AuthProvider>                    {/* 4.10 的登录态——也可以迁进 RootStore */}
  <StoreProvider>                {/* ← Mobx 中枢在这层注入 */}
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </QueryClientProvider>
  </StoreProvider>
</AuthProvider>
```

> 顺序无强制（这些 Provider 互不依赖），惯例是"业务全局态在外、技术层在内"。

---

### Step 5：observer 组件消费

```tsx
// src/pages/RecordsPage/index.tsx（节选）
import { observer } from 'mobx-react-lite'   // 函数组件用 lite 版（体积小，够用）
import { useStore } from '@/stores'

// observer(...) 是 HOC：把整个函数组件的 render 包成 reaction
const RecordsPage = observer(() => {
  const { records } = useStore()   // 拿到 store 实例（不是 state 快照！）

  // ① render 里"读属性"的动作会被依赖收集——
  //    读到 filteredRecords → 本组件订阅 filteredRecords 的依赖链
  const list = records.filteredRecords
  const balance = records.totalBalance

  return (
    <>
      <span>结余：{balance}</span>
      <List>
        {list.map((r) => (
          <List.Item key={r.id}>{r.category} {r.amount}</List.Item>
        ))}
      </List>
    </>
  )
})

export default RecordsPage

// ── 对照：Vue/Pinia 解构要 storeToRefs，Mobx 为什么不需要？──
// Pinia：解构 = 把值抄出来，丢掉 reactive 连接 → 必须 storeToRefs
// Mobx：追踪的是 render 执行期间的"属性访问"（getter 调用），
//        不存在"解构丢连接"——你读什么，就订阅什么。这是两套追踪模型的差异。
// ⚠️ 但顶层提前解构（render 第一行 const { totalBalance } = records）依然 OK：
//    只要"读取"发生在 render 函数体内，就在 reaction 的追踪范围内。
```

---

### 常见坑清单（接线期踩坑排行）

| # | 坑 | 现象 | 根因/修法 |
|---|----|------|----------|
| 1 | 忘了 `makeAutoObservable` | 改了数据界面不更新 | store 是死数据——启动开关没开 |
| 2 | 组件忘包 `observer` | store 变了组件不动 | 没有订阅；observer 是订阅入口 |
| 3 | `await` 后直改 state | enforceActions 报错 / 不通知 | 包 `runInAction` |
| 4 | 把 action 解构后当回调传 | `TypeError: Cannot read ... of undefined` | this 丢失 → `autoBind: true` |
| 5 | observable 直接 JSON 化 | 序列化异常/丢数据 | Proxy 壳 → 先 `toJS` |
| 6 | 组件里自己 useMemo 包 store 派生值 | 白包（computed 已缓存） | 直接读 getter |

---

### 一句话总结

- 五步接线：**类定义 → RootStore 组合 → Context 注入 → Provider 包裹 → observer 消费**
- 两条对照记忆：Pinia 的 `storeToRefs` 问题在 Mobx 不存在（属性访问追踪）；RootStore 模式就是你 4.10 AuthProvider 的多 store 版
- 坑都集中在"开关"上：makeAutoObservable、observer、runInAction——三个开了就通

## 考核过程

### 实战验收题：分类统计页接线全链路（两轮）

**首版用户回答**：CategoryStore（loading/root/topCategories getter/fetchAndExport）+ RootStore + observer 版 CategoryPage。8 处错误：

- 🔴 A 组概念：①`makeObservable(this, {root:false}, {autoBind:true})` 用混——排除表语义只属于 makeAutoObservable 的 overrides，makeObservable 的第二参数是完整标注映射，此写法下 store 全字段未标注（死的）；②`new CategoryStore()` 没传 root，this.root undefined 崩；③toJS 用在 `get loadingSnapshot(): boolean` 上（无意义），fetchAndExport 没写 POST
- 🔴 B 组语法：try() 多括号、`.map([category,total]=>…)` 解构缺括号、export const rootStore 写进 class 体
- 🔴 C 组逻辑：过滤条件 `!== 'income'`（题目要支出，1.6/4.6 老坑第 3 次回潮）、组件渲染 r.amount（对象是 {category,total}）

**点评**：未通过——五步结构与思路全对（observer/useStore/读 getter/autoBind 解构 action 当回调的组合用对了），错在细节。按 A/B/C 分组点评后修订。

**修订版**：8 处全部修正（makeAutoObservable+overrides、new CategoryStore(this)、toJS 进 POST、try{}、解构括号、export 类外、!=='expense'、r.total）。

**点评**：通过。两个微小残留（记为提醒不再返工）：①toJs→toJS 拼写（导出名大小写敏感）；②fetchAndExport(payload) 外部收参 + onClick 直传 = payload 实为 MouseEvent——**onClick 直传的 action 别指望从参数拿业务数据，快照应从 store 内部取**（如 `exportStats(toJS(this.root.records.records))`）。

---

通过时间：2026-11-25

  任务清单：

1. CategoryStore 类：
   - state：loading
   - 跨 store 派生：topCategories（从 recordsStore.filteredRecords 里取支出按 category  
    汇总，降序取前 5）
   - 异步 action：fetchAndExport()——调一个 mock API（await exportStats()），成功后用 toJS 把快照  
    POST 出去
2. RootStore：把 CategoryStore 挂进去（写出跨 store 引用的那一行）
3. 组件：observer 版 CategoryPage——展示 topCategories，按钮触发 fetchAndExport，loading 期间按钮禁用

   验收标准：

- ✅ 五步接线文件齐全（顺序对）
- ✅ 跨 store 引用的写法正确（root 传递 + overrides 排除）
- ✅ runInAction 用在正确的位置（await 之后）
- ✅ toJS 用在正确的位置（POST 的 payload）
- ✅ 按钮禁用直接读 loading（不需要 useState——想想为什么）

```ts
// 1、定义store
export class CategoryStore {
  loading:boolean = false
  root: RootStore   // 持有 root 引用 → 能读到所有兄弟 store

  constructor(root: RootStore) {
     this.root = root
    makeAutoObservable(this,{root: false },{autoBind: true}) //自动注入
  }

//getter
get topCategories():{ category: string; total: number }[] {
  const map = new Map<string, number>()
  const filteredRecords = this.root.records.filteredRecords // 跨store取recordStore的state
  for(let item of filteredRecords) {
    if (item.type !== 'expense') continue
    map.set(item.category,(map.get(item.category)??0) + item.amount)
  }
  // map 转数组并降序排序,取前五
  return [...map.entries()].map(([category,total]) => ({category,total})).sort((a,b) => b.total - a.total).slice(0,5)
}

//action
async fetchAndExport(){
  this.loading = true // await之前可以直接改state值
  try{
    await exportStats(toJS(this.root.records.records))
    runInAction(()=> {
      this.loading = false
    })
  }catch(e) {
    runInAction(()=> {
      this.loading = false
    })
    throw(e)
  }
}

}

//2、RootStore 组合（跨 store 交互的中枢）
export class RootStore {
  records: RecordsStore     // 各子 store 声明在前，构造器里赋值
  category: CategoryStore
  // ...other store
  constructor() {
    this.records = new RecordsStore() 
    this.category = new CategoryStore(this)//跨store引用
  }
}
export const rootStore = new RootStore()   // 全局唯一实例（单例）

// 组件 消费
const CategoryPage = observer(() => {
   const { category } = useStore()   // 拿到 store 实例

   const topCategories = category.topCategories
   const fetchAndExport = category.fetchAndExport
   const loading = category.loading

   return (
    <>
      <Button onClick={fetchAndExport} loading={loading}>导出</Button>
      <List>
        {topCategories.map((r) => (
          <List.Item key={r.category}>{r.category} {r.total}</List.Item>
        ))}
      </List>
    </>
  )
})

export default CategoryPage


```
