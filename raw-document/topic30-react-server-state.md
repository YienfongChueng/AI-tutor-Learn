# 3.3 数据请求与服务端状态（async / TanStack Query 选讲）

## 讲解原文

> 承接 2.7：你写的 `useFetch` 能跑，但把它放进真实项目三天你就会遇到三堵墙。本节讲清楚墙在哪、TanStack Query 怎么拆，以及一个重要的概念分层：**服务端状态 ≠ 客户端状态**。

### 一、useEffect + useFetch 的三堵墙

```tsx
const { data } = useFetch('/api/user/1')
```

**墙 1：重复请求。** 头像组件要 user，侧边栏也要 user，两个组件各调一次 `useFetch('/api/user/1')` -> 同一条数据**请求了两遍**。你的 useFetch 没有任何跨组件共享机制。

**墙 2：没有缓存。** 用户从 `/orders` 切到 `/` 再切回来，Orders 组件重新挂载 -> `useEffect` 重跑 -> **接口重新请求**，用户盯着 loading 转圈--明明 5 秒前刚拿过这份数据。

**墙 3：样板无底洞。** loading / error / 重试 / 分页 / 乐观更新……每个请求点都要手写一遍，写对了还得处理竞态（模块 2 你已经体验过这个心智负担）。

### 二、根因：它们是两种状态

| | 客户端状态 | 服务端状态 |
|---|---|---|
| 例子 | 弹窗开关、输入草稿、主题 | 用户信息、订单列表、商品详情 |
| 真相 | **你拥有**的数据，你说了算 | **远端数据的本地快照**，随时会过期 |
| 挑战 | 怎么组织、怎么避免多余渲染 | 怎么缓存、何时失效、怎么去重、怎么同步 |

`useState` 管前者天然合适；管后者就等于**用管理"自己的数据"的工具去管"别人的数据的照片"**--缓存、失效、去重全都得手搓。这就是三堵墙的根源。

### 三、TanStack Query：把墙拆成配置

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

// 读：
const { data, isLoading, error } = useQuery({
  queryKey: ['user', id],              // 缓存键（数组，可带参数）
  queryFn: () => fetch(`/api/user/${id}`).then(r => r.json()),
  staleTime: 60_000,                   // 1 分钟内视为新鲜，不重新请求
})

// 写：
const queryClient = useQueryClient()
const mutation = useMutation({
  mutationFn: (newUser) => api.updateUser(newUser),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['user'] })   // 写成功 -> 让相关缓存过期
  },
})
```

对照你的三堵墙看它怎么拆：

| 墙 | TanStack 的解法 |
|---|---|
| 重复请求 | 同 `queryKey` 的并发 useQuery **自动合并成一次请求**，结果共享 |
| 没有缓存 | 全局缓存池；`staleTime` 内直接回缓存；过期后**先给旧数据再后台刷新**（stale-while-revalidate，切回来秒出内容） |
| 样板 | loading/error/重试/焦点重新验证/窗口聚焦刷新全是内置开关 |

**心智模型的转变**：你不再"管理一次请求"，而是**声明一块缓存**（key + 获取函数 + 新鲜度），React Query 负责何时请求、何时复用、何时失效。`invalidateQueries` 是核心思想：**写操作不直接改缓存，而是宣布缓存作废，让读端自己重新拉**。

### 四、Vue 对照

- TanStack Query 有同门 **vue-query**（@tanstack/vue-query），API 几乎一致--学的是思想，两个框架通用
- Vue 里你更可能手写 `composable + 模块级 ref 缓存`（把 ref 放 setup 外面当全局缓存），本质是在手搓一个简陋版 query--墙 1 能解，墙 2/3 还是自己扛
- SWR 是同类竞品（Vercel 出品），思想一致，选一个学透即可

### 五、决策线

```
请求 < 5 个、无共享无缓存诉求     -> useEffect + 手写 useFetch 够用（模块 2 技能）
列表/详情/多组件共享/频繁增删改查 -> TanStack Query
超大型应用全局状态 + 服务端状态混合 -> 阶段 3 状态库里再谈（Zustand 管客户端态 + Query 管服务端态是主流组合）
```

**一句话：useState 装你的数据，TanStack Query 装服务端数据的照片，别让一个工具干两份活。**

---

## 补充详解：TanStack Query 从零到能用（应用户要求追加）

> 面向零基础，比正文细一档。以下基于 **v5**（2026 主流版本，与 v4 有几处 API 差异，文末标注）。

### 补1. 接入：全局一个 QueryClient

```tsx
// main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,        // 全局默认：缓存 1 分钟内视为新鲜
      retry: 2,                 // 失败自动重试 2 次（墙 3 的"重试"内置了）
    },
  },
})

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </QueryClientProvider>
  </StrictMode>
)
```

`QueryClient` 就是那个"全局缓存池"的实体（一个 Map，key 是 queryKey）。Provider 把它放进 Context--对照 Vue：相当于你在模块顶层 `export const queryClient = new QueryClient()` 然后 composable 里直接用，React 走了标准 Provider 路线。

### 补2. useQuery 的完整生命周期

```tsx
const { data, isPending, isError, error, isFetching, refetch } = useQuery({
  queryKey: ['orders', { page: 2, status: 'paid' }],
  queryFn: ({ signal }) =>
    fetch(`/api/orders?page=2&status=paid`, { signal }).then(r => r.json()),
})
```

一次查询的状态机（v5 语义）：

```
isPending（还没拿到过数据）
   │  queryFn 发出请求（isFetching=true）
   ├─ 成功 ──> 有 data，isPending=false
   │            此刻 data 进入缓存，staleTime 倒计时开始
   └─ 失败（重试耗尽）──> isError=true, error 有值
                            （注意：data 保持 undefined，不回退旧值，除非配置了 placeholderData）
