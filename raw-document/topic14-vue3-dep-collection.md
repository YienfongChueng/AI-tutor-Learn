# 主题块 2：依赖收集与触发

> Vue3 响应式源码 · vue@3.4 · 深度讲解模式（conceptual）· 参考 vue@3.4

## 讲解原文

### 子节 2.1：effect()：ReactiveEffect 与 activeEffect

**core_question：effect 与 ReactiveEffect 类、activeEffect 的关系**

---

#### 第 0 段：直觉锚定

把 `effect` 想成一个**订阅监工**：

- 监工上岗后巡视一圈仓库（执行 `fn`），每看一眼某样货物（读响应式数据），就在那个货物的**登记簿上签个名**（`track` 把当前监工记下）。
- 以后货物变了（`set` -> `trigger`），就按登记簿上的名单**通知监工重新巡视**（重新 `run`）。

「监工」= `ReactiveEffect` 实例；「巡视」= `run()` 执行 `fn`；「签名登记」= `track` 把 `activeEffect` 收进 dep；「当前正在巡视的监工」= `activeEffect` 全局变量。

> **回调主题块 1**：1.3 讲 get trap 调 `track(target, GET, key)`、set trap 调 `trigger`。但 `track` 怎么知道"**是谁在读**"？答案就是 `activeEffect`--effect `run` 时设的全局指针。这是主题块 1 埋的伏笔的答案。

---

#### 第 1 段：问题背景

主题块 1 讲了 handler 在 get 时调 `track`、set 时调 `trigger`。但留了两个未答的问题：

- `track` 收集"**谁**"？--当前正在运行的 effect。
- `trigger` 通知"**谁**"？--`track` 时收集的 effects。

要回答这两个，需要一个"当前 effect"的概念和"订阅者"的载体。`effect()` 就是创建并运行这个订阅者的入口。

> ⚠️ **常见先入为主的误解：** 很多人以为"effect 是 Vue 的渲染专用 API"。实际上 `effect` 是响应式系统的**底层原语**，组件渲染函数、`computed`、`watch` 都基于它实现。`effect` 本身与渲染无关，渲染只是在 effect 之上的一层应用。

---

#### 第 2 段：核心数据结构

```ts
// vue@3.4 · packages/reactivity/src/effect.ts
let activeEffect: ReactiveEffect | undefined   // 全局指针：当前正在运行的 effect

class ReactiveEffect<T = any> {
  active = true                              // 是否启用（stop 后 false）
  deps: Dep[] = []                           // 反向索引：该 effect 订阅的所有 dep（cleanup 用，2.6）
  parent: ReactiveEffect | undefined         // 嵌套 effect 父链（2.5）
  fn: () => T                                // 用户传入的函数
  scheduler: SchedulerFn | null = null       // 调度器（computed/watch 用，主题块3）

  constructor(fn, scheduler?) {
    this.fn = fn
    if (scheduler) this.scheduler = scheduler
  }

  run() { /* 见第 3 段 */ }
  stop() { cleanupEffect(this); this.active = false }   // 2.6
}

function effect(fn, options?) {
  const _effect = new ReactiveEffect(fn, options?.scheduler)
  if (!options?.lazy) _effect.run()          // 默认立即执行一次（收集依赖）
  const runner = _effect.run.bind(_effect)
  runner.effect = _effect                    // 暴露实例，供 stop 等使用
  return runner
}
```

字段关系：

```
effect(fn)
  └─> new ReactiveEffect(fn)
       ├ fn        ──> 用户函数（巡视任务）
       ├ deps[]    ──> 订阅的 dep 列表（反向索引，cleanup 用）
       ├ parent    ──> 外层 effect（嵌套恢复用）
       ├ active    ──> true（启用标志）
       └ scheduler ──> null（默认无，trigger 时同步 run）
  └─> run() 立即执行一次
       └─> activeEffect = this  ← 设全局指针
```

---

#### 第 3 段：运行流程

源码定位：`vue@3.4 · packages/reactivity/src/effect.ts · ReactiveEffect.run()`。

