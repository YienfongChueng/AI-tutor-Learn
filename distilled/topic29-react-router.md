# 3.2 React Router：路由与嵌套/守卫

**它解决什么问题**
SPA 需要 URL 与组件映射、嵌套布局、登录保护。React Router 用 JSX 组件树表达路由，守卫没有框架设施、自己写组件。

**核心机制**
- `<Routes><Route element>` JSX 嵌套，路由即组件树
- 布局路由 + `<Outlet/>`（= router-view）：切子路由布局不卸载
- 编程式全靠 Hook：useNavigate / useParams / useSearchParams
- 守卫 = RequireAuth 条件渲染组件 + `<Navigate state={{from}} replace>`
- replace 操作浏览器历史栈：替换当前条目，后退跳过

**最容易被追问的点**
- "未登录访问受保护路由发生什么？" -> RequireAuth 返回 Navigate -> 重定向 /login，Layout（若 /login 在外）随之卸载
- "布局里的 state 什么时候丢？" -> 跨出布局路由时；布局内部切换永远保留（洞里组件的 state 必丢）
- "登录后为什么用 replace？" -> 后退不该回登录页；与 state 无关，是历史栈操作
- "组件外能 navigate 吗？" -> 不能随手调，useNavigate 只在组件内（vs vue-router 全局实例）
- "守卫和 beforeEach 的思想差异？" -> Vue 集中式门卫，React 分布式乐高

**一句话类比**
布局路由像商场的固定店面框架，Outlet 是换展位：展位随便换（子路由 state 必丢），框架不动（Layout state 保留）；跨出商场（去 /login）整栋拆除重建。
