# 2.6 useMemo / useCallback 与重渲染优化

## 讲解原文

> 承接 2.5：你知道了父组件重渲染会带走全子树、Context 广播会带走所有消费者。本节讲怎么给这趟"顺风车"设关卡--以及为什么默认不该设。

### 一、先复习：重渲染到底何时发生

一个组件重渲染，只有三条路：

```
① 自身 setState          -> 自己重渲染
② 父组件重渲染           -> 全部子组件重新执行（props 变没变都执行）
③ 消费的 Context value 变 -> 自己重渲染
```

②是重灾区：99% 的"没必要的渲染"都是爹带来的。设关卡的思路只有一个：**在②的入口拦住--"props 没变就别重新执行我"。**

### 二、React.memo：props 浅比较关卡

```tsx
const ExpensiveList = memo(function ExpensiveList({ items, onSelect }) {
  // 假设这里渲染很重
})

// 父组件里：
<ExpensiveList items={items} onSelect={handleSelect} />
```

`memo` 包裹后，父组件重渲染时 React 会对每个 prop 做 `Object.is` 比较（**浅比较，逐 prop**）：

- 全部没变 -> **跳过这次重渲染**，复用上次结果
- 任何一个变了 -> 照常渲染

### 三、memo 的天敌：每次渲染都新鲜的引用

```tsx
function Parent() {
  const [query, setQuery] = useState('')

  // ❌ 每轮渲染都是【新函数引用】
  const handleSelect = (id) => console.log(id)
  // ❌ 每轮渲染都是【新数组引用】
  const visible = bigList.filter(item => item.name.includes(query))

  return <ExpensiveList items={visible} onSelect={handleSelect} />
}
```

Parent 因为 query 输入狂重渲染。`visible` 内容可能根本没变，但 `filter` 每次返回**新数组** -> `Object.is` 判"变了" -> **memo 关卡形同虚设**。函数同理。

这就是三件套的另外两件：

```tsx
const visible = useMemo(
  () => bigList.filter(item => item.name.includes(query)),
  [query]                       // query 不变 -> 返回上次的结果，引用相同
)

const handleSelect = useCallback(
  (id) => console.log(id),
  []                            // 依赖不变 -> 同一个函数引用
)
```

- **useMemo**：记住"算出来的**值**"，deps 不变就复用上次结果
- **useCallback**：记住"**函数本身**"，deps 不变就复用同一个引用。它就是 `useMemo(() => fn, deps)` 的语法糖

**三件套咬合**：`memo`（子组件设关卡）+ `useMemo`/`useCallback`（父组件保证传下去的引用稳定）。缺任何一环，关卡失效。

### 四、和 Vue 的根本差异（迁移者必读）

| | Vue computed | React useMemo |
|---|---|---|
| 触发 | **依赖追踪**：只有依赖真的被改过才重算 | 渲染时比对 deps 引用，变了就重算 |
| 缓存保证 | 强：没人读取时连重算都不发生 | 弱：React 官方声明它是"性能提示"，未来可能丢弃缓存重算 |
| 定位 | **派生状态**的一等公民，随处可用 | 主要是给 memo/子组件喂稳定引用；顺手缓存计算 |
| 用法密度 | Vue 鼓励多用 computed | React 鼓励少用 useMemo（它本身有存储和比较开销） |

**给 Vue 老手的纠偏**：在 Vue 里你想都不想就写 computed；在 React 里，`useMemo` 的默认答案是**不写**--除非传给 memo 组件、或计算确实昂贵。

### 五、优化纪律：先测量，后优化

1. 重渲染 ≠ 卡：组件重新执行是 React 的正常呼吸，多数组件渲染开销微不足道
2. 优化的唯一正当理由：**测量出问题**（Profiler 火焰图、明显输入卡顿）
3. 乱加 useMemo/useCallback 的代价：每次渲染多一次"存 deps + 逐项比较"，代码可读性下降，还可能把 bug 藏进漏写的依赖里
4. 顺序建议：先修**状态结构**（状态下沉：把 query 状态移到真正用它的组件，让别人不陪跑）-> 再上 memo 三件套

**一句话：memo 三件套是手术刀，不是营养品。**

## 考核过程

（此段在考核通过后追加填充）
