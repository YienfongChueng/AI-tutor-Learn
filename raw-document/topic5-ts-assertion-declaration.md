# 1.5 类型断言 & 类型声明

## 讲解原文

### 引入：当 TS 推断的类型，和你"知道"的真实类型对不上

你已经会用类型注解和 interface/type 描述形状了。但实战中常遇到这种情况：

```ts
const el = document.getElementById('app')  // TS 推断类型是 HTMLElement | null
el.innerHTML = 'hello'  // ❌ 报错：el 可能是 null
```

你**知道**页面上一定有 `#app` 这个元素，`el` 不可能是 `null`。但 TS 不知道--它只能根据 `getElementById` 的签名保守推断成 `HTMLElement | null`。

这就是类型断言要解决的问题：**你比 TS 更清楚真实类型，告诉它"相信我"**。

---

### 一、类型断言（Type Assertion）-- "我说它是什么类型"

#### 两种写法

```ts
// 写法 1：as 语法（推荐，JSX 中唯一可用）
const el = document.getElementById('app') as HTMLDivElement

// 写法 2：尖括号语法（不推荐，和 JSX 冲突）
const el2 = <HTMLDivElement>document.getElementById('app')
```

> 在 Vue3 的 `.vue` 文件里（模板是类 JSX 语法），**必须用 `as`**，尖括号写法会报错。所以养成习惯：永远用 `as`。

#### 断言的本质：只改类型，不改运行时值

**关键认知**：类型断言**不执行任何运行时转换**，它只改 TS 的静态类型标注，编译后消失。

```ts
const value: any = 'hello'
const num = value as number   // TS 类型变成 number
console.log(num.toFixed())   // 编译能过，但运行时炸：num 其实是字符串，没有 toFixed
```

> 断言是给编译器看的"信任声明"，不是给 JS 引擎看的转换。`value` 在运行时还是字符串 `"'hello'"`，调 `.toFixed()` 直接崩。**断言不等于安全。**

#### 断言的约束：不是你想断就能断

TS 不允许随意断言，必须满足**类型有重叠**：

```ts
const s: string = 'hi'
const n = s as number   // ❌ 报错：string 和 number 毫无重叠
const a = s as unknown as number  // ✅ 绕过：先转 unknown 再断言（危险操作）
```

- 合法断言：原类型和目标类型**有交集**（如 `HTMLElement | null` → `HTMLDivElement`，HTMLDivElement 是 HTMLElement 的子类型）
- 非法断言：两者完全不相关（如 `string` → `number`）
- `as unknown as X`：**强制断言**，相当于先扔进 `unknown`（所有类型的父类型），再断言成任意类型，能绕过所有检查，但等于关掉了类型保护

---

### 二、type vs interface 场景：`as` 与断言

回顾上一节，断言时用的都是**具体类型名**（`HTMLDivElement`、`number`）。联合类型、字面量类型也能断言：

```ts
const input = document.querySelector('input') as HTMLInputElement
const status = data.status as 'open' | 'closed'   // 断言成字面量联合
```

---

### 三、类型声明（Type Declaration）-- `.d.ts` 告诉 TS "这个模块存在"

#### 痛点

当你在 TS 项目里 import 一个没有类型定义的第三方库（老库、自写 JS 脚本）：

```ts
import oldLib from 'old-lib'   // ❌ Could not find declaration file for 'old-lib'
oldLib.doSomething()
```

TS 报错，因为它不知道 `oldLib` 是什么形状。

#### 解法：写 `.d.ts` 类型声明文件

创建 `old-lib.d.ts`：

```ts
declare module 'old-lib' {
  export function doSomething(x: string): number
  export default interface OldLib {
    version: string
  }
}
```

这样 TS 就知道 `oldLib` 的形状了。

#### `declare` 的语义

`declare` 关键字告诉 TS：**"这个变量/模块在运行时已经存在，我只是告诉你它的类型，不要生成任何 JS 代码。"**

```ts
declare const VERSION: string          // 全局常量
declare function g(tag: string): void // 全局函数
declare module 'lib-name' { ... }      // 整个模块
```

> `declare` 出现在编译产物里会**完全消失**，因为它纯粹是类型信息，不产生运行时代码。这和断言一样是"编译期产物"。

#### 全局变量声明（常见于 Vite/Webpack 注入的变量）

```ts
// vite-env.d.ts 里常见
declare const __DEV__: boolean        // 构建工具注入的全局变量
declare interface Window {
  myCustomProp: string                // 扩展 window（这里用 interface 合并！）
}
```

> 注意这里 `declare interface Window` 用到了上一节讲的**声明合并**--同名 interface 自动合并字段，所以能给全局 `Window` 安全补属性。

---

### 四、断言 vs 类型守卫：什么时候该用哪个

| 场景 | 用断言 | 用类型守卫（下节讲） |
|------|:------:|:---:|
| 你100%确定真实类型，且无法通过逻辑判断 | ✅ | ❌ |
| 类型可以通过 if/typeof 判断出来 | ❌（偷懒） | ✅ |
| 运行时数据来源不可信（API 返回、用户输入） | ❌ 危险 | ✅ 必须校验 |

