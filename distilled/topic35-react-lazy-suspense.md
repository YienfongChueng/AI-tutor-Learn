# 4.11 路由 lazy + Suspense

**它解决什么问题**
所有页面打进一个 637kB 的包，首屏就要下载一大半用不到的代码——路由边界是现成的切割线。

**核心机制**
- `React.lazy(() => import())` 把页面切成独立 chunk，按路由按需拉取
- Suspense 接住 chunk 加载空白，只在**首次**访问出现（chunk 缓存后 Promise 立即 resolve）
- 路由边界 = 天然分割点；Layout/Provider 是首屏骨架，不 lazy
- 打包器自动提取共享 chunk（公共组件抽公共盘，壳页面只有 0.14kB）
- 先测量后优化：build 前后对比主包大小与警告

**最容易被追问的点**
- "为什么 lazy 页面不 lazy 组件？" → 组件级切成微 chunk：请求瀑布、fallback 遍地、缓存碎片化；只在用户"本来就预期等待"的边界 lazy，切页面是唯一天然边界
- "fallback 会一直闪吗？" → 只首次加载出现，之后 chunk 命中缓存立即 resolve
- "直访 /stats 拉一堆小 css 正常吗？" → 正常，那是该页的按需集合；对比术前是全量单文件，HTTP/2 小文件并发无压力
- "LoginPage 要 lazy 吗？" → 可以不；lazy 了就必须在 Layout 外另配 Suspense boundary
- "Suspense 放哪？" → Routes 外（简单全覆盖）或 Outlet 外（导航不闪，管不到 login）

**一句话类比**
lazy 像自助餐按取餐口上菜：路由是取餐口，进哪个口拿哪个盘；打包器还会把两个口都点的菜自动拼成共享拼盘（共享 chunk），谁都能夹。
