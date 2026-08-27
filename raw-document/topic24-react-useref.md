# 2.4 useRef：可变盒子与 DOM 引用

## 讲解原文

> 承接 2.3：你已经用 ref 盒子救过过期闭包（监听器里读 `nameRef.current` 拿到活值）。本节回答两个还没拆开的问题：**改 `.current` 为什么不触发重渲染？ref 除了装数据还能干什么？**

### 一、useRef 的本质：一个跨渲染存活的同一个盒子

```tsx
const box = useRef(0)
// 初次渲染后 box === { current: 0 }，此后每一轮渲染拿到的都是同一个对象
```

```
渲染 #1 ──> 创建盒子 { current: 0 }，存进 fiber
渲染 #2 ──> 取出同一个盒子（引用相同！）
渲染 #3 ──> 还是同一个盒子
```

这就是它和普通变量的本质区别：组件函数每轮重新执行，`let x = 0` 每轮都归零；`useRef` 的盒子存在 React 外部（fiber 节点上），每轮原样还给你。**和 useState 存在同一个地方，但待遇完全不同。**

### 二、为什么改 `.current` 不触发重渲染

回忆 1.2/2.1 的模型：**触发重渲染的只有两条路--setState（自身）或父组件重渲染**。React 决定"要不要重新渲染"时，只对比 state 和 props；它**根本不追踪 `.current` 的变化**。

```tsx
function Counter() {
  const count = useRef(0)
  return (
    <button onClick={() => { count.current++; console.log(count.current) }}>
      点了 {count.current} 次   {/* ❌ 永远显示 0 */}
    </button>
  )
}
```

点击后 console 正常递增（盒子确实在变），但界面永远显示"点了 0 次"--没有任何机制通知 React"该重渲染了"。

**一句话记忆：ref 是 React 世界的"局外变量"--React 看得见盒子，看不见盒子里装了什么。**

这也是一个筛选器：**需要界面跟着变 -> state；只是"记下来后面用" -> ref**。

### 三、用途 1：可变盒子（2.3 已见，补全清单）

| 场景 | 例子 |
|------|------|
| 让过期闭包读到活值 | 2.3 的 `nameRef.current` |
| 存实例句柄 | `timerRef.current = setInterval(...)`，卸载时清理 |
| 存"上一轮的值" | `prevRef.current` 对比前后变化 |
| 跨渲染的中间数据 | 不想触发渲染的临时标记（`isLoadingRef`） |

### 四、用途 2：DOM 引用

React 里你通常不碰 DOM（声明式），但有些需求绕不开真实节点：聚焦输入框、测量尺寸、播放/暂停 `<video>`、接入第三方库（图表、地图）。

```tsx
function SearchBar() {
  const inputRef = useRef<HTMLInputElement>(null)   // 初始 null：挂载前还没有 DOM

  useEffect(() => {
    inputRef.current?.focus()    // 挂载完成后，current 指向真实 <input> 节点
  }, [])

  return <input ref={inputRef} />
}
```

机制：把盒子交给 React（`ref={inputRef}`），**DOM 挂载完成后 React 自动把真实节点塞进 `current`**，卸载时塞回 `null`。你全程没碰 `document.querySelector`。

**时序细节（值得记住）**：`ref.current` 的赋值发生在 **commit 阶段（DOM 已插入页面之后、useEffect 执行之前）**。所以 effect 里一定能拿到 DOM，但渲染期间（JSX 里）它还是 `null`。

### 五、两条铁律（陷阱区）

1. **渲染期间不要读写 `.current`**--渲染应该是 props/state 的纯函数，掺入 ref 会让同样的输入出现不同的渲染结果（StrictMode 双调用下直接露馅）。读/写都放在事件处理或 effect 里。
2. **不要用 ref 存"渲染依赖的数据"**--界面要显示的数据放 state，否则改了不刷新（见第二节的反例）。

### 六、Vue 对照：同名不同命

| | Vue `ref()` | React `useRef()` |
|---|---|---|
| 命名来源 | reference（响应式引用） | reference（引用），纯属撞名 |
| 改值触发更新 | ✅ `.value` 一改，视图跟着变 | ❌ `.current` 随便改，视图装死 |
| 底层 | Proxy 包装 + 依赖收集 | 一个普通 `{ current }` 对象 |
| 定位 | **状态**（数据驱动视图） | **逃生舱**（视图管不到的地方） |

迁移期最危险的习惯：把 Vue 里"ref 就是状态"的直觉带过来，用 `useRef` 存界面数据。反过来记：**Vue 的 ref 是台前的主角，React 的 ref 是后台的工具人。**

> 给 2.5 埋点：ref 还有一个进阶玩法--`forwardRef` + `useImperativeHandle`，让父组件拿到子组件暴露的方法（对照 Vue 的 `defineExpose`），到组件通信复盘时串起来。

## 考核过程

### 题目 1：Recorder 三问（ref 显示值 / 派生状态 / 视频播放器选型）

**用户回答**：
1. p 显示 0，ul 显示第0/1/2次记录；React 只对比 state/props，不追踪 .current
2. useState 修法可行，但更聪明的是直接用 log.length 推导，不用新增变量
3. (a) useState；(b) setCurrentTime(c => c + 10)

**点评**：未通过（概念性错误为主）。
- 第 1 问：❌ 正确答案 p 显示 3、ul 显示第1/2/3次。混淆了两条规则：改 .current 不"引起"渲染（✅记住），但渲染发生时会读活值（❌漏掉）--setLog 每次都引起重渲染，JSX 重新读盒子拿到新值。"永远显示 0"的前提是**没有任何 state 变化**。另有 off-by-one（++ 在 setLog 前，首条是第1次）
- 第 2 问：✅ 一次答对，派生状态思想迁移到位
- 第 3 问：(a) ✅；(b) ❌ 把 currentTime 当 state，需求是直接拨真实 `<video>` 节点，应该用 videoRef.current.currentTime += 10

### 补充对话

- 纠正第 1 问后给出规则拆分："渲染读活值，但只有 state/props 变化才触发渲染"
- 重测题（Timer 暂停组件）：首轮"界面无变化✅、用错了应改 useState✅"，但预测"再过1秒显示 6s 已暂停"❌--实际冻结在"5s 运行中"永远不变（暂停期间 setSeconds 被跳过，重渲染从未发生，"已暂停"永不出现=界面撒谎）
- 微调确认题（再点一下恢复运行后）：答对"6s 运行中"（下一秒 setSeconds 恢复->渲染->文案自愈）

---
通过时间：2026-08-27（重测+微调确认通过）
