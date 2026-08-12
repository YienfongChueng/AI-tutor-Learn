# 主题块 1：响应式数据

> Vue3 响应式源码 · [vue@3.4](mailto:vue@3.4) · 深度讲解模式（conceptual）

## 讲解原文

### 子节 1.1：为什么用 Proxy（vs Vue2 defineProperty）

**core_question：Vue2 defineProperty 的根本缺陷是什么，Proxy 如何解决**

---

#### 第 0 段：直觉锚定

想象给一栋办公楼装「门禁监控」。

- **Vue2 的做法**：挨个房间门口装传感器（每个属性一个 `defineProperty`）。新盖的房间（后新增的属性）没装传感器，监控不到，得手动补装（`$set`）。而且装修时必须一次性把整栋楼所有房间都装完（递归遍历），哪怕很多房间根本没人用。
- **Vue3 的做法**：在办公楼唯一的出入口设一个总门禁（`Proxy` 代理整个对象）。无论谁进出哪个房间、哪怕是新盖的房间、甚至是查目录（`Object.keys`），都要先过门禁登记。

「总门禁」= Proxy（整体代理）；「逐房间传感器」= defineProperty（逐属性劫持）。

---

#### 第 1 段：问题背景（defineProperty 的根本缺陷）

Vue2 用 `Object.defineProperty(obj, key, { get, set })` 对**每个属性**定义访问器。这带来四个硬伤：

1. **检测不到属性新增/删除**：defineProperty 只对初始化时遍历到的 key 生效。后加的 `obj.newKey = 1` 没有劫持，视图不更新 → Vue2 被迫提供 `Vue.set / $set`、`$delete` 让用户手动补救。
2. **数组监听受限**：直接 `arr[0] = x` 或 `arr.length = 0` 监听不到。Vue2 只能重写 `push/pop/splice/...` 等 7 个变更方法，改索引仍要 `$set`。
3. **初始化全量递归**：`observe()` 一开始就递归遍历整个对象所有层级，深层对象一次性全劫持 → 大对象初始化卡顿。
4. **无法代理原始值**：defineProperty 作用于「对象的属性」，原始值（number/string）没有属性可挂 → Vue2 的 `data` 必须是对象。

> ⚠️ **常见先入为主的误解**：很多人以为 Vue3 换 Proxy「主要为了性能更好」。实际上**根本动机是能力更强**——Proxy 能拦截 defineProperty 做不到的操作（新增/删除属性、数组索引赋值、`in`/`has`、`Object.keys` 遍历等共 13 种）。性能提升是「整体代理 + 惰性递归」带来的附带结果，不是换 Proxy 的主因。带着这个理解往下看。

---



#### 第 2 段：核心数据结构

Proxy 的结构很简单：

```ts
const proxy = new Proxy(target, handler)
//            ↑原始对象   ↑trap 集合
```

`handler` 是一组「拦截器函数（trap）」，Proxy 能拦截 **13 种**操作，本节先关注最常用的：


| trap                                | 触发时机                 | 作用                            |
| ----------------------------------- | -------------------- | ----------------------------- |
| `get(target, key, receiver)`        | 读 `proxy[key]`       | 拦截读取，可在此 `track` 收集依赖         |
| `set(target, key, value, receiver)` | 写 `proxy[key] = v`   | 拦截赋值，可在此 `trigger` 触发更新       |
| `has`                               | `key in proxy`       | 拦截 in 操作                      |
| `deleteProperty`                    | `delete proxy[key]`  | 拦截删除                          |
| `ownKeys`                           | `Object.keys(proxy)` | 拦截遍历                          |
| ...                                 | （共 13 种）             | defineProperty 只有 get/set 2 种 |


对比 defineProperty：只能给**已存在的属性**装 get/set，且必须**逐个**装。

---



#### 第 3 段：运行流程（两条路径对比）

**Vue2 defineProperty 路径**（逐属性、初始化全量）：

```
observe(obj)
  └─ 遍历 obj 的每个 key
       └─ defineReactive(obj, key)
            └─ Object.defineProperty(obj, key, { get, set })
                 （若 value 是对象，递归 observe —— 初始化时全量递归）
```

**Vue3 Proxy 路径**（整体代理、惰性）：

```
reactive(obj)
  └─ new Proxy(obj, baseHandlers)   ← 一次性包一层，不递归
       └─ 返回 proxy
（访问 proxy.child 时，get trap 里才把 child 也 reactive —— 惰性递归，见 1.5）
```

关键差异：Vue2 在**初始化**就递归完；Vue3 在**访问时**才递归（1.5 详讲）。新增属性？Vue2 要 `$set`，Vue3 的 set trap 天然拦得到。

---



#### 第 4 段：设计动机与权衡

**Proxy 优化的核心约束**：响应式系统的「能力完整性 + 初始化性能」。

- **能力**：13 种 trap 覆盖增删改查遍历，无需 `$set`，数组索引天然支持。
- **初始化性能**：整体代理 + 惰性递归 → 大对象初始化只包一层，子对象用到才代理。
- **原生支持**：Proxy 是引擎内置，V8 对其有优化。

**代价 / 权衡**：

- **不能代理原始值**：Proxy 的 target 必须是对象。这是 `ref` 存在的根本原因（1.4 详讲）。
- **兼容性**：IE 不支持 Proxy（Vue3 已放弃 IE）。
- **单次访问开销**：每次属性读写都过 trap，有微小开销；但整体仍优于 Vue2 的全量初始化递归。

**与系统其他机制的关系**：Proxy 只是「数据层」的代理外壳；真正的依赖收集（`track`）和触发（`trigger`）挂在 get/set trap 上，这是主题块 2 的核心。

---



#### 第 5 段：次级误解和边界

1. **误解：「Proxy 能监听新增属性，所以 Vue3 完全不需要 $set 了」**
  - 对**响应式对象**确实不需要 `$set`（已被移除）。但 `ref` 的 `.value`、解构失去响应式等场景仍有坑（后续讲）。「响应式断链」以新形式存在，不是完全没有。
2. **误解：「Proxy 性能一定比 defineProperty 好」**
  - 不绝对：单次属性访问 Proxy 有 trap 拦截开销。但初始化的惰性递归 + 无需 `$set` 的能力，使**整体**表现更优，尤其大对象。
3. **边界**：
  - 原始值不能 Proxy → `ref` 解决（1.4）。
  - `Map/Set/WeakMap` 等集合类型，普通 get/set trap 拦不住其方法调用（`map.get` 是方法）→ 需要 `collectionHandlers`（1.6）。
  - 对同一对象多次 `reactive()` 返回同一个代理（有缓存，1.2 详讲）。

---

**子节交接**：

> 现在我们知道了：Vue3 用 Proxy 整体代理对象，解决了 Vue2 defineProperty 检测不到新增/删除、数组受限、初始化全量递归的缺陷。但这只回答了「为什么用 Proxy」——`reactive()` **具体是怎么把一个对象变成 Proxy 的？handler 里到底写了什么？** 这是 1.2 要处理的事。



### 子节 1.2：reactive()：Proxy 创建 + handler 总览

**core_question：reactive 如何用 Proxy 包裹对象（createReactiveObject 流程）**

---



#### 第 0 段：直觉锚定

