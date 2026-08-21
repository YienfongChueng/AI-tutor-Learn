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


---

### 子节 2.2：targetMap 三层结构

**core_question：WeakMap->Map->Set 三层结构为什么这样设计**

---

#### 第 0 段：直觉锚定

把依赖存储想成一个**图书馆的"到书通知"系统**：

- 读者（`ReactiveEffect`）在某本书（`target` 原始对象）的某一章（`key`）末尾的**通知名单**上留了电话
- 书一到新版本（`set` 发生），管理员按"哪本书 -> 哪一章 -> 名单"三级查找，逐个通知名单上的读者

结构对应：

| 类比 | 真实机制 |
|------|---------|
| 整个图书馆的总索引柜（按"书"查） | `targetMap: WeakMap<target, Map>` |
| 某本书内部的章节目录（按"章名"查） | `depsMap: Map<key, Set>` |
| 某章末尾的读者通知名单 | `dep: Set<ReactiveEffect>` |
| 读者 | `ReactiveEffect` 实例 |
| 留电话 | `dep.add(activeEffect)`（track） |
| 按名单通知 | 遍历 `dep` 执行 `run()/scheduler()`（trigger） |

> **回调 2.1**：2.1 说 track 把 `activeEffect`"收进 dep"——dep 就是这个名单；"收进哪"就是按 `(target, key)` 在这三层里定位。

---

#### 第 1 段：问题背景

2.1 解决了"track 知道**是谁**在读"（`activeEffect`）。还剩存储问题：

- **存**：track 拿到 `(target, key, activeEffect)` 三元组，要存到一个"查询仓库"里
- **取**：trigger 发生在 set trap 里，手里只有 `(target, key)`，要**反查出所有依赖这个 key 的 effects**

这个仓库必须满足四个要求：

1. **以对象为键**——依赖按"哪个对象"分组
2. **属性粒度**——`state.count` 变了不能通知只读 `state.name` 的 effect
3. **订阅者去重**——同一 effect 的 fn 里读两次 `state.count`，只能进名单一次
4. **自动垃圾回收**——`state` 对象整个不用了，它的依赖关系应该跟着被回收，不能内存泄漏

> ⚠️ **常见先入为主的误解：** 很多人以为依赖信息"挂在响应式对象身上"（比如 proxy 的某个属性上）。实际上依赖存在一个**模块级全局仓库 `targetMap`** 里，和 proxy 对象本身无关——proxy 只是在 get/set trap 里把 `(target, key)` 转交给 track/trigger。另外一个反直觉点：`targetMap` 的键是**原始对象 target**，不是 proxy。

---

#### 第 2 段：核心数据结构

```ts
// vue@3.4 · packages/reactivity/src/effect.ts
export type Dep = Set<ReactiveEffect>        // 名单：订阅这个 key 的所有 effect
export type KeyMap = Map<string | symbol, Dep>

const targetMap = new WeakMap<object, KeyMap>()  // 总仓库：raw target -> KeyMap
```

三层实例图（两个原始对象、三个 dep、三个 effect）：

```
targetMap (WeakMap)
  │
  ├─ key: state (原始对象①)
  │    └─ value: depsMap (Map)
  │         ├─ key: 'count' ──值──> dep_A (Set) ──包含──> effect_1 (渲染effect)
  │         │                                      └──包含──> effect_2 (watch effect)
  │         └─ key: 'name' ──值──> dep_B (Set) ──包含──> effect_1
  │
  └─ key: other (原始对象②)
       └─ value: depsMap (Map)
            └─ key: 'age' ────值──> dep_C (Set) ──包含──> effect_3 (computed effect)
```

对应代码场景：

```js
const state = reactive({ count: 0, name: 'a' })   // state 是 proxy，target 是 { count:0, name:'a' }
const other = reactive({ age: 1 })

effect_1 = () => console.log(state.count, state.name)  // 进 dep_A 和 dep_B
effect_2 = watchEffect(() => state.count)               // 进 dep_A
effect_3 = computed(() => other.age)                    // 进 dep_C
```

注意两条**双向引用**（2.1 埋过伏笔）：

- 正向：`dep_A` 包含 `effect_1`（名单里有读者）——trigger 用
- 反向：`effect_1.deps = [dep_A, dep_B]`（读者记着自己留过电话的名单）——cleanup 用（2.6）

---

#### 第 3 段：运行流程

track 和 trigger 是这个仓库唯一的两个使用者，一个写入、一个读取。

**源码定位**（四步格式）：

