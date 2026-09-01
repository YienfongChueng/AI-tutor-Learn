# 2.7 自定义 Hook 实战

## 讲解原文

> 承接整个模块 2：useState/useEffect/useRef/useContext/useMemo/useCallback 你都过了一遍。本节把它们组装成自己的武器--自定义 Hook。核心认知：**它是逻辑复用的搬运工，不是生命周期钩子。**

### 一、自定义 Hook 没有魔法：就是个普通函数

```tsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial)
  const toggle = useCallback(() => setOn(v => !v), [])
  return [on, toggle]
}

// 用起来：
const [modalOpen, toggleModal] = useToggle()
const [sidebarOpen, toggleSidebar] = useToggle()
```

"自定义 Hook"的全部定义就两条：

1. **名字以 `use` 开头**--纯约定，但 ESLint 靠它启用 hooks 规则检查
2. **内部调用了其他 Hook**--所以它必须遵守 hooks 规则（只在顶层调用、只在 React 函数里调用）

除此之外它没有任何特殊待遇：React 不注册它、不追踪它、不知道它存在。

### 二、本质：把"状态 + 操作 + 副作用"打包搬走

不写自定义 Hook 时，逻辑复用的困境：

```tsx
// 弹窗要开关，侧边栏要开关，主题要开关……
// 每个组件都复制一遍 useState + toggle 函数
```

组件的本质矛盾：**UI 结构要复用 -> 抽组件；状态逻辑要复用 -> 抽什么？** 状态不能真的"搬出去"（state 必须挂在组件上），能搬的只有**"创建和管理状态的这套代码"**。自定义 Hook 就是这套代码的搬运工：

```
┌─ useToggle（代码模板，无状态）
│    每次调用 -> 在【调用者组件】身上创建一份全新的 state
│
├─ modalOpen 那次调用  -> state 挂在 ModalHost 组件的 fiber 上
└─ sidebarOpen 那次调用 -> state 挂在 Layout 组件的 fiber 上（完全独立，互不干扰）
```

**每次调用 = 一份独立的 state**。这不是"共享状态"工具--想共享得靠 Context 或状态库（阶段 3）。

### 三、实战：useFetch（本节的主角）

把 2.2/2.3 学的竞态处理、cleanup、依赖数组全部装进去：

```tsx
function useFetch(url) {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)

  useEffect(() => {
    let ignore = false                    // 2.3 的竞态旗标
    setLoading(true)

    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(res.statusText)
        return res.json()
      })
      .then(json => { if (!ignore) setData(json) })
      .catch(err => { if (!ignore) setError(err) })
      .finally(() => { if (!ignore) setLoading(false) })

    return () => { ignore = true }        // cleanup：url 变化重跑前作废旧请求
  }, [url])

  return { data, loading, error }
}

// 任何组件一行接入：
const { data, loading, error } = useFetch('/api/user/1')
```

看它替调用者扛了多少事：请求生命周期（loading/error/data）、竞态、cleanup、依赖重发。**组件里只剩 UI**。这就是 Hooks 体系设计的初衷：状态逻辑与视图解耦。

### 四、Vue 对照：你其实早就见过这个思想

| | Vue3 组合式函数（composables） | React 自定义 Hook |
|---|---|---|
| 形式 | `useXxx()` / `useXxx` 命名 | 同款约定 |
| 本质 | 把 setup 里的逻辑抽成函数 | 把组件里的 Hook 调用抽成函数 |
| 状态归属 | 每次调用独立的 `ref` | 每次调用独立的 state |
| 限制 | 无调用顺序限制（响应式系统不需要） | 必须顶层调用（hooks 靠调用顺序定位） |

Vue3 的 composables 本身就是受 React Hooks 启发的同门思想，你写 `useMouse()`/`useLocalStorage()` 的经验可以直接迁移。**核心差异还是那一个**：Vue 靠依赖追踪，Hook 想什么时候调就什么时候调；React 靠调用顺序索引 state，所以顺序就是生命线。

### 五、什么时候该抽自定义 Hook

- 同一套"状态 + 副作用"组合出现在 **2 个以上**组件
- 组件里 useEffect 块太长，和 UI 混在一起读不动
- 想给逻辑写独立测试（Hook 可以在测试环境单独渲染）