把 `reactive()` 想成一个「响应式化工厂」的取件窗口：

- 你递进一个普通对象，它给你**包一层 Proxy 外壳**返回。
- 窗口有本**登记簿**（reactiveMap）：同一个对象第二次来，直接把上次包好的代理还给你，不重复包装。
- 工厂内部分两条**流水线**：普通对象/数组走 baseHandlers，Map/Set 走 collectionHandlers。

「登记簿」= reactiveMap（WeakMap 缓存）；「分流水线」= 按 targetType 分发 handler。

---



#### 第 1 段：问题背景

`reactive()` 要解决：

- 把普通对象变成可追踪的 Proxy（外壳）。
- **不重复代理**：同一对象 reactive 两次应返回同一个 proxy（否则 track/trigger 关系混乱、内存浪费）。
- 识别"不该代理"的目标：原始值、已 frozen、被标记 skip -> 直接返回。
- 对不同对象类型用不同 handler（普通对象 vs 集合类型）。
- **不在此递归**（惰性，留给 get trap 在访问时做，见 1.5）。

> ⚠️ **常见误解**：很多人以为 `reactive(obj)` 会「深拷贝」对象或「递归把所有子对象都代理」。实际上它只**包一层外壳**，原对象还是同一个（共享引用），子对象的代理是访问时才发生的（1.5）。reactive 不拷贝数据。

---



#### 第 2 段：核心数据结构

```ts
// vue@3.4 · packages/reactivity/src/reactive.ts
const reactiveMap = new WeakMap<object, any>()  // 缓存：原始对象 -> 代理

// 特殊标记 key（通过 get trap 识别"是否已是代理"）
enum ReactiveFlags {
  IS_REACTIVE = '__v_isReactive',
  IS_READONLY = '__v_isReadonly',
  IS_SHALLOW  = '__v_isShallow',
  SKIP        = '__v_skip',
  RAW         = '__v_raw',
}

// 三类目标
enum TargetType { INVALID = 0, COMMON = 1, COLLECTION = 2 }
// INVALID: 原始值 / frozen / __v_skip / 不可扩展 -> 不代理
// COMMON:  普通对象/数组 -> baseHandlers (mutableHandlers)
// COLLECTION: Map/Set/WeakMap/WeakSet -> collectionHandlers
```

handler 分发：


| targetType | 对象              | handler                       |
| ---------- | --------------- | ----------------------------- |
| COMMON     | `{}`, `[]`      | baseHandlers（mutableHandlers） |
| COLLECTION | Map/Set/...     | collectionHandlers            |
| INVALID    | 原始值/frozen/skip | 不代理，直接返回                      |


---



#### 第 3 段：运行流程

源码定位：`vue@3.4 · packages/reactivity/src/reactive.ts · createReactiveObject(target, isReadonly, baseHandlers, collectionHandlers, proxyMap)`（具体行号因版本而异）。

伪代码（保留所有控制流分支）：

```ts
function createReactiveObject(target, isReadonly, baseHandlers, collectionHandlers, proxyMap) {
  // 1. 非对象（原始值）直接返回 + 警告
  if (!isObject(target)) {
    if (__DEV__) warn(`value cannot be made reactive: ${target}`)
    return target
  }
  // 2. 已是代理且不是「把 reactive 再转 readonly」-> 返回本身
  //    （靠 get trap 拦 ReactiveFlags.IS_REACTIVE 判断）
  if (target[ReactiveFlags.IS_REACTIVE] && !(isReadonly && !target[ReactiveFlags.IS_READONLY])) {
    return target
  }
  // 3. 命中缓存 -> 返回已有代理
  const existingProxy = proxyMap.get(target)
  if (existingProxy) return existingProxy
  // 4. 判断目标类型；INVALID 直接返回不代理
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) return target
  // 5. 按类型选 handler，创建 Proxy，缓存，返回
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  proxyMap.set(target, proxy)
  return proxy
}
```

流程图：

```
reactive(obj)
   │
   ├─ 非对象? ────────────yes──> 返回原值(warn)
   │ no
   ├─ 已是代理(非转readonly)? ─yes──> 返回本身
   │ no
   ├─ 缓存命中? ──────────yes──> 返回已有 proxy
   │ no
   ├─ targetType == INVALID? ─yes──> 返回原对象
   │ no
   └─ new Proxy(target, COMMON?baseHandlers:collectionHandlers)
        └─ 存 reactiveMap，返回 proxy
```

---



#### 第 4 段：设计动机与权衡

- **WeakMap 做缓存**：key 是原始对象（弱引用）。好处①同对象不重复代理，保证 `reactive(o) === reactive(o)`；好处②原始对象被回收时缓存项自动消失，**不阻 GC**。用普通 Map 会内存泄漏。
- **ReactiveFlags 标记**：不往对象加真实字段，而是用 `__v_isReactive` 等"特殊 key"，靠 get trap 拦截返回 true/false 来识别代理状态。原始对象保持干净。
- **分发 handler**：普通对象 get/set 拦属性即可；但 Map/Set 的 `map.get(k)` 是**方法调用**，普通 get trap 拦不住方法内部 -> 必须用 collectionHandlers 重写方法（1.6）。
- **不递归**：createReactiveObject 只包一层，性能 + 惰性（1.5）。
- **getTargetType 排除 INVALID**：frozen、`__v_skip`、不可扩展对象 -> 不代理（代理了也改不动或没意义）。

---



#### 第 5 段：次级误解和边界

1. **误解：「reactive 会深拷贝对象」** -> 错。只包一层 Proxy 外壳，原对象与代理**共享同一份数据**，改代理就是改原对象。
2. **误解：「同一对象 reactive 两次得到两个代理」** -> 错。reactiveMap 缓存，返回同一个。
3. **误解：「reactive(1) 会得到响应式数字」** -> 错。原始值非对象，直接返回原值并 warn。原始值响应式必须用 `ref`（1.4）。
4. **边界**：
  - `reactive(reactive(o))` 短路返回本身（步骤2），不会包两层。
  - `readonly(reactive(o))` 是允许的（特殊分支），得到「只读的响应式」。
  - `reactive(Object.freeze({}))` -> INVALID，返回原对象不代理。

---

**子节交接**：

> 现在我们知道了 reactive() 的创建流程：非对象拦截、代理短路、WeakMap 缓存、按 targetType 分发 handler、只包一层。但 handler 内部到底怎么写--**get 拦截怎么触发依赖收集、set 怎么触发更新？前置检测里讲的 receiver 在这里怎么用？** 这是 1.3 要处理的事。



### 子节 1.3：baseHandlers：get 触发 track、set 触发 trigger

**core_question：get/set 拦截器如何与依赖系统衔接**

---



#### 第 0 段：直觉锚定

回到 1.2 的「门禁」比喻。baseHandlers 就是门禁的两个核心动作：

- **get 是「读卡登记」**：谁（哪个 effect）读了哪个房间（哪个 key），记一笔 -> 这就是 `track`（收集依赖）。
- **set 是「变更广播」**：谁改了哪个房间，把当初登记过这个房间的所有人通知一遍 -> 这就是 `trigger`（触发更新）。

「读卡登记」= get 里调 `track`；「变更广播」= set 里调 `trigger`。track/trigger 的内部实现在主题块 2，本节只看 handler 怎么调它们。