1. 定位：`vue@3.4 · packages/reactivity/src/effect.ts · track()` 和 `trigger()`（`createDep` 在 `packages/reactivity/src/dep.ts`，具体行号随版本浮动，以函数名为准）
2. 签名解读：`track(target, type, key)`——target 是**原始对象**（get trap 的第一个参数），type 是操作类型（GET/SET/HAS/DELETE...，决定 2.4 里是否追加 ITERATE_KEY 依赖），key 是属性名；`trigger(target, type, newValue?, oldValue?)` 内部自己从 `type` 推出 key
3. 阅读入口：先只看两个函数里操作 `targetMap` 的那几行，其余（`newTracked` 标记、`effectsToRun`、ADD/DELETE 分支）分别留给 2.3 和 2.4
4. 关键行标注：track 里的 `targetMap.get(target)` 三步取-or-建，和 trigger 里的 `depsMap.get(key)` 是本节核心

**track 侧（写仓库）骨架：**

```ts
function track(target, type, key) {
  if (!activeEffect) return                    // 没有正在运行的 effect：不收集
  // —— 三步取-or-建（本节核心）——
  let depsMap = targetMap.get(target)          // 第一层：WeakMap 按原始对象取
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))  // 没有：建第二层 Map
  }
  let dep = depsMap.get(key)                   // 第二层：Map 按 key 取
  if (!dep) {
    depsMap.set(key, (dep = createDep()))      // 没有：建第三层 Set（dep.ts）
  }
  // —— 收进名单（2.3 详讲：含 dep.add + effect.deps 反向注册 + 重复收集防护）——
  dep.add(activeEffect)
}
```

**trigger 侧（读仓库）骨架：**

```ts
function trigger(target, type, newValue?, oldValue?) {
  const key = type 触发属性（SET/ADD/DELETE 对应 key，ADD/DELETE 还涉及 ITERATE_KEY，2.4 讲）
  let depsMap = targetMap.get(target)          // 第一层
  if (!depsMap) return                         // 这个对象从没被 track 过：无依赖，直接返回
  const dep = depsMap.get(key)                 // 第二层+第三层
  if (!dep) return                             // 这个 key 没人依赖

  const effects = [...dep]                     // 拷贝再遍历，防止遍历中增删 Set（2.6 的无限循环主题）
  for (const effect of effects) {
    if (effect !== activeEffect) {             // 防自己触发自己（2.6）
      if (effect.scheduler) effect.scheduler() // 有调度器走调度（computed/watch，主题块3）
      else effect.run()                        // 否则同步重跑
    }
  }
}
```

端到端流程图（与 2.1 的监工图衔接）：

```
【收集阶段】effect_1 run:
  activeEffect = effect_1
  └─ fn 读 state.count
       └─ get trap(target, 'count')
            └─ track(target, GET, 'count')
                 ├─ targetMap.get(target) ─── 已有 depsMap（复用）
                 ├─ depsMap.get('count')  ─── 已有 dep_A（复用）
                 └─ dep_A.add(effect_1)        ← 名单上签名

【触发阶段】state.count = 2:
  └─ set trap(target, 'count', 2)
       └─ trigger(target, SET, 2)
            ├─ targetMap.get(target)   ───> depsMap（没有则 return）
            ├─ depsMap.get('count')    ───> dep_A（没有则 return）
            └─ 遍历 dep_A：effect_1.run() / effect_2.scheduler()
```

---

#### 第 4 段：设计动机与权衡

三个"为什么这样选型"：

1. **最外层为什么是 WeakMap？**
   核心约束是**内存回收**（上面四要求的第 4 条）。WeakMap 对 key 是弱引用：当原始对象 `state` 在业务代码里再无任何强引用、可被 GC 时，`targetMap` 里以它为键的整条依赖记录（depsMap + 所有 dep）**自动跟着消失**，不需要任何手动清理。若用 `Map`/普通对象，仓库会永久强引用每个 target，组件销毁后依赖表成为内存泄漏点。代价：WeakMap 不可遍历（无法统计"全仓库有多少依赖"）——Vue 不需要这个能力，可接受。

2. **中间层为什么是 Map？**
   key 可以是 `string | symbol`（Vue3 内部用 symbol 做特殊 key，比如 `ITERATE_KEY` 标记"for...in/数组迭代"这类对整个集合的依赖，2.4 展开）；普通对象的 key 只能字符串，且会撞上原型链（1.3 讲过 `hasOwn` 判断，这里同理避坑）。

3. **最内层为什么是 Set？**
   核心约束是**去重 + O(1) 增删查**。fn 里读 `state.count` 两次、或 effect 被多次触发后重新收集，`Set.add` 天然幂等；`Array.push` 需要先 `includes` 判重（O(n)），删除更麻烦。

