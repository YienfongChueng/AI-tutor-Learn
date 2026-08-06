# 1.2 数组、元组 & 对象类型

## 讲解原文

### 一、数组类型

TS 数组有两种等价写法：

```ts
let a: number[] = [1, 2, 3]              // 写法 1：类型[]
let b: Array<number> = [1, 2, 3]        // 写法 2：泛型写法（泛型在 3.1 详讲）
```

日常用 `类型[]` 即可，泛型写法在写复杂类型/工具类型时更清晰。

**联合类型的数组--小心括号：**

```ts
let a: number[]            // 全是 number
let b: number | string[]  // ⚠️ 这是「number 或 string[]」，不是「全是 number 或 string」
let c: (number | string)[] // ✅ 这是「number 和 string 混合的数组」，元素可以是任一
```

> 关键：`|` 的优先级比 `[]` 低。想表达「数组里元素是联合类型」，**必须加括号** `(number | string)[]`。这是极高频的坑。

### 二、元组类型（Tuple）

元组 = **固定长度、每个位置类型可不同**的数组。用 `[类型1, 类型2, ...]` 表示：

```ts
let point: [number, number] = [10, 20]        // 二维坐标：两个 number
let rec: [string, number] = ['Tom', 18]       // 姓名 + 年龄
let httpRes: [number, string] = [200, 'OK']   // 状态码 + 消息
```

**对比数组 vs 元组：**

| | 数组 `number[]` | 元组 `[number, string]` |
|---|---|---|
| 长度 | 任意 | 固定 |
| 每个位置类型 | 全部相同 | 可以不同 |
| 推断 | `let a = [1,2,3]` -> `number[]` | `let t = ['x', 1]` -> `(string|number)[]`（非元组！） |

**关键陷阱：字面量数组初值不会被推断成元组。** `let t = ['x', 1]` 推断成 `(string|number)[]`，**不是** `[string, number]`。要得到元组，必须**显式标注** `let t: [string, number] = ['x', 1]`，或用 `const` 断言（`as const`，1.5 讲）。

**元组的实战场景：**
- `useState` 的返回值 `const [count, setCount] = useState(0)` 返回的就是元组 `[number, (v:number)=>void]`
- CSV 一行、坐标点、`Promise.all` 的结果
- 旧的 HOC / render prop 返回值

### 三、对象类型

用 `{}` 描述对象形状，**字段: 类型**逐个标注：

```ts
let user: { name: string; age: number } = { name: 'Tom', age: 18 }
```

**1. 可选字段 `?`**

```ts
type User = { name: string; age?: number }  // age 可有可无
const u: User = { name: 'Tom' }             // ✅ 不写 age 也合法
```

`?` 表示「这个字段可能不存在」。**重要后果：读取可选字段时，TS 认为它是 `类型 | undefined`**：

```ts
function getAge(u: User) {
  return u.age.toFixed()   // ❌ 报错：u.age 可能是 undefined，不能直接 .toFixed()
}
```

要先判空（类型守卫 2.2 详讲）或用可选链 `u.age?.toFixed()`。

**2. 只读字段 `readonly`**

```ts
type Config = { readonly apiKey: string }
const c: Config = { apiKey: 'xxx' }
c.apiKey = 'yyy'   // ❌ 报错：只读字段不可赋值
```

> `readonly` 是**编译期**检查，运行时仍可改（不像 `const` 锁定变量绑定）。它防的是「代码里误改」，不防「恶意篡改」。

**3. 索引签名 `[key: 类型]: 类型`**

当对象字段名不固定时，用索引签名描述「任意 key 都是某类型」：

```ts
type Scores = { [k: string]: number }
const s: Scores = { math: 90, english: 85 }  // 任何 key 都行，值必须是 number
```

**高频坑：索引签名会让所有字段都受其约束。**

```ts
type User = {
  name: string        // ❌ 报错！
  [k: string]: number // name 是 string，不兼容 number 索引签名
}
```

因为索引签名 `[k: string]: number` 声明「所有字符串 key 的值都是 number」，而 `name: string` 违反了它。修正：要么索引签名值改成 `string | number`，要么别把固定字段和索引签名混用。