---



#### 第 1 段：问题背景

handler 要衔接两件事：

- 读属性时，把「当前正在运行的 effect」与「这个 key」绑定（track）。
- 写属性时，找到绑定的 effects 并执行（trigger）。

还要处理：识别 ReactiveFlags 特殊 key、数组的特殊方法、原型链 receiver、ref 自动解包、避免原型链重复触发。

> ⚠️ **常见误解**：很多人以为 get 里"每次读取都新建一个依赖关系"。实际上 track 收集进的是 Set，**幂等**--同一个 effect 读同一个 key 多次，只记一笔，不是每次都新建。

---



#### 第 2 段：核心数据结构

`mutableHandlers = { get, set, deleteProperty, has, ownKeys }`，核心是 get/set：

```ts
// vue@3.4 · packages/reactivity/src/baseHandlers.ts
export const mutableHandlers = {
  get: createGetter(false, false),
  set: createSetter(false),
  deleteProperty,  // delete 时 trigger(DELETE)
  has,             // in 操作时 track(HAS)
  ownKeys,         // Object.keys 时 track(ITERATE)
}
```

get/set 都通过工厂 `createGetter/createSetter` 生成（参数 isReadonly/shallow 控制变体，1.6）。操作类型枚举：

```ts
enum TrackOpTypes   { GET, HAS, ITERATE }
enum TriggerOpTypes { SET, ADD, DELETE, CLEAR }
```

---



#### 第 3 段：运行流程

源码定位：`vue@3.4 · packages/reactivity/src/baseHandlers.ts · createGetter / createSetter`。

**createGetter 伪代码（保留控制流）**：

```ts
function createGetter(isReadonly, shallow) {
  return function get(target, key, receiver) {
    // 1. ReactiveFlags 特殊 key：直接返回标记，不走 track
    if (key === ReactiveFlags.IS_REACTIVE) return !isReadonly
    else if (key === ReactiveFlags.IS_READONLY) return isReadonly
    else if (key === ReactiveFlags.RAW) return target

    // 2. 数组特殊方法：用 instrumentations 重写版（避免 length track 死循环）
    const targetIsArray = isArray(target)
    if (!isReadonly && targetIsArray && hasOwn(arrayInstrumentations, key)) {
      return Reflect.get(arrayInstrumentations, key, receiver)
    }

    // 3. 取值 + 收集依赖（只读不收集，因为它不会变）
    const res = Reflect.get(target, key, receiver)   // ← receiver 原样登场！
    if (!isReadonly) track(target, TrackOpTypes.GET, key)

    // 4. 浅模式直接返回
    if (shallow) return res

    // 5. ref 自动解包：对象属性是 ref 时，访问自动取 .value
    if (isRef(res)) return targetIsArray && isIntegerKey(key) ? res : res.value

    // 6. 惰性深层：res 是对象则递归 reactive（1.5 详讲）
    if (isObject(res)) return isReadonly ? readonly(res) : reactive(res)

    return res
  }
}
```

**createSetter 伪代码**：

```ts
function createSetter(shallow) {
  return function set(target, key, value, receiver) {
    let oldValue = target[key]
    if (!shallow) oldValue = toRaw(oldValue)   // 旧值若被代理过，取 raw 比较

    const hadKey = isArray(target) && isIntegerKey(key)
      ? Number(key) < target.length
      : hasOwn(target, key)

    const result = Reflect.set(target, key, value, receiver)   // ← receiver 同样用

    // 关键：只在 receiver 的 raw === target 时才 trigger
    // 避免原型链 set 时，原型和实例各触发一次（重复）
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)            // 新增属性
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)  // 修改属性
      }
    }
    return result
  }
}
```

> **回调前置检测**：createGetter 第 3 步的 `Reflect.get(target, key, receiver)` 就是你之前说"不懂"的那个 receiver--这里原样登场。它保证：若 key 是原型链上的 getter，getter 内部的 `this` 指向代理而非原始 target，响应式沿原型链不断。

get/set 流程图：

```
读 proxy.key:
  get(target,key,receiver)
   ├─ 特殊flag? -> 返回标记
   ├─ 数组方法? -> 返回重写版
   ├─ Reflect.get(target,key,receiver) 取值
   ├─ track(target, GET, key)      ← 收集依赖
   └─ 值是对象? -> reactive(值) 惰性深层

写 proxy.key = v:
  set(target,key,value,receiver)
   ├─ 取 oldValue（toRaw）
   ├─ Reflect.set(target,key,value,receiver)
   ├─ target === toRaw(receiver)? （防原型链重复触发）
   │    ├─ 新增 key -> trigger(ADD)
   │    └─ 已存在且值变 -> trigger(SET)
   └─ return result
```

---



#### 第 4 段：设计动机与权衡

- **lazy track（读时才收集）**：只在 get 时 track 实际被读取的 key，没读的不收集 -> 精准，避免无效更新。
- **ADD vs SET 区分**：新增属性（!hadKey）trigger ADD，修改属性 trigger SET。ADD 还要通知 ITERATE 类依赖（如 `Object.keys`/`v-for`），范围更大。
- **hasChanged 守卫**：`Object.is(value, oldValue)` -- 值没变就不 trigger，避免无意义更新。
- **receiver + Reflect**：保原型链响应式（前置检测已说）。
- `target === toRaw(receiver)` **防重复触发**：对象继承自另一个 reactive 对象时，set 会先走原型 setter（receiver 是子实例）再走实例本身。此判断确保只在"真正属主"上 trigger 一次。
- **数组 instrumentations**：`push/pop/forEach` 等会读 `length`。若不处理，读 length 会 track(length)，写元素又 trigger(length) -> 可能死循环。Vue 用 `pauseTracking` 包裹这些方法避开。
- **ref 自动解包**：reactive 对象里的 ref 属性，访问时自动取 `.value`（写时反向赋到 `.value`）--`reactive` + `ref` 混用的便利设计。

---



#### 第 5 段：次级误解和边界

1. **误解：「get 每次都重建依赖」** -> 错。track 写入 Set，幂等，同 effect+key 只记一笔。
2. **误解：「set 一定触发更新」** -> 错。值没变（hasChanged false）不 trigger；只读对象的 set 不赋值也不 trigger（开发模式抛 warn）。
3. **误解：「读任意 key 都会触发渲染」** -> 错。只有"在 effect（如渲染函数）运行期间读的 key"才被 track。effect 外的普通读取不收集。
4. **边界**：
  - 数组索引赋值 `arr[5]=x`：hadKey 用 `Number(key) < length` 判断，区分 ADD/SET。
  - 内建 Symbol key（如 `Symbol.iterator`）不 track（无意义）。
  - `delete proxy.key` -> deleteProperty -> trigger(DELETE)。

---

**子节交接**：

> 现在我们知道了：get 里 `Reflect.get + track` 收集依赖，set 里 `Reflect.set + trigger` 触发更新，receiver 保原型链响应式，ADD/SET/hasChanged 精准控制。但 track/trigger 的内部（依赖存哪、怎么找）是主题块 2 的事。在此之前，1.4 先解决遗留问题：**原始值不能被 Proxy，那** `ref()` **怎么让一个 number 变成响应式？**



### 子节 1.4：ref()：为什么原始值不能 Proxy + 实现

