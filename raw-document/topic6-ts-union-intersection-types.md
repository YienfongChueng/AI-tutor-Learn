# 2.1 联合类型 & 交叉类型

## 讲解原文

### 一、核心直觉：或门 vs 与门

| | 联合类型 `A \| B` | 交叉类型 `A & B` |
|---|---|---|
| 逻辑关系 | 或（OR） | 与（AND） |
| 含义 | 值"是 A **或** B 中的某一个" | 值"同时具备 A **和** B 的全部特征" |
| 成员可达性 | 只能访问 A、B 的**公共**成员 | 可访问 A、B 的**所有**成员 |
| 典型用途 | 表示"多种可能之一"（id 可能是数字或字符串） | 组合 / 混入（把多个类型拼成一个） |

一句话记忆：**联合做"选择"，交叉做"叠加"。**

### 二、联合类型 `|`

#### 1. 基本用法

```typescript
let id: string | number;
id = "abc-123";   // ✅
id = 1001;        // ✅
id = true;        // ❌ 既不是 string 也不是 number
```

#### 2. 只能访问公共成员（收窄前）

```typescript
interface Bird { fly(): void; layEggs(): void; }
interface Fish { swim(): void; layEggs(): void; }

function operate(pet: Bird | Fish) {
  pet.layEggs();   // ✅ 两边都有，安全
  pet.fly();       // ❌ Fish 没有 fly，类型报错
  pet.swim();      // ❌ Bird 没有 swim，类型报错
}
```

要访问 `fly()` / `swim()` 这种"非公共"成员，必须先**收窄**类型——这正是下一节 2.2「类型守卫」要解决的问题。这里先记住：**联合类型在收窄前，只能碰公共部分。** TypeScript 这么限制是因为编译器无法确定运行时这个值到底是 A 还是 B，只能保证"不管哪个，公共成员一定存在"。

#### 3. 字面量联合（最常用形态）

联合类型最常见的形态是字面量联合，相当于轻量级枚举：

```typescript
type Status = 'active' | 'inactive' | 'pending';
type Method = 'GET' | 'POST' | 'PUT' | 'DELETE';

function setStatus(s: Status) { /* ... */ }
setStatus('active');   // ✅
setStatus('done');     // ❌ 'done' 不在联合范围内
```

好处：写法轻、IDE 有自动补全、有编译期约束，且收窄后能精确区分每个字面量分支（配合 switch 等做穷尽检查）。

### 三、交叉类型 `&`

交叉类型把多个类型"叠加"成一个，结果拥有全部成员。

#### 1. 基本用法（组合）

```typescript
interface Person { name: string; age: number; }
interface Loggable { log(): void; }

type Employee = Person & Loggable;
// Employee 同时有 name, age, log()

const e: Employee = {
  name: 'Alice',
  age: 30,
  log() { console.log(this.name); }
};
```

#### 2. 本质：成员取"并集"，同名属性类型取"交集"

对于对象类型，交叉结果的**属性集合是并集**（A 的字段 + B 的字段都有）。但如果同名属性的类型不同，该属性类型会取**类型层面的交集**——若两个类型无法共存，就会坍缩成 `never`：

```typescript
type A = { name: string; age: number };
type B = { name: number; email: string };

type C = A & B;
//   age: number;       ✅
//   email: string;     ✅
//   name: string & number  →  never  ⚠️
```

`string & number` 是不可能存在的值（一个值不可能同时是字符串又是数字），于是变成 `never`。一旦 `name` 是 `never`，给它赋任何值都会报错。这是交叉类型最隐蔽的坑。

### 四、ASCII 图示：成员可达性对比

```
联合 A | B（"或"关系）
 ┌──────────┐        ┌──────────┐
 │  类型 A   │        │  类型 B   │
 │  name    │        │  name    │ ← 收窄前只能安全访问 name
 │  age     │        │  email   │   age / email 需先收窄
 └──────────┘        └──────────┘

交叉 A & B（"与"关系）
 ┌───────────────────────┐
 │  name  (A、B 都有)      │
 │  age   (来自 A)         │ ← 全部成员都可访问
 │  email (来自 B)         │
 └───────────────────────┘
```

### 五、易错点速记

1. **联合只能访问公共成员**：访问特有成员前必须收窄（→ 2.2 类型守卫）。
2. **交叉的同名冲突会变 `never`**：`string & number = never`，赋值必报错。
3. **联合不是"合并对象"**：`{a:1} | {b:2}` 不等于 `{a:1, b:2}`（这是"二选一"）；而 `{a:1} & {b:2}` 才等于 `{a:1, b:2}`（这是"全都要"）。
4. **字面量联合是收窄友好的**：`'a' | 'b'` 收窄后能精确匹配每个字面量分支，是替代枚举的常用手段。

---

## 考核过程

📝 考核题：内容平台用户系统

  ▎ 场景：你在做一个内容平台，用户有两种角色。

  // 已有定义，不要修改
  interface Reader { username: string; readCount: number; }
  interface Editor { username: string; editCount: number; }

  任务 1　定义联合类型 User = Reader | Editor，并实现函数 describe(u: User): string，返回该用户的用户名。函数体内只允许访问
  u.username。请说明：为什么不能直接在函数体内访问 u.readCount 或 u.editCount？（一句话原因）
  ```ts
  type User = Reader | Editor
  function describe(u: User): string {
    return u.username
  }
  // 联合类型不是并集，只能访问共同属性，访问非共同属性需要收窄类型
  ```

  任务 2　平台新增 VIP 能力：

  interface Vip { vipLevel: number; }

  你想得到一个"既是 Reader 又是 Vip"的类型 VipReader。请写出它的定义（用合适的类型运算），并指出 VipReader
  类型的变量能访问哪些字段。

  ```ts
  type VipReader = Reader & Vip

  // 可访问username、readCount、vipLevel
  ```

  任务 3　同事写了这样一段：

  type P = { id: string };
  type Q = { id: number };
  type R = P & Q;
  const r: R = { id: ??? };

  问：R['id'] 的类型是什么？??? 处能填什么合法的值？为什么？  

  nerver，string & number

**点评**：
任务1、2 全对。任务3 类型判断与原因正确（`never`，因 `string & number`），但漏答"`???` 处能填什么合法值"，判为接近正确（close），给提示后微调。另顺带点拨精确表述：联合的**值**是并集、**可访问成员**是交集；与交叉（值是交集、成员是并集）正好相反，便于对称记忆。

**补充对话（微调）**：
用户补答任务3"能填什么值"：先答"填任何值都不行，都不会报错"（错），随即自行打断纠正为"填任何值都不行，都会报错，因为不可能有既是数字又是字符串的值存在"。纠正后完全正确：`never` 不接受任何具体值，赋任何值都报错。通过。

---
通过时间：2026-08-08

