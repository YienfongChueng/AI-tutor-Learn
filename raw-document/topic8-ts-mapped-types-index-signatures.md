# 2.3 映射类型 & 索引签名

## 讲解原文

### 一、索引签名（Index Signature）：描述"任意键"的对象

当你不知道对象有哪些键、但知道所有值的类型相同时，用索引签名：

```typescript
interface StringMap {
  [key: string]: string;
}

const dict: StringMap = { a: '1', b: '2', anything: 'x' };
```

`[key: string]: string` 表示"任意 string 键，值都是 string"。

#### 关键约束：索引签名类型必须兼容所有显式属性

```typescript
interface Person {
  [key: string]: string;
  age: number;  // ❌ 报错！age 是 number，但索引签名说所有值都是 string
}
```

因为 `age` 也能用 `person['age']` 访问（走索引签名），它必须满足索引签名的类型。要么所有显式属性都兼容索引签名类型，要么不用索引签名。

#### number 索引的特别之处

JS 里对象的键最终都是字符串，所以 `number` 索引实际上会被转成字符串。TS 允许同时有 `string` 和 `number` 索引，但 **number 索引的类型必须是 string 索引类型的子类型**（因为访问 `obj[0]` 实际是访问 `obj['0']`）。

### 二、keyof 操作符：取出键的联合类型

`keyof T` 把类型 T 的所有键提取成一个**字面量联合类型**：

```typescript
interface User { id: number; name: string; }
type UserKeys = keyof User;  // 'id' | 'name'
```

注意这正好是 2.1 讲的联合类型！`keyof` 是映射类型的基石。

### 三、映射类型（Mapped Type）：遍历键生成新类型

映射类型基于一个键的联合，遍历每个键生成新类型。语法：

```typescript
type Mapped = { [K in UnionType]: ValueType };
```

`in` 关键字遍历联合中的每个键。**注意：这里的 `in` 是类型层面的"遍历"，不是 2.2 讲的运行时类型守卫 `in`**，别混淆。

#### 1. 基于 keyof 遍历现有类型

```typescript
type User = { id: number; name: string };

// 把每个属性的值类型都变成 string
type Stringify<T> = { [K in keyof T]: string };
type R = Stringify<User>;  // { id: string; name: string }
```

#### 2. 修饰符：readonly 和 ?

映射类型可以加 / 减 `readonly` 和 `?` 修饰符，用 `+`（加，可省略）或 `-`（移除）：

```typescript
// 全部变可选（手写 Partial）
type MyPartial<T>  = { [K in keyof T]?: T[K] };

// 全部变只读（手写 Readonly）
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };

// 移除 readonly
type Mutable<T>  = { -readonly [K in keyof T]: T[K] };

// 移除可选
type Required<T> = { [K in keyof T]-?: T[K] };
```

`T[K]` 是**索引访问类型**（indexed access），取出 T 中键 K 对应的值类型。

#### 3. 内置工具类型就是映射类型

很多内置工具类型底层就是映射类型：

- `Partial<T>`  = `{ [K in keyof T]?: T[K] }`
- `Readonly<T>` = `{ readonly [K in keyof T]: T[K] }`
- `Record<K,V>` = `{ [P in K]: V }`（K 是键的联合）

这些会在 3.4「内置工具类型实战」系统练，这里先建立"映射类型 = 类型层面的循环"的直觉。

### 四、映射类型 vs 索引签名：别混淆（高频考点）


|     | 索引签名 `[key: string]: T` | 映射类型 `[K in keyof T]: ...` |
| --- | ----------------------- | -------------------------- |
| 键   | 任意 string 键（不固定集合）      | 遍历 T 的具体键（固定集合）            |
| 用途  | 描述动态键的字典                | 基于已有类型变换生成新类型              |
| 产物  | "开放"对象（可有任意键）           | "封闭"对象（键集合确定）              |


```typescript
// 索引签名：任意键，值都是 number
type Dict = { [key: string]: number };

// 映射类型：键固定为 'id' | 'name'，每个变可选
type OptionalUser = { [K in keyof User]?: User[K] };
```

### 五、易错点速记

1. **索引签名类型必须兼容所有显式属性**，否则报错。
2. **映射类型的 `in` 是"遍历键"，不是运行时 in 操作符**，别和 2.2 的类型守卫 `in` 混。
3. **映射类型用 `keyof` 取具体键集合**，索引签名用 `string`/`number` 表示任意键，两者产物不同。
4. **修饰符 `-readonly` / `-?` 用来移除**；加修饰符时 `+` 可省略。

---

## 考核过程

📝 考核题：索引签名 & 映射类型

  任务 1　下面这段代码会报错吗？如果会，说明原因，并给出一种修复方式（要求 age 仍保持 number 类型）。

```ts
  interface User {
    [key: string]: string;
    name: string;
    age: number;
  }

  //会报错，索引类型必须兼容所有显式属性
  interface User {
    [key: string]: string | number;
  }
```

  任务 2　给定：

```ts
  type Todo = { id: number; title: string; done: boolean };
```

  手写一个映射类型 MyReadonly，把 T 的所有属性变只读。然后回答：MyReadonly的键集合是什么（用联合类型表示）？这些键是怎么得到的？

```ts
  type MyReadonly<T> = { readonly[K in keyof T]: T[K] }
  // id | title | done  K in keyof T,这里的in是遍历键
```

  任务 3　区分题：下面 a 和 b 哪个是索引签名、哪个是映射类型？分别说明它们产出的对象类型的"键"有什么不同。

  type a = { [key: string]: number };
  type b = { [K in 'x' | 'y']: number };

a是索引签名，b是映射类型
a的键是任意string，值是number；b的键是 x或者y

**点评**：
任务1：错误判断与原因正确（索引签名值类型须兼容所有显式属性）。初答漏了修复方式，微调后给出 `[key: string]: string | number` 正确（age 保持 number，name/age 均兼容）。点拨：代价是 `user['任意键']` 返回 `string | number`。
任务2：`MyReadonly<T> = { readonly[K in keyof T]: T[K] }` 正确；键集合 `id | title | done` 由 `keyof T` 得到，`in` 遍历键，全对。
任务3：区分正确（a 索引签名 / b 映射类型）。初答把 a 的值说成 string，微调后纠正为"键任意 string、值 number"，正确。
判为接近正确（close），微调后通过。

---

通过时间：2026-08-08