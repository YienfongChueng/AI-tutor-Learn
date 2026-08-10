# 3.4 内置工具类型实战

## 讲解原文

### 它解决什么问题

3.1-3.3 学了泛型、约束、条件类型、infer--这些是「零件」。TS 内置了一批用这些零件造好的「工具类型」，覆盖最常见的类型变换：属性变可选、挑字段、删字段、拿函数返回类型……不用每次自己造轮子。

本节不逐个背 API，而是**按变换类别掌握 + 理解底层实现**（用前面学的零件复现），忘了一个也能推出来。

### 四类核心工具类型

**① 属性修饰类：Partial / Required / Readonly**

```ts
interface User { id: number; name: string; email: string }

type PartialUser  = Partial<User>   // 全字段变可选 { id?: number; name?: string; email?: string }
type RequiredUser = Required<User>  // 全字段变必填（移除 ?）
type ReadonlyUser = Readonly<User>  // 全字段变只读
```

底层（映射类型 2.3 + 修饰符）：
```ts
type MyPartial<T>  = { [K in keyof T]?: T[K] }        // 加 ?
type MyReadonly<T> = { readonly [K in keyof T]: T[K] } // 加 readonly
```
`-?` 是移除可选（Required 的原理）。

**② 字段筛选类：Pick / Omit**

```ts
type UserPreview = Pick<User, 'id' | 'name'>   // 只留 id、name
type UserSafe    = Omit<User, 'email'>         // 去掉 email
```

底层：
```ts
type MyPick<T, K extends keyof T> = { [P in K]: T[P] }
type MyOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>
```
Pick 用映射类型遍历 K；Omit = Pick + Exclude。

**③ 构造类：Record**

```ts
type UserMap   = Record<string, User>                  // { [k:string]: User }
type StatusMap = Record<'open' | 'closed', string>     // key 必须是这两个之一
```

底层：
```ts
type MyRecord<K extends keyof any, V> = { [P in K]: V }
```
`keyof any` = `string | number | symbol`（所有可做 key 的类型）。

**④ 类型提取类：ReturnType / Parameters**

```ts
function fetchUser(id: number): User { /* ... */ }

type R = ReturnType<typeof fetchUser>    // User
type P = Parameters<typeof fetchUser>    // [number]
```

底层（3.3 条件类型 + infer）：
```ts
type MyReturnType<T>  = T extends (...args: any[]) => infer R ? R : never
type MyParameters<T>  = T extends (...args: infer P) => any ? P : never
```

### Exclude / Extract（联合操作）

```ts
type T = Exclude<'a' | 'b' | 'c', 'a'>          // 'b' | 'c'  剔除
type U = Extract<'a' | 'b' | 'c', 'a' | 'b'>    // 'a' | 'b'  提取
```
底层是分布性条件类型：`Exclude<T, U> = T extends U ? never : T`。

### 实战场景（前端常见）

**表单草稿态：Partial**
```ts
interface Form { name: string; age: number; email: string }
let draft: Partial<Form> = {}   // 初始空，逐步填写
draft.name = '张三'              // 只填一个也合法
```

**枚举映射：Record 收束 key**
```ts
const statusText: Record<'open' | 'done' | 'closed', string> = {
  open: '进行中', done: '已完成', closed: '已关闭'
}  // 少写一个 key 立刻报错
```

**消除重复：ReturnType 跟着函数走**
```ts
type User = ReturnType<typeof fetchUser>  // 函数改返回类型，User 自动跟着变
```

### 工具类型速查

| 类别 | 工具 | 作用 | 底层零件 |
|------|------|------|---------|
| 修饰 | Partial / Required / Readonly | 加/去 ?、加 readonly | 映射类型 |
| 筛选 | Pick / Omit | 留/删指定字段 | 映射 + keyof |
| 构造 | Record | 批量造同 value 的对象 | 映射 + keyof any |
| 提取 | ReturnType / Parameters | 拿函数返回/参数类型 | 条件类型 + infer |
| 联合 | Exclude / Extract | 剔除/提取联合成员 | 分布性条件类型 |

### 一句话类比
工具类型像「类型版 Lodash」--Partial/Pick/Omit 是对象变换，ReturnType/Parameters 是函数探针，Record 是批量造对象。底层全是你刚学的泛型 + 映射 + 条件类型 + infer 拼出来的，忘了能现推。

### 本节边界
- ✅ 本节讲：Partial/Required/Readonly、Pick/Omit、Record、ReturnType/Parameters、Exclude/Extract 的用法 + 底层 + 实战
- 🎉 这是 TypeScript 主题最后一个节点，通过后做模块 3 + 全主题总结

## 考核过程

📝 考核题 3.4（收官题 🎉）

  interface User { id: number; name: string; email: string; age: number }

  要求：

  ① 用内置工具类型从 User 派生出两个类型：一个「只含 id 和 name」，一个「去掉 email」。

  ② 用 Partial<User> 表示表单草稿态：写一行创建空草稿 draft，再赋值 draft.name = '张三'。说明：为什么 Partial
  能让"先建空对象、再逐个填字段"合法？（不写 Partial 会怎样？）

  ③ 用条件类型 + infer 自己实现 MyParameters<T>（拿到函数的参数元组类型）：
  declare function fetchUser(id: number, name: string): User
  type P = MyParameters<typeof fetchUser>   // 期望得到 [number, string]
  写出 MyParameters 的实现，并解释 infer 在这里提取的是什么。

```ts
type UserPick = Pick<User,'id' | 'name'>
type UserOmit = Omit<User,'email'>

let draft: Partial<User> = {}
draft.name = '张三'
// 先用Partial 将User对象属性都转为可选属性，然后只赋值一个值也不会报错
// 如果不写Partial，那么draft这个对象需要初始化所有成员，不然报错
type MyParameters<T> =  T extends (...args: infer P) => any ? P : never
declare function fetchUser(id: number, name: string): User
type P = MyParameters<typeof fetchUser>
// P在这里提取的是[number,string]
```
**点评**：
通过（attempts=1，零错误）。①Pick<User,'id'|'name'> 与 Omit<User,'email'> 完全正确；②Partial 草稿态代码与解释正确（Partial 转可选使空对象合法，不写则需初始化全部必填字段）；③MyParameters 实现正确 T extends (...args: infer P) => any ? P : never，infer P 提取函数参数元组类型，fetchUser 得 [number, string]。

---
通过时间：2026-08-10
🎉 TypeScript 主题全部 12 节点完成。
