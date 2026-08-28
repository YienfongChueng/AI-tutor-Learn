# 2.5 useContext 与状态提升的取舍

## 讲解原文

> 承接 1.6（组件通信）：当时你的工具箱是"状态提升 + 回调 props"。组件树一深，props 就要一层层手工传递（prop drilling）。本节讲 React 的第三条通道--Context--以及它最容易被忽视的代价：**重渲染粒度**。

### 一、prop drilling：问题长什么样

```
App（user 状态在这里）
 └─ Layout        ── user 只是为了往下传
     └─ Sidebar   ── user 只是为了往下传
         └─ UserMenu  ── 真正要用 user 的在这里
```

```tsx
// 中间两层什么都没做，只是"人肉传送带"
<Sidebar user={user} />
```

Vue 里你可能用 provide/inject 跳过中间层。React 的对应物就是 Context。

### 二、Context 三步走

```tsx
// ① 创建（参数是默认值）
const UserContext = createContext(null)

// ② 提供：从 App 直接铺到整棵子树
function App() {
  const [user, setUser] = useState(null)
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Layout />
    </UserContext.Provider>
  )
}

// ③ 消费：任意深度直接拿，中间层无感知
function UserMenu() {
  const { user, setUser } = useContext(UserContext)
  return <span>{user?.name}</span>
}
```

对照 Vue：`createContext` ≈ `createContext`，`Provider` ≈ `provide`，`useContext` ≈ `inject`。心智模型几乎一样，**但重渲染行为完全不同**。

### 三、关键机制：Provider 的 value 变化会让所有消费者重渲染

```
Provider value 更新（App 重渲染 -> value 是新对象）
        ↓
React 找到树里所有 useContext(UserContext) 的组件
        ↓
全部重渲染（哪怕它们渲染结果根本没变）
```

两个放大器，让这个代价经常失控：

**放大器 1：value 每次都是新对象。**

```tsx
// ❌ App 每次重渲染，value 都是新对象 { user, setUser }
<UserContext.Provider value={{ user, setUser }}>
```

App 因为**任何**无关 state（比如一个主题切换）重渲染 -> value 引用变了 -> **所有消费者**跟着重渲染。这和 2.1 学的 `Object.is` 判定是同一件事：引用不同 = 变了。

```tsx
// ✅ 修法（2.6 会正式讲 useMemo，这里先见个面）
const value = useMemo(() => ({ user, setUser }), [user])
<UserContext.Provider value={value}>
```

**放大器 2：消费者重渲染时，它的整棵子树跟着渲染。** `useContext(UserContext)` 的组件重渲染，它 return 的 JSX 里所有子组件（没被 memo 的话）全部重新执行。

### 四、决策表：什么时候用什么

| 场景 | 选择 | 原因 |
|------|------|------|
| 两个兄弟组件共享状态 | 状态提升到父级 | 一层传递，杀鸡不用牛刀 |
| 状态 + 修改它的函数成对跨层传递 | 自定义 Hook + Context（2.7） | 传送带太长 |
| 低频变化的"环境"数据：主题、语言、当前用户、权限 | Context 天选 | 变化少，重渲染代价几乎不存在 |
| 高频变化的数据：输入框草稿、鼠标位置、列表筛选 | ❌ 别用 Context | 每次变化全树消费者陪跑 |
| 高频 + 确实要全局 | 状态库（Zustand/Pinia 思路，阶段3） | 组件可订阅"切片"，只在自己关心的数据变化时渲染 |

**一句话原则：Context 的价值 = 变化频率低。它解决的是"传得远"，不解决"变得快"。**

### 五、Vue 对照

| | Vue provide/inject | React Context |
|---|---|---|
| 传递机制 | 原型链式查找，响应式 | Provider value 广播 |
| 消费者何时更新 | 依赖收集：只有真正用到该值的 effect/渲染才更新 | Provider value 一变，**所有** useContext 消费者重渲染，无细粒度筛选 |
| 高频数据 | 可以放（ref 本身响应式，按需收集） | 避免放（除非配合 memo 切割） |

Vue 的依赖收集让 inject 天然"按需"；React 的 Context 是"广播"，细粒度要靠你自己组合 useMemo/memo（2.6）或上状态库（阶段3）。

> 埋点 2.6：放大器 1 的修法是 useMemo，放大器 2 的修法是 React.memo --下一节把"重渲染优化三件套"一次讲透。

## 考核过程

### 题目 1：App 无关 state 变化的广播三问

**用户回答**：
1. 会重渲染。两层原因：① fontSize 更新 -> App 重渲染，子组件即使 props 没变也跟着渲染（组件树通则）；② Context 广播，App 重渲染后整棵树使用 useContext 的组件都会渲染
2. `const userMemo = useMemo(() => ({ user, setUser }), [user])`，传入 Provider
3. (a) 该用 ✅ (b) 不该用，用状态管理库 ✅ (c) 该用 ✅ (d) 不该用，用状态管理库 ✅

**点评**：✅ 通过（attempts=1 一次全对）。第 1 问两层原因齐全；第 2 问 useMemo + [user] 依赖正确（setUser 引用稳定无需进 deps）。补充精确化：广播的扳机是 value 每次渲染造新对象 -> Object.is 判变 -> 消费者全体检疫。

---
通过时间：2026-08-27