```

**易混三兄弟**：

| 字段 | 含义 | 什么时候 true |
|------|------|--------------|
| `isPending` | 从未拿到过数据 | 只有首次（缓存也没有） |
| `isFetching` | 正在请求中 | 首次 + 每次后台刷新 |
| `isLoading` | = isPending && isFetching | v5 里是组合值 |

典型 UI：`isPending` 显示骨架屏；`isFetching && !isPending` 显示"刷新中"小角标（数据还在，不闪骨架屏）。

`queryFn` 拿到的参数里有 `signal`（AbortSignal）--传给 fetch，缓存失效/组件卸载时 Query **自动中止旧请求**。你在 2.2 手写的 ignore 旗标，这里是一等公民参数。

### 补3. queryKey：缓存世界的身份证

**key 的设计规则：影响请求结果的每个变量都必须进 key。**

```tsx
// ❌ id 没进 key：切换用户时读的还是张三的缓存！缓存串台
useQuery({ queryKey: ['user'], queryFn: () => fetch(`/api/user/${id}`) })

// ✅ id 进 key：不同 id = 不同缓存条目；id 变化 = 自动重新请求
useQuery({ queryKey: ['user', id], queryFn: () => fetch(`/api/user/${id}`) })
```

**key 变化 = 自动重新请求**。这就是"声明式数据获取"的核心：你不写"什么时候请求"，你声明"这块缓存由哪些变量决定"，变量变，Query 自己去拉。对照 2.2：queryKey 干的就是 useEffect deps 的活，但作用域是**全局缓存**而不只是当前组件。

层级结构是惯例（和路由路径一个道理）：

```tsx
['orders']                          // 所有订单
['orders', { page: 1 }]             // 第 1 页
['orders', 'detail', 42]            // 42 号详情
['user', 1, 'permissions']          // 用户 1 的权限（嵌套资源）
```

### 补4. 缓存的状态与三个时钟

一条缓存条目的一生：

```
活跃（有组件在用）
  │  组件全部卸载 ──> 不活跃（inactive）
  │                    gcTime（默认 5 分钟）内没再被用 ──> 垃圾回收，数据删除
  │
  └─ 无论活跃与否，拿到数据后经过 staleTime ──> 过期（stale）
```

| 参数 | 管什么 | 默认 | 类比 |
|------|--------|------|------|
| `staleTime` | 数据多久算"不新鲜" | 0（立刻过期） | 食品保质期 |
| `gcTime` | 没人用之后缓存留多久 | 5 分钟 | 冷藏保质期 |

过期 ≠ 删除。**过期的数据还能用**（切回页面先秒显旧数据，再后台刷新换新）；垃圾回收了才是真的没了（重新走骨架屏）。

**什么时候触发重新请求**（前提：缓存已 stale）：

1. 挂载了新的同 key 组件
2. queryKey 变化
3. 窗口重新聚焦（`refetchOnWindowFocus`，默认开--切回浏览器 tab 自动更新，很惊艳也吓人，可关）
4. 网络恢复（`refetchOnReconnect`）
5. 手动 `refetch()` / `invalidateQueries()`

### 补5. useMutation 与失效的艺术

```tsx
const qc = useQueryClient()