**反指标**：只为一处使用提前抽象（YAGNI）；纯计算没有 Hook 调用（普通函数就够，别加 use 前缀）。

## 考核过程

第 3 问：先别投降，你缺的不是知识是拼装 courage--模块 2 的零件全学过。 给你三块积木，拼起来：

  积木1（2.1）：一个 state 存"防抖后的值"，初始值就是 value
  积木2（2.2）：effect 里开 setTimeout，到点后 setDebounced(最新 value)，
               deps 写 [value] —— value 一变，effect 就重跑
  积木3（2.3）：cleanup 里 clearTimeout —— 这就是防抖的"灵魂"

  组装提示：用户连续打字时，value 连续变 -> effect 连续重跑 -> 每次重跑前 cleanup 先执行 -> 上一个还没到点的定时器被掐掉 -> 只有用户停下来，最后一个定时器才能活到
  300ms。

  请按这三块积木写出完整代码（就 5-6 行），写出来并解释对一句话，这节就过。

  ```jsx
   // 用法：const debouncedQuery = useDebounce(query, 300)
  // 效果：用户停止输入 300ms 后，debouncedQuery 才变成最新的 query
  // function debounce(fn, delay = 300) {
  //   let timer = null
  //   return function (...args) {
  //     clearTimeout(timer)
  //     timer = setTimeout(() => {
  //       fn.apply(this, args)
  //     }, delay)
  //   }
  // }

  function useDebounce(value, delay = 300) {
    const [debounced,setDebounced] = useState(value)

    useEffect(()=> {
      let timer = null
      timer = setTimeout(()=> {
        setDebounced(value)
      },delay)
      return ()=> clearTimeout(timer)
    },[value])
    return debounced
  }
// 为什么 cleanup 让它成为“防抖”而不是“延迟”？（提示：没有 cleanup，连续打字 5 个字会发生什么？）
// 每次value变化时，执行useEffect前，会先执行cleanup，清理掉上一个定时器，不会执行setDebounced，直到value不变化了，才执行最后一个定时器
  ```

  （此段在考核通过后追加填充）

### 题目 1：useCounter 预测 + 两句辨析 + useDebounce 编码

**用户回答**：
1. A:3 B:1；每次调用一份独立 state，a 的 count 挂在 a 的 fiber 上，b 的挂在 b 的 fiber 上
2. 不是全局单例（每次调用新 state）；不是生命周期钩子（是创建和管理状态逻辑的复用抽离）✅
3. 编程题不会

**点评**：
- 第 1 问：结果✅，但 fiber 归属小错--a、b 不是组件，两个 state 都挂在 **App 的 fiber** 上，是 hooks 链表的两个槽位，靠**调用顺序**区分（这正是"必须顶层调用"的根因）
- 第 2 问：✅ 两句辨析全对
- 第 3 问：未尝试，给三块积木（state 初始化 / effect+setTimeout+[value] / cleanup 掐定时器）引导组装

### 题目 2：useDebounce 编码（积木引导后）

**用户回答**（首版）：`return clearTimeout(timer)` + `setDebounced(c => c = value)`

**点评**：结构全对但两处错：
1. **致命 bug**：return 的是执行 clearTimeout 的结果（undefined）且当场执行--定时器刚创建就被掐死，debounced 永远停在初始值。应为 `return () => clearTimeout(timer)`。⚠️ "把调用当传递"第 4 次（1.3 onClick={fn()}、1.6 addFn={setTodoList(...)}、2.6 多包箭头）。口诀：**括号是"现在就干"，不加括号是"到时候再干"**
2. `setDebounced(value)` 即可--函数式更新是给"新值依赖旧值"的场景（c => c+1），旧值无参与感

### 题目 3：修正 + 解释

**用户回答**：修正后代码全对（见上方代码块）；解释："每次 value 变化执行 effect 前先 cleanup 掉上一个定时器，不会执行 setDebounced，直到 value 不变了才执行最后一个定时器" ✅

**点评**：通过。本质：没有 cleanup = 5 个定时器错峰各响（延迟+多次更新）；有 cleanup = 只有最后一个活到 300ms（防抖+一次更新）。

---
通过时间：2026-09-01（模块2最后一节，s2 Hooks 深入完成）
