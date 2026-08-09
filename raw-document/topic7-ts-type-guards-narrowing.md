# 2.2 类型守卫 & 类型收窄

## 讲解原文

### 一、什么是类型收窄（Narrowing）

TS 编译器会沿着控制流，根据条件判断不断"收窄"一个变量的类型，从宽类型推导出更精确的子类型。

```typescript
function padLeft(value: string, padding: string | number) {
  // 这里 padding 是 string | number
  if (typeof padding === 'number') {
    // 这里 padding 收窄为 number
    return ' '.repeat(padding) + value;
  }
  // 这里 padding 收窄为 string
  return padding + value;
}
```

这正好解决 2.1 的遗留问题：联合类型收窄前只能访问公共成员，**收窄后才能访问特有成员**。"类型守卫"就是触发收窄的那个判断条件。

### 二、主要的收窄手段

#### 1. typeof（原始类型）

判断原始类型，`typeof` 返回 8 种之一：`string | number | boolean | bigint | symbol | undefined | object | function`。

```typescript
function fn(x: string | number) {
  if (typeof x === 'string') { x.length }   // x: string
  else { x.toFixed() }                       // x: number
}
```

⚠️ 坑：`typeof null === 'object'`（历史遗留 bug）。所以用 `typeof` 判断"是不是对象"不可靠，`null` 会被当成 object。

#### 2. in（区分对象结构，最配联合）

检查属性是否存在，常用于区分结构不同的对象类型：

```typescript
interface Bird { fly(): void; }
interface Fish { swim(): void; }

function move(pet: Bird | Fish) {
  if ('swim' in pet) { pet.swim() }   // pet: Fish
  else { pet.fly() }                   // pet: Bird
}
```

这直接解决了 2.1 里 `operate` 函数无法调用 `fly`/`swim` 的问题。

#### 3. instanceof（类实例）

```typescript
function log(x: Error | Date) {
  if (x instanceof Date) { x.toISOString() }   // x: Date
  else { x.message }                            // x: Error
}
```

注意：`instanceof` 只对 `class` 构造的实例有效，对 `interface`/纯对象字面量无效（运行时类型信息已被擦除，没有构造函数痕迹可查）。

#### 4. 相等性收窄 & switch 字面量

对字面量联合（2.1 讲过的字面量联合）特别好用：

```typescript
type Status = 'active' | 'inactive';
function handle(s: Status) {
  switch (s) {
    case 'active':   // s: 'active'
    case 'inactive': // s: 'inactive'
  }
  if (s === 'active') { /* s: 'active' */ }
}
```

#### 5. 自定义类型谓词 `x is Type`

当内置手段不够用时，可以写一个返回**类型谓词**的函数，让 TS 信任你的运行时判断：

```typescript
function isFish(pet: Bird | Fish): pet is Fish {
  return (pet as Fish).swim !== undefined;
}

function move(pet: Bird | Fish) {
  if (isFish(pet)) { pet.swim() }   // pet: Fish ✅
  else { pet.fly() }                 // pet: Bird ✅
}
```

`pet is Fish` 是关键：它告诉编译器"这个函数返回 `true` 时，参数 `pet` 就是 `Fish`"。许多工具库（如 lodash 的 `isString`）大量用这个机制。

### 三、判别联合（Discriminated Union）—— 最推荐的模式

当多个对象类型共享一个**同名字面量字段**（判别字段 / discriminant）时，用相等性收窄最稳妥：

```typescript
interface Success { ok: true;  data: string; }
interface Failure { ok: false; error: string; }
type Result = Success | Failure;

function handle(r: Result) {
  if (r.ok === true) { r.data }    // r: Success
  else { r.error }                  // r: Failure
}
```

`ok` 这个 `true | false` 字面量字段就是判别字段。它比 `in 'data'` 更稳，因为判别字段是**专门为区分而设计**的，业务字段（如 data）未来可能被两个类型共有，导致 `in` 失效，而判别字段不会。

### 四、穷尽检查（Exhaustiveness）+ never

把 switch 的 `default` 分支赋给 `never` 类型变量，能强制覆盖所有可能；新增字面量却忘了处理时，编译器会报错：

```typescript
type Status = 'active' | 'inactive' | 'pending';

function handle(s: Status) {
  switch (s) {
    case 'active':   return 1
    case 'inactive': return 0
    case 'pending':  return -1
    default:
      const x: never = s  // 若漏掉任一 case，s 会有具体类型，赋给 never 报错
      return x
  }
}
```

新增 `'pending'` 时若忘加 case，`default` 里 `s` 是 `'pending'`，赋给 `never` 立刻报错。这是类型驱动开发的安全网。

### 五、速记表

| 手段 | 适用场景 | 语法 |
|------|---------|------|
| typeof | 原始类型 | `typeof x === 'string'` |
| in | 对象结构区分 | `'k' in obj` |
| instanceof | 类实例 | `x instanceof Date` |
| 相等性 / switch | 字面量联合、判别联合 | `===` / `switch` |
| 类型谓词 | 复杂自定义判断 | `x is Type` |

### 六、易错点速记

1. **`typeof null === 'object'`**，别用 typeof 判 null，用 `x === null`。
2. **instanceof 对 interface/纯对象无效**（运行时类型信息已擦除）。
3. **类型谓词的判断逻辑要你自己保证正确**：TS 只信你的 `is` 声明，声明错了编译器不报，运行时坑你。
4. **优先用判别联合**：给联合类型设计一个字面量判别字段，比 `in` 检查业务字段更稳。

---

## 考核过程

📝 考核题：API 响应处理

  interface Success { ok: true;  data: string; }
  interface Failure { ok: false; error: string; }
  type Response = Success | Failure;

  任务 1　实现 getMessage(r: Response): string：成功返回 r.data，失败返回 r.error。请用相等性收窄（判断 r.ok）实现。
  ```ts
  function getMessage(r: Response): string{
    if(r.ok) return r.data
    return r.error
  }
  ```

  任务 2　再写一个版本：先定义自定义类型谓词 isSuccess(r: Response): r is Success，再用它收窄实现同样的 getMessage。
  ```ts
  function isSuccess(r: Response): r is Success {
    if(r.ok === true) return true
    return false
  }
  function getMessage(r: Response): string{
    if(isSuccess(r)) return r.data
    return r.error
  }
  ```

  任务 3　同事用 in 写了第三版：

  function getMessage3(r: Response) {
    if ('data' in r) { return r.data }
    return r.error
  }

  这版现在能跑，但从可维护性看有隐患。请说明：如果将来 Failure 也加了一个 data 字段，会发生什么？ 为什么用 ok 这种判别字段比用 in
  检查业务字段更稳妥？

  in的判别会失效，ok这种字面量判别专门为区别而设计的

**点评**：
三任务全对，一次通过。任务1 用 `if(r.ok)` 是真值收窄（对布尔判别符与 `=== true` 等价）；点拨：对字符串/数字字面量判别符真值收窄无效，须用 `===`/switch。任务2 类型谓词正确。任务3 核心对：in 失效、ok 专为区分设计。补具体机制：Failure 加 data 后两类型都有 data，`'data' in r` 对两者为真，if 分支 r 仍为联合（data 拓宽），else 分支收窄成 never，`r.error` 编译报错。

---
通过时间：2026-08-08

  
