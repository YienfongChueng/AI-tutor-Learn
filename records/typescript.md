---
mode: knowledge
start_date: 2026-08-06
nodes:
  "1.1":
    name: 原始类型 & 字面量类型
    module: 类型基础
    status: mastered
    attempts: 3
    last_tested: 2026-08-10
    mastery_level: 2
    failure_type: detail
  "1.2":
    name: 数组、元组 & 对象类型
    module: 类型基础
    status: mastered
    attempts: 6
    last_tested: 2026-08-10
    mastery_level: 2
    failure_type: conceptual
  "1.3":
    name: 函数类型（参数/返回值/重载）
    module: 类型基础
    status: mastered
    attempts: 2
    last_tested: 2026-08-10
    mastery_level: 2
    failure_type: conceptual
  "1.4":
    name: interface vs type
    module: 类型基础
    status: mastered
    attempts: 1
    last_tested: 2026-08-11
    mastery_level: 2
    failure_type: detail
  "1.5":
    name: 类型断言 & 类型声明
    module: 类型基础
    status: mastered
    attempts: 2
    last_tested: 2026-08-11
    mastery_level: 2
    failure_type: detail
  "2.1":
    name: 联合类型 & 交叉类型
    module: 进阶类型
    status: mastered
    attempts: 2
    last_tested: 2026-08-11
    mastery_level: 2
    failure_type: close
  "2.2":
    name: 类型守卫 & 类型收窄
    module: 进阶类型
    status: mastered
    attempts: 1
    last_tested: 2026-08-17
    mastery_level: 2
    failure_type: null
  "2.3":
    name: 映射类型 & 索引签名
    module: 进阶类型
    status: mastered
    attempts: 2
    last_tested: 2026-08-17
    mastery_level: 2
    failure_type: close
  "3.1":
    name: 泛型基础（函数/接口/类）
    module: 泛型与类型体操
    status: mastered
    attempts: 1
    last_tested: 2026-08-17
    mastery_level: 2
    failure_type: close
  "3.2":
    name: 泛型约束（extends）
    module: 泛型与类型体操
    status: mastered
    attempts: 2
    last_tested: 2026-08-10
    mastery_level: 1
    failure_type: conceptual
  "3.3":
    name: 条件类型 & infer
    module: 泛型与类型体操
    status: mastered
    attempts: 1
    last_tested: 2026-08-10
    mastery_level: 1
    failure_type: close
  "3.4":
    name: 内置工具类型实战
    module: 泛型与类型体操
    status: mastered
    attempts: 1
    last_tested: 2026-08-10
    mastery_level: 1
    failure_type: null
---

# TypeScript 学习记录

## 学习日志
| 日期 | 节点 | 结果 | 备注 |
|------|------|------|------|
| 2026-08-06 | 1.1 | 通过 | attempts=3；细节错误：List 未定义+o.status 隐式 any；概念：let 拓宽导致保护失效 |
| 2026-08-06 | 1.2 | 通过 | attempts=6；概念错误：可选字段与索引签名混淆、可选链与条件分支混淆 |
| 2026-08-07 | 1.3 | 通过 | attempts=2；概念错误：重载签名用宽类型 string 导致坍缩，修正为字面量区分入参 |
| 2026-08-07 | 1.4 | 通过 | attempts=1；细节错误：interface/type 共享命名空间，同名 PermissionUser 报 Duplicate identifier，改为 PermissionUser2 |
| 2026-08-07 | 1.5 | 通过 | attempts=2；细节错误：断言目标选 Element 应为 HTMLInputElement、.d.ts 默认导出对象语法多次修正 |
| 2026-08-08 | 2.1 | 通过 | attempts=2；接近正确：任务3漏答"能填什么值"，微调时误写"不会报错"后自行纠正为"都会报错"（never 不接受任何值）|
| 2026-08-08 | 2.2 | 通过 | attempts=1；一次通过。点拨：if(r.ok) 真值收窄对布尔判别符等价于 ===，但字符串/数字字面量判别符须用 ===/switch；Failure 加 data 后 in 失效、else 收窄 never |
| 2026-08-08 | 2.3 | 通过 | attempts=2；接近正确：任务1漏写修复方式（微调后给 string|number）、任务3初答把 a 的值说成 string（微调纠正为 number）。模块2完成 |
| 2026-08-10 | 1.1 | 复习通过 | Lv1->2；core保留，细节纠正：const 阻止全部重赋值，正确写法 `let s:'open'\|'shipped'='open'` |
| 2026-08-10 | 1.2 | 复习通过 | Lv1->2；fix正确，概念纠正：索引签名不"覆盖"name(兼容共存)，真正问题是禁止可选字段(undefined不兼容) |
| 2026-08-10 | 1.3 | 复习通过 | Lv1->2；重载坍缩原因答对，改法正确(number区分+实现签名联合) |
| 2026-08-10 | 3.1 | 通过 | attempts=1；接近正确：对象字面量误用`;`分隔(应为`,`)，泛型逻辑完全正确；③解释可更精确(T锁定后arr元素类型与data同类型) |
| 2026-08-10 | 3.2 | 通过 | attempts=2；①②代码完美(keyof约束+T[K]返回+&{id}约束)；③概念纠正：相比无约束<T>，extends新增「能用成员」而非「保留具体类型」(后者两者都有) |
| 2026-08-10 | 3.3 | 通过 | attempts=1；①Unwrap正确②A=number[]/B=string正确③分布性推导对(string[]|number[])但误坍缩成[]；联合点属2.1,明日复习重测 |
| 2026-08-10 | 3.4 | 通过 | attempts=1零错误；Pick/Omit正确；Partial草稿态解释正确；MyParameters实现正确(infer P提取参数元组[number,string])。模块3+TS主题全部完成 |
| 2026-08-11 | 1.4 | 复习通过 | Lv1->2；声明合并对称点确认(interface同名合并/type同名或混用报Duplicate) |
| 2026-08-11 | 1.5 | 复习通过 | Lv1->2；as断言+HTMLInputElement目标正确；尖括号断言tsx限制答错(补:与JSX标签歧义,故tsx只能用as) |
| 2026-08-11 | 2.1 | 复习通过 | Lv1->2；string&number=never不接任何值/联合取并集，上次漏答"能填什么值"本次答对 |
| 2026-08-17 | 2.2 | 复习通过 | Lv1->2；守卫场景全对+自定义守卫信任式断言本质；补:Array.isArray是内置类型谓词`arg is any[]`，与自定义守卫同一机制 |
| 2026-08-17 | 2.3 | 复习通过 | Lv1->2；映射核心对；漏答Partial改法(加`?`修饰符)，记入待追问点 |
| 2026-08-17 | 3.1 | 复习通过 | Lv1->2；泛型本质(保留入参->返回类型精确关系)与any绕过检查答得准 |
