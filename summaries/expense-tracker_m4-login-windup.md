# 记账本 · 里程碑4 登录与收尾（4.10-4.12）· 项目完结

> 本文包含 Mermaid 图表，推荐在支持 Mermaid 渲染的 Markdown 阅读器中查看。
> 完成日期：2026-11-19 · 对应节点：react.md 4.10 / 4.11 / 4.12 · **s4 实战项目全部 12 节点收官**

## 功能说明

- **mock 登录（4.10）**：AuthContext（Provider 惰性初始化播种 localStorage）+ useAuth + RequireAuth 顶层守卫 + axios 拦截器注入 Bearer token + state.from 回跳
- **性能收尾（4.11）**：6 页面路由 lazy + Suspense（Layout/Outlet 外，导航不闪），主包 637.73kB → 299.54kB（↓53%），500kB 警告消除；NotFoundPage 升级 ErrorBlock + 回首页
- **项目收尾（4.12）**：README（功能/选型/运行/结构）+ 技术亮点三篇（queryKey 目录树 / 单一真相源 / 登录态分工）

## 整体架构图（全项目视角）

```mermaid
graph TB
    subgraph 全局Provider
        P1[AuthProvider] -->|token 广播| RA[RequireAuth 守卫]
        P1 -->|login/logout| LP[LoginPage / Layout]
        P2[QueryClientProvider] --> RH[useRecords hooks]
        P3[BrowserRouter]
    end
    subgraph 页面层 lazy chunk
        RP[RecordsPage 列表+筛选+搜索]
        DP[RecordDetailPage :id]
        AP[Add/EditPage 复用 AddOrEditForm]
        SP[StatisticsPage 纯派生统计]
    end
    subgraph 状态源
        URL["URL query（type/date=唯一真相源）"]
        REC[("queryKey ['record','list',params]")]
        RDET[("queryKey ['record','detail',id]")]
    end
    RA --> 页面层
    URL -->|useSearchParams 派生| RP
    RH --> REC
    RH --> RDET
    SP -->|useRecords({}) 复用 lists 前缀| REC
    AP -->|mutation| MV[invalidate lists/detail 前缀]
    MV -.-> REC
    MV -.-> RDET
    RH --> API[api/ 层 client.ts 拦截器]
    API -->|Bearer token ← localStorage| JS[json-server db.json]
```

## 关键纪律（全项目沉淀的五条军规）

1. **前缀失效**：invalidate 参数是目录路径不是 key 集合，两棵树各打一次
2. **单一真相源**：URL 唯一真相源（读写分离）+ 派生不存储（统计零第二 state）
3. **括号口诀**：括号=现在就干，不加括号=交出去等人调（queryFn/onClick/事件处处适用）
4. **Hook 纪律**：use 前缀可被 lint 追踪；hook 返回回调，回调里零 Hook
5. **复盘讲事故**：只讲真实踩过的坑，泛化故事在"具体哪一行"露馅

## 踩坑总表（里程碑4 新增）

| 坑 | 根因 | 修法 |
|----|------|------|
| auth.tsx 死代码"连不上" | Context 不自动生效，不 import 不通电 | 两步接线：main.tsx 包 Provider + useAuth 取电 |
| onClick 里调 Hook | Hook 只能顶层同步调用 | hook 返回回调函数 |
| SearchBar 文字消失 | isLoading 早退卸载非受控组件 | 受控 value + placeholderData |
| RequireAuth 漏塞 state.from | state 是寄件人塞的 | Navigate state={{from: pathname}} |
| FormState 'string' | 字面量类型误写 | 去引号 |
| 主包 637kB + 警告 | 无代码分割 | 路由 lazy + Suspense，↓53% |

## 技术知识点清单（4.10-4.12 新增）

- Context 两步接线法（一写二供三取四持久五验证）
- 惰性初始化 useState(() => ...) 的求值时机差异
- location.state 机制：寄件人塞的、不进 URL、刷新即丢
- React.lazy + Suspense 两个落点（Routes 外 vs Outlet 外）与自洽条件
- rolldown 自动共享 chunk 提取（壳页面/公共组件/按组件分片）
- lazy 粒度判断法：只在用户自然等待的边界 lazy
- README 与复盘的表达方法论（讲事故不讲知识）

## 项目完结总结

四个月（2026-09-03 ~ 11-19），12 节点，从 Vite 脚手架到完整移动端应用：
MVP（CRUD+Query）→ 路由与详情（布局路由/两棵缓存树）→ 筛选与统计（URL 真相源/防抖进 key/派生统计）→ 登录与收尾（Context 守卫/路由分包/项目表达）。

## 进阶方向建议

- **测试**：Vitest + React Testing Library（首选单测 useDebounce / queryKeys 前缀语义）
- **乐观更新**：useMutation onMutate 乐观更新 + 失败回滚（TanStack Query 进阶必修）
- **真实后端**：把 json-server 换成自己写的 Node/Express（衔接全栈学习路线）
- **分页与虚拟列表**：账单量上来后 useQuery 分页 + antd-mobile 虚拟列表
- **部署**：Vercel/Netlify 上线 + mock 数据分离策略
- **下一站**：s5 源码原理（deep-dive 模式：Fiber 架构 / Hooks 链表 / 调度，衔接 Vue3 响应式对照）
