# React 学习笔记（topic15-31 整合）

> 基于 ai-tutor/raw-document topic15~topic31 整理，面向 Vue3 → React 迁移者。
> 结构：模块1 基础语法 → 模块2 Hooks 深入 → 模块3 工程化实战。

---

## 目录

- 模块1：基础
  - [1.1 JSX 本质](#11-jsx-本质与-vue-模板对照)
  - [1.2 函数组件与 Props](#12-函数组件与-props)
  - [1.3 State 与事件处理](#13-state-与事件处理usestate-基础)
  - [1.4 条件与列表渲染（key）](#14-条件与列表渲染key-的真实作用)
  - [1.5 受控组件与表单](#15-受控组件与表单vs-v-model)
  - [1.6 组件通信全景](#16-组件通信全景vs-emiteiposev-model)
- 模块2：Hooks 深入
  - [2.1 useState 深入](#21-usestate-深入快照语义与批量更新)
  - [2.2 useEffect](#22-useeffect副作用模型与依赖数组)
  - [2.3 闭包陷阱与 cleanup](#23-useeffect-闭包陷阱与-cleanup)
  - [2.4 useRef](#24-useref可变盒子与-dom-引用)
  - [2.5 useContext](#25-usecontext-与状态提升的取舍)
  - [2.6 useMemo / useCallback](#26-usememo--usecallback-与重渲染优化)
  - [2.7 自定义 Hook](#27-自定义-hook-实战)
- 模块3：工程化
  - [3.1 Vite 项目结构与 StrictMode](#31-vite--react-项目结构与脚手架)
  - [3.2 React Router](#32-react-router路由与嵌套守卫)
  - [3.3 服务端状态与 TanStack Query](#33-数据请求与服务端状态)
  - [3.4 性能优化清单](#34-性能优化清单memo--lazy--suspense)
- [附录：个人高频错误清单](#附录个人高频错误清单)

---

# 模块1：基础

## 1.1 JSX 本质与 Vue 模板对照

**一句话锚定：JSX 不是模板，是 `createElement` 函数调用的语法糖；它的值是一个普通 JS 对象（ReactElement）。**

### 编译产物

```jsx
const el = <h1 className="title">Hello {name}</h1>
```

构建时（Vite 用 esbuild/SWC）转译为（React 17+ 自动 runtime，不需要 `import React`）：

```js
import { jsx as _jsx } from 'react/jsx-runtime'
const el = _jsx('h1', { className: 'title', children: 'Hello ' + name })
```

多个 children 时用 `_jsxs`（结尾多个 s，仅是打包 children 数组的细节）。

`_jsx(...)` 运行时返回的对象：

```js
{
  $$typeof: Symbol(react.element),   // 防 XSS 注入的安全标记
  type: 'h1',                        // 字符串=DOM元素；函数=组件；这是分流依据
  key: null,
  ref: null,
  props: { className: 'title', children: '...' },  // children 只是普通 prop
}
```

### 三个立即可用的推论

1. **JSX 是表达式**：可以存变量、放数组、当参数传、条件返回。所以 React 里"指令"不需要存在——`v-if` 是三元表达式，`v-for` 是 `.map()`，`v-slot` 是 `props.children`
2. **element 是不可变描述**：只是"想要的 UI 长什么样"的快照对象，不是 DOM。更新 = 生成**新的** element 交给 React 算 diff
3. **组件就是 `type` 字段里的函数引用**：`<Comp />` 编译成 `_jsx(Comp, {...})`——首字母必须大写，小写会被当成字符串（DOM 标签）

### 与 Vue 编译的对照

```
Vue:  template --编译--> render() 内 h() 调用 --执行--> VNode 树
React: JSX     --编译--> 组件体内 _jsx() 调用 --执行--> Element 树
```

| 维度 | Vue 模板 | React JSX |
|------|---------|-----------|
| 语法 | 约束的 DSL | 任意 JS 表达式，图灵完备 |
| 编译优化 | 静态提升、patchFlag、block tree | 基本没有，运行时全量 diff |
| 优化路线 | 聪明放在编译时 | 聪明放在运行时（Fiber/并发） |

**一句话：Vue 用语法约束换编译优化，React 用语法自由换表达力、把性能问题留给运行时解决。** Vue 里"模板写复杂表达式有性能警告"的概念在 React 里不存在。

### Vue 习惯映射表

| Vue 写法 | React 等价 |
|---------|-----------|
| `:class="{active: isActive}"` | `className={isActive ? 'active' : ''}`（或 `clsx` 库） |
| `v-if` / `v-else` | `{cond ? <A/> : <B/>}` 或 `{cond && <A/>}` |
| `v-for="x in list" :key="x.id"` | `{list.map(x => <X key={x.id} />)}` |
| `{{ text }}` | `{text}` |
| `v-html="raw"` | `dangerouslySetInnerHTML={{ __html: raw }}` |

属性名 **camelCase**：`class -> className`、`for -> htmlFor`、`onclick -> onClick`（props 是 JS 对象的键，DOM 保留字冲突）。

### 分层总览

```
JSX（语法）→ Element（数据）→ Fiber（运行时）→ DOM（结果）
```

### 细节盲区（考核踩过）

- `typeof el` 返回 `'object'`，不是 "React Element"
- 文本+表达式混写时，children **逐段切分**：`<p>Hi {name} <b>!</b></p>` 编译为 `children: ['Hi ', name, _jsx('b', {children:'!'})]`——**只有元素才包成 `_jsx()`，文本段/表达式段是原始值直接进数组**；同行文本与表达式之间的空格保留（`'Hi '` 有尾随空格）

---

## 1.2 函数组件与 Props

**一句话锚定：React 组件就是一个函数：接收 props 对象，返回 element 树。调用它的人不是你，是 React。**

### 执行模型的关键差异（最重要的一张表）

```jsx
function Badge({ color, children }) {
  return <span className={color}>{children}</span>
}
// <Badge color="red">Hi</Badge> 编译为 _jsx(Badge, {...})——此刻还没执行
// React 渲染时才调用它：Badge(props)，拿到 element 树再递归处理
```

| | Vue SFC (setup) | React 函数组件 |
|---|---|---|
| 组件函数体执行次数 | **只执行 1 次**（挂载时） | **每次渲染都重新执行** |
| `let count = 0` 定义在函数体 | 闭包住整个生命周期 | 每次渲染重新创建、值重置 |
| 状态存哪 | setup 闭包 + 响应式代理 | React 外部代管（hooks） |

推论：

- **函数体每次渲染从零跑一遍**，里面的变量、内部函数每轮都是**新的一份**——这是"state 是快照"的根源
- **hooks 必须在顶层无条件调用**（React 靠调用顺序对号入座）
- Vue 的"setup 执行一次 + 依赖收集"换成了 React 的"反复执行整个函数 + diff 新旧两棵树"的暴力方案

### Props：只读快照，单向数据流

```jsx
// 解构接收（主流写法），默认值直接给
function Avatar({ src, size = 48, ...rest }) {
  return <img src={src} width={size} {...rest} />
}
```

- React 没有 `defineProps` 声明层，**函数签名就是声明**
- 解构默认值（`size = 48`）取代 `default:`
- `{...rest}` 透传取代 `$attrs`——Vue 非声明属性自动落到根元素，React 默认**什么都不做**，要透传得自己写
- props 是快照：父组件重渲染 → 子组件拿到**新 props 对象**重新执行（不管值变没变）

三个特殊 prop：

| prop | 去向 |
|------|------|
| `key` | 被 createElement 摘走，不进 props（diff 身份标识） |
| `ref` | 被摘走（DOM/实例引用） |
| `children` | **在 props 里**，内容分发 |

### children = 默认插槽；具名插槽 = 传 JSX/函数

```jsx
// 默认插槽
<Card><p>内容</p></Card>
// Card 内部：function Card({ children }) { return <div>{children}</div> }

// 具名插槽：传 JSX 当属性
<Layout header={<h1>标题</h1>} body={<Table/>} />

// 作用域插槽：传函数当属性（render props 模式）
<Foo render={(data) => <Item {...data}/>} />
```

**插槽就是"以 prop 为载体传 JSX/函数"**，Vue 只是用 `<template #>` 做了语法糖。

### 纯函数约定（重要）

组件函数必须是**近似纯函数**：同样的 props → 同样的 JSX，渲染期间不改外部状态、不发请求。因为**函数体会被反复调用**，副作用会以不可预期的次数执行。

⚠️ Vue 人习惯把请求写进 setup 顶层"碰巧"没出事；React 里**组件渲染几次请求就发几次**，直接出 bug。请求/订阅去 `useEffect`，变更状态去事件回调。

---

## 1.3 State 与事件处理（useState 基础）

**一句话锚定：Vue 是"你改数据，它自己发现"；React 是"你喊一嗓子（setter），它替你改并重跑函数"。**

### 基本形态

```jsx
function Counter() {
  const [count, setCount] = useState(0)   // [当前值, 触发器] = useState(初始值)
  return <button onClick={() => setCount(count + 1)}>{count}</button>
}
```

执行链：点击 → `setCount(1)` → React 在外部仓库更新该组件槽位的 state → 安排重渲染（异步合批）→ 重新调用 `Counter()` → 这次 `useState(0)` 不取初始值，而是从仓库取回 1 → 新 JSX → diff → 更新 DOM。

**`useState(初始值)` 的参数只在首次渲染生效。**

### 快照语义：连 set 两次读的是"旧照片"

```jsx
function handleClick() {
  setCount(count + 1)   // count 是本轮渲染的快照，此刻是 0
  setCount(count + 1)   // 还是读 0！
}
// count=0 时点击 -> 界面显示 1，不是 2
```

两次 set 都基于同一轮渲染的 count（0）计算。修复：**函数式更新**：

```jsx
setCount(c => c + 1)   // 接收最新 state
setCount(c => c + 1)   // 0 -> 2 ✅
```

> ⚠️ **Vue 人的条件反射陷阱**：Vue 里 `count++` 连写两次天经地义；React 里 `count` 是局部 const，**改它没用，读它读到旧值**。一切更新必须经过 setter。`count++` 直接 TypeError（const）。

**快递模型**：传值 = 寄出瞬间定格的成品；传函数 = 寄指令单，React 处理队列时才执行且把流水线最新值作参数。

### 事件处理对照

| Vue | React |
|-----|-------|
| `@click="handle"` | `onClick={handle}` |
| `@click="add(n)"` | `onClick={() => add(n)}` |
| `@click.prevent` | 无修饰符系统，函数体第一行 `e.preventDefault()` |
| 事件对象 | 合成事件 SyntheticEvent（池化已废弃，当普通对象用） |

```jsx
// 经典错误：渲染时立即执行了 add，把返回值 undefined 当事件处理器注册
<button onClick={add(n)}>+</button>
// 正确：传函数引用
<button onClick={() => add(n)}>+</button>
```

### 对象/数组 state：不可变更新（Vue 人最痛的一刀）

React **没有响应式代理**，判断"变没变"的唯一依据是 **`Object.is`（引用比较）**：

```jsx
function addBad(n) {
  list.push(n)          // 原地修改：引用没变
  setList(list)         // Object.is 判等 → React 认为没变，跳过渲染 ❌
}

function addGood(n) {
  setList([...list, n]) // 新数组、新引用 ✅
}
```

**心法：永远创建新引用，而不是修改旧引用**（"换人不是整容"）。

嵌套结构套路：

```jsx
setForm(prev => ({ ...prev, address: { ...prev.address, city: 'SH' } }))  // 改深层
setList(prev => prev.map(item => item.id === id ? { ...item, done: true } : item))  // 改某项
setList(prev => prev.filter(item => item.id !== id))  // 删某项
```

### 心智图

```
state 快照模型
├─ 读：本轮渲染的绑定，永远是"当轮照片"
├─ 写：只能 setXxx(新值) 或 setXxx(前值 => 新值)
├─ 更新判定：Object.is 引用比较（无代理）
└─ 多次 set：同轮快照算多遍 → 函数式更新兜底
```

---

## 1.4 条件与列表渲染（key 的真实作用）

### 条件渲染：JS 本身就是指令系统

```jsx
{isLoggedIn ? <Dashboard/> : <Login/>}   // v-if / v-else
{hasError && <ErrorBanner msg={msg}/>}   // v-if 单分支
```

**&& 的经典坑（Vue 人专属）**：

```jsx
{count && <Badge count={count}/>}
// count = 0 时：0 && x 求值为 0（不是 false），React 会渲染出裸的 "0"！
// 修法：
{count > 0 && <Badge count={count}/>}   // 让左边产出布尔值
```

### key 的真实作用：diff 时的身份判定

**key 不是"帮 diff 提速的优化项"，是元素的"身份证"——身份证变了，React 就把它当成另一个人（销毁重建，state 清零）；身份证没变，就是同一个人（复用更新，state 保留）。**

React 拿新旧 element 树**只做同层比较**，判定"是不是同一个"的依据是 `type + key`：

```
旧：[a, b, c]   新：[z, a, b, c]   // 头部插入，有稳定 key
React 按 key 配对：
  a/b/c -> 同一个人：DOM 复用，只挪位置，组件 state 保留
  z     -> 新面孔：创建新节点
```

没有 key（或全用 index）时，**位置就是身份**：

```
旧：[A, B]      新：[X, A, B]   // 头部插入，用 index
  位置0: A -> X  "同一个人换了内容" -> 更新 props（A 的 state 留给了 X！）
  位置1: B -> A  同上
  位置2: 新建 B
```

一次头部插入退化成全员更新 + 一次新建（性能损失是小事）；真正的 bug 是：**组件的 state 跟着"位置"而不是跟着"数据"走**。

经典翻车现场：TodoItem 里用 useState 存输入框草稿，`key={i}` 头部插入后草稿"跑"到别的 todo 上。

**核心口诀：props 跟数据走，state 跟位置走。**

### key 使用检查清单

| 场景 | key 该用什么 |
|------|-------------|
| 服务端数据列表 | 业务 id（稳定、唯一） |
| 静态配置列表（永不增删排序） | index 勉强可用 |
| 头部插入/排序/删除的动态列表 | **index 必出 bug** |
| 想强制重置组件 | 刻意切换 key（如 `${id}-${version}`） |

### 反向应用：故意用 key 重置组件

```jsx
<UserProfile key={userId} userId={userId} />
// 切换 userId 时整个组件重新挂载，不用手动清空表单/内部 state
```

> key 用错的症状往往不是"变慢"，而是"状态错乱、输入框串位、动画闪跳"这类诡异 bug。
> 踩坑记录：`key="item.id"` 写成字符串字面量——**引号里的永远不会变，变量才跟着数据走**。

---

## 1.5 受控组件与表单（vs v-model）

**一句话锚定：v-model 是 `:value` + `@input` 的语法糖，React 表单就是把糖剥掉后手写的那两行。**

### 受控闭环

```jsx
const [text, setText] = useState('')
<input value={text} onChange={e => setText(e.target.value)} />
```

敲键 → onChange → setText → 重渲染 → `value={新text}` 回填。**state 是唯一真相源，输入框只是 state 的"显示器 + 遥控器"。**

只写一半的后果：

```jsx
<input value={text} />           // 没有 onChange：输入没反应 + 控制台警告
<input onChange={...} />         // 没有 value：非受控，state 拿不到值
<input value={text} readOnly />  // 显式只读：合法
```

### 各控件受控写法

| 控件 | React 受控 |
|------|------------|
| text 输入框 | `value={text}` + `onChange` 取 `e.target.value` |
| 多行文本 | `<textarea value={text}>`（**不是 children**） |
| 复选框 | `checked={ok}` + 取 `e.target.checked`（**不是 value**） |
| 单选组 | 每个 `<input type="radio" checked={g==='male'}>` 手动对齐 |
| 下拉单选 | `<select value={city}>` 挂父级（不是 `<option selected>`） |
| 文件 | **只能非受控**：`defaultValue` + ref（file 对象不可序列化） |

### 一个 handler 管全表单（惯用法）

```jsx
const [form, setForm] = useState({ user: '', pass: '' })
function handleChange(e) {
  const { name, value } = e.target
  setForm(prev => ({ ...prev, [name]: value }))   // 不可变更新 + 计算属性名
}
<input name="user" value={form.user} onChange={handleChange} />
```

修饰符是**字段语义**，按字段分派而不是全局猜数据形状：

```jsx
if (type === 'checkbox') value = e.target.checked
else if (name === 'age')  value = Number(v) || v    // isNaN 兜底
else if (name === 'name') value = v.trim()
else                      value = v
```

### v-model 修饰符手工等价

| Vue 修饰符 | React 手写 |
|-----------|-----------|
| `.number` | `Number(e.target.value)` |
| `.trim` | `e.target.value.trim()` |
| `.lazy` | onChange 换成 `onBlur` |

### 受控 vs 非受控

| | 受控 | 非受控 |
|---|------|--------|
| 真相源 | React state | DOM |
| 每键触发重渲染 | 是 | 否 |
| 校验/联动 | 天然支持 | 要手动读 DOM |
| 适用 | 绝大多数场景 | 文件上传、超长表单、一次性读取 |

非受控写法：`<input defaultValue="初始值" ref={inputRef} />`，要值时读 `inputRef.current.value`。

> Vue 的 v-model 底层也是受控同步，表单习惯可直接迁移——只是糖没了，**每键一次重渲染在两边都存在**。

---

## 1.6 组件通信全景（vs emit/expose/v-model）

**一句话锚定：Vue 给每种通信方向都造了专用语法；React 只给你一个通道——props——把"事件"也变成 props 传下去。**

### 全景映射表

| 通信方向 | Vue | React 等价 |
|---------|-----|-----------|
| 父 → 子 | props | props |
| 子 → 父 | `emit('xxx')` | **回调当 props 传**：`<Child onConfirm={fn}/>`，子组件调 `props.onConfirm(data)` |
| 双向绑定 | v-model | 手动组合：`value={x} onChange={setX}` |
| 跨层注入 | provide / inject | Context + `useContext` |
| 父拿子实例 | defineExpose + 模板 ref | `forwardRef` + `useImperativeHandle`（React 文化更倾向改成受控） |
| 兄弟/任意 | eventBus/mitt | **状态提升**到共同父级 → Context → 状态库。React 生态**没有 eventBus 惯例** |
| 插槽 | slot | children / 传 JSX 当 prop |

### 回调 props（emit 的替身）

```jsx
// 父：把函数传下去
<EditDialog initial={name} onConfirm={(v) => setName(v)} />
// 子：调用它
function EditDialog({ initial, onConfirm }) {
  const [draft, setDraft] = useState(initial)
  return <button onClick={() => onConfirm(draft)}>确认</button>
}
```

命名惯例：`onXxx` 开头。

### v-model 组件版 = 受控模式推广

```jsx
function MyInput({ value, onChange }) {
  return <input value={value} onChange={onChange} />
}
// 父：<MyInput value={text} onChange={setText} />
```

**受控组件模式**：子组件不持有真相，只显示 props + 上报事件；谁传数据进来，谁负责改。
**推论**：Vue 人第一反应"内部 ref + expose 吐方法"，React 第一反应应是"**能不能把状态提上去变成受控**"。内部 draft（草稿态）是少数合理例外：确认前属于子组件私有。

### 状态提升：兄弟通信的正解

```jsx
function Panel() {
  const [celsius, setCelsius] = useState('')      // 状态放共同父级
  return (
    <>
      <CelsiusInput value={celsius} onChange={setCelsius} />
      <FahrenheitInput celsius={celsius} onChange={setCelsius} />
    </>
  )
}
```

**状态放在离所有使用者最近的共同父级。** 草稿（打字中）不提升——提升它会让父组件每键重渲染并污染职责。

### 判断数据流是否走偏（一条标准）

**数据变更能否追溯到一次 setState？** 能 → 单向数据流；要靠"别人直接改我"→ 走偏了。

作弊通道（能跑别用）：useRef 挂子组件调方法、模块级全局变量、Context 塞高频值。

---

# 模块2：Hooks 深入

## 2.1 useState 深入：快照语义与批量更新

### 快照的根源：一次渲染 = 一次函数调用 = 一份闭包

点 3 次按钮，`console.log(count)` 依次打印 `0, 1, 2, 3`（渲染时各打印一次）——**每次调用产生独立的 count 变量，它是 const，在函数体执行期间不会变**：

```
渲染 #1: 调用 Counter() → useState 返回 0 → count = 0（这份闭包里永远是 0）
渲染 #2: 调用 Counter() → useState 返回 1 → count = 1（新的一份闭包）
```

`useState` 内部：维护一个"与组件位置绑定的存储格"（这就是 hooks 不能写在 if 里的原因——**位置即身份**）。渲染时从格子取值；`setCount` 往格子写新值并**安排一次重新调用**。

### Vue 对照：活引用 vs 定妆照

| | Vue3 ref/reactive | React useState |
|---|---|---|
| 变量本质 | **活引用**：读 `.value` 总是最新值 | **值快照**：本次渲染内是常量 |
| 异步回调里读 | 最新值 | **那次渲染时的旧值**（闭包捕获） |
| 连续 set | 每次同步生效 | **批量合并**，一次渲染 |

经典翻车：`setCount(count + 1)` 后 `setTimeout(() => alert(count), 3000)`，3 秒内连点 5 次，**alert 弹 0**（handleClick 属于渲染 #1 的闭包）。修复要靠 useRef 或把逻辑移进 set，函数式更新不解决 alert 读旧值。

### 批量更新（React 18 自动批处理）

> 同一个事件里发出的所有 setState 进入更新队列，**合并成一次重渲染**。事件函数执行期间 state 一直是旧值。

```jsx
function handleClick() {
  setA(a + 1); setB(b + 1); setA(a + 1)
  // 显示 1 1（不是 2 1）——三条更新基于快照，最后一条覆盖
}
```

### 更新队列：传值 vs 传函数

- `setCount(count + 1)` → 放进**成品值**（用当前快照算好）
- `setCount(c => c + 1)` → 放进**指令单**（React 从快照值开始依次执行）

```jsx
setCount(count + 1) ×3  // 0→1，三次像一次 ❌
setCount(c => c + 1) ×3  // 0→1→2→3 ✅
```

**什么时候必须函数式**：新值依赖旧值（计数、开关、数组追加）。传值即可：新值与旧值无关。

### Object.is：跳过渲染的最后一道闸

```jsx
setCount(0)       // Object.is(0,0)=true → 跳过渲染
setObj({...obj})  // 引用变了 → false → 照样渲染！
```

### 四条心智模型（背下来）

1. **一次渲染一份闭包，state 在其中是 const 快照**——异步回调读旧值不是 bug，是闭包的本分
2. **一个事件一批更新，合并成一次渲染**（React 18 全场景自动批处理）
3. **传值 = 成品，传函数 = 指令单**——依赖旧值必须传函数
4. **Object.is 判等，相同跳过渲染**——更新永远换新引用

---

## 2.2 useEffect：副作用模型与依赖数组

### 渲染必须纯粹

**渲染函数只做一件事：根据 props 和 state 算出 JSX。** 渲染里写 `setInterval` 的灾难：React 不保证渲染几次（StrictMode 下开发环境跑两遍），每次渲染开新定时器旧的不管——**定时器泄漏**。React 没有生命周期钩子，答案是统一的副作用容器。

### 执行时机模型

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

- 挂载后必跑一次（≈ `onMounted`）；卸载时跑最后一次 cleanup（≈ `onUnmounted`）
- **cleanup 的心智模型不是"销毁钩子"，而是"上一次副作用的撤销"**：clearInterval、退订、作废过期请求。保证世界里永远只有"当前这一份"副作用生效

### 依赖数组：不是"声明我用了什么"，是"申请重跑"

| 写法 | 行为 | 等价 Vue |
|------|------|---------|
| `useEffect(fn, [a, b])` | 挂载一次 + a/b 任一变（Object.is）后重跑 | watch + immediate |
| `useEffect(fn, [])` | 只在挂载跑一次 | onMounted |
| `useEffect(fn)`（不写） | **每次渲染后都跑** | 几乎不该用 |

**为什么手动声明 deps**：Vue 用 Proxy 在 getter 里静默收集依赖；React 渲染是纯函数调用、没有拦截点，只能让你手动声明。这是两套响应式哲学的分水岭：**Vue 自动但需要 Proxy，React 手动但保持原生 JS**。

### 典型用法

```tsx
// 1. 请求 + 竞态处理（ignore 旗标）
useEffect(() => {
  let ignore = false            // 每轮 effect 是独立闭包，各管各的旗标
  fetchUser(id).then(u => { if (!ignore) setUser(u) })
  return () => { ignore = true }  // 慢的旧请求不能覆盖新数据
}, [id])

// 2. 定时器：重跑前先撤旧的
useEffect(() => {
  const t = setInterval(tick, 1000)
  return () => clearInterval(t)   // 不写这行 = 泄漏
}, [])
```

### 心智模型（背下来）

1. **渲染纯函数，副作用进 effect**
2. **执行时序：DOM 更新后跑本体；重跑前先跑上次 cleanup**
3. **deps 是重跑申请，不是用途清单**——漏写 = 主动选择旧快照
4. **Object.is 判 deps**——对象/数组引用变了就重跑
5. **React 依赖手动声明 vs Vue 自动收集**

### 考核易错点

- deps `[]` 且组件未卸载时，**cleanup 一次都不打印**（只在 deps 重跑前或卸载时触发）
- deps 改为 `[count]`、点 3 次后卸载的完整序列：
  `effect:0 → cleanup:0, effect:1 → cleanup:1, effect:2 → cleanup:2, effect:3 → cleanup:3`
- `ignore` 旗标变量困惑：每轮 effect 是**独立闭包**，`ignore①`/`ignore②` 是两个变量；时序是"旧 cleanup（翻旧变量）→ 新 effect body（定义新变量）"

---

## 2.3 useEffect 闭包陷阱与 cleanup

### 一句话定义

**每个 effect 闭包捕获的是"它那一轮渲染"的 state/props 快照。只要这个闭包活得比那轮渲染久，它读到的就是旧值。**

只在**异步/持久性副作用**里发作：定时器、事件监听、fetch 回调——出生在一轮渲染里，很久之后才执行。

### 症状 1：定时器数到 1 就停

```tsx
useEffect(() => {
  const t = setInterval(() => setCount(count + 1), 1000)
  return () => clearInterval(t)
}, [])   // 闭包 A 是渲染#1创建的，count 永远是 0 → 每秒 setCount(0+1)
// 现象：0 → 1 → 1 → 1 ... 卡死
```

两种修法，代价不同：

```tsx
// 修法①：函数式更新（推荐）——定时器只开一次，零重开开销
setInterval(() => setCount(c => c + 1), 1000)   // deps []

// 修法②：count 进 deps——每秒销毁重建定时器，能跑但有开销
```

**心法：能让闭包"不读旧值"的，就不要让它"读新值"。**（函数式更新、useRef 都是这个思路）

### 症状 2：事件监听器拿到旧 props

```tsx
useEffect(() => {
  const handler = (e) => console.log(keyword)   // 挂载时的旧值
  window.addEventListener('keydown', handler)
  return () => window.removeEventListener('keydown', handler)
}, [])   // ❌ 漏写 keyword → 修法：[keyword] 进 deps
```

### 症状 3：fetch 竞态 → ignore 旗标 / AbortController

ignore 旗标是过期闭包的**反向利用**：给旧闭包发"死亡通知书"。选择标准：**能取消就取消（AbortController），取消不了就作废（ignore 旗标）**。

### 症状 4：副作用修改自己的依赖 = 无限循环

```tsx
useEffect(() => { setCount(count + 1) }, [count])
// count 变 → effect 重跑 → 又改 count → 死循环
// 解法：审视这个更新到底该不该放 effect 里
```

### 诊断流程（背下来）

1. **这个闭包是哪轮渲染创建的？** 它读的每个值都是那轮的快照
2. **它执行时，那个值变了吗？** 变了 → 过期闭包实锤
3. 我要的是"更新时重跑"（加 deps）、"不读旧值"（函数式更新/useRef），还是"旧闭包作废"（cleanup + 旗标）？

### deps vs ref 选择口诀

- **行为依赖值**（值变了要重新执行逻辑）→ 用 deps
- **只是要读最新值**（不想重跑）→ 用 ref

### Vue 对照：为什么 Vue 里没这病

| | Vue | React |
|---|---|---|
| 回调里读状态 | `.value` 永远最新（Proxy 活引用） | 闭包快照，可能过期 |
| 依赖 | 自动收集 | 手动声明 deps |
| 应对 | 几乎不用想 | 心里常驻"这闭包几岁了" |

---

## 2.4 useRef：可变盒子与 DOM 引用

### 本质：跨渲染存活的同一个盒子

```
渲染 #1 → 创建盒子 { current: 0 }，存进 fiber
渲染 #2 → 取出同一个盒子（引用相同！）
```

组件函数每轮重新执行，`let x = 0` 每轮归零；`useRef` 的盒子存在 React 外部（fiber 节点上），每轮原样还给你。**和 useState 存在同一个地方，但待遇完全不同。**

### 为什么改 `.current` 不触发重渲染

触发重渲染只有两条路：**setState（自身）或父组件重渲染**。React 只对比 state 和 props，**根本不追踪 `.current` 的变化**。

```tsx
const count = useRef(0)
return <button onClick={() => { count.current++ }}>点了 {count.current} 次</button>
// ❌ 界面永远显示 0（在没有任何 state 变化的前提下）
```

**一句话记忆：ref 是 React 世界的"局外变量"——React 看得见盒子，看不见盒子里装了什么。**

⚠️ 两条规则要分开记（考核翻车点）：
- 改 `.current` 不**引起**渲染 ✅
- 但渲染**发生**时（因为别的 state 变了），JSX 会重新读盒子，拿到的是**新值**——所以"永远显示 0"的前提是**没有任何 state 变化**

筛选器：**需要界面跟着变 → state；只是"记下来后面用" → ref。**

### 用途 1：可变盒子

| 场景 | 例子 |
|------|------|
| 让过期闭包读到活值 | `nameRef.current` |
| 存实例句柄 | `timerRef.current = setInterval(...)` |
| 存"上一轮的值" | `prevRef.current` 对比前后变化 |
| 不触发渲染的临时标记 | `isLoadingRef` |

### 用途 2：DOM 引用

```tsx
function SearchBar() {
  const inputRef = useRef<HTMLInputElement>(null)   // 初始 null：挂载前还没有 DOM
  useEffect(() => {
    inputRef.current?.focus()    // 挂载后 current 指向真实节点
  }, [])
  return <input ref={inputRef} />
}
```

**时序细节**：`ref.current` 赋值发生在 **commit 阶段（DOM 插入页面后、useEffect 执行前）**。effect 里一定能拿到 DOM，但渲染期间（JSX 里）还是 `null`。

### 两条铁律

1. **渲染期间不要读写 `.current`**——渲染应是 props/state 的纯函数（StrictMode 双调用直接露馅）。读/写放事件处理或 effect 里
2. **不要用 ref 存"渲染依赖的数据"**——否则改了不刷新，界面撒谎

### Vue 对照：同名不同命

| | Vue `ref()` | React `useRef()` |
|---|---|---|
| 改值触发更新 | ✅ `.value` 一改视图跟着变 | ❌ `.current` 随便改，视图装死 |
| 底层 | Proxy + 依赖收集 | 普通 `{ current }` 对象 |
| 定位 | **状态**（台前主角） | **逃生舱**（后台工具人） |

---

## 2.5 useContext 与状态提升的取舍

### prop drilling 问题

```
App（user 在这里）
 └─ Layout     ── 只为往下传
     └─ Sidebar ── 只为往下传
         └─ UserMenu ── 真正要用
```

### Context 三步走

```tsx
// ① 创建（参数是默认值）
const UserContext = createContext(null)

// ② 提供
<UserContext.Provider value={value}><Layout /></UserContext.Provider>

// ③ 消费：任意深度直接拿，中间层无感知
const { user, setUser } = useContext(UserContext)
```

对照 Vue：`createContext` ≈ `provide`，`useContext` ≈ `inject`。心智模型几乎一样，**但重渲染行为完全不同**。

### 关键机制：Provider value 变化 → 所有消费者重渲染

两个放大器让代价经常失控：

**放大器 1：value 每次都是新对象**

```tsx
// ❌ App 因任何无关 state 重渲染 → value 引用变 → 所有消费者陪跑
<UserContext.Provider value={{ user, setUser }}>

// ✅ 修法：useMemo 稳住引用
const value = useMemo(() => ({ user, setUser }), [user])   // setUser 引用稳定，无需进 deps
```

**放大器 2：消费者重渲染时，它的整棵子树跟着渲染**（没被 memo 的话）。

### 决策表

| 场景 | 选择 |
|------|------|
| 两个兄弟共享状态 | 状态提升到父级 |
| 状态 + 修改函数成对跨层传递 | 自定义 Hook + Context |
| 低频"环境"数据：主题、语言、当前用户、权限 | **Context 天选** |
| 高频数据：输入草稿、鼠标位置、列表筛选 | ❌ 别用 Context |
| 高频 + 确实要全局 | 状态库（Zustand 等，可订阅切片） |

**一句话原则：Context 的价值 = 变化频率低。它解决"传得远"，不解决"变得快"。**

### Vue 对照

Vue 的依赖收集让 inject 天然"按需更新"；React Context 是"广播"，无细粒度筛选，要靠 useMemo/memo 自己组合。

---

## 2.6 useMemo / useCallback 与重渲染优化

### 复习：重渲染只有三条路

```
① 自身 setState → 自己重渲染
② 父组件重渲染 → 全部子组件重新执行（props 变没变都执行）★ 重灾区
③ 消费的 Context value 变 → 自己重渲染
```

### React.memo：props 浅比较关卡

```tsx
const ExpensiveList = memo(function ExpensiveList({ items, onSelect }) { ... })
```

父组件重渲染时对每个 prop 做 `Object.is` 比较（**浅比较，逐 prop**）：全部没变 → 跳过渲染；任一变了 → 照常渲染。

### memo 的天敌：每次渲染都新鲜的引用

```tsx
const handleSelect = (id) => console.log(id)          // ❌ 每轮新函数引用
const visible = bigList.filter(item => ...)            // ❌ 每轮新数组引用
```

`filter` 内容没变也返回**新数组** → memo 关卡形同虚设。

### 三件套咬合

```tsx
const visible = useMemo(
  () => bigList.filter(item => item.name.includes(query)),
  [query]                       // query 不变 → 复用上次结果
)
const handleSelect = useCallback(
  (id) => console.log(id),
  []                            // 依赖不变 → 同一函数引用
)
```

- **useMemo**：记住"算出来的**值**"
- **useCallback**：记住"**函数本身**"——它就是 `useMemo(() => fn, deps)` 的语法糖
- **三件套咬合**：`memo`（子组件设关卡）+ `useMemo`/`useCallback`（父组件保证引用稳定）。**缺任何一环，关卡失效**

### deps 判定规则

deps 写的是**函数体内用到的外部变量**（setter 等引用稳定的可省略）。考核踩坑：`useCallback((id) => setTodos(ts => ts.filter(...)), [todos])`——函数体没读 todos，`[todos]` 是错的。

### 和 Vue 的根本差异

| | Vue computed | React useMemo |
|---|---|---|
| 触发 | 依赖真的被改过才重算 | 渲染时比对 deps 引用 |
| 缓存保证 | 强（没人读连重算都不发生） | 弱（官方声明是"性能提示"，可能丢弃缓存重算） |
| 定位 | 派生状态一等公民 | 主要给 memo 组件喂稳定引用 |

**Vue 老手纠偏**：Vue 里想都不想就写 computed；React 里 useMemo 的默认答案是**不写**——除非传给 memo 组件、或计算确实昂贵。

### 优化纪律：先测量，后优化

1. 重渲染 ≠ 卡：重新执行是 React 的正常呼吸
2. 优化的唯一正当理由：**测量出问题**
3. 乱加三件套的代价：多一次存 deps + 比较，还可能把 bug 藏进漏写的依赖
4. 顺序：先修**状态结构**（状态下沉）→ 再上 memo 三件套

**一句话：memo 三件套是手术刀，不是营养品。**

---

## 2.7 自定义 Hook 实战

### 没有魔法：就是个普通函数

自定义 Hook 的全部定义就两条：

1. **名字以 `use` 开头**——纯约定，ESLint 靠它启用 hooks 规则检查
2. **内部调用了其他 Hook**——必须遵守 hooks 规则（顶层调用、只在 React 函数里调用）

React 不注册它、不追踪它、不知道它存在。

```tsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial)
  const toggle = useCallback(() => setOn(v => !v), [])
  return [on, toggle]
}
const [modalOpen, toggleModal] = useToggle()      // state 挂在调用者组件的 fiber 上
const [sidebarOpen, toggleSidebar] = useToggle()  // 完全独立，互不干扰
```

**每次调用 = 一份独立的 state**。不是"共享状态"工具——想共享靠 Context 或状态库。

fiber 归属澄清（考核踩坑）：同一组件里调两次，两个 state 都挂在**该组件的 fiber** 上，是 hooks 链表的两个槽位，**靠调用顺序区分**——这正是"必须顶层调用"的根因。

### 实战：useFetch

```tsx
function useFetch(url) {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)

  useEffect(() => {
    let ignore = false
    setLoading(true)
    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(res.statusText)
        return res.json()
      })
      .then(json => { if (!ignore) setData(json) })
      .catch(err => { if (!ignore) setError(err) })
      .finally(() => { if (!ignore) setLoading(false) })
    return () => { ignore = true }        // url 变化重跑前作废旧请求
  }, [url])

  return { data, loading, error }
}
```

它替调用者扛了：请求生命周期、竞态、cleanup、依赖重发。**组件里只剩 UI。**

### 实战：useDebounce（防抖）

```tsx
// clearTimeout → 取消前 N-1 次 setDebounced 调用
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value)
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay) 
    return () => clearTimeout(timer)      // 防抖的灵魂
  }, [value])
  return debounced
}

// 问题：
// 每次 effect 闭包捕获的是当时的 value
// 如果 delay 内 value 变了多次
// 最后一个 timer 触发时，它捕获的 value 可能是倒数第二次的
// 用法：const debouncedQuery = useDebounce(query, 300)
```
```tsx
// 用 useRef + 手动控制
// ref → 保证最后一次 setDebounced 拿到的是最新 value
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  const valueRef = useRef(value);

  useEffect(() => {
    valueRef.current = value;           // 同步最新值
    const timer = setTimeout(() => {
      setDebounced(valueRef.current);   // 拿到最新的值只更新一次 state
    }, delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}
```

**为什么 cleanup 让它成为"防抖"而不是"延迟"**：没有 cleanup = 5 个定时器错峰各响（延迟 + 多次更新）；有 cleanup = 每次重跑前掐掉上一个定时器，只有用户停下来后最后一个定时器活到 300ms（防抖 + 一次更新）。

### 如果只是为了请求，直接防抖请求本身
```jsx
function useDebounceFn(fn, delay) {
  const timerRef = useRef(null);

  return useCallback((...args) => {
    clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => fn(...args), delay);
  }, [fn, delay]);
}

// 用法
const debouncedSearch = useDebounceFn((q) => fetch(q), 300);

<input onChange={e => debouncedSearch(e.target.value)} />
```

### Vue 对照

| | Vue3 组合式函数 | React 自定义 Hook |
|---|---|---|
| 本质 | 把 setup 里的逻辑抽成函数 | 把组件里的 Hook 调用抽成函数 |
| 状态归属 | 每次调用独立的 ref | 每次调用独立的 state |
| 限制 | 无调用顺序限制 | 必须顶层调用（顺序就是生命线） |

### 什么时候该抽

- 同一套"状态 + 副作用"出现在 **2 个以上**组件
- useEffect 块太长，和 UI 混在一起
- 想给逻辑写独立测试

**反指标**：只为一处使用提前抽象（YAGNI）；纯计算没有 Hook 调用（普通函数就够，别加 use 前缀）。

---

# 模块3：工程化

## 3.1 Vite + React 项目结构与脚手架

```bash
npm create vite@latest my-app -- --template react-ts
```

- CRA **已停止维护**（2023 起），新项目不要用
- Vite 内核同 Vue：dev 用 esbuild 预构建 + 原生 ESM，生产 Rollup 打包

```tsx
// main.tsx
import { createRoot } from 'react-dom/client'
createRoot(document.getElementById('root')!).render(
  <StrictMode><App /></StrictMode>
)
```

| | Vue | React |
|---|---|---|
| 挂载 | `createApp(App).mount()` | `createRoot(el).render(<App/>)`（React 18 起 `ReactDOM.render` 已废弃） |
| 插件体系 | `app.use(router).use(pinia)` | 无全局 use，各库自带 Provider |
| 全局错误处理 | `app.config.errorHandler` | ErrorBoundary 组件 |

生产结构参考：

```
src/
├─ main.tsx      入口
├─ App.tsx       路由/Provider 组装层
├─ routes/       路由级页面
├─ components/   通用组件
├─ hooks/        自定义 Hook
├─ api/          请求封装 + 类型
└─ stores/       状态库
```

### 本节主角：StrictMode 双调用

`<StrictMode>` 下**仅开发模式**故意：

1. **每个组件函数渲染两次**（同输入调两遍）
2. **每个 effect 走一遍 mount → unmount → 再 mount**（cleanup 执行后再重跑）

生产构建完全没有。这是一次"疫苗"，帮你排查两类病：

- **渲染不纯**：两次渲染结果不一样（渲染里改外部变量/`ref.current`）→ 界面闪烁警告
- **effect 缺 cleanup**：副作用翻倍暴露（重复请求、重复监听）

**"见鬼时刻"**：Network 里同一个请求发了两遍。新手的错误反应是删掉 StrictMode（= 把体检报告撕了，病还在）。正确反应：检查 effect 是否该有 cleanup/ignore 旗标。

时序澄清：双渲染是**同一次提交的连续两次函数调用**（纯度体检，第一份结果丢弃）；effect 双跑是提交后的另一场体检。两次 render 之间没有 effect。

---

## 3.2 React Router（路由与嵌套/守卫）

**最大观念差异一句话：React Router 没有守卫，守卫是你自己写的一个组件。**（v6/v7，JSX 声明式路线）

### 最小可用版

```tsx
// main.tsx
<BrowserRouter><App /></BrowserRouter>

// App.tsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/orders" element={<Orders />} />
  <Route path="*" element={<NotFound />} />   {/* 兜底 404 */}
</Routes>
```

### vue-router 概念对照

| vue-router | React Router v6 |
|---|---|
| routes 配置数组 | `<Routes><Route/></Routes>` JSX 嵌套 |
| `<router-view>` | `<Outlet />` |
| `<router-link to>` | `<Link to>` |
| `router.push('/x')` | `navigate('/x')`（`useNavigate()`） |
| `route.params.id` | `useParams()` |
| `route.query.x` | `useSearchParams()` |
| redirect 配置 | `<Navigate to="/x" replace />` |
| `beforeEach` 守卫 | **没有**，自己封装组件 |

### 嵌套路由：布局路由 + Outlet

```tsx
<Routes>
  <Route path="/" element={<Layout />}>
    <Route index element={<Home />} />            {/* 默认子路由 */}
    <Route path="orders/:id" element={<OrderDetail />} />
  </Route>
</Routes>

function Layout() {
  return <div><header>导航栏</header><Outlet /></div>
}
```

URL 切换时只有 Outlet 洞里的内容换，Layout 保持挂载（**state 不丢**）。跨出布局路由再回来 = Layout 卸载 = state 销毁。

### 动态参数与编程式跳转

```tsx
const { id } = useParams()
const navigate = useNavigate()
const [params] = useSearchParams()   // params.get('tab')
navigate('/orders', { replace: true })
```

⚠️ `navigate` 是 Hook 拿的，**组件外不能随便调**（请求工具 401 踢回登录需 window.location 或把 navigate 传出去）——这是和 vue-router 全局 `router` 实例的实打实差异。

### push vs replace（考核重点）

- 操作的是**浏览器历史栈**：push 入栈（后退可回），replace 替换当前条目（后退跳过）
- 列表 → 详情：push；404 → 首页：replace
- **登录成功后用 replace**：按后退不应回到 /login（与 state 无关）

### 守卫：RequireAuth 包装组件

```tsx
function RequireAuth({ children }) {
  const { user } = useAuth()
  const location = useLocation()
  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />
  }
  return children
}

// 挂到路由上：
<Route path="/orders" element={<RequireAuth><Orders /></RequireAuth>} />

// Login 成功后送回原页：
navigate(location.state?.from?.pathname ?? '/', { replace: true })
```

对照：vue-router 的 `beforeEach` 是**集中式**门卫；RequireAuth 是**分布式**门卫——哪个路由要保护就在哪包一层。**Vue 给框架级设施，React 给乐高积木。**

---

## 3.3 数据请求与服务端状态

### 手写 useFetch 的三堵墙

1. **重复请求**：两个组件同 URL 各请求一遍，没有跨组件共享
2. **没有缓存**：路由切走再切回 → 重新挂载 → effect 重跑 → 重新请求盯着转圈
3. **样板无底洞**：loading/error/重试/分页/乐观更新每个请求点手写一遍

### 根因：服务端状态 ≠ 客户端状态

| | 客户端状态 | 服务端状态 |
|---|---|---|
| 例子 | 弹窗开关、输入草稿、主题 | 用户信息、订单列表 |
| 真相 | **你拥有**的数据 | **远端数据的本地快照**，随时过期 |
| 挑战 | 组织、避免多余渲染 | 缓存、失效、去重、同步 |

**一句话：useState 装你的数据，TanStack Query 装服务端数据的照片，别让一个工具干两份活。**

### TanStack Query（v5）核心

```tsx
// main.tsx：全局一个 QueryClient
const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 60_000, retry: 2 } },
})
<QueryClientProvider client={queryClient}><App /></QueryClientProvider>
```

```tsx
// 读
const { data, isPending, isError, error, isFetching, refetch } = useQuery({
  queryKey: ['user', id],              // 缓存键
  queryFn: ({ signal }) => fetch(`/api/user/${id}`, { signal }).then(r => r.json()),
  staleTime: 60_000,
})

// 写
const qc = useQueryClient()
useMutation({
  mutationFn: (newUser) => api.updateUser(newUser),
  onSuccess: () => qc.invalidateQueries({ queryKey: ['user'] }),
})
```

| 墙 | 解法 |
|---|---|
| 重复请求 | 同 queryKey 并发 useQuery **自动合并**，结果共享 |
| 没有缓存 | 全局缓存池；staleTime 内回缓存；过期后**先给旧数据再后台刷新**（stale-while-revalidate） |
| 样板 | loading/error/重试/聚焦刷新全是内置开关 |

**心智转变**：不再"管理一次请求"，而是**声明一块缓存**（key + 获取函数 + 新鲜度）。`invalidateQueries` 核心思想：**写操作不直接改缓存，而是宣布作废，让读端自己重新拉**。

### 易混三兄弟

| 字段 | 含义 | 什么时候 true |
|------|------|--------------|
| `isPending` | 从未拿到过数据 | 只有首次 |
| `isFetching` | 正在请求中 | 首次 + 每次后台刷新 |
| `isLoading` | = isPending && isFetching | 组合值 |

典型 UI：`isPending` 显示骨架屏；`isFetching && !isPending` 显示"刷新中"小角标（不闪骨架屏）。

`queryFn` 的 `signal`（AbortSignal）传给 fetch，自动中止旧请求——**手写 ignore 旗标在这里是一等公民参数**。

### queryKey：缓存世界的身份证

**规则：影响请求结果的每个变量都必须进 key。**

```tsx
// ❌ ['user']：id 变了读的还是张三的缓存（缓存串台）
// ✅ ['user', id]：不同 id = 不同条目；id 变化 = 自动重新请求
```

层级结构是惯例：`['orders', { page: 1 }]`、`['orders', 'detail', 42]`——因为 `invalidateQueries` 是**前缀匹配**，`{ queryKey: ['orders'] }` 作废所有 orders 系缓存。**key 树 = 你的失效武器库的目录。**

踩坑记录：`['user', 'userId']` 写成字符串字面量 ❌——**引号里的永远不会变，变量才跟着 props 走**。

### 两个时钟

| 参数 | 管什么 | 默认 | 类比 |
|------|--------|------|------|
| `staleTime` | 数据多久算"不新鲜" | 0 | 食品保质期 |
| `gcTime` | 没人用后缓存留多久 | 5 分钟 | 冷藏保质期 |

**过期 ≠ 删除**：过期的数据还能用（切回页面秒显旧数据再后台刷新）；gc 了才是真没了（重新骨架屏）。

重新请求的触发条件（前提：缓存已 stale）：挂载新同 key 组件 / queryKey 变化 / 窗口聚焦（`refetchOnWindowFocus` 默认开，可关）/ 网络恢复 / 手动 refetch、invalidate。

### 乐观更新（体验敏感点）

```tsx
useMutation({
  mutationFn: (name) => api.patchUser(id, { name }),
  onMutate: async (name) => {
    await qc.cancelQueries({ queryKey: ['user', id] })
    const prev = qc.getQueryData(['user', id])
    qc.setQueryData(['user', id], (old) => ({ ...old, name }))   // 先改缓存，界面立刻变
    return { prev }
  },
  onError: (_e, _v, ctx) => qc.setQueryData(['user', id], ctx.prev),  // 失败回滚
  onSettled: () => qc.invalidateQueries({ queryKey: ['user', id] }),  // 最终以服务器为准
})
```

两种缓存更新哲学：**失效流**（默认，永远是服务器真相，多一次请求）vs **直写流**（省请求，但要自己保证形状对，只配乐观更新用）。

### 实战开关

```tsx
enabled: !!id                          // 条件请求：id 没拿到时不发
placeholderData: keepPreviousData      // 分页保留上一页数据，不闪骨架屏
qc.prefetchQuery({ queryKey: ['orders', page + 1], ... })   // 预取下一页
```

### 决策线

```
请求 < 5 个、无共享无缓存诉求   → useEffect + 手写 useFetch 够用
列表/详情/多组件共享/频繁增删改查 → TanStack Query
```

---

## 3.4 性能优化清单（memo / lazy / Suspense）

### 优化金字塔（顺序本身就是考点）

```
⑤ memo 三件套            最贵、最后动
④ 虚拟化长列表            长列表才需要
③ code splitting         lazy + Suspense
② 状态结构设计            状态下沉/派生不存储：免费，收益最大
① 测量 Profiler           永远第一步：没有数据就没有优化
```

越往下越是"改设计"（一次修复全局受益）；越往上越是"打补丁"（自带开销）。新人最大错误：直接跳到⑤给全世界包 memo。

### 第一优先级：状态结构（免费但最值钱）

- **状态下沉（colocation）**：state 离用它的组件越近越好（query 放 SearchBar 而不是 App）
- **派生不存储**：能算出来的不存 state（如 `log.length`）

**最好的优化是让渲染根本不发生。**

### code splitting：React.lazy + Suspense

```tsx
const Orders = lazy(() => import('./pages/Orders'))

<Route path="/orders" element={
  <Suspense fallback={<PageSkeleton />}>
    <Orders />
  </Suspense>
} />
```

机制：`import()` 动态导入 → 该模块单独打成一个 chunk，访问时才下载。

| React | Vue |
|---|---|
| `lazy(() => import(...))` | `defineAsyncComponent(() => import(...))` |
| `<Suspense fallback>` | `<Suspense><template #fallback>` |

**fallback 出现的唯一原因**：import() Promise pending（chunk 不在本地），与"页面渲染快慢"无关。第二次访问 chunk 已在浏览器缓存，fallback 一闪难见；发版后 chunk 文件名带 hash 变了 → 缓存键失配 → 重新下载（Vite 内容变 hash 变，自动击穿旧缓存）。

分层建议：**路由级**（默认做，每页一个 chunk）→ **重型组件级**（图表/富文本编辑器按需 lazy）→ 不要过分割（每个 chunk 有请求/解析开销）。

### 虚拟化长列表

渲染 10000 条 = 卡死。虚拟列表只渲染视口内 20 条，滚动时复用 DOM。库：`react-window` / TanStack Virtual。**判断线**：超过几百行且真卡再上，普通分页常常已够。

### 打印贴墙版清单

| 序 | 手段 | 成本 | 何时用 |
|----|------|------|--------|
| 0 | Profiler 测量 | 0 | 每次优化的第一步 |
| 1 | 状态下沉 | 0 | 默认遵守 |
| 2 | 派生不存储 | 0 | 默认遵守 |
| 3 | key 稳定 | 0 | 列表渲染默认遵守 |
| 4 | 路由级 lazy | 低 | 项目起步就配 |
| 5 | 重组件 lazy | 低 | 图表/编辑器 |
| 6 | memo 三件套 | 中 | 测出热点后 |
| 7 | 虚拟化列表 | 中 | 千行级列表 |
| 8 | useDeferredValue / startTransition | 中 | 输入即时反馈卡顿（搜索联想） |

### Vue 迁移者特别提示

- React **没有内置 keep-alive**：路由切走 = 组件卸载 = state 丢失。要缓存用社区库或把状态放路由外/store。这是"怎么又重新请求了"的常见来源
- computed 自动缓存心智没有免费等价物：**派生计算默认每次渲染重算**，贵了才 useMemo
- 最佳实践：筛选条件放 URL query（`useSearchParams`），免费获得分享链接/后退还原

---

# 附录：个人高频错误清单

来自各主题考核记录，反复出现的错误模式（考前过一遍）：

| 错误模式 | 出现次数 | 口诀 |
|---------|---------|------|
| **把调用当传递**：`onClick={fn()}`、`onAdd={setTodoList(...)}`、`return clearTimeout(timer)` | 5+ 次 | **括号是"现在就干"，不加括号是"到时候再干"** |
| **字符串字面量 vs 变量**：`key="item.id"`、`['user', 'userId']` | 2 次 | **引号里的永远不会变，变量才跟着数据走** |
| 渲染期间写 ref / 用 ref 存界面数据 | 多次 | 需要 UI 跟着变 → state；只是记下来 → ref |
| 混淆"重渲染"与"effect 重跑" | 多次 | deps 挡的是 effect 重跑，挡不住组件重渲染 |
| 函数式更新滥用：`setDebounced(c => c = value)` | 1 次 | 函数式更新只给"新值依赖旧值"的场景 |
| useMemo 大括号不 return / 多包箭头 | 1 次 | 检查返回值 |
| 双括号手滑 `function Fn((children))` | 5 次 | 写完看一眼参数列表 |

## 十大心智模型（全课程浓缩）

1. **JSX → Element → Fiber → DOM**：JSX 是对象描述，不是 DOM
2. **组件即函数，渲染即调用**：函数体每次渲染从零跑一遍
3. **state 是快照**：一次渲染一份闭包，读永远是"当轮照片"
4. **传值 = 成品，传函数 = 指令单**：依赖旧值必须函数式更新
5. **Object.is 是唯一判等尺**：更新永远换新引用
6. **props 跟数据走，state 跟位置走**（key = 身份证）
7. **cleanup 是"上一次副作用的撤销"**，不只是卸载
8. **effect 闭包几岁，读到的值就停在几岁**
9. **ref 是局外变量**：React 看得见盒子，看不见里面装了什么
10. **Vue 活引用 vs React 死快照**：迁移期最危险的是带着 Vue 直觉写 React