> **经验法则**：断言是"我保证"，类型守卫是"我验证"。能用守卫就别用断言。下节 2.2 会专门讲类型守卫。

---

### 五、`.d.ts` 文件的两类常见用法

| 类型 | 例子 | 作用 |
|------|------|------|
| 模块声明 | `declare module 'lib' {...}` | 给无类型的第三方库补类型 |
| 全局声明 | `declare const X` / `declare global {...}` | 声明全局变量、扩展内置对象 |

#### 一个实战模板（Vue3 项目常见）

```ts
// env.d.ts
declare const __APP_VERSION__: string

declare module '*.vue' {
  import type { DefineComponent } from 'vue'
  const component: DefineComponent<{}, {}, any>
  export default component
}
```

---

### 六、本节核心认知（面试常问）

1. **断言只改类型不改值** -- 编译后消失，运行时数据原样不动，所以断言不保证安全
2. **断言有约束** -- 必须类型有重叠，`as unknown as` 是强制绕过（危险）
3. **`declare` 不产生代码** -- 纯类型信息，给运行时已存在的东西补类型说明
4. **`.d.ts` 给无类型模块/全局变量补类型** -- 是 TS 与 JS 生态的"桥接层"
5. **能用类型守卫就别用断言** -- 断言是"信任"，守卫是"验证"

---

## 考核过程

📝 考核题：修复后台项目的 3 个类型问题

你的 Vue3 后台项目里有 3 段代码各自报错，请分别修复。对每处：写出修复代码 + 一句话说明用了本节哪个概念。

---
问题 1：获取 DOM 输入框，要读它的 .value

```ts
const input = document.querySelector('.search-input')
console.log(input.value)   // ❌ 报错：input 类型是 Element | null，没有 value 属性
（提示：你确定页面上 .search-input 是个 <input> 元素）

const input = document.querySelector('.search-input') as HTMLInputElement
//自己清楚知道类型，用类型断言
```

---
问题 2：接口返回的数据，你知道 status 字段一定是 'success' | 'fail'，但 TS 把它推断成了 string

```ts
interface ApiResponse {
  status: string    // 后端文档说是 'success' | 'fail'，但这里写宽了
  data: any
}
const res: ApiResponse = await fetch('/api/list').then(r => r.json())
if (res.status === 'success') {    // ⚠️ 想要精确比较，但 status 是 string 太宽
  console.log(res.data)
}
```
要求：用本节学的方式，让 res.status 的类型变成精确的 'success' | 'fail'。（注意有不止一种改法，写出你选的那种并说明理由）

```ts
//写法1
interface ApiResponse {
  status: 'success' | 'fail'    // 联合字面量类型限制status值
  data: any
}
// 写法2
type Status = 'status' | 'fail'
if ((res.status as Status) === 'success') {    断言
  console.log(res.data)
}
```
---
问题 3（关键题）：项目 import 了一个老库 legacy-auth，它没有类型定义，TS 报错找不到声明：

import auth from 'legacy-auth'
auth.login('token')   // ❌ Could not find declaration file for module 'legacy-auth'
这个库的形状是：默认导出一个对象，有 login(token: string): boolean 方法。

要求：写出修复方案（创建什么文件、写什么内容），并说明 declare 关键字在这里的作用 -- 它会产生运行时代码吗？
创建legacy-auth.d.ts文件
内容：
```ts
declare module 'legacy-auth' {
  const auth: {login(token:string): boolean}
  export default auth
}
// declare是编译时产物，不会产生运行时代码
```

### LLM 点评

**评估结果：通过（2 轮，细节错误已纠正）**

三题核心认知全部到位：
- 问题 2 一轮通过：直接修正源接口为字面量联合是最佳实践（从源头精确化优于断言）
- 问题 3 关键认知一把答对：「declare 是编译时产物，不产生运行时代码」-- 本节最核心的点

**细节错误（已纠正）**：
1. 问题 1 初答 `as Element` -- 断言目标选错。`Element` 仍无 `.value` 属性，须断言成具体的 `HTMLInputElement`。要断言到「真正有目标属性的那个具体子类型」，而非任意去掉 null 的父类型。
2. 问题 3 `.d.ts` 语法多次迭代：初答把默认导出对象写成 `export function`（命名导出）+ 空 interface；后写成 `export default auth { login: login(...) }`（混用方法简写与属性式语法）；再写成 `export auth`（非默认导出）。最终 `const auth: {...}; export default auth` 正确。

**关键认知沉淀**：
- 断言目标必须选「带目标属性的具体子类型」，而非任意去掉 null 的父类型
- 对象类型里方法有两种合法写法：方法简写 `m(x): T` 或属性式 `m: (x) => T`，不可混用
- `.d.ts` 默认导出对象须分两步：先 `const x: {...}` 声明形状，再 `export default x`

---
通过时间：2026-08-07




---
