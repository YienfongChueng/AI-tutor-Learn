# 3.3 条件类型 & infer

## 讲解原文

### 它解决什么问题

泛型约束能要求 T「至少有某形状」，但有时你想**根据 T 的具体形态决定类型**--比如「如果 T 是数组，取出元素类型；如果是 Promise，取出内部类型」。这是类型层面的「if-else」。

条件类型 + infer 就是这套机制：**在类型层面做判断，并从类型里「提取」出一部分**。3.4 要学的内置工具类型几乎全靠它俩构建。

### 条件类型：类型层面的三元

`T extends U ? X : Y` 读作「如果 T 可赋值给 U，结果为 X，否则 Y」：

```ts
type IsString<T> = T extends string ? true : false

type A = IsString<'hi'>            // true
type B = IsString<42>              // false
type C = IsString<string | number> // boolean  ← 见「分布性」
```

### 分布性（Distributive）：对联合自动拆开

当 T 是**裸联合**（直接是 `string | number` 这种）时，条件类型会把联合**拆开逐个判断再合并**：

```ts
type IsString<string | number>
  = IsString<string> | IsString<number>
  = true | false
  = boolean
```

这不是 bug 是特性。想关掉分布性：用方括号包住 `[T] extends [U] ? ...`，T 被包进元组不再是「裸」的：

```ts
type IsStringNonDist<T> = [T] extends [string] ? true : false
type D = IsStringNonDist<string | number>  // false（整体判断，不再拆开）
```

### infer：从类型里「提取」

`infer R` 在 extends 子句里声明一个待提取的类型变量，TS 会在匹配成功时把对应部分绑给 R：

```ts
// 提取数组的元素类型
type ElementOf<T> = T extends (infer E)[] ? E : never
type R1 = ElementOf<number[]>   // number
type R2 = ElementOf<string[]>   // string
type R3 = ElementOf<number>     // never（不是数组，走 else）

// 解包 Promise
type Unwrap<T> = T extends Promise<infer R> ? R : T
type R4 = Unwrap<Promise<string>>  // string
type R5 = Unwrap<number>           // number（不是 Promise，原样返回）
```

读法：`T extends Promise<infer R>` 意思是「如果 T 形如 `Promise<某类型>`，把那个某类型叫作 R」。

### 实战：自己造一个 ReturnType

内置 `ReturnType<T>` 拿到函数的返回类型，底层就是条件类型 + infer：

```ts
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never

type F = () => { id: number }
type Ret = MyReturnType<F>   // { id: number }
```

`(...args: any[]) => infer R`：匹配「任意参数的函数」，把返回类型绑给 R。这就是 3.4 内置工具类型的底层原理。

### 条件类型可嵌套

```ts
type TypeName<T> =
  T extends string ? 'string' :
  T extends number ? 'number' :
  T extends boolean ? 'boolean' :
  T extends undefined ? 'undefined' :
  'other'

type N = TypeName<42>   // 'number'
```

### 一句话类比

条件类型像类型层面的「问诊台」--「你这个 T 像不像数组？像我就把你的元素类型抽出来（infer）」；不像就走 else。分布性像「群体体检」，把联合里每个人分开问诊再合并结果。

### 本节边界
- ✅ 本节讲：条件类型三元、分布性、infer 提取、嵌套、自造 ReturnType
- ❌ 不讲（留给后续）：内置工具类型全家族（3.4）

## 考核过程

📝 考核题 3.3

  ① 写一个条件类型 Unwrap<T>：若 T 是 Promise<某类型>，返回那个内部类型；否则原样返回 T。

  ② 给出下面两个的求值结果（写出推导）：
  type A = Unwrap<Promise<number[]>>   // ?
  type B = Unwrap<string>              // ?

  ③ 下面得到什么类型？为什么？（这题考分布性，想清楚再答）
  type ToArray<T> = T extends any ? T[] : never
  type R = ToArray<string | number>   // ?

```ts
type Unwrap<T> = T extends Promise<infer R> ? R : T
type A = Unwrap<Promise<number[]>> //number[] ，是promise，把返回类型绑给R
type B = Unwrap<string> //string，不是promise，原样返回 T类型

//得到数组类型，代码拆分解析如下：
type R = ToArray<string> | ToArray<number>
       = string[] | number[]
       = [] 
```
**点评**：
通过（接近正确，attempts=1）。①Unwrap 完全正确（infer 提取 Promise 内部类型）；②A=number[]、B=string 正确；③分布性推导对（string[]|number[]），但最后误坍缩成 []。string[]|number[] 是联合（满足其一即可），不等于 []；[] 只是同时满足两者的特例。联合点属 2.1，明日复习重测。

---
通过时间：2026-08-10
