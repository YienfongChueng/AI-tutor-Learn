# 1.4 interface vs type

## 讲解原文

### 引入：两个长得一样的东西

你已经会写：

```ts
type User = { name: string; age: number }
```

但很多代码里你也见过：

```ts
interface User {
  name: string
  age: number
}
```

这俩描述的对象形状**一模一样**。那为什么 TS 要提供两套？它们到底有什么不同、什么时候该用哪个？这是 TS 高频面试题，也是日常写代码几乎每天都要做的选择。

---

### 一、各自是什么

#### interface —— 对象形状的"契约"

`interface` 专门用来描述"一个对象 / 类应该长什么样"。它是 TS 里最接近 Java / C# `interface` 的东西（你有后端思维，应该感到熟悉）。

```ts
interface User {
  name: string
  age: number
}

const u: User = { name: 'Tom', age: 18 }
```

#### type —— 类型别名

`type` 是"给一个类型起个名字"，它**什么都能起名**：对象、联合、元组、基本类型……

```ts
type User = { name: string; age: number }   // 对象
type Status = 'open' | 'closed'             // 联合 ← interface 做不到
type Pair = [string, number]                // 元组 ← interface 做不到
type ID = string                            // 基本类型别名 ← interface 做不到
```

> 一句话：**interface 是"对象形状专用笔"，type 是"万能别名笔"。**

---

### 二、四个核心差异

#### 差异 1：声明合并（interface 独有）

同名 `interface` 会自动把字段**合并**到一起：

```ts
interface Window {
  title: string
}
interface Window {
  status: number
}
// 实际效果：Window = { title: string; status: number } ✅ 合并成功
```

`type` 同名直接报错：

```ts
type Window = { title: string }
type Window = { status: number }
// ❌ Error: Duplicate identifier 'Window'
```

> 这特性在"给第三方库补字段"时很有用——比如给全局 `window` 加一个自定义属性，或者扩展某个库导出的类型。重新声明同名 interface 即可，不用改源码。

#### 差异 2：扩展的写法不同

```ts
// interface 用 extends
interface Animal { name: string }
interface Dog extends Animal { bark(): void }

// type 用 &（交叉类型 intersection）
type Animal = { name: string }
type Dog = Animal & { bark(): void }
```

效果一样，但**语法不同**：`interface` 用 `extends`，`type` 用 `&`。

#### 差异 3：type 能表示的类型更多

前面已展示：联合 `|`、元组 `[...]`、基本类型别名，**只有 type 能做**。`interface` 只能描述对象形状，干不了这些活。

#### 差异 4：错误信息

`interface` 报错时通常把类型名显示出来（更友好）：

```
Type 'X' is not assignable to type 'User'.
```

`type` alias 有时会被展开成原始结构显示。这是细节，不影响功能，但调试体验略有差异。

---

### 三、决策表：什么时候用哪个

| 场景 | interface | type | 推荐 |
|------|:---------:|:----:|------|
| 描述对象 / 类的形状 | ✅ | ✅ | interface |
| 联合类型 `A \| B` | ❌ | ✅ | type |
| 元组 `[A, B]` | ❌ | ✅ | type |
| 基本类型别名 | ❌ | ✅ | type |
| 需要声明合并 / 扩展库类型 | ✅ | ❌ | interface |
| 工具类型 / 映射类型 | ❌ | ✅ | type |
| 函数类型 | ✅ | ✅ | type 更简洁 |

---

### 四、社区共识（经验法则）

1. **描述对象形状**（组件 props、API 响应结构、类 `implements` 的契约）→ 默认用 `interface`
2. **需要联合、交叉、元组、工具类型** → 必须用 `type`
3. **团队统一最重要** —— 别在同一个项目里一会儿 `interface` 一会儿 `type` 描述同一个对象，自找麻烦

> 在 Vue3 里，你写 `defineProps<{...}>()` 用的是 type 语法；而写组件 emit 的 payload、API 返回值结构、`class` 要 `implements` 的契约时，`interface` 更常见。

---

### 五、一句口诀

> **对象形状选 interface，组合别名选 type。**

记住这一句，90% 的日常选择就不会错。

---

## 考核过程

---
📝 考核题：后台用户权限模块

你在做后台管理系统的用户模块，有 4 个类型定义需求。请对每个需求：先写「用 interface 还是 type」，再写代码，并说明为什么。

需求 1：定义角色联合类型，只能是 'admin' | 'editor' | 'viewer'
需求 2：定义基础用户形状 { id: number; name: string; role: Role }
需求 3：定义「带权限的用户」，包含需求2的全部字段，再加 permissions: string[]。
（请用 interface 的方式和 type 的方式各写一遍，体会两种扩展语法）
需求 4（关键题）：项目里用了一个第三方库，它导出了：

interface DatabaseConfig { host: string }

你想给所有 DatabaseConfig 补一个 port: number 字段，但不能改第三方库源码。你该用 interface 还是 type？写出代码，并说明为什么另一种做不到。

```ts
type Role = 'admin' | 'editor' | 'viewer'
interface User {
  id: number,
  name: string,
  role: Role
}
interface PermissionUser extends User {
  permissions: string[]
}
type PermissionUser2 = User & {permissions: string[]}

interface DatabaseConfig { host: string }
interface DatabaseConfig {
  port: number
}
// interface 同名可以合并声明，type同名会报错

```
---
💡 提示：4 个需求分别对应本节讲的「4 个核心差异」。第 4 题是面试高频考点，答出关键能力名称即可。

### LLM 点评

**评估结果：通过（接近正确 -> 微调通过，1 轮通过）**

4 个核心差异全部命中，概念零错误。需求4 命名合并关键能力直接答出，并正确解释 type 同名报错。

**细节错误（已纠正）**：初答需求3 两份代码用了同一标识符 `PermissionUser`，TS 会报 `Duplicate identifier`。隐藏知识点--**interface 和 type 共用同一命名空间**，二者不是平行两套世界，抢同一个标识符不能重名。用户接受提示后改为 `PermissionUser2`，正确消除重名。

**关键认知沉淀**：interface vs type 不是「功能等价任选」，而是「能力有交集、各自有独占地盘」--interface 独占声明合并，type 独占联合/元组/基本类型别名；二者在命名空间里是竞争关系。

---
通过时间：2026-08-07