**core_question：ref 为何存在，原始值响应式如何实现**

---

#### 第 0 段：直觉锚定

1.1 说过 Proxy 是"办公楼总门禁"，但门禁只能装在"建筑物"（对象）上。原始值（number/string/boolean）像一颗"散装糖果"——没有外壳也没有门，门禁无处可装（`new Proxy(1, handler)` 直接抛 TypeError）。

要追踪这颗糖果，办法是：把它装进一个**糖果盒**（`RefImpl` 对象）。盒子上开一个窗口叫 `.value`：

- 你从窗口看糖果（读 `.value`）→ 盒子登记"谁来看过"（`track`）
- 你从窗口换掉糖果（写 `.value`）→ 盒子通知"所有登记过的人"（`trigger`）

「糖果盒」= `RefImpl` 实例（一个普通对象）；「窗口」= `.value` 访问器（getter/setter）。盒子本身是对象，所以它能被创建、能挂访问器，绕过了"原始值不能 Proxy"的限制。

---

#### 第 1 段：问题背景

1.1 讲过 Proxy 的四个硬伤之一：**无法代理原始值**。Proxy 的 target 在 JS 规范里必须是对象：

```ts
new Proxy(1, {})      // TypeError: Cannot create proxy with a non-object as target
new Proxy('hi', {})   // 同样报错
```

但响应式系统必须能追踪原始值（计数器 `count`、开关 `visible` 都是 number/boolean）。怎么让原始值也响应式？这就是 `ref` 存在的**根本原因**。

> ⚠️ **常见先入为主的误解：** 很多人以为"ref 就是 `reactive({ value: x })` 的语法糖"。实际上 ref 用了独立的 `RefImpl` 类，**不走 Proxy**，只用一个 `.value` 访问器实现 track/trigger，比 reactive 轻量得多。但 ref 包对象时，内部会复用 reactive（见第 3 段 `toReactive`）。带着这个理解往下看。

---

#### 第 2 段：核心数据结构

```ts
// vue@3.4 · packages/reactivity/src/ref.ts
class RefImpl<T> {
  private _value: T                    // 实际存储的值（对象值会被 toReactive 包成代理）
  private _rawValue: T                 // 原始值（非代理版，用于 hasChanged 比较）
  public readonly __v_isRef = true     // 标记：isRef() 靠它判断
  public readonly __v_isShallow: boolean

  constructor(value: T, public readonly __v_isShallow: boolean) {
    this._rawValue = __v_isShallow ? value : toRaw(value)
    this._value    = __v_isShallow ? value : toReactive(value)
  }

  get value() {                        // 访问器 getter
    track(this, TrackOpTypes.GET, 'value')
    return this._value
  }

  set value(newVal) {                  // 访问器 setter（见第 3 段）
    // ...
  }
}

function toReactive(value) {
  return isObject(value) ? reactive(value) : value   // 对象才 reactive，原始值原样返回
}
```

字段关系图：

```
RefImpl 实例
┌──────────────────────────────────────┐
│ _rawValue  ──> 原始值(用于比较)       │
│ _value     ──> 响应式值               │  ← 对象时 = reactive(原值)；原始值时 = 原值
│ __v_isRef  ──> true (只读标记)        │
│ __v_isShallow ──> 是否浅模式          │
├──────────────────────────────────────┤
│ get value() ──> track + 返回 _value  │
│ set value() ──> hasChanged + trigger │
└──────────────────────────────────────┘
```

关键：ref 的 track/trigger 用**固定 key = `'value'`**（不像 reactive 用动态属性 key）。一个 ref 就是一个独立的依赖单元，target 是 ref 实例本身。

---

#### 第 3 段：运行流程

源码定位：`vue@3.4 · packages/reactivity/src/ref.ts · ref() / createRef() / RefImpl`。

**创建流程**：

```ts
function ref(value) {
  return createRef(value, false)
}

function createRef(rawValue, shallow) {
  if (isRef(rawValue)) return rawValue      // 已是 ref -> 短路返回，不重复包
  return new RefImpl(rawValue, shallow)     // 否则 new 一个盒子
}
```

**set value 伪代码（保留控制流）**：

```ts
set value(newVal) {
  const useDirectValue = this.__v_isShallow || isShallow(newVal) || isReadonly(newVal)
  newVal = useDirectValue ? newVal : toRaw(newVal)     // 新值取 raw（去代理）用于比较

  if (hasChanged(newVal, this._rawValue)) {            // 值变了才 trigger（守卫）
    this._rawValue = newVal
    this._value = useDirectValue ? newVal : toReactive(newVal)  // 对象值重新 reactive
    trigger(this, TriggerOpTypes.SET, 'value', newVal)
  }
}
```

流程图：

```
ref(1)
  └─ createRef(1, false)
       └─ isRef(1)? no
            └─ new RefImpl(1, false)
                 ├─ _rawValue = toRaw(1) = 1
                 └─ _value = toReactive(1) = 1   (原始值 isObject=false，原样返回)

读 count.value:
  get value()
   ├─ track(this, GET, 'value')    ← 收集依赖（key 固定为 'value'）
   └─ return this._value

写 count.value = 2:
  set value(2)
   ├─ newVal = toRaw(2) = 2
   ├─ hasChanged(2, _rawValue=1)? yes
   │    ├─ _rawValue = 2
   │    ├─ _value = toReactive(2) = 2
   │    └─ trigger(this, SET, 'value', 2)   ← 触发更新
   └─ (值没变则不 trigger)
```

> **回调 1.3**：ref 的 track/trigger 就是 1.3 讲的 track/trigger，只是 key 固定为 `'value'`，target 是 ref 实例本身。依赖收集机制完全复用主题块 2 的 targetMap 结构。

---

#### 第 4 段：设计动机与权衡

- **根本约束**：Proxy 不能代理原始值（JS 规范）→ 必须用"对象包一层"绕过。这是 ref 存在的**唯一根本原因**。
- **为什么用 RefImpl 类而非 `reactive({value: x})`**：
  - **更轻量**：reactive 走 Proxy 的 13 种 trap，ref 只关心一个 `.value` key，class 访问器足够，无需 Proxy 开销。
  - **语义明确**：ref = 单值响应式，`__v_isRef` 标记便于 `isRef` 判断、模板自动解包（`{{ count }}` 不用写 `.value`）。
  - **对象值复用 reactive**：`toReactive` 让 ref 包对象时自动深层响应式，复用已有体系，不重复造轮子。
- **`_rawValue` 与 `_value` 分离**：
  - `_value` 存响应式版（对象是代理），用于返回给用户。
  - `_rawValue` 存原始版，用于 `hasChanged` 比较。否则比较代理对象会失效（代理 !== 原对象）。
- **hasChanged 守卫**：和 1.3 的 set 一样，值没变（`Object.is`）不 trigger，避免无意义更新。
- **代价**：使用上多一层 `.value`，容易忘写（模板里自动解包正是为了缓解这点）。

---

#### 第 5 段：次级误解和边界

