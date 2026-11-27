# 4.3 RTK：createSlice + Immer——纪律保留，模板消灭

## 讲解原文

### 一、RTK 是什么

Redux Toolkit = **官方钦定的 Redux 最佳实践打包**。它不改变 Redux 哲学（action/reducer/单向流/纯函数照旧），只做一件事：**把纪律的执行成本砍到最低**。三个组成：

- `configureStore`：替代 createStore，自动集成 devtools + redux-thunk 中间件 + 开发环境防呆（意外 mutation 直接报错）
- `createSlice`：一个对象声明里同时产出 **reducer + actionCreators + actionType**——三处模板收进一处
- 内置 **Immer**：reducer 里可以"直接改"，Immer 帮你产出不可变新对象

### 二、createSlice 完整语法（对照 4.1 原教旨版）

```ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit'

type CounterState = { count: number }
const initialState: CounterState = { count: 0 }

const counterSlice = createSlice({
  name: 'counter',                    // 命名空间：actionType 自动变成 'counter/increment'
  initialState,
  reducers: {
    // ⚠️ 注意这些"看起来在直接改 state"的写法——Immer 代理，后面详讲
    increment(state) { state.count += 1 },
    decrement(state) { state.count -= 1 },
    reset(state) { state.count = 0 },
    // 带参数的 action：PayloadAction<T> 标注 payload 类型
    addAmount(state, action: PayloadAction<number>) {
      state.count += action.payload
    },
  },
})

// ⭐ 三件产物一次到齐（4.1 手写的 actionCreator 工厂，这里自动生成）：
export const { increment, decrement, reset, addAmount } = counterSlice.actions
export default counterSlice.reducer
```

**模板消灭对照表**（4.1 逐项回收）：

| 4.1 原教旨版 | RTK |
|--------------|-----|
| 手写 actionType 常量（`const INCREMENT = 'INCREMENT'`） | `name` 自动生成 `'counter/increment'` |
| 手写 actionCreator 工厂函数 | `counterSlice.actions` 自动导出 |
| reducer 的 switch + default | 每个方法即一个 case，**default 自动处理**（没匹配 = 原样返回） |
| 手写不可变展开 `{...state, count: ...}` | Immer 代理，直接 `state.count += 1` |

### 三、Immer 原理一句话：写法可变，产物不可变

reducer 收到的 `state` **不是真的 state**——是 Immer 造的 **Proxy 草稿**：

```
你写 state.count += 1
   → Proxy 拦截这次赋值，记录"草稿被改了"
   → reducer 返回时（或结束），Immer 根据记录生成【不可变的新 state】
   → 只重建"被改路径"上的对象，没动的子树引用原样复用（性能好）
```

所以 **1.1 的"React 需要新引用"没有被违反**——组件拿到的还是新对象。写法自由是 Immer 给的，产物纪律是 Immer 保的。

**纯度纪律还剩多少？** Immer 只把"手写不可变样板"自动化了，**reducer 依然必须纯**：
- 副作用（请求、setTimeout）——照旧禁止（4.4 的 thunk 处理）
- 非确定性（Math.random/Date.now）——照旧禁止（照旧破坏重放）
- configureStore 开发模式甚至内置了 mutation 检测，reducer 外改 state 直接报错

**嵌套深改是最大收益**（手写 spread 时代最痛的场景）：

```ts
type TodoState = { list: Todo[] }

toggleTodo(state, action: PayloadAction<string>) {
  // Immer 支持深路径直接改——手写 spread 时代要逐层 map/展开：
  const todo = state.list.find((t) => t.id === action.payload)
  if (todo) todo.done = !todo.done        // ← 一行搞定嵌套更新
}
```

### 四、configureStore 接线（消费层零变化）

```ts
// store.ts
import { configureStore } from '@reduxjs/toolkit'
import counterReducer from './counterSlice'

export const store = configureStore({
  reducer: { counter: counterReducer },   // 多 slice 自动合并成树状 state
})

export type RootState = ReturnType<typeof store.getState>   // ⭐ 一行导出全局 state 类型
// 组件里 useSelector((s: RootState) => s.counter.count)——TS 全程无 any
```

