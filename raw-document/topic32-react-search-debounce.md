# 4.8 搜索防抖（自定义 Hook useDebounce 实战）

## 讲解原文

### 知识铺垫

核心新认知只有一条：**防抖后的值要参与 queryKey**。queryKey 是缓存的目录地址——把 debouncedKeyword 放进 key，每个搜索词各自有独立的缓存条目和请求；把原始输入放进去，每敲一个字都是新 key、新请求。

### useDebounce 机制（时间线）

```ts
export function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value)
  useEffect(() => {
    const handler = setTimeout(() => setDebounced(value), delay)
    return () => clearTimeout(handler)
  }, [value, delay])
  return debounced
}
```

```
输入:    a        ab        abc          ← 每敲一个字 value 变，effect 重跑
           │        │         │
cleanup:  清T1     清T2      （abc 后 300ms 没新输入）
                             ↓
                           T3 到点 → setDebounced('abc') → keyword 才变
```

每次 value 变化，cleanup 都掐掉上一个定时器——只有最后一次输入能活过 delay。

### 接线要点

- RecordsPage 两个 state：rawKeyword（原始输入，给 SearchBar）+ keyword = useDebounce(rawKeyword, 500)（给 Query）
- queryKey：`['records', filters, keyword]`；queryFn 把 `q: keyword` 混进 api 参数
- api 层：空字符串的 q 不传给 json-server（`...(keyword ? { q: keyword.trim() } : {})`）
- 配套优化：`placeholderData: (prev) => prev`——换 key 时借用上一个 key 的旧数据，isLoading 不闪

### 本节实战修复的 bug（讲解+排查记录）

**现象**：输入关键词触发查询后，搜索框文字清空、✕ 清除按钮消失。

**根因链**：keyword 进 key → 换新 queryKey → 无缓存 → isLoading=true → 命中早退分支 `if (isLoading) return <SpinLoading/>` → **整个页面子树被替换，SearchBar 被卸载**。SearchBar 未传 value 是非受控组件，文字存在自己实例内部 state 里，卸载即销毁；重挂后全新空框。rawKeyword 其实没丢（RecordsPage 未卸载，state 跟实例走）。

**修法**：① 受控化 `value={rawKeyword}`——文字由 props 驱动，卸载重挂后恢复（用户采用）；② `placeholderData: (prev) => prev`——换 key 不再触发 isLoading（数据层根治）；③ 结构层辅修（可选）：筛选区永远渲染，只有列表区在 loading/error/empty/list 四态间切换。

### 顺带纠正的 Hook 规范问题

`useInvalidateRecordLists` 内部调用 `useQueryClient()`（Hook），被在 `onClick={() => useInvalidateRecordLists()}` 里调用——**Hook 只能在组件顶层同步调用**（Rules of Hooks）。修法：hook 顶层拿 client、返回普通回调函数；组件顶层调用 hook 拿回调，onClick 里只调普通函数。此 bug 在函数改名为 use* 之前就存在（当时叫 invalidateRecordLists，无 use 前缀、问题被藏住）——ESLint rules-of-hooks 正是靠 use 前缀抓这类问题，tsc 查不出。

## 考核过程

### 思考题：为什么把防抖后的值放进 queryKey，而不是原始输入？

**用户回答**：因为把防抖值放入 queryKey，就会按照 delay 来触发拉数据；如果把原始值放入 queryKey，那么每敲一个字就拉取一次数据，达不到防抖效果。

**点评**：通过。核心正确——queryKey 变化 = 换缓存目录 = 发起新请求。补充连带伤害：直接放原始输入，每个中间字符串（k/ke/key）都会在缓存里留下一条没人再用的垃圾条目；防抖值进 key 同时解决请求风暴和缓存污染。

### 验收（Network 面板，5/5 通过）

快速输入 5 字仅 1 次请求且带 q、停手 500ms 才发、搜索期间列表不闪/文字不消失、清空回全量（无 q）、搜索与 type/month 筛选叠加。

### 代码审查记录（3 轮）

- 首版：useDebounce 机制一次写对（没踩"调用当传递"坑）；queryKey 工厂自发重构（官方推荐写法，加分）；api 层空串不传 + trim 细节
- 修 3 处：①useDebounce 写死 string → 泛型 `<T>`（体内只透传无需 extends 约束）②handleKeywordChange 死代码（if 分支被下一行覆盖）③invalidateRecordLists 改名 useInvalidateRecordLists（Hook 命名约定）
- 二轮：onClick 里调 Hook（Rules of Hooks 违规，改名前就存在）→ hook 返回回调、顶层调用修复；补 placeholderData

---
通过时间：2026-11-18