4. **键为什么是原始对象而不是 proxy？**
   两个原因：(a) get/set trap 回调的**第一个参数就是原始 target**，track/trigger 直接拿来用，不必反查；(b) 同一个原始对象可能被 `reactive()` 和 `shallowReactive()` 包出不同 proxy，用原始对象做键能让它们**共享同一份依赖表**，语义统一。

牺牲了什么：三层嵌套每次 track/trigger 都要三次哈希查找 + 两次"没有则建"的分支判断，这是**用查询开销换自动内存回收和属性粒度精确性**，对前端场景是划算的。

---

#### 第 5 段：次级误解和边界

1. **误解：「依赖挂在 proxy 对象上」** -> 错。依赖集中存在全局 `targetMap`，键是**原始对象**。proxy 只是"传感器"，把 `(target, key)` 转交给 track/trigger，自己不存任何依赖数据。
2. **误解：「dep 里存的是用户传的那个 fn 函数」** -> 错。存的是 **`ReactiveEffect` 实例**。trigger 遍历时需要读实例的 `scheduler` 字段决定"同步 run 还是走调度"（2.1 的字段表），存裸函数就丢了这些状态。
3. **误解：「每个 effect 有自己独立的 dep」** -> 错。dep 以 `(target, key)` 为粒度**全局共享**——effect_1 和 effect_2 都读 `state.count`，进的是**同一个** `dep_A`。这正是"改一个属性只通知依赖它的人"能成立的根基。
4. **边界**：
   - trigger 时对象/key 从未被 track 过 -> 第二层就 return，零成本
   - `for (let k in state)` / 数组迭代这类"依赖整个集合"的操作不是挂在某个具体 key 上，而是挂在特殊键 `ITERATE_KEY` 上（symbol）——这也是中间层必须用 Map 的原因之一，2.4 细讲
   - WeakMap 不可枚举，所以 Vue 生态的开发者工具（devtools 展示依赖树）走的是别的钩子，不是遍历 targetMap

---

**子节交接**：

> 现在我们知道了：track/trigger 共用一个全局三层仓库 `WeakMap<target, Map<key, Set<ReactiveEffect>>>`，track 三步取-or-建后 `dep.add(activeEffect)`，trigger 三步查找后遍历执行。但上面 track 骨架的最后一步被我们折叠了——`dep.add` 之前和之后其实还有**反向注册 `effect.deps`、防重复收集的标记位**等动作，完整的 `track()` 比骨架多做了什么？这是 2.3 要处理的事。


---

### 子节 2.3：track()：当前 effect 如何进 dep

**core_question：track 如何把当前 activeEffect 收进 dep**

---

#### 第 0 段：直觉锚定

延续 2.2 的图书馆系统。这次要解决的问题是：**老读者的"签到册"维护**。

读者（effect）每次重新巡视（`run`）时，签到逻辑不是简单地"在名单上再写一遍名字"：

- 巡视**开始前**，图书馆在读者**上次留过电话的所有名单**（`effect.deps`）上盖一个"旧"章（`w` 标记）
- 巡视中每到一个名单签名（`track`），就盖一个"新"章（`n` 标记）
- 巡视**结束后**结算：**只盖了旧章、没盖新章的名单** = 这次没来签到 = 该读者已不关心这个货物 -> **把他的电话划掉**

这就是"增量清理"：不拆掉整个签到册重建（全量 cleanup），而是用两个章判断哪些签名过期了。

> 为什么要清理？先记住一个场景：`effect(() => flag ? a.x : b.y)`--`flag` 从 true 变 false 后重跑，这次读的是 `b.y` 不再读 `a.x`，但 `a.x` 的名单上还留着他的电话，以后 `a.x` 变了还会通知他（多余的 run）。清理就是为了去掉这种**过期依赖**。完整故事在 2.6。

---

#### 第 1 段：问题背景

2.2 的骨架把 `dep.add(activeEffect)` 折叠了。展开后 track 要回答三个问题：

1. **该不该收？**--`activeEffect` 为空（effect 外读数据）或全局开关 `shouldTrack` 关闭时，不收
2. **怎么收不重？**--fn 里读两次 `state.count`、或重跑后再收集，不能重复注册（`Set.add` 天然幂等能兜底，但每次 `dep.has()` 查重有开销，且解决不了过期依赖）
3. **过期依赖谁来清？**--见第 0 段场景。两条路线：
   - **全量清理**：每次 run 前，把 effect 从它订阅的**所有** dep 里删掉，再全部重新收集（简单粗暴，代价 O(所有依赖数)，且有一触发无限循环的坑，2.6 讲）
   - **增量清理（Vue3.2+ 实际采用）**：run 前"盖旧章"、收集中"盖新章"、run 后删"只有旧章的"——平时不删，结算时删，均摊代价接近 O(本次实际访问数)

