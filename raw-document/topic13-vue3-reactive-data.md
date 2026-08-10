# 主题块 1：响应式数据

> Vue3 响应式源码 · vue@3.4 · 深度讲解模式（conceptual）

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

| trap | 触发时机 | 作用 |
|------|---------|------|
| `get(target, key, receiver)` | 读 `proxy[key]` | 拦截读取，可在此 `track` 收集依赖 |
| `set(target, key, value, receiver)` | 写 `proxy[key] = v` | 拦截赋值，可在此 `trigger` 触发更新 |
| `has` | `key in proxy` | 拦截 in 操作 |
| `deleteProperty` | `delete proxy[key]` | 拦截删除 |
| `ownKeys` | `Object.keys(proxy)` | 拦截遍历 |
| ... | （共 13 种） | defineProperty 只有 get/set 2 种 |

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

> 现在我们知道了：Vue3 用 Proxy 整体代理对象，解决了 Vue2 defineProperty 检测不到新增/删除、数组受限、初始化全量递归的缺陷。但这只回答了「为什么用 Proxy」——**`reactive()` 具体是怎么把一个对象变成 Proxy 的？handler 里到底写了什么？** 这是 1.2 要处理的事。

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

| targetType | 对象 | handler |
|---|---|---|
| COMMON | `{}`, `[]` | baseHandlers（mutableHandlers） |
| COLLECTION | Map/Set/... | collectionHandlers |
| INVALID | 原始值/frozen/skip | 不代理，直接返回 |

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
- **`target === toRaw(receiver)` 防重复触发**：对象继承自另一个 reactive 对象时，set 会先走原型 setter（receiver 是子实例）再走实例本身。此判断确保只在"真正属主"上 trigger 一次。
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

> 现在我们知道了：get 里 `Reflect.get + track` 收集依赖，set 里 `Reflect.set + trigger` 触发更新，receiver 保原型链响应式，ADD/SET/hasChanged 精准控制。但 track/trigger 的内部（依赖存哪、怎么找）是主题块 2 的事。在此之前，1.4 先解决遗留问题：**原始值不能被 Proxy，那 `ref()` 怎么让一个 number 变成响应式？**

## 考核过程

（主题块 1 全部讲完后统一考核，此处待填充）