### 四、一句话总结

- 数组：`类型[]` 或 `Array<类型>`；联合数组必须加括号 `(A|B)[]`
- 元组：`[A, B, C]` 固定长度异构数组；字面量初值**不会**推断成元组，要显式标注
- 对象：`?` 可选、`readonly` 只读、`[k:类型]:类型` 索引签名；可选字段读取时是 `类型|undefined`，索引签名会约束所有字段
- 有具体字段名 → 不用方括号；key 不固定 → 才用方括号索引签名。

## 考核过程
### 考核题：给后端接口响应设计类型

  你在写后台管理系统，对接一个查询接口。后端返回的数据有三种情况，请你为每种情况设计类型，并补全一
  个处理函数。
  - 背景

  ```ts
  // 情况 A：单个用户
  // { "name": "Tom", "age": 18, "role": "admin", "avatar": "http://..." }

  // 情况 B：CSV 导出的一行（固定顺序：姓名, 年龄, 评分）
  // ["Tom", 18, 9.5]

  // 情况 C：动态键值对（后端返回的「自定义配置」，键名不固定，值都是字符串）
  // { "theme": "dark", "language": "zh-CN", "timezone": "UTC+8" }
  ```
  - 任务：
  
  1. 分别为情况 A / B / C 定义类型，命名为 User / CsvRow / Config
  2. 情况 A 的 role 只能是 'admin' | 'editor' | 'viewer'，age 可选，avatar可选但只要存在就必须是字符串
  3. 情况 B 用元组表示（为什么这里用元组而不是数组？）
  4. 情况 C 用索引签名表示
  5. 补全下面这个函数并让它类型安全：
   ```ts
    // 接收情况 A 的用户，返回展示用的字符串
    function formatUser(u: /* 你来填 */) {
      // 要求：返回 "姓名(角色)"，如 "Tom(admin)"
      // 注意：age 和 avatar 都可能不存在，你的代码要处理这种情况
    }
    
   ```

  ```ts
   type Role = 'admin' | 'editor' | 'viewer';
   type User = { 
    "name": string, 
    "age"?: number, 
    "role": Role, 
    'avatar'?:string 
  };
  type CsvRow = [string,number,number];
  const csvRow:CsvRow = ["Tom", 18, 9.5];
  type Config = {
    [k: string]: string 
  };
  function formatUser(u: User) {
    let result = `${u.name}(${u.role})`
    if(u.age !== undefined) {
      result = `${u.name}(${u.role},${u.age})`
    }
    return result
  }
  //回答问题3:因为CsvRow的对象属性值的数据类型是不固定的，元组支持每个位置类型可不同；而数组里的成员只能是相同类型的数据
  ```

### LLM 点评

CsvRow 选元组、Config 用索引签名 -- 判断正确，第 3 题解释到位。最终代码类型安全、逻辑正确，通过。

**考核迭代过程（6 轮，2 个概念性错误）：**
1. 首轮：`const type` 混用类型定义与变量声明；avatar 写成 `['avatar':string]?: "http://..."`（索引签名语法与可选字段混用 + 误用字面量类型）；formatUser 跳过可选字段处理。
2. 第 2 轮：avatar 仍带方括号（第 2 次栽在方括号）；formatUser 三分支返回相同字符串，age 未拼入。
3. 第 3 轮：User 正确（avatar 改为 `avatar?: string`）。
4. 第 4 轮：formatUser 用可选链 `u?.age` 全拼，age 不存在时输出 "undefined" 字样。**关键认知：可选链只管访问不报错，不管显示格式。**
5. 第 5 轮：改用条件拼接但漏闭右括号 `)`。
6. 第 6 轮：最终版补全闭括号、`if(u.age !== undefined)` 严谨判空、分支内不用多余 `?.`。通过。

**两个核心认知突破（必须牢记）：**
1. **可选字段 vs 索引签名**：有固定字段名 -> `avatar?: string`（不用方括号）；key 不固定 -> `[k:string]: string`（才用方括号）。
2. **可选链 vs 条件分支**：`?.` 管访问不报错，管不了显示格式；要控制「不显示」必须用 `if` 条件拼接。

---
通过时间：2026-08-06