> ⚠️ **常见先入为主的误解：** 很多人以为 track 的核心就是 `dep.add(activeEffect)` 一行，顶多加个 `dep.has` 查重。实际上这行前后各有一半机制：**前一半**判断"该不该收、收过没有"（标记位优化），**后一半**反向注册 `activeEffect.deps.push(dep)`（2.1 埋的反向索引伏笔，在这里兑现）。收进去只是一半，**登记双向关系**才是完整动作。

---

#### 第 2 段：核心数据结构

```ts
// vue@3.4 · packages/reactivity/src/dep.ts / effect.ts（字段名以函数名为准，随小版本可能微调）
type Dep = Set<ReactiveEffect> & { w: number; n: number }
//                    │            │       └ newTracked：本层 run 中"新收集"标记
//                    │            └ wasTracked：本层 run 开始前"已存在"标记
//                    └ 名单本体（2.2 讲的 Set）

// effect.ts
let effectTrackDepth = 0     // effect 嵌套深度（2.5）
let trackOpBit = 1           // 位掩码：1 << effectTrackDepth，每层嵌套占一个二进制位
```

`w`/`n` 不是布尔而是**位掩码**：嵌套 effect 场景（2.5）里，内层和外层 run 各占一位，互不覆盖，这样内层结算时不会误删外层刚盖的章。普通单层场景可以把它当布尔理解。

双向引用实例图（3 个节点，箭头带语义）：

```
dep_A (state.count 名单)              effect_1
 ├ 成员：effect_1                     ├ fn: () => state.count + state.name
 ├ w: 0b10 (外层run盖的旧章)           ├ deps: [dep_A, dep_B]
 └ n: 0b01 (内层run盖的新章)           │         │
       ▲                             │         │
       └──────包含(正向)──────────────┘         │
       ┌────────────────────────────────────────┘
       └──effect_1.deps 注册(反向)──> dep_B (state.name 名单) 也含 effect_1

规则（finalizeDepMarks 结算时）：
  w=1 且 n=0  -> 从 dep 删除该 effect（过期依赖）
  n=1         -> 保留（本轮还在用 / 本轮新加）
```

---

#### 第 3 段：运行流程

**源码定位（四步格式）：**

1. 定位：`vue@3.4 · packages/reactivity/src/effect.ts · track()`、`trackEffects()`；标记相关：同文件 `initDepMarkers()`、`finalizeDepMarks()`（`run()` 的 try/finally 中调用；具体行号随版本浮动，以函数名为准）
2. 签名解读：`track(target, type, key)`--2.2 已讲；`trackEffects(dep)`--把"当前 activeEffect 收进指定 dep"的全部动作封装在这里
3. 阅读入口：先读 `track()`（约 10 行，含守卫 + 三步取-or-建），再读 `trackEffects()`（核心判断都在这），最后对照 `run()` 里 init/finalize 两个调用点
4. 关键行标注：`trackEffects` 里的 `shouldTrack` 三分支判断 + `dep.add` / `activeEffect.deps.push` 两行，是本节核心

**完整伪代码（保留全部分支）：**

```ts
// get trap 入口（1.3 讲过）
function track(target, type, key) {
  if (!shouldTrack || !activeEffect) return    // 守卫：开关关着 或 没有正在运行的 effect
  // -- 三步取-or-建（2.2 讲过）--
  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))
  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, (dep = createDep()))   // createDep 同时初始化 w/n
  trackEffects(dep)
}

function trackEffects(dep) {
  let shouldTrack = false
  if (effectTrackDepth <= maxMarkerBits) {     // 嵌套 ≤30 层：走标记位优化
    if (!newTracked(dep)) {                    // 本轮还没收过这个 dep
      dep.n |= trackOpBit                      //   盖"新"章
      shouldTrack = !wasTracked(dep)           //   之前就有 -> 本轮不需要重复 add（Set里已有）
    }                                          //   newTracked 过 -> shouldTrack=false，跳过
  } else {                                     // 嵌套超过 30 位：标记位不够用
    shouldTrack = !dep.has(activeEffect)       //   退化为 Set 查重
  }
  if (shouldTrack) {
    dep.add(activeEffect)                      // ← 正向：名单收人
    activeEffect.deps.push(dep)                // ← 反向：读者记名单（2.1 伏笔兑现）
    // dev 模式：onTrack 钩子（devtools 用），忽略
  }
}

// run() 的首尾（2.1 只画了中段，这里补全）：
run() {
  ...
  activeEffect = this
  effectTrackDepth++                           // 嵌套深度 +1
  trackOpBit = 1 << effectTrackDepth           // 本层的位
  initDepMarkers(this)                         // ← 给 this.deps 里每个 dep 盖"旧"章(dep.w |= trackOpBit)
  try {
    return this.fn()                           // fn 里每次读响应式数据 -> track -> trackEffects 盖"新"章+收人
  } finally {
    finalizeDepMarks(this)                     // ← 结算：w有 n无 的 dep 删掉该 effect；effect.deps 压缩
    effectTrackDepth--
    trackOpBit = 1 << effectTrackDepth
    activeEffect = this.parent
    ...
  }
}

function finalizeDepMarks(effect) {
  const { deps } = effect
  let ptr = 0
  for (let i = 0; i < deps.length; i++) {
    const dep = deps[i]
    if (wasTracked(dep) && !newTracked(dep)) { // 上轮就有 + 本轮没再收集 = 过期
      dep.delete(effect)                       //   从名单划掉电话
    } else {
      deps[ptr++] = dep                        //   保留并原地压缩数组
    }
  }
  deps.length = ptr
}
```