组件层（useSelector/useDispatch/Provider）与 4.2 **一字不差**——RTK 只改 store 定义层，消费层零学习成本。

### 五、一个 Immer 专属陷阱：mutate 和 return 别混写

```ts
reducers: {
  badCase(state, action) {
    state.count += 1
    return { count: 999 }    // ❌ return 一个新对象 = 提前退出草稿模式，
                             //    前面 state.count += 1 的改动【整个丢弃】！
  }
}
// 规则：一个 reducer 要么全程 mutate（不 return），要么 return 新对象（不 mutate），
// 二选一。全 mutate 是常态。
```

### 六、一句话总结

- createSlice = name + initialState + reducers 三合一，**actionCreator 自动生成、actionType 自动命名、default 自动处理**——4.1 的三处模板全部回收
- Immer：写法可变、产物不可变——纪律保留（副作用/非确定性照旧禁止），样板消灭
- 深路径直改是相对手写 spread 的最大收益；mutate 和 return 别混写
- configureStore 自动合并 reducer + 一行导出 RootState 类型；消费层与 4.2 零差异

### 补充：RTK 完整案例——todos 从定义到组件消费全接线（带注释）

```
文件流：types → todosSlice.ts（定义） → store.ts（组合） → main.tsx（Provider） → 组件（消费）
        与你 4.2 写的接线完全同构，只有"定义层"从手写 reducer 换成了 createSlice
```

**第 1 步：类型（`src/features/todos/types.ts`）**

```ts
export type Todo = {
  id: string
  text: string
  done: boolean
}

// Slice 的 state 形状——createSlice 的泛型 + initialState 都用它
export type TodosState = {
  list: Todo[]
  lastId: number      // 自增 id 用（真实项目 id 由服务端生成，mock 场景本地自增）
}
```

**第 2 步：Slice 定义（`src/features/todos/todosSlice.ts`）**

```ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit'
import type { Todo, TodosState } from './types'

const initialState: TodosState = { list: [], lastId: 0 }

const todosSlice = createSlice({
  name: 'todos',                       // actionType 命名空间：'todos/addTodo'
  initialState,
  reducers: {
    // ── actionCreator 自动生成：addTodo(text) ──
    addTodo(state, action: PayloadAction<string>) {
      state.lastId += 1                                    // 深路径直改 ✅
      state.list.push({                                    // push 而不是 [...spread]——Immer 兜底
        id: `todo-${state.lastId}`,                        // 用刚自增的 lastId 造 id
        text: action.payload,
        done: false,
      })
      // ⚠️ 全程 mutate、没有 return——符合"二选一"规则
    },

    // ── toggleTodo：深路径直改的教科书场景 ──
    toggleTodo(state, action: PayloadAction<string>) {
      const todo = state.list.find((t) => t.id === action.payload)
      if (todo) todo.done = !todo.done
      // 手写 spread 时代的等价物（对比 Immer 的收益）：
      // return { ...state, list: state.list.map(t =>
      //   t.id === action.payload ? { ...t, done: !t.done } : t
      // )}  ← 逐层展开，又长又容易漏层
    },

    // ── removeDone：filter 天然产出新数组，这里"混合风格"也合法——
    //    因为赋值给草稿字段（state.list = 新数组）依然是 mutate 草稿，没有 return ✅
    removeDone(state) {
      state.list = state.list.filter((t) => !t.done)
    },
  },
})

export const { addTodo, toggleTodo, removeDone } = todosSlice.actions  // actionCreator 自动生成
export default todosSlice.reducer
```

**第 3 步：store 组合（`src/store.ts`）**