1. **误解：「ref 是 `reactive({value:x})` 的语法糖」** → 错。ref 用独立 `RefImpl` 类，不走 Proxy，只一个访问器；但对象值内部会 `toReactive` 复用 reactive。
2. **误解：「ref 包对象时只有 .value 这一层响应式，深层不响应」** → 错。`_value = toReactive(value)`，对象值被 reactive 包一层，深层属性访问照样 track/trigger（1.5 详讲惰性递归）。
3. **误解：「.value 是普通字段」** → 错。`.value` 是 getter/setter 访问器，读触发 track、写触发 trigger。直接 `count._value = 2` 绕过 setter 不会触发更新（且 `_value` 是 private）。
4. **边界**：
   - `ref(ref(x))` → isRef 短路返回内层 ref，不包两层。
   - `shallowRef(obj)` → `__v_isShallow=true`，`_value` 直接存原对象不 toReactive，深层不响应式（1.6）。
   - 模板里 ref 自动解包：`<div>{{ count }}</div>` 等价 `count.value`（编译器处理，仅模板层；JS 里仍需 `.value`）。
   - `ref(reactive(o))` → `_value` 是已存在的 reactive 代理，toReactive 再调 reactive 会命中 reactiveMap 缓存返回同一个代理。

---

**子节交接**：

> 现在我们知道了：原始值不能 Proxy，所以 ref 用 `RefImpl` 对象包一层，靠 `.value` 访问器实现 track/trigger，对象值则通过 `toReactive` 复用 reactive。但这里埋了一个问题：`toReactive` 把对象包成 reactive 后，**子对象是何时被代理的？是 `reactive()` 调用时就全量递归，还是访问时才递归？** 这关系到 1.2 留下的"惰性递归"承诺。这是 1.5 要处理的事。


### 子节 1.5：深层响应式：惰性递归代理

**core_question：子对象何时被代理，惰性代理的性能好处**

---

#### 第 0 段：直觉锚定

回到 1.2 的「响应式化工厂」。工厂给主楼包门禁时，**只包了大楼入口一层**。但楼里有楼层，楼层里有房间，房间里有抽屉。

- **Vue2 的做法**：建楼时一次性把所有楼层、房间、抽屉**全装上门禁**（递归遍历每一层）。楼再大也得一口气装完。
- **Vue3 的做法**：只装大楼入口门禁。当你**推开某层楼的门**（访问子对象）时，才给那层楼装门禁；再推开某房间门时，才给房间装。**用到才装，没推开的永远不装。**

「主楼入口门禁」= `reactive(obj)` 只包一层外壳；「推门时才装」= get trap 里对返回的子对象调 `reactive(res)`。

> **回调 1.3**：1.3 讲 `createGetter` 时，第 6 步那行 `if (isObject(res)) return reactive(res)` 就是"推门时装门禁"的动作--它正是惰性递归的入口。

---

#### 第 1 段：问题背景

1.2 讲过 `createReactiveObject` **只包一层、不递归**。1.4 讲 ref 包对象时 `toReactive` 也只包一层。但用户的对象往往是嵌套的：

```ts
const state = reactive({ user: { profile: { age: 20 } }, list: [...] })
state.user.profile.age = 21   // 这行能不能触发更新？
```

要能触发，`state.user`、`state.user.profile` 都必须是响应式代理。但 `reactive(state)` 只包了最外层。子对象何时变响应式？

> ⚠️ **常见先入为主的误解：** 很多人以为"`reactive(obj)` 会递归把所有子对象都代理"。实际上它**只包一层外壳**，子对象是**访问时**才被代理的（get trap 里 `reactive(res)`）。这也解释了 Vue2 与 Vue3 在初始化性能上的根本差异。

---

#### 第 2 段：核心数据结构

惰性递归复用两个已有结构，没有新增：

- **`reactiveMap`（WeakMap 缓存）**：1.2 讲过，`原始对象 -> 代理`。保证同一子对象**不重复代理**。
- **`__v_raw` 标记**：代理的 get trap 拦截 `ReactiveFlags.RAW` 返回 `target`（取原始对象，用于去代理比较）。

代理本身**不存储"子代理"引用**--子代理是访问时即时创建（或命中缓存）的。数据流实例：

```
原始对象 o = { a: { b: { c: 1 } } }

reactive(o) ──────────────────────> proxy_o   （只包一层！a/b/c 都还没动）

访问 proxy_o.a:
  get trap ─> res = o.a = {b:{c:1}}
           ─> isObject(res) ─> reactive(res) ─> proxy_a  （此刻才包，缓存进 reactiveMap）

访问 proxy_o.a.b:
  proxy_a 的 get trap ─> res = o.a.b = {c:1}
                      ─> reactive(res) ─> proxy_b

访问 proxy_o.a.b.c:
  proxy_b 的 get trap ─> res = 1
                      ─> isObject(1)? no ─> 直接返回 1   （原始值，不再递归）
```

关键：每一层代理都是**访问到才创建**，且通过 `reactiveMap` 保证 `proxy_o.a === proxy_o.a`（同一引用）。

---

#### 第 3 段：运行流程

源码定位：`vue@3.4 · packages/reactivity/src/baseHandlers.ts · createGetter` 第 6 步（1.3 已展开，这里聚焦惰性递归那一行）。

```ts
// createGetter 内（保留与惰性递归相关的控制流）
function get(target, key, receiver) {
  // ... 特殊 flag、数组方法（略）
  const res = Reflect.get(target, key, receiver)
  if (!isReadonly) track(target, TrackOpTypes.GET, key)
  if (shallow) return res                          // shallow 模式不递归（1.6）
  if (isRef(res)) return /* ... */ res.value        // ref 解包（1.3）
  if (isObject(res)) return isReadonly ? readonly(res) : reactive(res)  // ← 惰性递归入口
  return res                                        // 原始值直接返回
}
```

惰性递归流程图：

```
读 proxy.key
  ├─ Reflect.get(target, key, receiver) 取 res
  ├─ track(target, GET, key)              ← 收集依赖
  ├─ res 是 ref? ──yes──> 解包返回 res.value
  ├─ res 是对象? 
  │    yes ─> reactive(res)               ← 惰性：此刻才递归包一层
  │            ├─ reactiveMap 命中? ─yes─> 返回已有子代理（不重复包）
  │            └─ 未命中? ────────────> new Proxy(res, baseHandlers)，缓存，返回
  │    no  ─> 直接返回 res（原始值）
```

**为什么不会重复代理、且数据一致**：

- 每次访问 `proxy.a` 都会走 get trap -> `reactive(o.a)` -> `reactiveMap` 命中 -> 返回**同一个** `proxy_a`。
- 所以 `proxy.a === proxy.a`（同引用），对 `proxy.a` 的修改走 `proxy_a` 的 set trap -> `trigger`，依赖正常触发。
- 子代理与原始子对象**共享数据**（1.2 讲过 reactive 不拷贝），改 `proxy.a.x` 就是改 `o.a.x`。

---

#### 第 4 段：设计动机与权衡

- **惰性递归 vs Vue2 全量递归**：

| 维度 | Vue2 `observe()` | Vue3 `reactive()` |
| ---- | ----------------- | ----------------- |
| 时机 | 初始化时全量递归 | 访问时按需递归 |
| 初始化开销 | O(n)（n = 所有层级属性总数） | O(1)（只包一层） |
| 首次访问子对象 | 无额外开销 | 一次 `reactive()`（查缓存 + 可能 new Proxy） |
| 未使用的深层 | 仍被代理（浪费） | 永不代理（省 CPU/内存） |