单次 run 的时序图：

```
effect_1.run()
  ├─ activeEffect = effect_1, effectTrackDepth++, trackOpBit=本层位
  ├─ initDepMarkers: effect_1.deps 里的 dep_A.w |= bit（盖旧章）
  ├─ 执行 fn:
  │    ├─ 读 state.count -> track -> trackEffects(dep_A)
  │    │    ├─ newTracked? 否 -> dep_A.n |= bit（盖新章）
  │    │    ├─ wasTracked? 是 -> shouldTrack=false → 不重复 add（已在名单）
  │    └─ (若 flag 翻转后没读 a.x，dep_a.x 只有 w 没有 n)
  └─ finally: finalizeDepMarks
       ├─ dep_A: w=1,n=1 -> 保留
       └─ dep_a.x: w=1,n=0 -> dep_a.x.delete(effect_1)  ← 过期依赖被清理
```

---

#### 第 4 段：设计动机与权衡

- **核心约束：重跑时依赖会变（分支切换），过期依赖必须清掉，但清理本身要便宜。**
  全量清理（每次 run 前从所有 dep 删除再重收集）逻辑最简单，但每次重跑都要 O(总依赖数) 次删除+可能的重加；且 2.6 会讲到它有触发无限循环的坑。增量清理把删除推迟到结算时，只处理"确实过期"的条目，均摊 O(本轮访问的属性数)。
- **为什么用位掩码而不是布尔？** 服务于嵌套 effect（2.5）：外层和内层 run 同时在栈上，同一个 dep 可能被两层先后盖章。布尔会互相覆盖（内层结算误删外层的章）；每位一层，结算时各看各的位。
- **`maxMarkerBits`（30 位）溢出兜底**：JS 位运算 32 位，嵌套超过 30 层就退回 `dep.has` 查重 + 保守策略。真实业务嵌套不了这么深，这是防御性设计。
- **`activeEffect.deps.push(dep)` 的意义**：反向索引是 finalizeDepMarks 能工作的前提--结算时遍历的是 `effect.deps`（我订阅了谁），不是全仓库扫描。双向引用换 O(本 effect 依赖数) 的清理。
- **牺牲**：`Dep` 从纯 Set 变成带 hack 属性的对象（`w`/`n` 直接挂在 Set 实例上），类型不干净；且 dep 被多个 effect 共享（2.2），w/n 是"按层"共享的标记，逻辑精妙但难读。这是**用实现复杂度换运行时性能**的典型取舍。

---

#### 第 5 段：次级误解和边界

1. **误解：「`dep.add` 就完成收集了」** -> 不完整。收集是**双向注册**：`dep.add(effect)` + `effect.deps.push(dep)`。少了反向注册，增量清理、`stop()`（2.6）都无法工作。
2. **误解：「每次 track 都要查重 `dep.has`」** -> 3.4 主路径不用。标记位优化让"收过没有"变成读一个整数字段（`newTracked`/`wasTracked`），比 Set.has 更快，且顺便携带了"这轮收集是不是新的"信息。只有嵌套超 30 层才退回 `dep.has`。
3. **误解：「过期依赖是 trigger 时清的」** -> 错。过期依赖在**下一次 run 的结算阶段**（`finalizeDepMarks`）清理。trigger 只管"通知名单上的人"，不管名单过期。
4. **边界**：
   - `shouldTrack` 全局开关（数组方法内部 `pauseTracking` 等，1.3/2.1 提过）能让 track 直接短路
   - 开发模式下 `trackEffects` 末尾触发 `onTrack` 调试钩子（devtools 依赖面板靠它）
   - **版本边界（重要）**：`vue@3.5` 起响应式重写为**版本计数 + 双向链表**（`dep.subs` / `sub.deps` 链表，`SubEffect.version`），不再用 w/n 标记位。但解决的问题是同一个（重跑时高效清理过期依赖），本节的"增量清理"思想仍是理解 3.5+ 的地基

