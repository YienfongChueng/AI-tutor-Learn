# 3.4 性能优化清单（memo / lazy / Suspense）

## 讲解原文

> 承接 2.6（三件套是手术刀不是营养品）和 3.2 埋点（路由懒加载）。本节把优化手段排成一张**按性价比排序的清单**--顺序本身就是考点。

### 一、总原则：优化金字塔

```
        ╱ memo 三件套 ╲        ⑤ 最贵、最后动（2.6 已学）
       ╱  虚拟化长列表  ╲       ④ 长列表才需要
      ╱  code splitting ╲      ③ 本节主角：lazy + Suspense
     ╱  状态结构设计     ╲      ② 状态下沉/派生不存储：免费，收益最大
    ╱  测量 Profiler      ╲     ① 永远第一步：没有数据就没有优化
```

**为什么这个顺序**：越往下越是"改设计"（一次修复，全局受益，不引入额外代码）；越往上越是"打补丁"（逐点优化，自带开销和复杂度）。新人最大的错误是直接跳到⑤给全世界包 memo。

### 二、第一优先级：状态结构（免费但最值钱）

两个已经在模块 2 学过的招，这里正式入册：

**① 状态下沉（colocation）**：state 放得离用它的组件越近越好。

```tsx
// ❌ query 放在 App -> 整棵树跟着输入陪跑
// ✅ query 放进 SearchBar 组件 -> 只有 SearchBar 自己重渲染
function SearchBar() {
  const [query, setQuery] = useState('')
  ...
}
```

**② 派生不存储**：能算出来的不存 state（2.4 的 `log.length`）。

这两招不动一行"优化 API"，却直接消灭了大部分重渲染--**最好的优化是让渲染根本不发生**。

### 三、code splitting：React.lazy + Suspense

**问题**：默认打包是一个大 bundle，用户打开首页要把"订单页/设置页/图表库"全部下载完才能用。

**手段**：路由级分割（收益最大、成本最低）。

```tsx
import { lazy, Suspense } from 'react'

const Orders = lazy(() => import('./pages/Orders'))    // 注意：返回 import() 的 Promise
const Settings = lazy(() => import('./pages/Settings'))

<Routes>
  <Route path="/orders" element={
    <Suspense fallback={<PageSkeleton />}>            {/* chunk 下载期间的占位 */}
      <Orders />
    </Suspense>
  } />
</Routes>
```

机制：`import()` 是 Vite 的**动态导入**，被它引用的模块（及其依赖）会**单独打成一个 chunk 文件**。首页 bundle 里只剩一小段"加载 Orders chunk"的指令，用户访问 `/orders` 时才下载。

**Vue 对照**（你已会的映射）：

| React | Vue |
|---|---|
| `lazy(() => import(...))` | `defineAsyncComponent(() => import(...))` |
| `<Suspense fallback>` | `<Suspense><template #fallback>` |
| 路由懒加载 | `component: () => import(...)` |
| 组件预取名 | `defineAsyncComponent({ loader })` |

**Suspense 边界的粒度**：fallback 是"整块替换"式的体验（类似路由切换的白屏/骨架屏）。可以嵌套多层 Suspense，让页面骨架先出、局部数据区再各自等待。

**分层建议**：

1. **路由级**（默认做）：每个页面一个 chunk
2. **重型组件级**（按需做）：富文本编辑器、图表库这种几百 KB 的家伙，`lazy` 到"用到时"（点了编辑按钮才加载编辑器）
3. 不要过分割：每个 chunk 都有请求/解析开销，"按路由 + 按重组件"是甜点位

### 四、虚拟化长列表（清单第④位）

渲染 10000 条 = 10000 个 DOM 节点 = 卡死。虚拟列表只渲染**视口内的 20 条**，滚动时复用 DOM（顶部补/尾部裁）。库：`react-window` / `react-virtualized` / TanStack Virtual。**判断线**：列表超过几百行且真卡再上，普通分页常常已够。

**Vue 对照**：v-virtual-scroller / vueuse 的 useVirtualList，思想完全一致。

### 五、完整清单（打印贴墙版）

| 序 | 手段 | 成本 | 何时用 |
|----|------|------|--------|
| 0 | **Profiler 测量** | 0 | 每次优化的第一步，找热点组件 |
| 1 | 状态下沉 | 0（改设计） | 默认遵守 |
| 2 | 派生不存储 | 0 | 默认遵守 |
| 3 | key 稳定（1.4） | 0 | 列表渲染默认遵守 |
| 4 | 路由级 lazy | 低 | 项目起步就配 |
| 5 | 重组件 lazy | 低 | 图表/编辑器等 |
| 6 | memo 三件套（2.6） | 中 | 测出重渲染热点后 |
| 7 | 虚拟化列表 | 中 | 千行级列表 |
| 8 | useDeferredValue / startTransition | 中 | 输入即时反馈卡顿时（搜索联想） |

第 8 项是 React 18 并发特性，s5 源码篇会正式讲，这里知道名字和场景即可。

### 六、Vue 迁移者特别提示

- React **没有内置 keep-alive**：路由切走 = 组件卸载 = state 丢失（3.2 亲测）。要缓存得用社区库或自己把状态放路由外/store。这是 Vue 迁移者最常见的"怎么又重新请求了"的来源
- Vue 的 computed 自动缓存心智在 React 里没有免费等价物：**派生计算默认每次渲染重算**，贵了才 useMemo（2.6 纪律）

## 考核过程

### 题目 1：优化排序 / lazy 机制 / 场景配对 / keep-alive 迁移

**用户回答**：
1. 排序：B 测量 → C 状态下沉 → D 代码分割 → E 虚拟列表 → A memo 三件套 ✅
2. (a) 单独 chunk 减轻首屏体积 ✅；(b) "第二次也会看到 fallback，页面没渲染完成都会看到" ❌
3. 四场景配对：状态下沉 / 虚拟化 / 重组件 lazy / 先测量 ✅
4. React 无 keep-alive，切路由组件卸载状态丢失；状态放路由外或 store ✅

**点评**：仅 (b) 错。fallback 出现的唯一原因是 import() Promise pending（chunk 不在本地），与"页面渲染快慢"无关；第二次访问 chunk 已在浏览器缓存，Promise 近乎立即 resolve，fallback 一闪难见。补一个最佳实践：筛选条件放 URL query（useSearchParams），免费获得分享链接/后退还原。

### 补充对话

- 微调确认（弱网 + 发版后 chunk 文件名变了，会看到 fallback 吗）：✅ 答对--文件名变 = 缓存键失配 = 重新下载。顺势讲透 Vite chunk 名加 hash 的设计目的（内容变 hash 变，自动击穿旧缓存）

---
通过时间：2026-09-03（s3 工程化最后一节，模块完成）
