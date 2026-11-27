# 3.2 Zustand 中间件与爆火原因

## 讲解原文

### 一、中间件机制：柯里化形状终于派上用场

中间件 = **包装 setup 函数的高阶函数**——它接管 `set/get`，在"改 state"这条路上插自己的私活。3.1 说的"中间件在里，类型在外"就是为它准备的：

```ts
export const useRecordsStore = create<RecordState>()(
  persist(                              // ← 中间件从内往外包
    devtools(
      (set, get) => ({ /* ...你 3.1 写的那套... */ }),
      { name: 'records-store' },        // devtools 面板里显示的名字
    ),
  ),
)
//洋葱模型：外层中间件先拦截 set，处理完往里传——
// 每一层滤芯都对"水流"加工一次（净水器串联类比）
```

### 二、常用中间件三件

**① persist（对照你 Pinia 的持久化插件——完全同构）**

```ts
import { persist, createJSONStorage } from 'zustand/middleware'

export const useRecordsStore = create<RecordState>()(
  persist(
    (set) => ({ /* state + actions */ }),
    {
      name: 'records-store',                                   // storage 的 key（必填）
      storage: createJSONStorage(() => localStorage),          // 存哪（默认 localStorage）
      partialize: (s) => ({ filterType: s.filterType }),       // 只持久化哪些字段（对照 Pinia 的 pick）
      // ⚠️ 默认会把整个 state 存下来——action 函数会被忽略（不可序列化），
      //    但 records 这类大数组要不要存要想清楚（数据主权在服务端就别存）
    },
  ),
)
```

三问框架看 persist 的本质：它挂在"怎么通知"的链路上——每次 set 后写 storage，初始化时读回**合并**进 state。和你用过的 `piniaPluginPersistedstate` 逐概念对得上（key / pick / 读回）。

**② devtools（Zustand 也能接 Redux DevTools！）**

```ts
import { devtools } from 'zustand/middleware'
create<RecordState>()(devtools((set) => ({...}), { name: 'records-store' }))
// 浏览器装 Redux DevTools 扩展 → 能看到这个 store 的每次 set——时间旅行可用
// （对照 1.1：时间旅行需要"每次变更可序列化"——Zustand 的 set 就是 action 名单）
```

**③ immer（让 set 里可以"直接改"）**

```ts
import { immer } from 'zustand/middleware/immer'
create<State>()(immer((set) => ({
  records: [],
  addRecord: (r) => set((s) => { s.records.push(r) }),  // ← 直接 mutate！
  // immer 用 Proxy 记录你的改动，帮你产出不可变新对象
  // 对照 1.1 的光谱：Zustand 默认"不可变手动"，immer 中间件 = 借来 Proxy 的自动追踪体验
})))
```

**叠加顺序**：两个都要拦截 set，一般 **devtools 放外层**（保证 devtools 记录到最终合并结果）；拿不准时按洋葱模型推演——**外层先接触 set**。顺序细节官方文档为准。

### 三、面试叙事：Zustand 为什么爆火（背这三条）

1. **零仪式**：无 Provider、无 actionType、无 reducer 模板——API 一页纸，心智 = "React setState 的全局版"。对比 Redux（RTK 之前 action/constants/reducer 三处声明）生态位完全重合的部分
2. **粒度可控且天然正确**：selector 手动定粒度，不写 selector 也不会错得离谱（默认行为可用）；对比 react-redux 旧版 connect 的 mapState 优化负担、Mobx 的类体系学习成本
3. **模块级单例 = 组件外可用**：`useStore.getState()` 在任何 JS 上下文可用（axios 拦截器、工具函数、事件回调）——对比 react-redux 离开 React 树就失灵。你 4.10 的 token 拦截器读 localStorage 的妥协，Zustand 场景下可以直接 getState()

**补充第四条（加分）**：pmndrs 团队（react-three-fiber 作者群）出品、TS 体验极佳、体积 <1kB。

### 四、边界：什么时候不用 Zustand（防坑）

- **服务端状态别用它管**——这是最高频误用：接口数据需要缓存、失效、去重、后台刷新（3.3 学的 TanStack Query 全套），Zustand 只是"全局 useState"。面试标准答案：**"客户端状态 Zustand，服务端状态 TanStack Query，两者互补不替代"**
- 大团队需要**强规范**（强制 action 语义、完整时间旅行审计、中间件生态）→ RTK 的价值区
- 复杂派生关系想要**自动追踪**（不想手写 selector）→ Mobx 的价值区

### 五、一句话总结