- **性能好处**：
  - 大对象但只用到部分字段 -> 未用部分**永不代理**，省 CPU 和内存。
  - 初始化快：首屏渲染前 `reactive` 一个大数据结构不卡顿。
  - 总开销**摊到访问时**，用得多才付得多，公平。
- **`reactiveMap` 缓存保证不重复**：同一子对象多次访问只代理一次，惰性不会退化成"每次访问都 new Proxy"。
- **代价**：
  - 首次访问子对象有一次 `reactive()` 调用开销（查缓存 + 可能 `new Proxy`）。
  - 深层嵌套访问链路每次都过 get trap（微开销，但 V8 对 Proxy 有优化）。
- **与 ref 的衔接**：1.4 讲的 `toReactive` 走同一个 `reactive()`，所以 `ref` 包对象时，深层访问**同样惰性递归**--ref 与 reactive 在深层响应式上行为一致。

---

#### 第 5 段：次级误解和边界

1. **误解：「`reactive(obj)` 会一次性把所有子对象代理」** -> 错。只包一层外壳，子对象**访问时**才代理。
2. **误解：「每次访问子对象都 new 一个新 Proxy」** -> 错。`reactiveMap` 缓存，同一子对象返回同一个代理（`proxy.a === proxy.a`）。
3. **误解：「深层属性修改不会触发更新」** -> 错。访问时子对象已被代理成 `proxy_a`，修改 `proxy_a.x` 走它的 set trap，照样 `trigger`。
4. **边界·解构失去响应式的真正根因**（这是 Vue3 最经典的坑，用惰性递归能彻底解释）：

   ```ts
   const state = reactive({ user: { age: 20 }, count: 0 })

   // 情况 A：解构对象属性
   const { user } = state      // 触发 get trap -> user = reactive(o.user) = 子代理 ✅ 仍响应式
   user.age = 21               // 走子代理 set trap，触发更新 ✅

   // 情况 B：解构原始值属性
   const { count } = state     // 触发 get trap -> res = 0 -> isObject(0)? no -> 直接返回 0
   count = 1                   // count 就是个普通数字，没有代理，不触发更新 ❌ 失去响应式
   ```

   **根因**：原始值没有"对象外壳"可被代理（1.1/1.4），get trap 直接返回原值；解构出来的是值拷贝，与响应式系统脱钩。对象属性解构后仍是子代理，所以不失响应式。这就是为什么 Vue3 要用 `toRefs` 把原始值属性转成 `ref` 再解构。

5. **其他边界**：
   - `shallowReactive`：只包一层，子对象访问**不递归**代理（1.6 详讲）。
   - 新增对象属性 `proxy.newKey = {x:1}` -> set trap -> `trigger(ADD)`；之后访问 `proxy.newKey` -> get trap -> `reactive({x:1})` 惰性代理。新增的对象属性也会在访问时变响应式。
   - `toRaw(proxy)` 沿 `__v_raw` 一层层取回最原始对象（穿透所有代理层）。

---

**子节交接**：

> 现在我们知道了：reactive 默认**深层响应式**，但靠"访问时才递归代理"实现惰性--初始化只包一层，子对象用到才代理，`reactiveMap` 保证不重复。这也解释了"解构原始值失响应式、对象不失"的根因。但有时我们**不需要深层**：只想关心第一层变化（性能），或只想只读不可改（安全）。这就需要 `readonly` / `shallow` 变体--它们的 handler 和默认 `mutableHandlers` 有什么差异？这是 1.6 要处理的事。


### 子节 1.6：readonly / shallow 变体

**core_question：不同响应式变体的 handler 差异**

---

#### 第 0 段：直觉锚定

回到 1.2 的响应式化工厂。工厂其实有四条流水线，对应四种"门禁权限套餐"：

- **普通门禁（reactive）**：能进能出能改，整栋楼逐层装到底（深层 + 可改）。
- **只读门禁（readonly）**：只能看不能改，也是逐层装到底（深层 + 只读）。
- **浅门禁（shallowReactive）**：只给大门这层装门禁，里面的楼层房间是"毛坯"（原始对象，没装门禁）。
- **浅只读门禁（shallowReadonly）**：大门这层只能看，里面也是毛坯。

「门禁权限套餐」= 四种 handler，由 `isReadonly` × `isShallow` 两个布尔参数组合生成（1.3 讲过 `createGetter(isReadonly, shallow)` / `createSetter(shallow)` 的工厂签名，这里看参数怎么组合出四变体）。

---

#### 第 1 段：问题背景

默认 `reactive` 是"深层 + 可改"。但实际场景需要另三种：

- **只读不可改**：父组件传给子组件的 `props` 不应被子组件修改 -> `readonly`。
- **只关心第一层**：大对象深层不需要响应式，省代理开销 -> `shallowReactive`。
- **第一层只读**：组件 `props` 实际就是 `shallowReadonly`（props 通常整体替换，不深改）。

> ⚠️ **常见先入为主的误解：** 很多人以为 `readonly` 是"深拷贝一份只读副本"。实际上 `readonly` 只包一层**只读外壳**，与原对象**共享数据**（和 reactive 一样不拷贝）。改原对象，`readonly` 视图也会变（如果是 `readonly(reactive(o))`）。

---

#### 第 2 段：核心数据结构

四种 handler（vue@3.4 · `packages/reactivity/src/baseHandlers.ts`），都由 `createGetter/createSetter` 的两个参数生成：

```ts
// 1. reactive：深层 + 可改
export const mutableHandlers = {
  get: createGetter(false /*isReadonly*/, false /*isShallow*/),
  set: createSetter(false),
  deleteProperty, has, ownKeys,
}

// 2. readonly：深层 + 只读
export const readonlyHandlers = {
  get: createGetter(true, false),
  set(target, key) {
    if (__DEV__) warn(`Set key "${key}" failed: target is readonly.`)
    return true                 // 不赋值，返回 true 不抛错
  },
  deleteProperty(target, key) { if (__DEV__) warn(...); return true },
  has, ownKeys,
}

// 3. shallowReactive：浅层 + 可改
export const shallowReactiveHandlers = {
  get: createGetter(false, true /*isShallow*/),
  set: createSetter(true),
  deleteProperty, has, ownKeys,
}

// 4. shallowReadonly：浅层 + 只读
export const shallowReadonlyHandlers = {
  get: createGetter(true, true),
  set(target, key) { /* warn */ return true },
  deleteProperty(target, key) { /* warn */ return true },
  has, ownKeys,
}
```

四个独立缓存（`packages/reactivity/src/reactive.ts`），互不影响：

```ts
const reactiveMap        = new WeakMap()   // reactive
const shallowReactiveMap = new WeakMap()   // shallowReactive
const readonlyMap        = new WeakMap()   // readonly
const shallowReadonlyMap = new WeakMap()   // shallowReadonly
```

四变体矩阵：

```
                 isShallow=false          isShallow=true
isReadonly=false  reactive                 shallowReactive
                  (深层可改)                (浅层可改)
isReadonly=true   readonly                 shallowReadonly
                  (深层只读)                (浅层只读)
```

---

#### 第 3 段：运行流程

差异全部落在 `createGetter` 的三个分支（按 `isReadonly`/`isShallow` 走不同路径）：

