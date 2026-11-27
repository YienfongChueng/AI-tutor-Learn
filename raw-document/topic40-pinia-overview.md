# 1.1 Pinia 全流程校准 + 为什么 Vue 生态不需要 reducer

## 讲解原文

### 一、统一分析框架：三问

所有状态管理库只回答三个问题，之后每个库都用这个框架过一遍：

| 三问 | Pinia 的答案 |
|------|-------------|
| 状态放哪？ | `defineStore` 定义 store，组件外单例 |
| 谁改？ | **谁都能改**——action 里改、组件里直接 `store.count++` 都合法 |
| 怎么通知？ | **proxy 自动追踪**——改了谁，谁用的组件自动更新，无需手动订阅 |

### 二、全流程语法快扫（校准你已有的知识）

```ts
// stores/counter.ts —— setup 写法（你熟悉的主力写法）
export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)                          // state
  const double = computed(() => count.value * 2) // getter
  function increment() { count.value++ }         // action
  return { count, double, increment }
})

// options 写法（老项目会见到，能读懂即可）
export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0 }),
  getters: { double: (state) => state.count * 2 },
  actions: { increment() { this.count++ } },
})

// 组件里
const store = useCounterStore()
store.increment()                        // action 调用
store.count++                            // 直接改也合法（响应式流派特色）
const { count, double } = storeToRefs(store)  // ⚠️ 解构必须 storeToRefs，否则丢响应性
store.$patch({ count: 100 })             // 批量改
store.$reset()                           // 回初始值（setup 写法需自己实现 $reset）
```

**持久化插件**（你用过的）：

```ts
pinia.use(piniaPluginPersistedstate)
// store 里第三参数 { persist: true } / { persist: { pick: ['count'] } }
// 本质：订阅 store.$subscribe → 每次变化写 localStorage → 初始化时读回
// —— 用三问框架看：它只是给"怎么通知"挂了一个副作用钩子
```

**易忘点清单**（项目实战全流程里的坑）：

1. `storeToRefs` 解构——直接解构 store 丢响应性（state/getter 转 ref），**函数（action）不能 storeToRefs**，直接取
2. 跨 store 调用：在一个 store 的 action 里直接 `const other = useOtherStore()`——天然组合
3. setup 写法没有免费 `$reset`：state 被展开成散装 ref，需要自己写 `$reset` 或用 options 写法
4. devtools 时间旅行 Redux 才完整（见第四节），Pinia 有 state 快照但回放能力弱

### 三、核心问题：为什么 Vue 生态不需要 mutation/reducer？

这是本节的面试锚点。你在 Vue3 主题块学过答案，现在把它接到状态管理上：

**Vue3 的响应式系统（proxy + 依赖收集）免费提供了"精确追踪"**——组件渲染时读了 `store.count`，这个 effect 就被记录为 `count` 的订阅者；`store.count++` 时 proxy 的 set 拦截器**精确通知**这些订阅者。

```
Pinia:  store.count++ ──proxy set 拦截──▶ 精确通知"读过 count"的组件
        （可变数据 + 自动细粒度追踪）

Redux:  必须生成新对象 ──浅比较──▶ useSelector 重新算 selector
        （不可变数据 + 订阅者自己比较"变没变"）
```

**Redux 为什么非要不可变？** React 没有依赖收集，订阅靠 `useSelector(store => ...)`——它判断"要不要重渲染"只能靠**引用比较**（Object.is）。可变数据下引用永远不变，比较失效；所以 Redux 强制 reducer 返回新对象，让"变化"体现在引用上。

> 一句话锚定：**响应式流派"改数据，框架替你追"；单向数据流派"改引用，订阅者自己比"。** 这不是品味差异，是 React/Vue 底层能力差异的必然推论。

### 四、Pinia vs Vuex（面试顺带）

| 维度 | Vuex 4 | Pinia |
|------|--------|-------|
| mutation | 必经（时间旅行的代价） | 取消（devtools 改为记录 state 快照） |
| 模块化 | modules + 嵌套命名空间 | 天然扁平（一 store 一文件） |
| TS | 弱（类型推导差） | 完整推导（setup 写法尤佳） |
| 组合 | getter 跨模块繁琐 | store 之间直接互相调用 |

Vue2 → Vuex 的 mutation 之所以存在，正是为了**让每次变更可序列化、可记录、可回放**（时间旅行）；Pinia 用"state 快照 diff"换掉了这层仪式。

### 五、一句话总结

- Pinia = 三问的"响应式流派"答案：放 store、随便改、proxy 自动通知
- 你的日常操作链：defineStore(setup) → storeToRefs 解构 → action/直接改 → persist 插件
- **不需要 reducer 的根因**：Vue3 响应式已提供精确追踪；React 没有追踪能力，只能靠不可变引用比较——这是两个生态状态管理差异的总根因

## 考核过程

### 题目：React 同事抱怨"改个数字又要 dispatch 又要 reducer"

**用户回答**：
1. React 没有依赖收集，可变数据下引用永远不变，Object.is 失效，所以生成新对象让变化体现在引用上；Vue/Pinia 有响应式，靠 proxy+依赖收集追踪，直接改 store 的 state 时 proxy 的 set 会拦截并通知订阅者。
2. 候选库选 Mobx。

**点评**：
- 第 1 问满分——双向机制（不可变+引用比较 / proxy set 拦截+精确通知）完整且方向正确
- 第 2 问库选对（Mobx），机制追问答"响应式系统"——哲学层面正确但表述偏泛，面试需具体化：proxy 拦截 + 依赖收集 + observer 精确订阅（与 Vue3 同款哲学、独立实现）。补充分已达成共识，判通过

---
通过时间：2026-11-24