- 中间件 = 洋葱滤芯：persist（存取）、devtools（时间旅行）、immer（可变写法），柯里化形状承接
- persist 与 Pinia 插件同构：name/partialize(pick)/读回合并
- 爆火三叙事：零仪式 / 粒度可控 / 组件外可用；边界：服务端状态归 TanStack Query

## 考核过程
任务 1（编码）：给 3.1 的 useRecordsStore 加两个中间件——
  - persist：只持久化 filterType（partialize），key 随意
  - devtools：加上并起名
  - 写出完整柯里化嵌套形状（谁包谁）
  
  ```ts
  type RecordState = {
    records: RecordItem[]
    filterType: 'all' | 'expense' | 'income'
    setFilter: (t: RecordState['filterType']) => void
    addRecord: (record: Omit<RecordItem,'id'>) => void
    resetAll:()=> void
  }
  export const useRecordsStore = create<RecordState>()(persist(
    devtools(
      (set,get)=>({
        records: [],
        filterType: 'all',

        setFilter:(t)=> set({filterType: t}),

        addRecord:(record) => set(s => ({records:[...s.records,record]})),

        resetAll: ()=> set({records:[],filterType: 'all'},true)
      }),
      { name: 'records-store' },
  ),
  { name: 'records-store' , partialize: (s) => ({ filterType: s.filterType }) }))
  ```

  任务 2（口头两问）：

  ① 面试官问：“你们项目为什么选 Zustand 不用 Redux/RTK？”——用爆火三叙事回答，但每条后面带一句“RTK 什么时候反而更合适”（防止背成 religious war）

  - **零仪式**：无 Provider、无 actionType、无 reducer 模板，API 一页纸——中小项目接入成本几乎为零。
    （对照：如果是大团队、需要**强规范**——强制 action 语义、变更审计、统一代码风格——RTK 的 createSlice 约定反而能防住"各写各的"）
  - **粒度可控且天然正确**：selector 手动定订阅粒度，不写也不会错得离谱。
    （对照：如果需要**完整的时间旅行生态**（Redux DevTools 全套 action 面板/回放/导入导出），RTK 的 devtools 集成更标准化）
  - **模块级单例，组件外可用**：`useStore.getState()` 在 axios 拦截器、工具函数里直接用，离开 React 树不失灵。
    （对照：如果服务端请求也想纳入同一套体系——**RTK Query 与 store 一体化**（缓存/失效/重新注入标签），RTK 是更完整的全家桶）

  收尾句式（防 religious war）："我的场景是中小应用、客户端状态为主，所以选 Zustand；如果场景变成大团队强规范或服务端缓存深度集成，RTK 是更对的选择。"

  ② 同事说："我打算把所有接口数据都塞 Zustand 管，统一状态。"——这句话错在哪？

  - 把**客户端状态**和**服务端状态**混为一谈了。Zustand 的本质是"全局 useState"——只解决**客户端状态三问**（放哪/谁改/怎么通知）。
  - 但接口数据是**服务端状态的镜像**，它需要一套 Zustand 完全没有的机制：**缓存**（staleTime 内不发重复请求）、**失效**（mutation 成功后 invalidate 相关 key）、**请求去重**（同 key 并发请求合并成一个）、**后台刷新**（stale-while-revalidate：先渲旧数据再静默拉新）——这套东西 TanStack Query 内建，用 Zustand 管接口数据 = 每个接口手写这四件事，等于重造一个劣化版 TanStack Query。
  - 纠正句式："客户端状态（表单草稿、筛选条件、登录态）用 Zustand；服务端状态（接口数据）归 TanStack Query——两者互补不替代。"

（此段在考核通过后追加填充）

### 点评

**任务 1**：首版把 `{name, partialize}` 挂到了 **devtools 的 options** 位置，persist 裸奔（partialize 未生效）；提示"谁的功能找谁——每层中间件的 options 跟在自己的 setup 之后"后自行修对：`persist(devtools(setup, {name}), {name, partialize})` ✅。判断口诀沉淀：**partialize 是持久化概念，就在 persist 参数表里**。

**任务 2**：①三叙事背出但缺题目要求的"RTK 何时更合适"对照；②只答结论缺机制理由。由 LLM 补参考稿（①每条叙事对照 RTK 价值区 + 防 religious war 收尾句式；②客户端/服务端状态区分 + Zustand 缺失的四大机制：缓存/失效/去重/后台刷新）。记参考补充，复习重点：**"客户端状态 Zustand，服务端状态 TanStack Query，互补不替代"** 这句面试标准答案。

---
通过时间：2026-11-25