```ts
import { configureStore } from '@reduxjs/toolkit'
import todosReducer from './features/todos/todosSlice'

export const store = configureStore({
  reducer: {
    todos: todosReducer,        // 树状 state：s.todos.list / s.todos.lastId
    // counter: counterReducer,  ← 其他 slice 继续往这棵树挂
  },
  // configureStore 默认自带：redux-thunk（4.4 用）+ Redux DevTools（开发环境自动开）
  // + 意外 mutation 检测（reducer 外改 state 直接报错，开发期防呆）
})

// ⭐ 一行导出全局 state 类型——组件里 useSelector 的 s 就有完整提示
export type RootState = ReturnType<typeof store.getState>
// 展开即：{ todos: TodosState }（每加一个 slice 自动跟着长）
```

**第 4 步：Provider 挂载（`src/main.tsx`，4.2 同款接线）**

```tsx
import { Provider } from 'react-redux'
import { store } from './store'

<Provider store={store}>
  <App />
</Provider>
```

**第 5 步：组件消费（`src/features/todos/TodoList.tsx`）**

```tsx
import { useSelector, useDispatch } from 'react-redux'
import { addTodo, toggleTodo, removeDone } from './todosSlice'
import type { RootState } from '@/store'

function TodoList() {
  // selector：从树状 state 选切片——返回原始字段/稳定引用，Object.is 比较有效（4.2 纪律）
  const todos = useSelector((s: RootState) => s.todos.list)
  const dispatch = useDispatch()

  const handleAdd = () => {
    const text = window.prompt('输入待办')
    if (text) dispatch(addTodo(text))     // actionCreator 自动生成，直接 dispatch
  }

  return (
    <>
      <button onClick={handleAdd}>添加</button>
      <button onClick={() => dispatch(removeDone())}>清除已完成</button>
      <ul>
        {todos.map((t) => (
          <li key={t.id}>
            {/* 深改 toggle：点击即 dispatch 语义事件，改数据是 slice 的事 */}
            <label onClick={() => dispatch(toggleTodo(t.id))}>
              <input type="checkbox" checked={t.done} readOnly />
              <span style={{ textDecoration: t.done ? 'line-through' : 'none' }}>
                {t.text}
              </span>
            </label>
          </li>
        ))}
      </ul>
    </>
  )
}

export default TodoList
```

**接线复盘（对照你的知识地图）**：

| 步骤 | 对应学过的 | 差异点 |
|------|-----------|--------|
| 定义层 | 4.1 手写 reducer + 4.1 优化版 actionCreator | createSlice 三合一自动化 |
| 组合层 | 3.2 RootStore / 4.2 configureStore | combineReducers 自动、RootState 一行导出 |
| Provider | 4.10 AuthProvider / 4.2 Provider | 完全同款 |
| 消费层 | 4.2 useSelector/useDispatch | 零变化 |

**查漏点**：组件里 selector 的 `s` 记得标 `RootState`；actionCreator 全部 import 直用（不在 store 树里，4.2 的坑）。

## 考核过程

 🎯 考核（编码 + 口头混合）

  任务 1（编码）：用 createSlice 写一个 todosSlice：

  - state：{ list: Todo[], lastId: number }（Todo = { id: string; text: string; done: boolean }）
  - reducers：addTodo(text)（用 lastId 造 id + 追加 + lastId 自增）、toggleTodo(id)（深路径直改，不许逐层展开）、removeDone()（过滤掉所有 done）
  - 导出 actions + reducer + RootState 类型声明

  任务 2（口头两问）：

  ① Immer 之后 reducer 还算纯吗？纯的三重回报（可测试/可重放/可追溯）保住了吗？逐条说。

  ② 为什么“mutate 和 return 混写”会丢改动？用“草稿 Proxy”的机制讲。

---------------------------

  任务 1（编码，在 todosSlice 上补两个 reducer）：
  - editTodo(id, text)：深路径直改某条的 text
  - restoreSnapshot(snapshot: TodosState)：用 return 整体替换 state（练习“二选一”的 return 侧——注意这次不能有任何 mutate）

（此段在考核通过后追加填充）
