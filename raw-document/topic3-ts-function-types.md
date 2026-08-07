# 1.3 函数类型（参数/返回值/重载）

## 讲解原文

### 一、参数类型 & 返回值类型

JS 函数不拦类型，运行时才暴露问题：

```js
function add(a, b) { return a + b }
add(1, 2)      // 3
add(1, '2')    // '12'  ← JS 不拦，静默变成字符串拼接
```

TS 给参数和返回值都加注解：

```ts
function add(a: number, b: number): number {
  return a + b
}
add(1, 2)      // ✅
add(1, '2')    // ❌ 编译报错：'2' 不是 number
add(1)         // ❌ 报错：缺少参数
```

- 参数注解：`参数名: 类型`
- 返回值注解：在参数括号后 `): 类型`
- 返回值类型通常可推断、可省略；但**对外公开的 API 建议显式标注**，避免实现悄悄改了返回类型却没人发现。

### 二、函数作为类型（函数类型字面量）

当你想把「一个函数」当作变量的类型时（回调、事件处理器、formatter），用箭头形式的类型字面量：

```ts
type Formatter = (value: number) => string

const toMoney: Formatter = (n) => '$' + n.toFixed(2)
//        ↑ 类型是 Formatter       ↑ 参数 n 自动推断为 number
```

> 类比 Vue：`(value: number) => string` 就像你给 props 里的 `validator` 或表格列的 `formatter` 声明签名——调用方必须按这个形状传函数。

**箭头类型里的参数名只是占位，不必和实现里的名字一致：**

```ts
type Formatter = (value: number) => string
const f: Formatter = (x) => String(x)  // x 推断为 number，名字不必叫 value
```

### 三、可选参数、默认参数、剩余参数

| 特性 | 语法 | 说明 |
|------|------|------|
| 可选参数 | `name?: type` | 可不传；**必须在必填参数后面** |
| 默认参数 | `name: type = 默认值` | 不传时用默认值；带默认值后该参数可选 |
| 剩余参数 | `...args: type[]` | 收集剩余实参为数组 |

```ts
function greet(name: string, greeting?: string): string {
  return `${greeting || 'Hi'}, ${name}`
}
greet('Tom')              // ✅ "Hi, Tom"
greet('Tom', 'Hello')     // ✅ "Hello, Tom"

function log(msg: string, level: string = 'info') { /* ... */ }
log('done')               // level 默认 'info'

function sum(...nums: number[]): number {
  return nums.reduce((a, b) => a + b, 0)
}
sum(1, 2, 3)              // 6
```

**高频坑：可选参数的位置。**

```ts
function f(a?: number, b: number) { }  // ❌ 报错：必填参数不能在可选参数后面
```

TS 按位置匹配参数，可选参数后面再放必填会导致「无法不传 a 却传 b」。规则：**可选参数必须在必填参数之后**。

### 四、void 返回类型

表示「函数不返回有意义的值」（只执行副作用）：

```ts
function log(msg: string): void { console.log(msg) }
```

- `void` 表示调用方**不应该使用返回值**。
- 常用于回调签名（如 `Array.forEach` 的回调、事件处理）。
- `void` vs `undefined`：`void` 允许实现返回任意值（调用方不用），`undefined` 严格要求返回 undefined。日常用 `void`。

### 五、函数重载（Function Overloading）

**问题场景：** 一个函数根据入参类型返回**不同类型**的结果，用单一签名无法精确表达：

```ts
// 期望：传 'list' 返回 string[]，传 'count' 返回 number
function query(arg: string): ???  // 单一签名说不清
```

**TS 的解法：多个重载签名 + 一个实现签名。**

```ts
// 重载签名（对外暴露的精确类型，可多个）
function query(type: 'list'): string[]
function query(type: 'count'): number

// 实现签名（对外不可见，内部用，必须兼容所有重载）
function query(type: string): string[] | number {
  if (type === 'list') return ['a', 'b']
  return 0
}

const r1 = query('list')   // string[]
const r2 = query('count')  // number
```

**三条铁律：**

1. **重载签名在外，实现签名在内**，实现签名对外不可见（调用方看不到）。
2. 实现签名的参数/返回类型必须**兼容**所有重载（一般是它们的联合）。
3. 调用方看到的是重载签名，不是实现签名——所以能精确推断返回类型。

