# 2.2 useEffect：副作用模型与依赖数组

## 讲解原文

> 承接 2.1：渲染 = 纯粹的"算 UI"。可真实组件还要发请求、开定时器、订阅事件--这些"渲染之外的事"，React 用一个统一的模型管理。

### 一、为什么渲染必须是纯粹的

先建立一个铁律：**渲染函数应该只做一件事--根据 props 和 state 算出 JSX**。

```tsx
function Timer() {
  setInterval(() => console.log('tick'), 1000) // ❌ 写在渲染里
  return <div>hi</div>
}
```

这段代码的灾难：React 不保证渲染几次（StrictMode 下开发环境就跑两遍），每次渲染都开一个新定时器，旧的不关--**定时器泄漏，越积越多**。

Vue 里你习惯把这类代码放 `onMounted`/`onUnmounted`。React 没有生命周期钩子，它的答案是：**用一个统一的副作用容器，声明"这段逻辑依赖哪些数据"**。

### 二、useEffect 的执行时机模型

```tsx
useEffect(() => {
  // 副作用本体
  return () => {
    // cleanup（可选）
  }
}, [deps])
```

时序上只记一件事（对照 Vue mounted）：

```
渲染（调用组件函数，算出 JSX）
   ↓
DOM 已更新（浏览器还没画）
   ↓
跑 effect 本体          ← 挂载后跑一次 + 每次 deps 变化后跑
   ↓
下次 effect 跑之前
先跑上一次的 cleanup    ← 老逻辑"退场"，新逻辑"进场"
```

- **挂载后必跑一次**（不看你写不写 deps）--相当于 `mounted`
- deps 变化 -> 先跑旧 cleanup -> 再跑新 effect
- 卸载时 -> 跑最后一次 cleanup --相当于 `unmounted`

**cleanup 的心智模型不是"销毁钩子"，而是"上一次副作用的撤销"**：定时器要 clearInterval、订阅要退订、请求要忽略过期结果。每次 effect 重跑前都先撤销旧的，保证世界里永远只有"当前这一份"副作用在生效。

### 三、依赖数组：不是"声明我用了什么"，是"申请重跑"

Vue 对照：`watch(count, cb)` 里你指定**监听源**；`useEffect(fn, [count])` 里你指定**重跑条件**。方向反过来：watch 是"数据变了通知我"，effect 是"这个数据变了我就重做一遍"。

三种写法：


| 写法                      | 行为                               | 等价 Vue            |
| ----------------------- | -------------------------------- | ----------------- |
| `useEffect(fn, [a, b])` | 挂载跑一次 + a/b 任一变（Object.is 判等）后重跑 | watch + immediate |
| `useEffect(fn, [])`     | 只在挂载跑一次                          | onMounted         |
| `useEffect(fn)`（不写）     | **每次渲染后都跑**                      | 几乎不该用             |


**为什么 deps 用 Object.is 判等**：和 setState 跳过渲染是同一把尺子。所以 deps 里放**对象/数组**要小心--引用一变就重跑，哪怕内容没变（这是 2.6 useMemo 存在的原因之一）。

### 四、依赖数组的本质：让 React 替你记住闭包

用 2.1 的快照视角重新理解 effect：

```tsx
useEffect(() => {
  document.title = `count is ${count}`
}, [count])
```

每份 effect 闭包里捕获的是**那次渲染的 count 快照**。React 做的事：渲染后拿新闭包的 `[count]` 和上一次存的 `[count]` 做 Object.is，不同才重跑。

这就解释了两个高频现象：

1. **漏写 deps = 用旧快照**。写 `[]` 却在 effect 里读 `count`，那个闭包永远停在第一次渲染的 count=0--**过期闭包**（2.3 专门拆）。
2. **React 无法自动收集依赖**。Vue 用 Proxy 在 getter 里静默收集依赖，React 的渲染是纯函数调用、没有拦截点，只能**让你手动声明**。这是两套响应式哲学的分水岭：Vue 自动但需要 Proxy 包裹，React 手动但保持原生 JS。



