# TypeScript 学习笔记（topic1-12 整合）

> 基于 ai-tutor/raw-document topic1~topic12 整理，2026-08-06 ~ 2026-08-10 全部考核通过。
> 结构：模块1 类型基础 → 模块2 类型系统进阶 → 模块3 泛型与工具类型。
> 每节含「一句话锚定 + 核心机制 + 细节盲区（考核踩过）」。

---

## 目录

- 模块1：类型基础
  - [1.1 原始类型与字面量类型](#11-原始类型与字面量类型)
  - [1.2 数组、元组与对象类型](#12-数组元组与对象类型)
  - [1.3 函数类型与重载](#13-函数类型与重载)
  - [1.4 interface vs type](#14-interface-vs-type)
  - [1.5 类型断言与 declare 声明](#15-类型断言与-declare-声明)
- 模块2：类型系统进阶
  - [2.1 联合类型与交叉类型](#21-联合类型与交叉类型)
  - [2.2 类型守卫与类型收窄](#22-类型守卫与类型收窄)
  - [2.3 映射类型与索引签名](#23-映射类型与索引签名)
- 模块3：泛型与工具类型
  - [3.1 泛型基础](#31-泛型基础)
  - [3.2 泛型约束 extends](#32-泛型约束-extends)
  - [3.3 条件类型与 infer](#33-条件类型与-infer)
  - [3.4 内置工具类型实战](#34-内置工具类型实战)
- [附录：个人高频错误清单](#附录个人高频错误清单)

---

# 模块1：类型基础

## 1.1 原始类型与字面量类型

**一句话锚定：类型注解是给变量盒子贴标签；`let` 推宽类型、`const` 推字面量——字面量联合类型是消灭魔法字符串的利器。**

### 原始类型与注解

TS 原始类型与 JS 一一对应，共 7 种：`string` / `number` / `boolean` / `null` / `undefined` / `symbol` / `bigint`。注解语法 `let 变量: 类型 = 值`。

### 类型推断的关键陷阱：let vs const

```ts
let a = 'hello'    // 推断为 string（宽类型，因为 let 可被重新赋值）
const b = 'hello'  // 推断为 'hello'（字面量类型，因为 const 不可变）
```

**后果**：`let currentStatus = 'pending'` 被拓宽成 `string` 后，类型保护失效——拼错 `'paid'` → `'padi'` 不会编译报错，错误从「编译期被拦」退化成「运行时静默」。想要字面量精确性：显式标注 `let s: Status = 'open'`，或用 `const`。

### 字面量联合类型 = 轻量级枚举

```ts
type Status = 'open' | 'closed' | 'done'
type Role = 'admin' | 'editor' | 'viewer'

function setStatus(s: Status) { /* ... */ }
setStatus('open')    // ✅
setStatus('opened')  // ❌ 拼写错误当场抓到
```

三个实战价值：① 替代魔法字符串，拼错编译期抓；② 类型即文档，一眼看清业务状态全集；③ 收窄后能精确区分每个分支（配合 2.2 穷尽检查）。

> 类比：`string` 是不限量自助餐券，字面量联合是套餐菜单——更严格，但点错菜立刻提醒。

### 细节盲区（考核踩过）

- 「`let s = 'open'` 的问题」不是「赋 `'paid'` 会报错」，而是**拓宽后拼错也静默通过**——保护失效而非报错
- 用 const 保字面量的前提是变量不需要重新赋值；需要重赋值时只能显式注解

---

## 1.2 数组、元组与对象类型

**一句话锚定：数组 `类型[]`、元组 `[A, B]` 固定长度异构、对象逐字段标注；联合数组必须加括号 `(A|B)[]`——`|` 优先级比 `[]` 低。**

### 数组：小心括号

```ts
let a: number[]             // 全是 number
let b: number | string[]    // ⚠️ 这是「number 或 string[]」
let c: (number | string)[]  // ✅ 元素是 number|string 混合的数组
```

### 元组：字面量初值不会推断成元组

| | 数组 `number[]` | 元组 `[number, string]` |
|---|---|---|
| 长度 | 任意 | 固定 |
| 每位置类型 | 全部相同 | 可以不同 |
| 推断 | `[1,2,3]` → `number[]` | `['x', 1]` → `(string\|number)[]`（**不是元组！**） |

```ts
let t: [string, number] = ['x', 1]   // 必须显式标注，或用 as const
```

实战场景：`useState` 返回值就是元组 `[number, (v: number) => void]`、CSV 行、坐标点、`Promise.all` 结果。

### 对象类型：三个修饰

```ts
type User = { name: string; age?: number }          // ? 可选
type Config = { readonly apiKey: string }            // readonly 只读
type Scores = { [k: string]: number }                // 索引签名
```

- **可选字段读取时是 `类型 | undefined`**：`u.age.toFixed()` 报错，需判空或 `u.age?.toFixed()`
- **`readonly` 是编译期检查**，运行时仍可改——防误改不防恶意篡改
- **索引签名会约束所有字段**：

```ts
type User = {
  name: string        // ❌ 报错！string 与索引签名 [k: string]: number 不兼容
  [k: string]: number
}
```

修正：索引签名值改成 `string | number`，或不混用。

### 细节盲区（考核踩过）

- **可选字段 vs 索引签名的选择**：有固定字段名 → `avatar?: string`（不用方括号）；key 不固定 → 才用 `[k: string]: string`（考核中两次栽在方括号上）
- **可选链 vs 条件分支**：`?.` 只管「访问不报错」，管不了显示格式——`u?.age` 全拼时 age 不存在会输出 `"undefined"` 字样；要控制「不显示」必须用 `if (u.age !== undefined)` 条件拼接
- 元组选型理由：CSV 行每个位置类型可不同且顺序固定，数组表达不了这种「位置→类型」的对应

---

## 1.3 函数类型与重载

**一句话锚定：函数类型字面量 `(a: T) => U` 描述签名；重载 = 多个对外签名 + 一个兼容实现，价值是保留「入参→返回」的精确对应。**

### 基础注解

```ts
function add(a: number, b: number): number { return a + b }
type Formatter = (value: number) => string   // 函数作为类型
const f: Formatter = (x) => String(x)        // 参数名只是占位，不必叫 value
```

返回值类型多可推断省略，但**公开 API 建议显式标注**，防止实现悄悄改了返回类型没人发现。

### 参数规则

| 特性 | 语法 | 说明 |
|------|------|------|
| 可选参数 | `name?: type` | **必须在必填参数之后** |
| 默认参数 | `name: type = 值` | 带默认值后自动可选 |
| 剩余参数 | `...args: type[]` | 收集实参为数组 |

```ts
function f(a?: number, b: number) {}   // ❌ 必填不能在可选后面
```

`void` 表示「调用方不应该使用返回值」，常用于回调签名；`void` 允许实现返回任意值，`undefined` 严格要求返回 undefined。

### 函数重载：三条铁律

```ts
// 重载签名（对外暴露，可多个）
function query(type: 'list'): string[]
function query(type: 'count'): number
// 实现签名（对外不可见，必须兼容所有重载）
function query(type: string): string[] | number {
  if (type === 'list') return ['a', 'b']
  return 0
}
const r1 = query('list')   // string[]，精确！
```

1. 重载签名在外，实现签名在内（调用方看不到实现签名）
2. 实现签名的参数/返回类型必须兼容所有重载（一般是它们的联合）
3. 运行时只有一个函数——TS 重载不是同名多份实现，是**一套实现 + 多套对外类型声明**

**为什么不直接写联合返回？** `query('list')` 拿到 `string[] | number` 后还得自己收窄，丢掉了「这个入参对应那个返回」的精确关系。

### 细节盲区（考核踩过）

- **重载签名必须用能区分的参数类型当分派依据**：两个重载签名都写宽类型 `string` 时，TS 按顺序匹配永远命中第一个，`query('count')` 实际返回 `string[]`——重载坍缩失效。**字面量参数类型是重载精确分派的基石**（菜单上两道菜不能都叫"炒饭"）
- 实现签名的返回类型写少了（如只写 `string[]`）违反铁律 2，必须是联合
- 实现分支要覆盖全部路径，消除末尾隐式 `undefined`

---

## 1.4 interface vs type

**一句话锚定：对象形状选 interface，组合别名选 type——能力有交集、各有独占地盘，且共用同一个命名空间。**

### 四个核心差异

| 差异 | interface | type |
|------|-----------|------|
| 声明合并 | ✅ 同名自动合并（给第三方库/全局 Window 补字段的关键） | ❌ 同名报 Duplicate identifier |
| 扩展写法 | `extends` | `&`（交叉类型） |
| 能表示的类型 | 只能描述对象形状 | 联合、元组、基本类型别名、映射/工具类型都能做 |
| 报错信息 | 显示类型名，更友好 | 有时展开成原始结构 |

```ts
// 扩展对照
interface Dog extends Animal { bark(): void }
type Dog = Animal & { bark(): void }

// 声明合并实战：不改源码给第三方库补字段
interface DatabaseConfig { host: string }
interface DatabaseConfig { port: number }   // ✅ 合并成功
```

### 决策表

| 场景 | 推荐 |
|------|------|
| 对象/类的形状（props、API 响应、implements 契约） | interface |
| 联合 / 元组 / 基本类型别名 | type（只能用 type） |
| 需要声明合并 / 扩展库类型 | interface（只能用 interface） |
| 工具类型 / 映射类型 | type |
| 函数类型 | type 更简洁 |

> 口诀：**对象形状选 interface，组合别名选 type。** 团队统一最重要。

### 细节盲区（考核踩过）

- **interface 和 type 共用同一命名空间**：二者不是平行两套世界，同一个标识符不能既声明 interface 又声明 type（Duplicate identifier）
- 对象类型里方法两种写法不混用：方法简写 `m(x): T` 或属性式 `m: (x) => T`

---

## 1.5 类型断言与 declare 声明

**一句话锚定：断言是「我保证」（只改类型不改运行时值），declare 是「它已存在，我只补类型说明」（不产生任何 JS 代码）——都是纯编译期产物。**

### 类型断言

```ts
const el = document.getElementById('app') as HTMLDivElement   // 推荐写法
const el2 = <HTMLDivElement>document.getElementById('app')    // ❌ 与 JSX 冲突，.vue 文件里必须用 as
```

**三个关键认知：**

1. **断言不执行任何运行时转换**，编译后消失。`value as number` 后运行时还是字符串，调 `.toFixed()` 直接崩——**断言不等于安全**
2. **断言有约束**：原类型和目标类型必须有重叠；完全不相关要 `as unknown as X` 强制绕过（危险，等于关掉类型保护）
3. **断言目标要选「带目标属性的具体子类型」**：

```ts
const input = document.querySelector('.search-input') as HTMLInputElement
// ❌ as Element 仍然没有 .value 属性——要断言到真正有目标属性的那个具体子类型
```

### declare 与 .d.ts

痛点：import 无类型的第三方库报 `Could not find declaration file`。解法：写 `.d.ts` 声明文件。

```ts
// legacy-auth.d.ts —— 默认导出对象分两步：先声明形状再 export default
declare module 'legacy-auth' {
  const auth: { login(token: string): boolean }
  export default auth
}

// vite-env.d.ts 常见：构建工具注入的全局变量 + 扩展 Window（靠 interface 声明合并）
declare const __APP_VERSION__: string
declare interface Window { myCustomProp: string }
```

`declare` 的语义：「这个变量/模块在运行时已存在，我只是告诉你它的类型」——编译产物里**完全消失**，不产生运行时代码。

### 断言 vs 类型守卫（决策表）

| 场景 | 用断言 | 用类型守卫 |
|------|:------:|:---------:|
| 你 100% 确定且无法通过逻辑判断 | ✅ | ❌ |
| 类型可以通过 if/typeof 判断 | ❌（偷懒） | ✅ |
| 运行时数据来源不可信（API、用户输入） | ❌ 危险 | ✅ 必须校验 |

> 断言是「我保证」，守卫是「我验证」。**能用守卫就别用断言**；能从源头改精确类型（如把 `status: string` 改成字面量联合）优于事后断言。

### 细节盲区（考核踩过）

- 断言目标选错：`as Element` 去掉 null 但没拿到 `.value`——要断言到**具体子类型**
- `.d.ts` 默认导出对象的语法坑：不能写成 `export function`（那是命名导出）或 `export auth { ... }`（混用语法），必须先 `const auth: {...}` 再 `export default auth`

---

# 模块2：类型系统进阶

## 2.1 联合类型与交叉类型

**一句话锚定：联合做「选择」、交叉做「叠加」；联合收窄前只能访问公共成员，交叉的同名冲突会坍缩成 never。**

### 核心对照

| | 联合 `A \| B` | 交叉 `A & B` |
|---|---|---|
| 逻辑 | 或（OR） | 与（AND） |
| 值的集合 | A 的值 ∪ B 的值（并集） | 同时满足 A 和 B（交集） |
| 可访问成员 | 只能访问**公共**成员 | 可访问**所有**成员 |
| 典型用途 | 多种可能之一 | 组合 / 混入 |

> 对称记忆：联合的**值**是并集、**可访问成员**是交集；交叉正好相反（值是交集、成员是并集）。

```ts
// 联合：收窄前只能碰公共部分
function operate(pet: Bird | Fish) {
  pet.layEggs()   // ✅ 两边都有
  pet.fly()       // ❌ Fish 没有 fly，须先收窄（→ 2.2）
}

// 交叉：组合
type Employee = Person & Loggable   // 同时有全部成员

// 交叉最隐蔽的坑：同名冲突 → never
type A = { name: string; age: number }
type B = { name: number; email: string }
type C = A & B   // name: string & number → never，赋任何值都报错
```

易混点：`{a:1} | {b:2}` 是「二选一」，不等于 `{a:1, b:2}`；`{a:1} & {b:2}` 才是「全都要」。

### 细节盲区（考核踩过）

- `never` 不接受任何具体值：`const r: { id: string & number } = { id: ??? }` 填任何值都报错——因为不可能存在既是 string 又是 number 的值
- 表述精确性：「联合不是并集」的说法是错的——联合的**值**确实是并集，只是**可访问成员**是交集

---

## 2.2 类型守卫与类型收窄

**一句话锚定：守卫触发收窄，收窄后才能访问联合的特有成员；给联合设计一个字面量判别字段（判别联合）是最稳的模式。**

### 五种收窄手段

| 手段 | 适用场景 | 语法 |
|------|---------|------|
| typeof | 原始类型 | `typeof x === 'string'` |
| in | 对象结构区分 | `'swim' in pet` |
| instanceof | 类实例 | `x instanceof Date` |
| 相等性 / switch | 字面量联合、判别联合 | `===` / `case` |
| 类型谓词 | 复杂自定义判断 | `x is Type` |

```ts
// 类型谓词：TS 只信你的 is 声明，判断逻辑要自己保证正确
function isFish(pet: Bird | Fish): pet is Fish {
  return (pet as Fish).swim !== undefined
}
```

### 判别联合（Discriminated Union）—— 最推荐

```ts
interface Success { ok: true;  data: string }
interface Failure { ok: false; error: string }
type Result = Success | Failure

function handle(r: Result) {
  if (r.ok === true) { r.data }    // r: Success
  else { r.error }                  // r: Failure
}
```

判别字段比 `in` 检查业务字段更稳：业务字段未来可能被两个类型共有导致 `in` 失效（Failure 加了 `data` 后，`'data' in r` 对两者为真，if 分支仍是联合，else 分支收窄成 `never` 直接编译报错）；判别字段是专为区分设计的，不会撞车。

### 穷尽检查（Exhaustiveness）+ never

```ts
type Status = 'active' | 'inactive' | 'pending'
function handle(s: Status) {
  switch (s) {
    case 'active':   return 1
    case 'inactive': return 0
    case 'pending':  return -1
    default:
      const x: never = s   // 新增字面量忘加 case 时，这里赋值报错——编译期安全网
      return x
  }
}
```

### 细节盲区（考核踩过）

- **`typeof null === 'object'`**（历史遗留 bug），判 null 用 `x === null`
- **instanceof 对 interface/纯对象字面量无效**——运行时类型信息已擦除，没有构造函数痕迹可查
- **真值收窄的边界**：对布尔判别符 `if (r.ok)` 与 `=== true` 等价；但对字符串/数字字面量判别符真值收窄无效，必须用 `===`/switch

---

## 2.3 映射类型与索引签名

**一句话锚定：索引签名描述「任意键的开放对象」，映射类型是「类型层面的循环」，遍历 keyof 得到的固定键集合生成新类型——内置工具类型的基石。**

### 索引签名

```ts
interface StringMap { [key: string]: string }
```

**关键约束：索引签名类型必须兼容所有显式属性**（`age: number` 撞上 `[key: string]: string` 直接报错——因为 `person['age']` 也要走索引签名）。修复代价：索引签名放宽成 `string | number` 后，`user['任意键']` 的返回值也变宽了。

number 索引类型必须是 string 索引类型的子类型（JS 对象键最终都是字符串，`obj[0]` 实际是 `obj['0']`）。

### keyof 与映射类型

```ts
interface User { id: number; name: string }
type UserKeys = keyof User              // 'id' | 'name' ← 字面量联合

type Stringify<T> = { [K in keyof T]: string }    // 遍历键，值变 string
```

**修饰符加减：** `+`（可省略）加、`-` 移除：

```ts
type MyPartial<T>  = { [K in keyof T]?: T[K] }            // 加 ?
type MyReadonly<T> = { readonly [K in keyof T]: T[K] }    // 加 readonly
type Mutable<T>    = { -readonly [K in keyof T]: T[K] }   // 移除 readonly
type Required<T>   = { [K in keyof T]-?: T[K] }           // 移除可选
```

`T[K]` 是索引访问类型：取出 T 中键 K 对应的值类型。

### 索引签名 vs 映射类型（高频考点）

| | 索引签名 `[key: string]: T` | 映射类型 `[K in keyof T]: ...` |
|---|---|---|
| 键 | 任意 string 键（不固定集合） | 遍历 T 的具体键（固定集合） |
| 产物 | 「开放」对象 | 「封闭」对象 |
| 用途 | 描述动态键的字典 | 基于已有类型变换出新类型 |

### 细节盲区（考核踩过）

- **两个 `in` 不是一回事**：映射类型的 `in` 是类型层面「遍历键」，2.2 的 `in` 是运行时类型守卫——别混
- 读题要抠字眼：`{ [key: string]: number }` 的值是 number 不是 string（考核中答错过）

---

# 模块3：泛型与工具类型

## 3.1 泛型基础

**一句话锚定：泛型是类型的参数化——`T` 不是「任意类型」，而是「调用那一刻被锁定的某个具体类型」，同一次调用里 T 保持一致，保住了入参→返回的类型关系，这正是 any 做不到的。**

### 泛型 vs any

```ts
function identity(value: any) { return value }
const n = identity(42)      // n: any —— 类型信息丢了，n.toUpperCase() 不报错但运行时炸

function identity<T>(value: T): T { return value }
const n = identity(42)      // n: number，n.toUpperCase() 编译期报错 ✅
```

| 维度 | `any` | 泛型 `<T>` |
|------|-------|-----------|
| 类型关系 | 全部擦除 | 保留入参→返回对应 |
| 编译期检查 | 放弃 | 保留 |
| 适用 | 临时绕过、迁移期 | 正经写可复用逻辑 |

### 三种声明形式

```ts
// ① 泛型函数
function first<T>(arr: T[]): T { return arr[0] }
// ② 泛型接口
interface Box<T> { value: T }
// ③ 泛型类
class Stack<T> { private items: T[] = []; push(x: T) { this.items.push(x) }; pop(): T | undefined { return this.items.pop() } }
```

### 实战：泛型 API 响应包装（axios 封装的底层原理）

```ts
interface ApiResponse<T> {
  code: number
  message: string
  data: T
}
const userRes: ApiResponse<{ id: number; name: string }> = {
  code: 200, message: 'ok', data: { id: 1, name: '张三' }
}
userRes.data.name   // ✅ 有类型提示——复用结构，不丢类型
```

多类型参数命名约定：`T`/`U`/`V` 通用类型，`K` 键、`V` 值、`E` 集合元素。

### 细节盲区（考核踩过）

- **对象字面量（值）属性分隔用 `,`，interface 内用 `;`**——考核中把返回对象写成 `success: true;` 报错
- 泛型版本类型安全的精确解释：T 在一次调用里被锁定为同一类型，`arr: T[]` 与 `data: T` 共享此 T；any 版本的 arr 与 data 各自独立为 any，无联动

---

## 3.2 泛型约束 extends

**一句话锚定：`T extends Shape` 读作「T 至少满足 Shape 形状」（约束不是继承）——约束既让函数体能安全访问 Shape 的成员，又保留 T 的具体类型。**

### 解决的问题

无约束的 T 上什么都做不了（`x.length` 报错），断言 `as any` 又不安全。约束 = 在「保留具体类型」和「能用特定成员」之间取得平衡：

| 写法 | 能否用 `.length` | 保留具体类型 |
|------|------|------|
| `x: any` | 能（不安全） | 否 |
| `x: T`（无约束） | 不能 | 是 |
| `x: T extends HasLength` | 能（安全） | 是 |

```ts
interface HasLength { length: number }
function getLength<T extends HasLength>(x: T): T {
  return x   // 返回 T，不是 HasLength——约束不擦除具体类型
}
getLength('abc')     // ✅ s: string
getLength(123)       // ❌ number 不满足 HasLength
```

### keyof 约束：安全访问对象属性（经典必会）

```ts
function getValue<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
getValue(user, 'name')    // ✅ 返回 string（精确到值类型）
getValue(user, 'emial')   // ❌ 编译期报错：不是 user 的 key
```

多约束用 `&` 叠加：`<T extends { id: number } & { name: string }>`。类型参数可给默认值：`interface Paginated<T = any>`。

### 细节盲区（考核踩过）

**「extends 解决了什么问题」要区分对照组**：
- 相比**基类型**（`x: HasLength`）：新增的是**保留具体类型**（基类型返回值丢掉 string 身份）
- 相比**无约束泛型**（`x: T`）：新增的是**函数体能安全使用约束形状的成员**

两个价值别答串。

---

## 3.3 条件类型与 infer

**一句话锚定：`T extends U ? X : Y` 是类型层面的 if-else；`infer R` 是「如果 T 形如这个模式，把匹配的那部分叫 R」——内置工具类型的底层原理。**

### 条件类型与分布性

```ts
type IsString<T> = T extends string ? true : false

type C = IsString<string | number>   // boolean ← 分布性：拆开逐个判断再合并
// = IsString<string> | IsString<number> = true | false = boolean
```

**分布性只对裸联合生效**。想整体判断，用方括号包住：

```ts
type IsStringNonDist<T> = [T] extends [string] ? true : false
type D = IsStringNonDist<string | number>   // false（不拆开）
```

### infer：从类型里提取

```ts
type ElementOf<T> = T extends (infer E)[] ? E : never
type Unwrap<T>    = T extends Promise<infer R> ? R : T   // 不是 Promise 原样返回

type R1 = ElementOf<number[]>          // number
type R2 = Unwrap<Promise<string>>      // string

// 自造 ReturnType
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never
```

读法：`T extends Promise<infer R>` = 「如果 T 形如 `Promise<某类型>`，把那个某类型绑给 R」。条件类型可嵌套（如 TypeName 链式判断）。

### 细节盲区（考核踩过）

- **联合不坍缩**：`ToArray<string | number>` 分布后得 `string[] | number[]`，这是联合（满足其一即可），**不等于 `[]`**（`[]` 是同时满足两者的特例）——考核中误答坍缩成 `[]`，本质是 2.1 联合概念的回潮

---

## 3.4 内置工具类型实战

**一句话锚定：工具类型是「类型版 Lodash」——底层全是泛型 + 映射类型 + 条件类型 + infer 拼出来的，忘了能现推。**

### 五类速查

| 类别 | 工具 | 作用 | 底层零件 |
|------|------|------|---------|
| 修饰 | Partial / Required / Readonly | 加/去 `?`、加 readonly | 映射类型 |
| 筛选 | Pick / Omit | 留/删指定字段 | 映射 + keyof |
| 构造 | Record | 批量造同 value 的对象 | 映射 + keyof any |
| 提取 | ReturnType / Parameters | 拿函数返回/参数类型 | 条件类型 + infer |
| 联合 | Exclude / Extract | 剔除/提取联合成员 | 分布性条件类型 |

### 用法 + 底层实现

```ts
interface User { id: number; name: string; email: string; age: number }

// ① 属性修饰
type PartialUser = Partial<User>     // 全变可选——表单草稿态：let draft: Partial<Form> = {}
type RequiredUser = Required<User>   // -? 移除可选
type ReadonlyUser = Readonly<User>
type MyPartial<T>  = { [K in keyof T]?: T[K] }
type MyReadonly<T> = { readonly [K in keyof T]: T[K] }

// ② 字段筛选
type UserPreview = Pick<User, 'id' | 'name'>   // 只留 id、name
type UserSafe    = Omit<User, 'email'>         // 去掉 email
type MyPick<T, K extends keyof T> = { [P in K]: T[P] }
type MyOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>

// ③ 构造：Record 收束 key（少写一个 key 立刻报错）
const statusText: Record<'open' | 'done' | 'closed', string> = {
  open: '进行中', done: '已完成', closed: '已关闭'
}
type MyRecord<K extends keyof any, V> = { [P in K]: V }   // keyof any = string | number | symbol

// ④ 类型提取
function fetchUser(id: number, name: string): User { /* ... */ }
type R = ReturnType<typeof fetchUser>   // User
type P = Parameters<typeof fetchUser>   // [number, string] ← 提取的是参数元组
type MyReturnType<T> = T extends (...args: any[])   => infer R ? R : never
type MyParameters<T> = T extends (...args: infer P) => any  ? P : never

// ⑤ 联合操作
type T1 = Exclude<'a' | 'b' | 'c', 'a'>        // 'b' | 'c'
type T2 = Extract<'a' | 'b' | 'c', 'a' | 'b'>  // 'a' | 'b'
// 底层：Exclude<T, U> = T extends U ? never : T（分布性）
```

### 三个实战心法

1. **消除重复**：`type User = ReturnType<typeof fetchUser>` —— 函数改返回类型，User 自动跟着变
2. **Record + 字面量联合**替代 switch 里的魔法字符串映射，key 拼漏/拼错编译期抓
3. **Partial 草稿态**：`let draft: Partial<Form> = {}` 合法，因为全字段可选；不写 Partial 则必须初始化全部必填字段

---

# 附录：个人高频错误清单

> 12 节考核中实际踩过的坑，复习优先级最高。

## 概念级错误（考核中被判「概念错误」）

| # | 错误 | 正确认知 | 出处 |
|---|------|---------|------|
| 1 | 认为 `let s = 'open'` 赋错值会报错 | 拓宽成 string 后**拼错静默通过**——保护失效而非报错 | 1.1 |
| 2 | 重载签名参数用宽类型 `string` | 重载签名必须用**字面量**做分派依据，宽类型导致重载坍缩、永远命中第一个签名 | 1.3 |
| 3 | 认为 extends 的新价值是「保留具体类型」 | 相比基类型才是保留类型；相比无约束泛型，新增的是「函数体能安全访问约束成员」 | 3.2 |
| 4 | `string[] \| number[]` 误坍缩成 `[]` | 联合不坍缩——满足其一即可，`[]` 是同时满足的特例（联合概念回潮） | 3.3 |

## 语法级错误（反复栽）

| # | 错误 | 正确写法 | 出处 |
|---|------|---------|------|
| 5 | 可选字段写成 `['avatar']: string` 方括号 | 固定字段名用 `avatar?: string`，key 不固定才用索引签名 | 1.2 |
| 6 | 可选链 `u?.age` 全拼、age 缺失时输出 "undefined" | `?.` 只管访问不报错；控制显示格式必须 `if (u.age !== undefined)` 条件拼接 | 1.2 |
| 7 | interface 和 type 用同一标识符 | 二者共用命名空间，不能重名 | 1.4 |
| 8 | 断言目标写成父类型 `as Element` | 断言到**带目标属性的具体子类型** `as HTMLInputElement` | 1.5 |
| 9 | `.d.ts` 默认导出写成 `export auth {...}` | 先 `const auth: {...}` 再 `export default auth`，两步 | 1.5 |
| 10 | 对象字面量属性分隔用 `;` | 对象字面量（值）用 `,`；interface 内才用 `;` | 3.1 |

## 易混概念对照（一表防混）

| 容易混 | 区分 |
|--------|------|
| `number \| string[]` vs `(number \| string)[]` | `\|` 优先级低于 `[]`，联合元素必须加括号 |
| 数组 vs 元组 | 字面量初值 `['x', 1]` 推断成 `(string\|number)[]`，不是元组；须显式标注 |
| 断言 vs 类型守卫 | 断言=「我保证」，守卫=「我验证」；能守卫就别断言 |
| 两个 `in` | 映射类型 `[K in keyof T]` 是类型层遍历；`'k' in obj` 是运行时守卫 |
| 索引签名 vs 映射类型 | 任意键开放对象 vs 固定键封闭对象 |
| `void` vs `undefined` 返回 | void 允许实现返回任意值；undefined 严格 |
| `?.` vs `if` 判空 | `?.` 管访问安全，`if` 管显示逻辑 |
| never | 不接受任何值；穷尽检查 + 交叉冲突坍缩两个场景都靠它 |

## 学习记录

| 模块 | 节点 | 通过时间 |
|------|------|---------|
| 模块1 | 1.1 ~ 1.5 | 2026-08-06 ~ 08-07 |
| 模块2 | 2.1 ~ 2.3 | 2026-08-08 |
| 模块3 | 3.1 ~ 3.4 | 2026-08-10（3.4 收官零错误一次通过 🎉） |