> 注意：TS 的重载不是真的「同名多份实现」（像 Java/C++ 那样），而是**一套实现 + 多套对外类型声明**。运行时只有一个函数。

**为什么不直接写联合返回？**

```ts
function query(type: string): string[] | number { /* ... */ }
const r = query('list')  // 类型是 string[] | number，用之前还得收窄
```

联合返回会把所有可能的类型「揉」在一起，调用方拿到 `string[] | number` 后还得自己判断类型，**丢掉了「这个入参对应那个返回」的精确对应关系**。重载就是为了保留这种对应。

### 六、一句话总结

- 参数 `name: type`，返回值 `): type`；返回值多可省略，公开 API 建议显式标。
- 函数作为类型用字面量 `(a: T) => U`；箭头里参数名是占位。
- 可选 `?` / 默认 `= 值` / 剩余 `...args`；**可选参数必须在必填参数之后**。
- `void` 表示「别用返回值」；回调签名常用。
- 重载 = 多个对外签名 + 一个兼容实现，运行时只有一个函数；价值是保留「入参→返回」的精确对应。

## 考核过程

考核题：给查询函数设计重载

你在写后台管理系统的数据看板，有一个查询函数 query，需求如下：
- 传 type: 'list'：返回订单列表（string[]，订单号数组）
- 传 type: 'count'：返回订单总数（number）

调用方期望拿到精确类型，不要被联合类型 string[] | number 糊弄。
任务
1. 用函数重载为 query 编写完整类型声明（含重载签名 + 实现签名）
2. 在实现签名里用 if 分支返回不同的值（'list' 返回 ['#1001', '#1002']，'count' 返回 2）
3. 实现签名的返回类型必须是所有重载返回类型的联合（验证你理解了兼容规则）
额外思考（写完代码后回答）
```ts
function query(type: 'list'):string[]
function query(type: 'count'):number
function query(type:string): string[] | number {
  if(type === 'list') return ['#1001', '#1002']
  return 2
}
```

4. 有一位新人同事想偷懒，把你的重载改成了下面这样「单一签名」版本。请判断：这种写法有什么具体坏处？调用方拿到返回值后会遇到什么麻烦？

```ts
function query(type: string): string[] | number {
  if (type === 'list') return ['#1001', '#1002']
  return 2
}
const result = query('list')
// result 此时的类型是什么？result.length 能直接访问吗？为什么？
// result 此时的类型是 string[]和number, result.length不能直接访问，需要进行类型收窄，判断结果的类型。
```

5. 下面这个重载声明哪里违反了「三条铁律」？怎么改？

```ts
function query(type: 'list'): string[]
function query(type: 'count'): number
function query(type: 'list' | 'count'): string[] {
  if (type === 'list') return ['#1001', '#1002']
  return 2
}
// 实现签名的返回值类型需要兼容所有重载
function query(type: 'list' | 'count'): string[] | number {//...}
```
把任务 1-3 的完整代码 + 第 4、5 题的判断一起发给我。重点看你对「重载为什么存在」和「实现签名兼容规则」的理解。

### LLM 点评

第 4、5 题完全正确。但任务 1-3 的首版重载有致命概念错误。

**考核迭代过程（2 轮，1 个概念性错误）：**
1. 首轮：两个重载签名参数都写成宽类型 `string`，而 `'list'`/`'count'` 都是 string，TS 按顺序匹配时永远命中第一个签名，`query('count')` 实际返回 `string[]` 而非 `number`，重载坍缩失效。错误类型：概念性错误（重载签名必须用「能区分的参数类型」当分派依据）。
2. 修正版：两个重载签名改用字面量 `'list'`/`'count'`，实现签名把第二个 if 改为兜底 `return 2`（消除末尾隐式 undefined，与 `string[] | number` 兼容）。通过。

**核心认知突破（必须牢记）：**
重载签名像菜单点菜，菜名（参数类型）必须不同服务员才分得清；两道菜都叫"炒饭"（都是 string），只能端出第一道。**字面量联合/字面量参数类型是重载能「按入参精确分派返回」的基石**（关联 1.1 字面量类型）。

---
通过时间：2026-08-07