---

**子节交接**：

> 现在我们知道了：track 在三步定位 dep 后，用"旧章/新章"标记做查重与增量清理，双向注册 `dep.add(effect)` + `effect.deps.push(dep)`，run 结束时结算掉过期依赖。收的方向闭环了。但付的方向还有悬念：set trap 调 `trigger` 后，除了"按 key 找 dep 执行 effects"，还有 **ITERATE_KEY（for...in/数组遍历的依赖）、新增/删除属性时的连锁触发、拷贝后再遍历防遍历中修改** 这些细节。这是 2.4 要处理的事。


---

### 子节 2.4：trigger()：找到 dep 并执行 effects

**core_question：trigger 如何按 key 找到 dep 并执行 effects**

---

#### 第 0 段：直觉锚定

图书馆通知环节。表面上"某章新版到了，按那章的名单打电话"就够了，但真实系统多两条规则：

1. **目录订阅者**：有一类特殊读者订阅的不是某一章，而是"**这本书的目录本身**"（`ITERATE_KEY`，对应 `for...in`/数组遍历/展开）。你**新增或删除一章**，目录变了，即使没改任何现有章节的内容，也必须通知他们。
2. **先复印名单再打电话**：打电话过程中，读者可能会当场要求把名字**加进或划出**其他名单（run 时重新收集/清理依赖）。对着原件边打电话边改名单，会漏打或重打--所以通知前先把名单**复印一份**（拷贝 Set），对复印件打电话。

| 类比 | 真实机制 |
|------|---------|
| 目录订阅 | `ITERATE_KEY`（symbol）上的 dep |
| 新增/删除章节 | `trigger` 的 ADD / DELETE 类型 |
| 复印名单 | `[...dep]` 拷贝后再遍历 |
| 电话接通后的动作 | `effect.scheduler()` 或 `effect.run()` |

---

#### 第 1 段：问题背景

set trap 之外，delete、数组下标赋值、`length` 赋值、`map.clear()` 都会改变响应式数据且各有"该通知谁"的语义：

- **改已有属性的值**（SET）-> 通知这个 key 的 dep。但注意：值没变不用通知（`hasChanged` 判断在 baseHandlers 的 set trap 里，1.3 讲过 `!hasChanged` 则直接 return，根本不进 trigger）
- **新增属性**（ADD，`obj.newKey = 1`）-> 除了这个 key，**键的集合变了**，还要通知 `ITERATE_KEY` 的 dep；数组下标新增还可能改变 `length`，要通知 `length` 的 dep
- **删除属性**（DELETE）-> 同 ADD，要通知 `ITERATE_KEY`
- **数组 `length` 赋值缩短**（`arr.length = 2`）-> 所有 `>= 新length` 的下标对应的 dep 都要通知（那些元素"没了"），外加 `length` 自己
- **`map.clear()`**（CLEAR）-> 键全没了，**所有** dep 都通知

所以 trigger 不是"找一个 dep"，而是"**按操作类型算出一个 dep 列表**，聚合、去重、执行"。

> ⚠️ **常见先入为主的误解：** 很多人以为 trigger 只通知"被改的那个 key"的 dep。实际上一次 `obj.newKey = 1` 会通知**两个** dep（`newKey` 的 + `ITERATE_KEY` 的），`arr[5] = x`（原 length 3）会通知**两个**（下标 5 的 + `length` 的），`map.clear()` 通知**全部**。通知范围由操作语义决定，不由"你改了什么"单独决定。

---

#### 第 2 段：核心数据结构

```ts
// vue@3.4 · packages/reactivity/src/effect.ts / dep.ts
const effects: ReactiveEffect[] = []   // trigger 内的聚合容器（可能来自多个 dep）

// packages/reactivity/src/baseHandlers.ts（constants）
export const ITERATE_KEY = Symbol(__DEV__ ? 'iterate' : '')        // for...in / 数组迭代的依赖键
export const MAP_KEY_ITERATE_KEY = Symbol(__DEV__ ? 'Map key iterate' : '')  // map.keys() 类依赖
```

一个 depsMap 里的完整视图（含 ITERATE_KEY）：

```
depsMap (Map) —— target = list (数组原始对象)
  ├─ '0' ────────值──> dep_0 (Set) ──包含──> effect_A（读了 list[0]）
  ├─ '1' ────────值──> dep_1 (Set) ──包含──> effect_B
  ├─ 'length' ───值──> dep_len (Set) ──包含──> effect_C（渲染函数 v-for）
  └─ ITERATE_KEY ─值──> dep_iter (Set) ──包含──> effect_C（v-for 也遍历了数组）

触发示例：
  list[0] = 9        (SET, key='0')   -> 通知 dep_0
  list.push(9)       (ADD, key='3')   -> 通知 dep_3 + dep_len（下标3是新增，length也变了）
  list.length = 1    (SET length=1)   -> 通知 dep_1 + dep_len（下标1的元素没了 + length变了）
  delete list.x      (DELETE)         -> 通知 dep_x + dep_iter（对象才走 dep_iter）
```

