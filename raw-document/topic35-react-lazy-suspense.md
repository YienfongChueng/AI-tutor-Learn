# 4.11 性能收尾（路由 lazy + Suspense）

## 讲解原文

### 知识铺垫

1. **纪律：先测量后优化**——优化前 `npm run build` 记录主包大小和警告（"术前片"）
2. `React.lazy(() => import('@/pages/...'))` 把组件变成按需加载的独立 chunk；`<Suspense fallback={...}>` 接住 chunk 加载期的空白
3. **路由切换是天然分割点**——用户没进过的页面没必要进首屏；Layout/Provider 是首屏骨架，不 lazy

### Suspense 的两个落点

- **推荐简单方案**：包 App.tsx 的 `<Routes>` 外——一处管所有 lazy 页（含 Layout 外的 /login）
- **进阶方案**：包 Layout 里 `<Outlet />` 外——切页时导航栏不闪（4.4"换页不换壳"延续），但管不到 Layout 外的 /login，login 要么不 lazy 要么另配 boundary
- 用户选了进阶方案（Layout/Outlet 外），LoginPage 保持 static import——恰好自洽闭环

### 术前/术后对比（实测）

| 指标 | 术前 | 术后 |
|------|------|------|
| JS 主包 | 637.73 kB（gzip 208.68），单包 | 299.54 kB（gzip 97.12），↓53% |
| chunk 数 | 1 | ~17 个按需分片 |
| 500kB 警告 | 有 | 消失 |

### build 输出的隐藏细节（打包器自动行为）

- **共享 chunk 自动提取**：AddOrEditForm 没被手动 lazy，但 Add/Edit 两页共用 → rolldown 抽成 4.62 kB 公共 chunk；`AddRecordFormPage` 只有 0.14 kB——纯"壳"，内容在 AddOrEditForm chunk 里
- **antd-mobile 按组件分片**：date-picker 52 kB 独立 chunk，只有用到它的页面才拉；统计页用户永远不用下载这 52 kB
- **直访 /stats 拉很多小 css/文件是正常的**：那是统计页的按需集合（0.6~4.6 kB each），对比术前是全量单文件；HTTP/2 并发小文件无压力，chunk 可跨页复用缓存

## 考核过程

### 思考题：为什么 lazy 粒度选"页面"而不是"每个小组件"？

**用户回答**：按页面符合用户体验，如果按照组件，每个小组件都要按需拉取，网络请求没回来时组件都是 loading 状态，用户体验不友好。

**点评**：通过（UX 角度正确）。补充两个连带伤害：①请求瀑布——chunk 依赖链可能串行拉；②缓存碎片化——微 chunk 依赖图复杂，改一行 hash 变一片。**判断法：只在用户"本来就预期等待"的边界上 lazy，切页面是唯一天然边界。**

### 验收

- build 术前/术后对比数据齐全 ✅
- Network：首次切统计页才拉 StatisticsPage chunk，第二次进入无请求、不闪 fallback ✅（验证 3.4 考点：chunk 已缓存则 Promise 立即 resolve）
- 刷新直访 /stats 正常，按需集合判断正确 ✅
- 顺带收掉 m2 遗留：NotFoundPage 升级 ErrorBlock + 回首页按钮 ✅
- tsc 0 错误 ✅

---
通过时间：2026-11-19
