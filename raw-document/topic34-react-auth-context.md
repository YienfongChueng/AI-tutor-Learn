# 4.10 mock 登录（Context + localStorage）+ RequireAuth 守卫

## 讲解原文

### 知识铺垫

1. **登录态 = 典型 Context 场景**：多页面共享（守卫/请求层/Layout）+ 低频变化——2.5 的两条判断全占
2. **单一真相源分工**：`localStorage` 只负责"刷新后还记得"（持久层），运行时真相是 Context state——初始化读一次播种，之后只信 state
3. **Context 两步接线法**（用户最初的困惑点）：①main.tsx 套 `<AuthProvider>`（全局供电）②组件里 `useAuth()` 取电。没人 import = 没通电的发电站

### auth.tsx 三段式模板

```tsx
interface AuthContextType { token: string | null; login: (...) => Promise<boolean>; logout: () => void }
const AuthContext = createContext<AuthContextType | null>(null)   // null = "没人管"占位

export function AuthProvider({ children }) {
  const [token, setToken] = useState<string | null>(() => localStorage.getItem('token'))  // 惰性初始化
  const login = async (u, p) => { /* 校验→500ms 假延迟→mock token→双写→return true */ }
  const logout = () => { /* 双清：removeItem + setToken(null) */ }
  return <AuthContext.Provider value={{ token, login, logout }}>{children}</AuthContext.Provider>
}

export function useAuth(): AuthContextType {
  const ctx = useContext(AuthContext)
  if (!ctx) throw new Error('必须在 AuthProvider 内使用')   // 提前暴露绕过，错误离病因最近
  return ctx
}
```

- **惰性初始化**：`useState(localStorage.getItem(...))` 的参数每轮渲染都求值（结果被丢弃），白读一遍；`() => ...` 只在首轮调一次
- **为什么还要 state 而不只写 localStorage**：localStorage 变化不广播——logout 清了档案，守卫不知道；state 变化才触发重渲染传播

### location.state 机制（用户主动提问）

**state 是"寄件人"塞的，不是目标页自己长的**：Link / navigate / Navigate 跳转时塞随行包裹 → 不进 URL、刷新即丢 → 目标页 `useLocation().state` 读。

```
未登录访问 /stats → RequireAuth 无 token
  → <Navigate to="/login" replace state={{ from: location.pathname }} />
  → 登录页 location.state = {from:'/stats'}
  → 登录成功 navigate(location.state?.from ?? '/')   // 直接敲 /login 时兜底回首页
```

RequireAuth 首版漏塞 state（3.2 考过仍漏）——state 不凭空出现，`Navigate` 不塞登录页永远收到 null。

## 考核过程

### 首版架构分叉（重要教学事件）

用户按骨架一次写对 auth.tsx（惰性初始化/双写/useAuth 判空 throw 全对），但接线时绕过 Context 走了纯 localStorage 路线：LoginPage 调 api/user.ts（json-server /users 查询）+ 手动 setItem，RequireAuth 直读 localStorage，Layout 无退出——**auth.tsx 成死代码**。用户主动提出"连接不上 auth.tsx，不知道怎么使用"。

**教学处理**：讲解两步接线法后改回 Context 路线（main.tsx 包 Provider、三处 useAuth、Layout 退出按钮），api/user.ts 保留（将来接真接口时把 mock 延迟换成调它）。

### 审查修正

- 🔴 `FormState { username: 'string' }` 字面量类型 → `username: string`（值只能是字符串 "string"）
- 🟡 登录成功跳 `/` 丢 state.from → `location.state?.from ?? '/'`
- 🟡 登录中无反馈 → loading state + Button loading（500ms 可重复点）
- RequireAuth 漏塞 state（见上）

### 做对的

axios 拦截器补 token（从 localStorage 读——axios 层在 React 树外拿不到 state，正是"localStorage 是持久层"的意义）；RequireAuth 套 Layout 层一处守卫管全部子路由，/login 在外天然不受守卫；tsc 0 错误。

