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

## 考核过程

（主题块 1 全部讲完后统一考核，此处待填充）
