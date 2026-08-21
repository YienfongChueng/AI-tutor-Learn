# 组件通信方式全景（vs emit/expose/v-model）

## 讲解原文

### 一句话锚定

**Vue 给每种通信方向都造了一个专用语法（emit/expose/v-model/provide）；React 只给你一个通道--props--然后把"事件"也变成 props 传下去。**先记住这个不对称，再看映射表。

---

### 1. 全景映射表

| 通信方向 | Vue 语法 | React 等价 |
|---------|---------|-----------|
| 父 -> 子（数据） | props | props（唯一正道） |
| 子 -> 父（事件） | `emit('xxx')` | **回调当 props 传**：`<Child onConfirm={fn}/>`，子组件里 `props.onConfirm(data)` |
| 双向绑定 | `v-model`（modelValue + update:modelValue 的糖） | 手动组合：`value={x} onChange={setX}`（1.5 的受控模式**推广到组件**） |
| 跨层注入 | provide / inject | Context：`createContext` + `useContext`（2.5 详讲） |
| 父拿子实例/方法 | `defineExpose` + 模板 ref | `forwardRef` + `useImperativeHandle`（2.4 详讲）；**但 React 文化更倾向改成受控，不暴露实例** |
| 兄弟/任意组件 | eventBus/mitt（Vue 社区常见） | **状态提升**到共同父级；再远用 Context 或状态库（数据流模块）。React 生态**没有 eventBus 惯例** |
| 插槽内容 | slot | children / 传 JSX 当 prop（1.2 讲过） |

---

### 2. "回调 props"：emit 的替身

```jsx
// 父组件：把函数传下去（Vue 里你会写 @confirm="onConfirm"）
function Parent() {
  const [name, setName] = useState('')
  return <EditDialog initial={name} onConfirm={(v) => setName(v)} />
}

// 子组件：调用它（Vue 里你会写 emit('confirm', v)）
function EditDialog({ initial, onConfirm }) {
  const [draft, setDraft] = useState(initial)
  return (
    <div>
      <input value={draft} onChange={e => setDraft(e.target.value)} />
      <button onClick={() => onConfirm(draft)}>确认</button>
    </div>
  )
}
```

机制上完全同构：Vue 的 emit 最终也是找到父组件挂在组件上的监听函数去调用。区别只是**注册形式**：Vue 用 `@事件名` 指令糖，React 把函数当普通 prop 用 JSX 属性传。

命名惯例：`onXxx` 开头（和 DOM 的 `onClick` 一脉相承）。

---

### 3. v-model 组件版：受控模式的推广

Vue3 的组件 v-model：

```html
<MyInput v-model="text" />
<!-- 等价 <MyInput :modelValue="text" @update:modelValue="text = $event" /> -->
```

React 没有这个糖，但你自己封一个"双向组件"就是三行：

```jsx
function MyInput({ value, onChange }) {
  return <input value={value} onChange={onChange} />
}
// 父组件：<MyInput value={text} onChange={setText} />
```

**受控组件模式**（1.5 的表单套路）是 React 里比"双向绑定"更基础的组件设计范式：

- 子组件**不持有真相**，只显示 props + 上报事件
- 谁传数据进来，谁负责改--单向数据流不破

推论：Vue 里你习惯给组件内部 `ref` 个状态再用 `expose` 吐方法；React 的第一反应应该是"**能不能把状态提上去变成受控**"。内部 draft（草稿态，像上面的 EditDialog）是少数合理例外：确认前属于子组件私有。

---

### 4. 状态提升（Lifting State Up）：兄弟通信的正解

```jsx
// TemperaturePanel：摄氏/华氏两个输入框互相联动
function Panel() {
  const [celsius, setCelsius] = useState('')      // 状态放在共同父级
  return (
    <>
      <CelsiusInput value={celsius} onChange={setCelsius} />
      <FahrenheitInput celsius={celsius} onChange={setCelsius} />  {/* 内部做换算再上报 */}
    </>
  )
}
```

两个套路合体：**状态放父级（提升）+ 每个子组件受控（props 进、回调出）**。Vue 里你也做过状态提升，但遇到麻烦时 Vue 人容易掏 eventBus；React 生态没有这个拐杖，**提升 -> Context -> 状态库**是唯一升级路线，这条路线正好是 s2（useContext）和 s3/数据流模块的主线。

---

### 5. 什么算"作弊"通道（能跑但别用）

- `useRef` 挂到子组件上调方法（绕过数据流，2.4 再讲边界）
- 模块级全局变量/单例 store 直接读写（无响应性，改了也不渲染）
- Context 塞高频变化的值（所有消费者全量重渲染，2.5 详讲）

判断标准就一条：**数据变更能否追溯到一次 setState/useReducer？** 能 -> 单向数据流；要靠"别人直接改我"或"我直接改别人"-> 走偏了。

## 考核过程

（Phase 3 通过后追加填充）