```ts
run() {
  if (!this.active) return this.fn()         // 已 stop：直接执行，不 track
  let parent = activeEffect
  let prevShouldTrack = shouldTrack
  try {
    this.parent = parent
    activeEffect = this                      // ← 设当前 effect
    shouldTrack = true
    return this.fn()                         // ← 执行 fn，读响应式数据 -> track 收集 this
  } finally {
    activeEffect = this.parent               // ← 恢复外层 activeEffect
    this.parent = undefined
    shouldTrack = prevShouldTrack
  }
}
```

端到端流程图：

```
effect(() => { console.log(state.count) })
  │
  ├─ new ReactiveEffect(fn)
  └─ _effect.run()
       ├─ activeEffect = this（ReactiveEffect 实例）   ← 设全局指针
       ├─ 执行 fn:
       │    └─ 读 state.count
       │         └─ get trap -> track(target, GET, 'count')
       │              └─ 把 activeEffect 收进 dep（2.3 详讲"收进哪"）
       └─ finally: activeEffect = parent（恢复，这里 parent=undefined）

  之后 state.count = 2:
       └─ set trap -> trigger(target, SET, 'count')
            └─ 找到 dep 里的 effect，重新 run()   ← 自动重新执行
```

> **回调主题块 1**：`track` 的 `(target, key)` 来自 get trap，但"**谁**"就是 `activeEffect`。effect `run` 时设 `activeEffect = this`，fn 里读响应式数据触发 track，track 就能拿到当前 effect。effect 结束恢复 `activeEffect = parent`，避免污染外层。

---

#### 第 4 段：设计动机与权衡

- **`activeEffect` 全局指针**：`track` 需知道"当前谁在读"，但 get trap 参数里没有 effect 信息。用全局 `activeEffect` 是最简方案--effect `run` 时设、`finally` 恢复。代价：非线程安全（JS 单线程，无碍）。
- **立即执行一次**：`effect(fn)` 默认立即 `run`，让首次 track 收集依赖。`lazy` 选项（`computed` 用）延迟到首次访问时才 `run`（主题块 3）。
- **`parent` 链恢复**：嵌套 effect 时，内层 `run` 设 `activeEffect=内层`，`finally` 恢复 `activeEffect=外层`。保证 track 收集到正确的 effect（2.5 详讲）。
- **`deps` 反向索引**：effect 记住自己订阅了哪些 dep，`stop`/`cleanup` 时能快速从所有 dep 移除自己（2.6 详讲）。这是"双向引用"--dep 记 effect，effect 也记 dep。
- **`scheduler` 可选**：默认 `trigger` 时同步 `run`；有 `scheduler` 则调 `scheduler`（`computed`/`watch` 用它控制时机，主题块 3）。
- **`runner` 返回值**：`effect` 返回绑定 `run` 的函数，可手动调用；`runner.effect` 暴露 `ReactiveEffect` 实例（`stop` 等用）。

---

#### 第 5 段：次级误解和边界

1. **误解：「effect 是渲染专用 API」** -> 错。`effect` 是底层原语，渲染/computed/watch 都基于它。
2. **误解：「`effect(fn)` 不立即执行」** -> 错。默认立即 `run` 一次收集依赖（除非 `lazy`）。
3. **误解：「`activeEffect` 是每个响应式数据自己的」** -> 错。`activeEffect` 是**全局唯一**的指针，effect `run` 时设，表示"当前正在运行的 effect"。
4. **边界**：
   - effect **外**直接读响应式数据不收集依赖（`activeEffect` 是 `undefined`，track 跳过）。
   - `stop()` 后 `effect.active=false`，`run` 时不 track（但 `fn` 仍执行）。
   - `shouldTrack` 全局开关可暂停 track（如数组方法 `push` 内部 `pauseTracking`，1.3 提过）。
   - 嵌套 effect 靠 `parent` 链正确恢复 `activeEffect`（2.5 详讲）。

---

**子节交接**：

> 现在我们知道了：`effect` 创建 `ReactiveEffect` 实例，`run` 时设 `activeEffect = this`，fn 里读响应式数据时 `track` 就能收集到当前 effect。但 `track` 把 effect 收进**哪个数据结构**？`trigger` 又怎么按 `(target, key)` 找到该通知的 effects？这需要一个依赖的存储结构--`targetMap`。这是 2.2 要处理的事。


## 考核过程

（主题块 2 全部讲完后统一考核，此处待填充）