```ts
function get(target, key, receiver) {
  if (key === ReactiveFlags.IS_REACTIVE) return !isReadonly
  if (key === ReactiveFlags.IS_READONLY) return isReadonly
  // ...
  const res = Reflect.get(target, key, receiver)
  if (!isReadonly) track(target, GET, key)   // ① readonly 不 track
  if (shallow) return res                     // ② shallow 直接返回，不递归、不解包
  if (isRef(res)) return /* ... */ res.value
  if (isObject(res)) return isReadonly ? readonly(res) : reactive(res)  // ③ 深层递归：同种变体向下传
  return res
}
```

三个差异点：

- **① `readonly` 不 track**：只读对象不会被改 -> 无需收集依赖来触发更新。但 `readonly(reactive(o))` 的依赖收集由**底层 reactive** 负责（见下方）。
- **② `shallow` 直接返回 `res`**：不递归 `reactive`、不解包 `ref`。所以 `shallowReactive` 只有第一层响应式，深层是原始对象。
- **③ 深层递归保持同种变体**：`readonly` 的子对象包 `readonly`，`reactive` 的子对象包 `reactive`（变体向下传递）。

set 差异：

- `readonly` 的 set：`warn + return true`（不赋值、不 `trigger`）。
- `shallow` 的 set：`createSetter(true)`，不 `toRaw` 比较（直接用新值）。

四变体行为对比表：

```
变体              track?  惰性递归?      ref解包?  set 行为
─────────────────────────────────────────────────────────────
reactive          ✅      ✅ (reactive)  ✅        Reflect.set + trigger
readonly          ❌      ✅ (readonly)  ✅        warn 不赋值
shallowReactive   ✅      ❌             ❌        Reflect.set + trigger（仅第一层）
shallowReadonly   ❌      ❌             ❌        warn 不赋值
```

**关键机制：为什么 `readonly` 不 track，但 `readonly(reactive(o))` 视图能更新？**

```
readonly(reactive(o))   访问 proxy_ro.key
  └─ readonly 的 get: isReadonly=true -> 不 track
       └─ Reflect.get(reactiveProxy, key)
            └─ reactive 的 get: isReadonly=false -> track ✅  依赖收集在这层！
```

依赖收集发生在**底层 reactive**，`readonly` 只是套了个只读外壳。而纯 `readonly(plainObj)` 无底层 reactive -> 不 track -> 永不更新（合理：它本就不可变）。

---

#### 第 4 段：设计动机与权衡

- **`readonly` 的意义**：保护数据不被改（props、配置常量）。自身不 track 是因为"只读不会变，无需收集"；配合 `reactive` 可作"响应式数据的只读视图"。
- **`shallow` 的意义**：性能优化。大对象深层不需要响应式时，`shallowReactive` 只代理第一层，深层是原始对象（访问快、不占代理内存）。组件 `props` 用 `shallowReadonly`（props 通常整体替换，不深改）。
- **工厂函数复用**：四种 handler 都由 `createGetter/createSetter` 的两个参数生成 -> **参数化差异**，避免四份重复代码。这是把"可变的维度"（readonly? shallow?）抽成参数的典型设计。
- **独立缓存 Map**：`reactiveMap` / `readonlyMap` 等分开，保证 `reactive(o)` 和 `readonly(o)` 是**不同代理**互不影响；同一变体重复调用仍命中各自缓存。
- **代价**：`readonly` 不能改（开发模式 warn，生产模式静默 `return true`）；`shallow` 深层不响应式（需手动深层 `reactive` 或用 `ref`）。

---

#### 第 5 段：次级误解和边界

1. **误解：「`readonly` 是深拷贝只读副本」** -> 错。只包一层只读外壳，与原对象共享数据。
2. **误解：「`readonly` 完全不收集依赖」** -> 部分对。`readonly` 自身不 track，但 `readonly(reactive(o))` 的依赖收集由底层 reactive 负责，视图能更新。
3. **误解：「`shallowReactive` 的深层修改能触发更新」** -> 错。深层是原始对象（`shallow` 直接返回 `res` 不递归），改 `proxy.deep.x` 走的是原始对象的 set，**不 trigger**。
4. **误解：「`reactive(o)` 和 `readonly(o)` 是同一个代理」** -> 错。两个独立 WeakMap 缓存，是不同的代理对象。
5. **边界**：
   - **组件 `props` 实际是 `shallowReadonly`**：第一层只读（开发模式改 props 会 warn），深层仍是原始对象引用。
   - **`markRaw(obj)`**：给对象打 `__v_skip` 标记，`reactive` 时 `getTargetType` 返回 `INVALID`，**永不代理**（逃生舱，用于某些不想被响应式的对象，如第三方实例）。
   - **`readonly(reactive(o))` vs `reactive(readonly(o))`**：前者是"响应式数据的只读视图"（可感知底层变化，常用）；后者无意义（`readonly(o)` 已是代理，再 `reactive` 会短路返回本身）。
   - **四种变体都共享同一套 track/trigger 机制**（主题块 2 详讲），差异只在 handler 的拦截行为。

---

**子节交接（主题块 1 收尾）**：

> 现在我们知道了四种响应式变体（`reactive` / `readonly` / `shallowReactive` / `shallowReadonly`）的 handler 差异，由 `isReadonly` × `isShallow` 两个参数组合生成。回顾整个主题块 1：1.1 讲为什么用 Proxy，1.2 讲 `reactive()` 创建流程，1.3 讲 baseHandlers 的 get/set，1.4 讲 ref 如何处理原始值，1.5 讲惰性递归，1.6 讲变体。但这六节都在讲"**数据层**"--如何把对象变成代理、get/set/变体如何拦截。而 handler 里反复调用的 `track` / `trigger` 的内部--依赖到底存在哪？`trigger` 如何找到该通知哪些 effect？这是主题块 2「依赖收集与触发」要回答的核心。


---

### 补充串讲：响应式数据源码追踪调用链（1.2 + 1.3 + 1.5 串联）

> 本段为考核题 1 未通过后的补充讲解。用一条完整调用链把「创建 -> 读 -> 写」三阶段串起来，对照源码分支逐行走。

场景：

```ts
const state = reactive({ user: { age: 20 } })
state.user.age        // 读
state.user.age = 21   // 写
```

#### 阶段 A：`reactive(state)` 创建（1.2）

```
reactive({user:{age:20}})
  └─ createReactiveObject(target, false, baseHandlers, collectionHandlers, reactiveMap)
       ├─ ① isObject(target)? ─────── yes({}是对象) ──> 继续
       ├─ ② target[IS_REACTIVE]? ──── no(普通对象)  ──> 继续
       ├─ ③ reactiveMap.get(target)? ─ no(首次)     ──> 继续
       ├─ ④ getTargetType(target)? ── COMMON(非INVALID) ──> 继续
       └─ ⑤ new Proxy(target, baseHandlers) ─> reactiveMap.set ─> 返回 proxy
```

**三个易错点**：

1. **没有 isRef 判断**。isRef 属于 `ref.ts`，不在 `createReactiveObject` 流程里。
2. **返回的是 `proxy`（state 的代理）**，不是 `proxy.user`。`proxy.user` 要等访问时才产生。
3. **`user` 此刻没被代理**。reactiveMap 里只有 `{ state原始对象 -> proxy }` 一项。子对象访问时才惰性代理（1.5）。