注意 `effect_C` 同时在 `dep_len` 和 `dep_iter` 里（v-for 既读了 length 又遍历了元素）--这就是"聚合后必须去重"的根源。

---

#### 第 3 段：运行流程

**源码定位（四步格式）：**

1. 定位：`vue@3.4 · packages/reactivity/src/effect.ts · trigger()`、`triggerEffects()`、`triggerEffect()`（行号随版本浮动，以函数名为准）
2. 签名解读：`trigger(target, type, newValue?, oldValue?, oldTarget?)`--type 是 TriggerOpTypes（SET/ADD/DELETE/CLEAR），newValue 在 `length` 赋值时用来比较下标；oldTarget 用于开发模式调试信息
3. 阅读入口：`trigger()` 前半段是"**算 dep 列表**"（本节核心），后半段"聚合执行"调 `triggerEffects`；`triggerEffect` 只有一个 if/else，看 `scheduler` 分流
4. 关键行标注：`switch (type)` 的三个分支 + `effect !== activeEffect` 判断 + `effect.scheduler ? scheduler() : run()`

**伪代码（保留全部分支）：**

```ts
function trigger(target, type, newValue?, oldValue?) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return                          // 从未被 track：无人可通知

  // ---- 第一阶段：按操作类型算出 dep 列表 ----
  let deps: (Dep | undefined)[] = []

  if (type === CLEAR) {                         // map.clear()
    deps = [...depsMap.values()]                // 全部 dep
  } else if (key === 'length' && isArray(target)) {   // 数组 length 赋值
    depsMap.forEach((dep, key) => {
      if (key === 'length' || key >= newValue)  // length 自身 + 被砍掉的下标
        deps.push(dep)
    })
  } else {
    if (key !== undefined) deps.push(depsMap.get(key))   // 常规：key 自己的 dep
    switch (type) {
      case ADD:                                 // 新增属性/新增下标
        if (!isArray(target)) {
          deps.push(depsMap.get(ITERATE_KEY))   // 对象：目录变了
          if (isMap(target)) deps.push(depsMap.get(MAP_KEY_ITERATE_KEY))
        } else if (isIntegerKey(key)) {
          deps.push(depsMap.get('length'))      // 数组下标新增：length 可能变
        }
        break
      case DELETE:                              // 删除属性
        if (!isArray(target)) {
          deps.push(depsMap.get(ITERATE_KEY))
          if (isMap(target)) deps.push(depsMap.get(MAP_KEY_ITERATE_KEY))
        }
        break
      case SET:                                 // 改已有值
        if (isMap(target)) {
          deps.push(depsMap.get(MAP_KEY_ITERATE_KEY))  // map.set 改值：keys() 顺序没变但语义上通知
        }
        break                                   // 普通对象：只通知 key 自己（改值不动目录）
    }
  }

  // ---- 第二阶段：聚合 + 去重 ----
  const effects: ReactiveEffect[] = []
  for (const dep of deps) {
    if (dep) effects.push(...dep)               // 多个 dep 的成员倒进同一个数组（可能有重复，如 effect_C）
  }

  // ---- 第三阶段：执行 ----
  triggerEffects(new Set(effects))              // 用 Set 去重：同一 effect 只跑一次
}

function triggerEffects(dep: Set<ReactiveEffect>) {
  const effects = isArray(dep) ? dep : [...dep] // ← 复印名单（第 0 段的类比）
  for (const effect of effects) {
    // computed 有单独分支（涉及 dirty 标记，主题块 3 讲，此处折叠）
    triggerEffect(effect)
  }
}

function triggerEffect(effect) {
  if (effect !== activeEffect || effect.allowRecurse) {  // 防自己触发自己（2.6 详讲）
    if (effect.scheduler) effect.scheduler()    // 有调度器：交给调度器（渲染/watch 的异步批量入口，主题块3）
    else effect.run()                           // 没有调度器：同步立即重跑
  }
}
```

分支决策图（横向状态机：操作类型 -> 通知范围）：