const updateName = useMutation({
  mutationFn: (name: string) => api.patchUser(id, { name }),

  // 乐观更新：不等服务器，先改缓存，失败再回滚
  onMutate: async (name) => {
    await qc.cancelQueries({ queryKey: ['user', id] })
    const prev = qc.getQueryData(['user', id])           // 存旧值
    qc.setQueryData(['user', id], (old) => ({ ...old, name }))  // 直接改缓存 -> 界面立刻变
    return { prev }
  },
  onError: (_e, _v, ctx) => qc.setQueryData(['user', id], ctx.prev),  // 回滚
  onSettled: () => qc.invalidateQueries({ queryKey: ['user', id] }),  // 最终以服务器为准
})
```

`invalidateQueries` 的 key 匹配是**前缀匹配**：

```tsx
qc.invalidateQueries({ queryKey: ['orders'] })
// 命中：['orders']、['orders', {page:1}]、['orders', 'detail', 42] ... 全部作废
```

所以 key 的层级设计直接决定失效粒度--这也是补 3 说"层级是惯例"的深层原因：**key 树 = 你的失效武器库的目录**。

**两种更新缓存的哲学**：

```
失效流（invalidate）：宣布作废 -> 重新拉     ✅ 简单、永远是服务器真相，多一次请求
直写流（setQueryData）：直接改本地缓存        ✅ 少一次请求，但要自己保证形状对
```

默认写失效流，体验敏感的点（改名立刻生效）再上乐观更新。

### 补6. 三个高频实战开关

```tsx
// ① 条件请求：id 还没拿到时不发（依赖请求）
useQuery({
  queryKey: ['user', id],
  queryFn: () => fetch(`/api/user/${id}`),
  enabled: !!id,             // id 为空时挂起，不发也不报错
})

// ② 分页时保留上一页数据（切页不闪骨架屏）
const { data, isPlaceholderData } = useQuery({
  queryKey: ['orders', page],
  queryFn: () => fetchOrders(page),
  placeholderData: keepPreviousData,   // 翻页瞬间先显示上一页 + isPlaceholderData=true
})

// ③ 预取（下一页提前拉好）
qc.prefetchQuery({ queryKey: ['orders', page + 1], queryFn: () => fetchOrders(page + 1) })
```

### 补7. v4 -> v5 改名速查（面试防踩坑）

| v4 | v5 |
|----|----|
| `isLoading`（首载） | `isPending`（isLoading 变组合值） |
| `cacheTime` | `gcTime` |
| `onSuccess` 参数直接拿 data | 回调签名调整，`select` 推荐用于派生 |
| 数组 key 直接 `['x', {a}]` | 同款（哈希算法换了，行为一致） |

### 补8. 心智总结（对照表收尾）

| 你手写过的 | TanStack 对应物 |
|-----------|----------------|
| useFetch 里的 useEffect + deps | queryKey（全局作用域的 deps） |
| ignore 旗标 + cleanup | AbortSignal 自动中止 |
| 模块级 ref 缓存 | QueryClient 缓存池 + staleTime/gcTime |
| 手写 loading/error | isPending/isError + retry |
| 手动 refetch 按钮 | refetch() / invalidate / 聚焦刷新 |

**学完这节你应该能回答的问题**：为什么"服务端状态"值得单独一类工具管？--因为它自带**缓存/去重/失效/重试**四件你手写必丢三件的事。

## 考核过程

### 题目 1：queryKey bug / 缓存时间线 / 失效哲学 / 并发去重

**用户回答**：
1. 诊断：key 没变导致读旧缓存 ✅；修法写成 `['user', 'userId']`（字符串字面量）❌
2. 2 分钟后切回：旧数据秒出 ✅（漏了"同时后台刷新"）；10 分钟后：骨架屏重新请求 ✅
3. 前缀匹配命中三个 orders 系 key、不中 user ✅；"为什么推荐失效流"漏答（后自述"不懂"）
4. 请求一次 ✅，但理由说成"获取缓存数据"（实为并发去重，首次并无缓存）

**点评**：主体正确，三处修正：
- `['user', 'userId']` 是 **1.6 `key="item.id"` 同款错误第 2 次**：字符串字面量 vs 变量。口诀：引号里的永远不会变，变量才跟着 props 走
- 失效流三条理由重新讲解：①永远是服务器真相（直写只改手拼的字段，漏了服务器端其他变化）②不用手拼缓存形状 ③只需知道"什么变了"不用知道"变成什么样"；直写流唯一优点省请求，只配乐观更新用
- 并发去重 ≠ 缓存命中：两个观察者同 key 并发挂载，合并成一次网络请求结果共享

### 补充对话

- 用户主动要求 TanStack Query 详解（零基础），追加补1-补8 八个小节（接入/状态机/queryKey 规则/staleTime vs gcTime/乐观更新/enabled+分页+预取/v5 改名/手写对照表）
- 补答 1：正确 queryKey 一行 ✅
- 补答 3（userId 1→2 时 Query 做什么）：首次答串（重贴第 4 问答案），二次答"重新请求 user2 数据" ✅，补全"['user',1] 转 inactive 躺池等 gcTime"

---
通过时间：2026-09-03