#### 阶段 B：读取 `state.user.age`（1.3 + 1.5）

`state.user.age` 是**两次属性访问**，每次都触发一次 get trap：

```
state.user           ← 第 1 次 get trap
  └─ get(target=state原始, key='user', receiver=proxy)
       ├─ Reflect.get(state, 'user', proxy) ─> res = {age:20}
       ├─ track(state, GET, 'user')              ← track ①
       ├─ shallow? no
       ├─ isRef(res)? no
       └─ isObject(res)? yes ─> reactive(res)    ← 惰性递归入口！(1.5)
            └─ createReactiveObject(user...)
                 └─ ③缓存未命中 ─> ⑤new Proxy(user) ─> proxy_user
       └─ 返回 proxy_user

proxy_user.age       ← 第 2 次 get trap
  └─ get(target=user原始, key='age', receiver=proxy_user)
       ├─ Reflect.get(user, 'age', proxy_user) ─> res = 20
       ├─ track(user, GET, 'age')               ← track ②
       └─ isObject(20)? no ─> 返回 20
```

**关键**：

- **2 次 get trap、2 次 track**。两次 track 对应不同 `(target, key)`：`(state,'user')` 和 `(user,'age')`。
- `user` 在第 1 次 get trap 的 `isObject(res) -> reactive(res)` 处被代理（惰性递归入口，1.5）。不是创建时代理的。

#### 阶段 C：修改 `state.user.age = 21`（1.3）

```
state.user           ← 先走 get（同阶段B第1次）─> 拿到 proxy_user
proxy_user.age = 21  ← set trap
  └─ set(target=user原始, key='age', value=21, receiver=proxy_user)
       ├─ oldValue = user['age'] = 20，toRaw(20)=20
       ├─ hadKey = hasOwn(user, 'age') = true
       ├─ Reflect.set(user, 'age', 21, proxy_user)   ← 真正写入 user.age=21
       ├─ target === toRaw(receiver)?
       │    user === toRaw(proxy_user)=user ─> yes ✅
       └─ hadKey && hasChanged(21,20) ─> trigger(user, SET, 'age', 21, 20)
```

**`target === toRaw(receiver)` 防的是什么？**

防**原型链 set 重复触发**。场景：若 `user` 继承自另一个 reactive 对象 `proto`：

```
proxy_user.age = 21
  ├─ user 上没有 age ─> 沿原型链到 proto 的 setter
  │    └─ proto 的 set trap 被调用（receiver 仍是 proxy_user，但 target=proto）
  │         └─ target === toRaw(receiver)?  proto === user? ─> no ─> 不 trigger
  └─ user 上有 age ─> user 的 set trap
       └─ target === toRaw(receiver)?  user === user? ─> yes ─> trigger
```

所以此判断确保**只在"真正属主"（属性实际写在哪层）上 trigger 一次**，避免原型链场景下 proto 和 user 各触发一次（重复更新）。

**它不是**"防不需要代理的对象"--那是创建阶段 ①`isObject` / ④`getTargetType===INVALID` 的事，发生在 `new Proxy` 之前，跟 set trap 无关。

#### 三阶段串联记忆图

```
创建(1.2):  reactive(o) ─createReactiveObject─> 只包一层 proxy（子对象未代理）
读(1.3+1.5): proxy.key ─get trap─> track + isObject?reactive(res):res（惰性递归）
写(1.3):    proxy.key=v ─set trap─> Reflect.set + (target===toRaw(receiver)?) trigger
```


## 考核过程

 题目 1：源码追踪题（必须）

  const state = reactive({ user: { age: 20 } })

  state.user.age        // 第 1 行：读取
  state.user.age = 21   // 第 2 行：修改

  追踪三件事的调用链（列出经过的函数节点 + 顺序）：

  1. reactive(state) 执行时，createReactiveObject 依次走了哪些判断分支？最终返回什么？此时子对象 user 被代理了吗？
   - 是否isReactiive、是否isReadonly、是否isRef、是否ifObject判断；最终返回 proxy.user,子对象user被代理了
  2. 读取 state.user.age 时：触发了几次 get trap？几次 track？user 是在哪个环节被代理的？（惰性递归入口是哪行）
   - 1次，1次；在isObject(user)? reactive(user):user
  3. 修改 state.user.age = 21 时：set trap 的执行流程？trigger 如何被调用？代码里 target === toRaw(receiver)
  这个判断在这里防的是什么问题？
  - 防不需要代理的对象，直接返回原是对象

  ---
  📌 题目 2：机制推理题（必须）

  const a = ref({ x: 1 })
  const b = shallowReactive({ x: 1, deep: { y: 1 } })
  const c = readonly(reactive({ x: 1 }))

  假设各自有一个 effect 读取了对应属性。推理并说明为什么：

  1. a.value.x = 2 能否触发「读取了 a.value.x 的 effect」？为什么？（提示：toReactive）
    - 触发，因为ref是对象类型，会toReactive包一层
  2. b.x = 2 和 b.deep.y = 2，哪个能触发更新、哪个不能？为什么？
    - b.x能触发，d.deep.y不能触发更新，因为shallowReactive只代理第一层对象，深层对象不代理，直接返回原是值
  3. c.x = 2 会发生什么？若通过 reactive 代理改底层值（reactiveProxy.x = 3），c 的视图会更新吗？为什么 readonly 自己不 track却还能更新？
    - 在开发模式，会warn提示；会更新，因为readonly只在外层包装，里层的走toReactive({x:1})

  ---
  📌 题目 3：设计理解题（必须）

  1. ref 为什么用独立的 RefImpl 类，而不直接用 reactive({ value: x })？说出核心权衡（至少 2 点）。
   - 因为proxy只能代理对象，基本类型不能代理；
   - 需要区分引用类型还是基本类型，如果直接用 reactive({ value: x })，区分不了
  2. reactiveMap 为什么用 WeakMap 而不是普通 Map？如果用普通 Map 会有什么具体问题？
   - WeakMap 的key是原始对象，能够在不引用的时候进行CG
   - this指向问题，WeakMap的this只想原始对象，这样能保证代理对象和原始对象是同一个引用，改代理对象的值就是改原始对象的值

1.1 是否isReactiive、是否isReadonly、是否isRef、是否ifObject判断；最终返回 proxy.user,子对象user被代理了
1.2 1次，1次；在isObject(user)? reactive(user):user
1.3 防不需要代理的对象，直接返回原是对象
2.1 触发，因为ref是对象类型，会toReactive包一层
2.2 b.x能触发，d.deep.y不能触发更新，因为shallowReactive只代理第一层对象，深层对象不代理，直接返回原是值
2.3 在开发模式，会warn提示；会更新，因为readonly只在外层包装，里层的走toReactive({x:1})
3.1  - 因为proxy只能代理对象，基本类型不能代理；
     - 需要区分引用类型还是基本类型，如果直接用 reactive({ value: x })，区分不了
3.2  - WeakMap 的key是原始对象，能够在不引用的时候进行CG
     - this指向问题，WeakMap的this只想原始对象，这样能保证代理对象和原始对象是同一个引用，改代理对象的值就是改原始对象的值