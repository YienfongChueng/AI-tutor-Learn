# 3.1 泛型基础（函数/接口/类）

## 讲解原文

### 它解决什么问题

先看一段「能用但不安全」的代码：

```ts
function identity(value: any) {
  return value
}
const n = identity(42)      // n: any —— 类型信息丢了
n.toUpperCase()             // 不报错，但运行时炸：number 没有 toUpperCase
```

`any` 让函数能装任何类型，代价是**交还了类型保护**：返回值是 `any`，编译器不再帮你查错，错误退回运行时。

泛型解决的就是这个矛盾：**写一份逻辑，但保留「入参类型 -> 返回类型」的精确关系**。它给类型留一个「占位符」`T`，调用时再由实参决定 `T` 具体是什么。

### 核心机制：类型参数化

把类型也当成「参数」来传。`<T>` 是类型参数列表，就像 `()` 是值参数列表：

```
值层面:   函数(值参数)      ->  identity(42)            传「值」
类型层面: 函数<T>(值参数)   ->  identity<number>(42)    传「类型」
```

```ts
function identity<T>(value: T): T {
  return value
}

const n = identity<number>(42)   // 显式指定 T = number -> n: number
const s = identity('hi')         // 推断 T = string     -> s: string
n.toUpperCase()                  // ✅ 现在会报错：number 没有 toUpperCase
```

关键认知：**`T` 不是「任意类型」，而是「调用那一刻被锁定的某个具体类型」**。同一次调用里 `T` 是同一个类型，这保证了入参和返回的类型关系不被擦除——这正是 `any` 做不到的。

### 三种声明形式

同一个机制，落在三种语法载体上：

**① 泛型函数**

```ts
function first<T>(arr: T[]): T {
  return arr[0]
}
first<number>([1, 2, 3])   // number
first(['a', 'b'])          // 推断 string
```

**② 泛型接口** —— 描述「形状里某个字段的类型待定」

```ts
interface Box<T> {
  value: T
}
const numBox: Box<number> = { value: 42 }
const strBox: Box<string> = { value: 'hi' }
```

**③ 泛型类** —— 类的实例字段和方法都能用这个类型参数

```ts
class Stack<T> {
  private items: T[] = []
  push(x: T) { this.items.push(x) }
  pop(): T | undefined { return this.items.pop() }
}
const s = new Stack<number>()
s.push(1)
s.push('x')   // ❌ 报错：string 不能赋给 number
```

### 类型推断 vs 显式指定

绝大多数时候**不用手写**类型参数，编译器从实参推断：

```ts
function wrap<T>(x: T): T { return x }
wrap(42)        // 推断 T = number
wrap('hi')      // 推断 T = string
```

什么时候需要显式写 `<number>`？
- 推断不够精确时（例如实参是字面量，推断成字面量类型，而你要更宽的类型）
- 调用方想主动强制约束时

### 多个类型参数 & 命名约定

可以有多个类型参数 `<T, U>`，约定俗成的命名：

| 名字 | 含义 |
|------|------|
| `T` (Type) | 第一个泛型类型 |
| `U`、`V` | 第二、第三个 |
| `K` (Key) | 对象键 |
| `V` (Value) | 对象值 |
| `E` (Element) | 集合元素 |

```ts
function pair<K, V>(k: K, v: V): [K, V] {
  return [k, v]
}
pair('age', 18)   // [string, number]
```

### 实战：泛型 API 响应包装

前端最常见场景——统一接口返回结构：

```ts
interface ApiResponse<T> {
  code: number
  message: string
  data: T
}

// 用户接口返回
const userRes: ApiResponse<{ id: number; name: string }> = {
  code: 200, message: 'ok',
  data: { id: 1, name: '张三' }
}
userRes.data.name   // ✅ 有类型提示，data 不是 any

// 列表接口返回
const listRes: ApiResponse<string[]> = {
  code: 200, message: 'ok',
  data: ['a', 'b']
}
```

一个 `ApiResponse<T>` 框住结构，`T` 让 `data` 的类型随接口变化——**复用结构，不丢类型**。这正是你写 Vue 项目封装 axios 时 `request<T>(...)` 返回 `Promise<T>` 的底层原理。

### any vs 泛型 对比

| 维度 | `any` | 泛型 `<T>` |
|------|-------|-----------|
| 类型关系 | 全部擦除 | 保留入参->返回对应 |
| 编译期检查 | 放弃 | 保留 |
| 复用性 | 能复用（但不安全） | 能复用且安全 |
| 适用 | 临时绕过、迁移期 | 正经写可复用逻辑 |

### 一句话类比

泛型像「带占位符的表单模板」：模板上印着「___ 处填什么类型由你交表时决定」，复印成「数字版」「字符串版」各有各的类型，但模板只写了一份。`any` 像空白菜单——什么都能点，但厨房不检查你点的菜存不存在。

### 本节边界

- ✅ 本节讲：泛型是什么、三种形式（函数/接口/类）、类型推断、命名约定
- ❌ 不讲（留给后续）：`extends` 约束（3.2）、条件类型 `infer`（3.3）、内置工具类型（3.4）

## 考核过程

📝 考核题 3.1

  现状：用 any 实现了一个「取数组第一个元素并包装成响应」的函数，调用方拿到的 data 是 any，类型信息丢了：

  function firstResponse(arr: any[]): { success: boolean; data: any } {
    return { success: true, data: arr[0] }
  }
  const r = firstResponse([1, 2, 3])
  r.data  // any，没有提示，也查不出错

  要求：

  ① 用泛型重写 firstResponse：传 number[] 时返回的 data 必须是 number（类型精确对应，不能是 any）。

  ② 定义一个泛型接口 FirstResponse<T> 描述返回结构（不要用内联对象类型），让 ① 的函数返回值用它标注。

  ③ 写一行调用 firstResponse(['a', 'b'])，说明此时 data 是什么类型；并解释：相比 any 版本，泛型版本为什么能保证类型安全？

  ```ts
  interface FirstResponse<T> {
    success: boolean;
    data: T;
  }
  function firstResponse<T>(arr: T[]): FirstResponse<T> {
    return {
      success: true;
      data: arr[0]
    }
  }
  //firstResponse(['a', 'b']),传入的是string[]，所以data是string类型;用类型占位符定义了函数或者接口的入参和出参类型，具体参数类型由调用放决定；而any，是任何类型都可以，编译会绕过检测，不做类型检测
  ```

**点评**：
通过（接近正确，attempts=1）。泛型核心逻辑完全正确：`firstResponse<T>(arr: T[]): FirstResponse<T>` 精确保留「数组元素 T -> data: T」的对应关系；接口定义正确；调用推断正确。
一个细节错误：返回的对象字面量用 `;` 分隔属性（`success: true;`），对象字面量必须用 `,`；interface 内 `;` 合法，对象字面量（值）只能用 `,`。
③解释方向对，可更精确：T 在一次调用里被锁定为同一类型，`arr: T[]` 与 `data: T` 共享此 T，故 data 类型被保证与数组元素类型一致；any 无此联动，arr 与 data 各自独立为 any。

---
通过时间：2026-08-10