```
                    ┌─ SET  普通对象 ──────────────> [key 自己]
                    ├─ SET  Map.set改值 ───────────> [key 自己, MAP_KEY_ITERATE]
 trigger(type,key) ─┼─ ADD  普通对象 ──────────────> [key 自己, ITERATE_KEY]
                    ├─ ADD  数组 isIntegerKey ─────> [key 自己, 'length']
                    ├─ DELETE 普通对象 ────────────> [key 自己, ITERATE_KEY]
                    ├─ SET length=更小 (数组) ─────> ['length', 所有 >= newLength 的下标]
                    └─ CLEAR (map.clear) ─────────> [全部 dep]
                                    │
                                    ▼
                    聚合 effects 数组 -> new Set 去重 -> 逐个执行
                                    │
                                    ▼
                    effect !== activeEffect ?
                      ├─ 是 -> scheduler ? scheduler() : run()
                      └─ 否 -> 跳过（防 count++ 自触发）
```

---

#### 第 4 段：设计动机与权衡

- **`effects` 数组 + `new Set` 去重**：一个 effect 可能同时出现在多个被通知的 dep 里（渲染函数读 `length` 又遍历元素）。不去重就 run 两次--纯浪费。用 Set 聚合是**O(n) 一次去重**，比两两比对 dep 便宜。
- **遍历前拷贝 `[...dep]`**：run 会重新走 track -> 可能向正在被遍历的 dep `add`/`delete` 成员（增量清理、2.3 的 finalizeDepMarks）。对正在遍历的 Set 做增删，行为未定义/会跳项。拷贝一份隔离读和写。代价：一次浅拷贝。
- **`effect !== activeEffect` 防自环**：`effect(() => count.value++)` 里 run 期间 set 又触发 trigger，名单里正是自己--不挡就是无限递归。挡掉的代价是**合法的递归更新也不生效**，所以留了 `allowRecurse` 逃生口（watch 深度递归场景用）。完整攻防在 2.6。
- **scheduler 优先于 run**：这是**异步批量更新的地基**--渲染 effect 和 watch 都带 scheduler，trigger 时不真跑，只把 job 入队，多次同步修改合并成一次执行（queueJob / nextTick，主题块 3）。trigger 本身保持"通知"语义，**什么时候跑、跑几次**交给调度层。这样 trigger 函数保持纯粹（找 dep + 通知），职责分离。
- **ITERATE_KEY 放进同一个 depsMap**：而不是单独建一张"迭代依赖表"。好处：track 侧（1.3 的 `get` trap 里 has/iterate 分支）和 trigger 侧用同一套三层结构、同一套 trackEffects/triggerEffects，无特判代码路径。代价：depsMap 里混进了一个"假键"，遍历 depsMap 的代码都要意识到它存在。

---

#### 第 5 段：次级误解和边界

1. **误解：「改了就通知」** -> 不对，**值没变不通知**。`hasChanged(newValue, oldValue)`（`Object.is`）在 set trap 里就拦下了，`state.count = state.count` 不会进 trigger。这是很多"为什么赋值没触发更新"问题的反面--赋了相同的值本来就不该触发。
2. **误解：「trigger = 立即重新执行」** -> 只对"裸 effect"成立。组件渲染和 watch 的 effect 都带 scheduler，trigger 只是**入队**，真正执行推迟到微任务批量阶段（主题块 3）。所以同步代码里连续改三次 `state.count`，渲染只发生一次。
3. **误解：「`arr.push(x)` 只通知新增下标」** -> 漏了 `length`。push 在 baseHandlers 里被特殊代理（1.3 讲过数组方法的拦截），最终对 `length` 做一次 SET/对下标做 ADD，v-for 依赖的 `length` 必须被通知，否则列表不更新。
4. **边界**：
   - 数组 `length` **赋值增大**（`arr.length = 10`）：不通知任何下标（没有元素被删），只通知 `length` 自己
   - `for...in` 一个对象只 track `ITERATE_KEY`；`Object.keys` 走 `ownKeys` trap 同样收敛到 `ITERATE_KEY`；Map 的 `keys()` 额外 track `MAP_KEY_ITERATE_KEY`
   - dev 模式下 `triggerEffect` 有 `onTrigger` 调试钩子（devtools）
   - computed 的 trigger 有"dirty 才重新标脏"的分支细节，故意留到主题块 3.2

---

**子节交接**：

> 现在我们知道了：trigger 按操作类型算出 dep 列表（key 自身 + ITERATE_KEY/length/全部），聚合去重后拷贝遍历，按 `scheduler ? scheduler() : run()` 分流执行。至此"收集 -> 存储 -> 触发"主链路闭环。但 2.1 埋的 `parent` 字段还没兑现：trigger 执行 `run()` 时 `activeEffect` 会被重新赋值，如果 effect A 运行中又启动了 effect B（**嵌套 effect**），track 收到的必须是 B 而不是 A。`activeEffect` 的栈式管理怎么实现？这是 2.5 要处理的事。


## 考核过程

（主题块 2 全部讲完后统一考核，此处待填充）
