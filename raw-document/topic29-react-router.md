# 3.2 React Router：路由与嵌套/守卫

## 讲解原文

> 前置：vue-router 你熟（routes 配置 + `<router-view>` + beforeEach 守卫）。React Router（v6/v7，两者 API 基本一致，以下统称 v6）走的是** JSX 声明式**路线，映射过来很快。最大观念差异一句话：**它没有守卫，守卫是你自己写的一个组件。**

### 一、最小可用版

```tsx
// main.tsx
import { BrowserRouter } from 'react-router-dom'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
)
```

```tsx
// App.tsx
import { Routes, Route, Link } from 'react-router-dom'

function App() {
  return (
    <>
      <nav>
        <Link to="/">首页</Link>
        <Link to="/orders">订单</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/orders" element={<Orders />} />
        <Route path="*" element={<NotFound />} />     {/* 兜底，404 */}
      </Routes>
    </>
  )
}
```

### 二、与 vue-router 逐概念对照

| vue-router | React Router v6 | 备注 |
|---|---|---|
| `routes: [...]` 配置数组 + `createRouter` | `<Routes><Route/></Routes>` JSX 嵌套 | 路由即组件树 |
| `<router-view>` | `<Outlet />` | 嵌套路由的"洞" |
| `<router-link to>` | `<Link to>` | 几乎同款，最终都渲染 `<a>` |
| `router.push('/x')` | `navigate('/x')`（`useNavigate()` 拿到） | 编程式跳转 |
| `route.params.id` | `useParams()` | 路径参数 |
| `route.query.x` | `useSearchParams()` | 查询串，注意不是 useQuery |
| `redirect` / 路由 redirect 配置 | `<Navigate to="/x" replace />` | 声明式重定向组件 |
| `beforeEach` 全局守卫 | **没有**，自己封装组件 | 见下文 |
| 嵌套 children + 父路由不带组件 | 嵌套 `<Route>` + 父 `element` 里放 `<Outlet/>` | 也可用无 element 的布局路由 |

### 三、嵌套路由：布局路由 + Outlet

```tsx
<Routes>
  <Route path="/" element={<Layout />}>      {/* 布局路由 */}
    <Route index element={<Home />} />        {/* index：默认子路由，渲染在 / 时 */}
    <Route path="orders" element={<Orders />} />
    <Route path="orders/:id" element={<OrderDetail />} />
  </Route>
  <Route path="/login" element={<Login />} />
</Routes>

function Layout() {
  return (
    <div>
      <header>导航栏（访问 /orders 时它不消失）</header>
      <Outlet />                               {/* 子路由在这里换，布局不重挂载 */}
    </div>
  )
}
```

对照记忆：`<Outlet/>` 就是 `<router-view>`，布局路由就是带 `children` 的父路由 + App.vue 里的 `<router-view/>`。URL 从 `/` 切到 `/orders/3` 时，只有 Outlet 洞里的内容换，Layout 保持挂载（它的 state 不丢）。

### 四、动态参数与编程式跳转

```tsx
function OrderDetail() {
  const { id } = useParams()                    // /orders/:id 里的 id
  const navigate = useNavigate()
  const [params] = useSearchParams()            // /orders/3?tab=logs -> params.get('tab')

  return (
    <button onClick={() => navigate('/orders', { replace: true })}>
      返回（replace：不留历史记录，后退跳过本页）
    </button>
  )
}
```

注意 `navigate` 是 Hook 拿的（`useNavigate`），不能在组件外随便调--组件外跳转（如请求工具里 401 踢回登录）需要特殊处理（window.location 或把 navigate 传出去），这是和 vue-router 全局 `router` 实例的实打实差异。

### 五、守卫：React Router 的正确打开方式

没有 `beforeEach`。守卫 = **一个条件渲染的包装组件**：

```tsx
function RequireAuth({ children }) {
  const { user } = useAuth()                          // 你的登录态（Context/状态库）
  const location = useLocation()

  if (!user) {
    // 记住用户想去哪，登录后送回去
    return <Navigate to="/login" state={{ from: location }} replace />
  }
  return children                                     {/* 放行 */}
}

// 挂到路由上：
<Route path="/orders" element={
  <RequireAuth><Orders /></RequireAuth>
} />
```

```tsx
// Login 成功后送回原页：
const location = useLocation()
const navigate = useNavigate()
// 登录成功：
navigate(location.state?.from?.pathname ?? '/', { replace: true })
```

对照：vue-router 的 `beforeEach(to, from, next)` 是**集中式**门卫；React 的 RequireAuth 是**分布式**门卫--哪个路由要保护，就在哪包一层。没有对错，思想差异：Vue 给你框架级设施，React 给你乐高积木。

### 六、给 s3.3 埋点

路由懒加载（`lazy(() => import('./Orders'))` + `<Suspense>`）放到 3.4 性能优化一起讲，那边有完整的 code splitting 图景。

## 考核过程

实现 RequireAuth，要求：
  1. 未登录重定向到 /login，且记住用户本来想去哪
  2. 登录成功后能回到原来的目标页（写出 Login 里登录成功那行跳转代码）
   
   ```jsx
   function RequireAuth((children)) {
    const {user} = useAuth()
    const location = useLocation()
    if(!user) {
      return <Navigate to='/login' state={{from:location}} replace />
    }
    return children 
   }

   // login 跳转的代码
   const location = useLocation()
   const navigate = useNavigate()
   navigate(location.state?.from?.pathname ?? '/',{replace: true})
   ```
   navigate('/x') 和 navigate('/x', { replace: true }) 的区别是什么？登录成功后的跳转该用哪个，为什么？
   一个不替换路由，一个替换路由，登陆成功后跳转应该用替换路由，因为需要更新state数据

（此段在考核通过后追加填充）

### 题目 1：未登录访问受保护路由三连预测 + RequireAuth 编码 + replace 语义

**用户回答**：
1. (a) Outlet 替换成 Orders 内容 ❌；(b) 搜索框字不在，"login和layout不同组件，state被覆盖了"（结论对理由不准）；(c) Home 的 state 被清掉，各路由组件 state 独立 ✅
2. RequireAuth 代码逻辑全对（Navigate + state.from + replace + return children），Login 跳转行一字不差；但 `function RequireAuth((children))` 双括号语法错误（第 5 次手滑）
3. 登录后用 replace，理由"需要更新 state 数据" ❌

**点评**：未通过。
- (a) 漏审题："用户未登录"被无视--RequireAuth 返回 `<Navigate to='/login'>`，用户根本看不到 Orders，被踢到登录页（Layout 随之卸载）
- (b) 理由精确化：不是"覆盖"是**卸载销毁**--React 里 state 只有"组件活着在"和"组件死了没了"两种
- replace 语义：操作的是**浏览器历史栈**（push 入栈后退可回，replace 替换当前条目后退跳过）。登录用 replace 的理由是登录后按后退不该回到 /login，与任何 state 无关

### 补充对话

- 补答 1（已登录版：`/` → `/orders` → `/`，搜索框的字还在吗）：✅ 在，Layout 没卸载只是 Outlet 换内容。本质区别：这次全程没跨出布局路由
- 补答 2（列表进详情 push / 404 返回首页 replace）：✅ 全对

---
通过时间：2026-09-01（补答通过）