### 五、典型用法对号入座

```tsx
// 1. 挂载后请求一次数据（最常见）
useEffect(() => {
  let ignore = false            // cleanup 旗标：组件卸载/重跑时作废旧请求
  fetchUser(id).then(u => {
    if (!ignore) setUser(u)     // 竞态：慢的旧请求不能覆盖新的
  })
  return () => { ignore = true }
}, [id])                        // id 换了重新请求，旧的作废

// 2. 定时器：每次重跑前先撤旧的
useEffect(() => {
  const t = setInterval(tick, 1000)
  return () => clearInterval(t) // 不写这行 = 泄漏
}, [])

// 3. 订阅外部事件
useEffect(() => {
  window.addEventListener('resize', onResize)
  return () => window.removeEventListener('resize', onResize)
}, [])
```

第 1 个例子的 `ignore` 旗标值得多看一眼：它治的是**竞态**--快速切换 id 时，前一个请求若更慢返回，会把新数据覆盖成旧数据。cleanup 在重跑前把它作废。

### 六、本节心智模型（背下来）

1. **渲染纯函数，副作用进 effect**--渲染次数不受你控制，effect 的重跑条件归你声明
2. **执行时序：DOM 更新后跑本体；重跑前先跑上次 cleanup**--cleanup 是"撤销上一次"，不只是"卸载"
3. **deps 是重跑申请，不是用途清单**--漏写 = 主动选择旧快照
4. **Object.is 判 deps**--对象/数组引用变了就重跑，和 setState 同一把尺子
5. **React 依赖手动声明 vs Vue 自动收集**--没有拦截点，只能你亲口告诉它



## 考核过程

### 题目 1：A 预测（[] deps 的 log）+ B 找错（UserCard 两个 bug）

**用户回答**：
A：effect 和 cleanup 各打印一次，`effect: 0`、`cleanup: 0`；"没写依赖项只会初始化执行一次"。
B：补 `[id]` + `ignore` 旗标防竞态，代码完整正确。

**点评**：
B 满分（旗标 + 依赖一次到位）。A 细节错误：本场景组件未卸载、deps 恒定，`cleanup:` 打印 **0 条**；cleanup 只在 deps 重跑前或卸载时触发（卸载时打印 `cleanup: 0`，闭包快照）。

### 补充对话 1：ignore 旗标的变量困惑

用户主动提问：新 effect 体内重新定义 `ignore=false`，旧 cleanup 的 `true` 会不会被覆盖？--暴露"两轮共享同一变量"的误解。补讲：每轮 effect 是独立闭包，`ignore①`/`ignore②` 是两个变量，各自被各自轮的 `fetch.then` 捕获；时序是"旧 cleanup（翻旧变量）→ 新 effect body（定义新变量）"。用户表示理解。

### 题目 2（确认题）：deps 改为 `[count]`，点 3 次后卸载，完整 log 序列

**用户首答**："1 条，effect:1, cleanup:0" --概念性错误：以为 deps 只在"最终状态"变化时重跑一次，漏掉每次变化都触发 旧cleanup→新effect 的循环，也漏了卸载收尾。

**纠正过程**：画出点第 1 次的三段式（渲染→Object.is 比较→cleanup:0→effect:1），让用户照此补表。用户正确补全：`(cleanup:1, effect:2)`、`(cleanup:2, effect:3)`，并答出卸载时 `cleanup:3`。

**最终正确序列**：`effect:0 → cleanup:0, effect:1 → cleanup:1, effect:2 → cleanup:2, effect:3 → cleanup:3`。**通过**（微调后）。

---
通过时间：2026-08-24

```js
function UserCard({ id }) {

    const [user, setUser] = useState(null)

    useEffect(() => {
      let ignore = false

      fetchUser(id).then((res) => {
        if(!ignore) setUser(res)
      })
      return ()=> {
        ignore = true
      }
    }, [id])

    if (!user) return <p>加载中...</p>

    return <div>{user.name}</div>

  }
```