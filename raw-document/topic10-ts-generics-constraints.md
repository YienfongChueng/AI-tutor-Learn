# 3.2 泛型约束（extends）

## 讲解原文

### 它解决什么问题

上一节的泛型函数里，T 是「任意类型」--但你没法对 T 做任何有意义的操作：

```ts
function getLength<T>(x: T): number {
  return x.length   // ❌ 报错：Property 'length' does not exist on type 'T'
}
```

编译器只知道 T 是「某个类型」，不知道它有没有 `.length`。直接用会报错，强行断言（`x as any`）又不安全。

泛型约束解决这个：**给 T 设一个「下限」--T 至少要满足某个形状**，这样函数体里就能安全访问这个形状的成员。

### 核心机制：extends 约束

`<T extends Shape>` 意思是「T 必须满足 Shape 的形状」。约束后，函数体里可以安全访问 Shape 的成员：

```ts
interface HasLength {
  length: number
}

function getLength<T extends HasLength>(x: T): number {
  return x.length   // ✅ 安全，约束保证了 T 一定有 length
}

getLength('abc')        // ✅ string 有 length
getLength([1, 2, 3])    // ✅ array 有 length
getLength(123)          // ❌ 报错：number 不满足 HasLength
```

关键认知：**extends 在这里是「约束」不是「继承」**。读作「T 至少是 HasLength 形状」，不是「T 继承 HasLength」。

### 约束保留具体类型

约束不擦除 T 的具体类型。传入什么，T 还是什么，只是多了一层保证：

```ts
function getLength<T extends HasLength>(x: T): T {
  console.log(x.length)
  return x   // 返回 T，不是 HasLength
}
const s = getLength('abc')   // s: string，不是 HasLength
```

对比：若把参数写成 `x: HasLength`（不用泛型），返回值就是 `HasLength`，丢掉了 string 的具体类型。**约束让 T 既有最低保证，又保留具体身份**--这是它相比直接用基类型的最大价值。

### keyof 约束：安全访问对象属性

经典场景：取对象某个属性的值。要保证 key 真的存在：

```ts
function getValue<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

const user = { name: '张三', age: 18 }
getValue(user, 'name')   // ✅ 返回 string
getValue(user, 'age')    // ✅ 返回 number
getValue(user, 'email')  // ❌ 报错：'email' 不是 user 的 key
```

- `K extends keyof T`：K 被约束为「T 的所有 key 之一」，传错 key 直接编译报错。
- 返回类型 `T[K]`：精确到那个 key 对应的值的类型（不是 any，也不是宽联合）。

这正是你写 Vue 时「按字段名安全取值」的类型层武器。

### 多约束：交叉 `&`

用 `&` 叠加多个约束：

```ts
function logBoth<T extends { id: number } & { name: string }>(x: T) {
  console.log(x.id, x.name)
}
```
T 必须同时有 `id: number` 和 `name: string`。

### 默认类型参数（顺带）

类型参数可以给默认值，调用方不传时用它：

```ts
interface Paginated<T = any> {
  data: T[]
  total: number
}
const p: Paginated = { data: [1, 2], total: 2 }   // T 默认 any
```

### 三种写法对比

| 写法 | 函数体能否用 `.length` | 保留具体类型 |
|------|----------------------|-------------|
| `x: any` | 能（但不安全，运行时可能炸） | 否（any） |
| `x: T`（无约束） | 不能（T 上没有 length） | 是 |
| `x: T extends HasLength` | 能（安全，约束保证） | 是 |

约束 = 在「保留具体类型」和「能用特定成员」之间取得平衡。

### 一句话类比

无约束泛型像「收一个完全未知的包裹」--你不敢拆，因为不知道里面有没有你要的东西；`extends` 约束像「要求包裹至少贴了易碎标签」--保证你拆箱时能安全地按易碎品处理，但包裹本身是什么还是什么。

### 本节边界
- ✅ 本节讲：extends 约束、约束保留具体类型、keyof 约束、多约束、默认类型参数
- ❌ 不讲（留给后续）：条件类型 `infer`（3.3）、内置工具类型（3.4）

## 考核过程

📝 考核题 3.2

  现状：想写一个「从对象取指定属性值」的函数，但这个 any 版本不安全：

  function getValue(obj: any, key: string): any {
    return obj[key]
  }
  getValue({ name: '张三' }, 'name')    // ok
  getValue({ name: '张三' }, 'emial')   // 拼错也不报错，运行时返回 undefined

  要求：

  ① 用泛型 + keyof 约束重写 getValue，让传错的 key（如 'emial'）在编译期就报错，且返回值类型精确对应那个 key 的值类型（传
  'name' 返回 string，传 'age' 返回 number）。

  ② 如果你还想额外限制：obj 必须是「有 id: number 字段的对象」。你的函数签名该怎么加这个约束？（写出完整签名即可，不用实现）

  ③ 一句话：extends 约束解决了「无约束泛型 <T>」的什么问题？
  ```ts

  function getValue<T extends object & { id: number },K extends keyof T>(obj: T,key: K): T[K] {
    return obj[key]
  }

  //解决了函数体能安全使用约束形状的成员
  ```

**点评**：
通过（接近正确，attempts=2）。①②代码完全正确：`K extends keyof T` + 返回 `T[K]` 精确到值类型，`& { id: number }` 正确加 id 约束。小提醒：基础 ① 版应是 `<T extends object, K extends keyof T>`（不要求 id）；`object &` 略冗余，`T extends { id: number }` 即可。
③初次答「解决了保留具体类型」--这是 extends 相比*基类型*的价值，而非相比*无约束泛型*。无约束泛型本就保留具体类型，extends 相比它新增的是「让函数体能安全使用约束形状的成员」。微调后答对，通过。

---
通过时间：2026-08-10