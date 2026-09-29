# 3.4 性能优化清单（memo/lazy/Suspense）

**它解决什么问题**
React 应用越写越卡、首屏 bundle 越打越大。需要一个按性价比排序的手段清单，避免上来就给全世界包 memo。

**核心机制**
- 优化金字塔：测量 → 状态结构 → lazy 分割 → 虚拟化 → memo 三件套
- 状态下沉 + 派生不存储：免费、最值钱（让渲染根本不发生）
- `lazy(() => import())` = 单独 chunk；Suspense fallback 兜下载期
- 分割甜点位：按路由 + 按重组件，不过分割
- React 无内置 keep-alive：切路由即卸载，state 丢失

**最容易被追问的点**
- "为什么不先包 memo？" -> 越靠金字塔顶部越是打补丁：有开销、逐点修；状态结构是一次修复全局受益
- "第二次访问还看得到 fallback 吗？" -> 基本看不到；fallback 只跟 chunk 在不在本地有关，chunk 已缓存 Promise 立即 resolve
- "发版后为什么 fallback 又出现了？" + 弱网 -> chunk 名带内容 hash，内容变 hash 变 = 缓存键失配重新下载
- "返回列表筛选丢了怎么办？" -> URL query（useSearchParams，还能分享/后退还原）/ store / 状态放路由外（Layout）
- "万行列表卡？" -> 虚拟化只渲染视口内条目；几百行内通常分页就够

**一句话类比**
优化像看病：先拍片（Profiler）再治病；状态下沉是改生活习惯（免费断根），memo 三件套是吃药（按疗程有副作用）。
