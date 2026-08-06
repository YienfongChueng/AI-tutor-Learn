# 1.1 原始类型 & 字面量类型

## 讲解原文

### 一、原始类型（Primitive Types）

TypeScript 的原始类型与 JavaScript 一一对应，共 7 种。**类型注解**语法是 `let 变量名: 类型 = 值`：

```ts
let name: string = 'Tom'
let age: number = 18
let isOn: boolean = true
let nothing: null = null
let undef: undefined = undefined
let sym: symbol = Symbol('id')
let big: bigint = 100n
```

| JS 值 | TS 类型 |
|-------|---------|
| `'Tom'` | `string` |
| `18` | `number` |
| `true` | `boolean` |
| `null` | `null` |
| `undefined` | `undefined` |
| `Symbol()` | `symbol` |
| `100n` | `bigint` |

> 类比：把类型注解想象成「变量盒子的标签」。JS 的盒子贴的是「随便装」，TS 的盒子贴的是「只准装 number」——装错了编译期就报警。

### 二、类型推断（Type Inference）—— 你不写，TS 替你猜

大多数时候你**不必手写注解**，TS 会根据初值自动推断：

```ts
let count = 0       // 推断为 number
let title = 'TS'    // 推断为 string
```

**关键陷阱：`let` vs `const` 的推断不同：**

```ts
let a = 'hello'   // 推断为 string（宽类型，因为 let 可被重新赋值）
const b = 'hello'  // 推断为 'hello'（字面量类型，因为 const 不可变）
```

这就引出了「字面量类型」。

### 三、字面量类型（Literal Types）

在 TS 中，**一个具体的值本身也可以当作类型**。当变量只能是某个特定值时，它的类型就是字面量类型：

```ts
let x: 'hello' = 'hello'   // x 只能是 'hello'，赋别的就报错
let y: 42 = 42             // y 只能是 42
let z: true = true
```

单独一个字面量类型没太大用。真正的威力在**联合字面量类型**——用一个类型表达「有限的取值集合」：

```ts
type Status = 'open' | 'closed' | 'done'
type Direction = 'up' | 'down' | 'left' | 'right'
type Role = 'admin' | 'editor' | 'viewer'

function setStatus(s: Status) {
  // ...
}
setStatus('open')    // ✅
setStatus('opened')  // ❌ 编译报错：拼写错误当场抓到
```

> 类比：原始类型 `string` 是「任何字符串都行」，像一张不限量的自助餐券；字面量联合类型 `Status` 是「只能从这三道菜里选」，像套餐菜单。套餐更严格，但也更精确——点错菜立刻提醒。

### 四、为什么字面量联合类型重要（实战要点）

1. **替代「魔法字符串」**：后端开发里你常会写 `if (status === 'done')`，把 `'done'` 这种散落各处的字符串抽成 `Status` 类型后，拼错 `'don'` 编译期就能抓到，不用等运行时。
2. **让类型成为文档**：看到 `Status = 'open' | 'closed' | 'done'`，立刻知道业务上有哪些状态，不用翻接口文档。
3. **`let` 推断会「拓宽」到宽类型**：`let s = 'open'` 推断成 `string` 而非 `'open'`。若你确实想要字面量类型，要么用 `const`，要么显式标注 `let s: Status = 'open'`。

### 五、一句话总结

- 原始类型 = JS 的 7 种基础值的类型，注解语法 `变量: 类型`。
- 类型推断 = 不写注解 TS 也能猜；`let` 推宽类型，`const` 推字面量。
- 字面量联合类型 = 用「值即类型」精确表达有限取值集合，是消灭魔法字符串的利器。

## 考核过程

### 题目 1：用 TS 改造魔法字符串代码（定义 OrderStatus 联合 + 补注解 + 抓拼错） 

**用户回答**:        
```ts
type OrderStatus = 'pending' | 'paid' | 'shipped' | 'done'
let currentStatus: OrderStatus = 'pending'

type Order = {
  id: number,
  productName: string,
  status:OrderStatus
}
function filterOrders(orders: Order[],status:OrderStatus) {
  return orders.filter(o => o.status === status)
}
filterOrders(list,'pending')
filterOrders(list,'shipped')
filterOrders(list,'shiped')

```
- 问题：如果你写 let currentStatus = 'pending'（不加注解），TS 会把 currentStatus
  推断成什么类型？这会导致后面 currentStatus = 'paid' 时出现什么问题？你会怎么解决？
> `let` 的拓宽让你失去了字面量级的精确性，错误从「编译期被拦」退化成「运行时静默」。 解决：显式注解 `let currentStatus: OrderStatus = 'pending'`（你已经会了），或用` const`（但 `const` 不能重新赋值，这里场景不适用）。
- 问题：为什么 let currentStatus = 'pending' 反而比 const currentStatus = 
  'pending' 在类型上「更不安全」？
> 因为`let` 会拓宽类型，值拼错了也静默通过，编译不会出错；而`const`会推断成字面量类型，如果值不对，会在编译的时候报错

**点评**:
1. 三个硬性要求（联合定义、注解、编译期抓拼错）全部满足。第 4 题复述准确，抓住了`let`拓宽vs `const`字面量的本质差异。
2. 两处需修正：
   - 细节错误：`orders: List` 未定义、`o.status` 隐式 any → 需补 `Order` 对象类型，`orders: Order[]`，`status: OrderStatus`，使 `o.status === status` 两边都是合法状态。
   - 概念澄清：`let currentStatus = 'pending'` 的「问题」不是「`paid` 报错」，而是字面量被拓宽成 `string` 后保护失效--拼错也静默通过。错误从「编译期被拦」退化为「行时静默」。

 ### 补充对话
- 引导补 `Order` 类型：用户给出 `{ id: number, productName: string, status: OrderStatus }`，正确。
- 引导修 `orders` 参数：从 `orders: Order` 改为 `orders: Order[]`（`.filter()` 是数组方法），用户答对。
---                                                                             
通过时间：2026-08-06   