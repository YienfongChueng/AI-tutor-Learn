# 4.10 mock 登录（Context + RequireAuth）

**它解决什么问题**
登录态要跨守卫/请求层/多页面共享，且变化要广播——props 层层传太脏，纯 localStorage 不通知。

**核心机制**
- AuthProvider 唯一持有 token state，惰性初始化从 localStorage 播种
- 组件 useAuth() 取电；createContext(null) + 判空 throw 提前暴露绕过
- login/logout 双写：state（广播）+ localStorage（持久）
- RequireAuth 顶层守卫：无 token 时 Navigate 塞 state.from 踢回 /login
- axios 拦截器从 localStorage 读 token（React 树外拿不到 state）

**最容易被追问的点**
- "为什么不只存 localStorage？" → localStorage 变化不广播，logout 清了守卫不知道；state 才触发重渲染
- "location.state 是哪来的？" → 跳转方 Navigate 塞的随行包裹，不进 URL、刷新即丢，目标页 useLocation().state 读
- "登录后为什么能回原页？" → RequireAuth 塞 state.from，登录页 `state?.from ?? '/'` 兜底
- "useState 为什么要惰性初始化？" → 普通参数每轮渲染都求值白读一遍，`() =>` 只在首轮执行
- "守卫放哪层？" → 套 Layout 父路由一处管全部子路由，/login 放外面天然豁免

**一句话类比**
Context 是广播电台（Provider 实时播 token 变化），localStorage 是录音带（断电后重放，刷新播种）；axios 拦截器在电台覆盖不到的地方（React 树外），只能听录音带——所以"运行时信 state，持久层信录音带，初始化时用录音带给电台开台"。