### 验收（5/5 过）

未登录敲 /stats 被踢且 URL 无 from、登录回 /stats、刷新不丢登录态、退出后再访问仍被拦、请求头带 Bearer token。

## 补充归档（2026-11-19）：Context 登录完整接线流程

> 用户反馈"接线不太熟悉"，把本次从死代码改回 Context 路线的完整流程沉淀如下。

### 接线五步法：一写 · 二供 · 三取 · 四持久 · 五验证

**Step 1：写 Context 文件**（`src/context/auth.tsx`，三段式）

```
① interface AuthContextType   —— 声明广播值的形状（token + login + logout）
② createContext<T | null>(null) —— null 是"没人管"的占位初始值
③ AuthProvider                  —— 唯一持有 state 的地方（惰性初始化读 localStorage 播种）
④ useAuth()                     —— 取电入口（判空 throw，防绕过）
```

**Step 2：供——main.tsx 套 Provider**（全局通电，只此一处）

```tsx
<AuthProvider>                    {/* 放最外层：它不依赖 router/query */}
  <QueryClientProvider client={queryClient}>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </QueryClientProvider>
</AuthProvider>
```

规则：**所有要 useAuth 的组件必须挂在这棵树内**——漏包 = `useContext` 拿到 null → 有 throw 直接报错定位。

**Step 3：取——三个消费端，各取所需**

| 消费端 | 取什么 | 用途 |
|--------|--------|------|
| `RequireAuth`（App.tsx） | `token` | 没有就 Navigate 踢到 /login（塞 state.from） |
| `LoginPage` | `login` | 登录成功后 Context 自动广播，页面**不再手动写 localStorage** |
| `Layout` | `logout` | 退出按钮：logout() + navigate('/login') |

**Step 4：持久层——axios 拦截器读 localStorage**（client.ts）

```ts
client.interceptors.request.use(config => {
  const token = localStorage.getItem('token')   // React 树外拿不到 useAuth，只能读档案
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})
```

**Step 5：验证清单**（自上而下过一遍数据流）

- 未登录敲受保护路由 → 被踢 /login（URL 无 from）→ 登录 → 回 state.from
- 刷新 → 惰性初始化从 localStorage 播种 → 不丢登录态
- 退出 → 双清 → 守卫立刻拦（state 广播生效的证据）
- Network 请求头带 Bearer token（持久层生效的证据）

### 数据流全景

```
                    ┌── 写入：login() ──→ setState（广播→守卫/Layout 重渲染）
登录页 useAuth ──→──┤
                    └── 同时 localStorage.setItem（持久）
                              ↓
刷新 → 惰性初始化播种 ←──── localStorage
                              ↓
main.tsx AuthProvider ──广播──→ RequireAuth(token) / Layout(logout)
                              ↓
axios 拦截器 ──读──→ localStorage ──→ 请求头 Bearer token
```

### 当时的踩坑回顾

| 坑 | 现象 | 根因 |
|----|------|------|
| auth.tsx 写了没人用（死代码） | "连接不上" | Context 不是自动生效的——**不 import 不通电** |
| LoginPage 手动 localStorage.setItem | 双写两处、绕过广播 | 持久化写进 Provider 的 login 里，页面只管调 |
| RequireAuth 直读 localStorage | 能跑但不响应 logout | 运行时真相是 Context state，不是档案 |
| RequireAuth 漏塞 state | 登录后回不了原页 | state 是寄件人塞的，Navigate 不塞目标页永远是 null |

### 记忆口诀

> **一写 Context、二供 Provider、三处 useAuth、四持久拦截器、五按数据流验证。**
> 页面里永远不直接摸 localStorage 的 token——摸档案的活儿只属于 Provider（写）和拦截器（读）。

---
通过时间：2026-11-19
