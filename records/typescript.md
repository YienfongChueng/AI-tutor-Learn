---
mode: knowledge
start_date: 2026-08-06
nodes:
  "1.1":
    name: 原始类型 & 字面量类型
    module: 类型基础
    status: mastered
    attempts: 3
    last_tested: 2026-08-06
    mastery_level: 1
    failure_type: detail
  "1.2":
    name: 数组、元组 & 对象类型
    module: 类型基础
    status: mastered
    attempts: 6
    last_tested: 2026-08-06
    mastery_level: 1
    failure_type: conceptual
  "1.3":
    name: 函数类型（参数/返回值/重载）
    module: 类型基础
    status: mastered
    attempts: 2
    last_tested: 2026-08-07
    mastery_level: 1
    failure_type: conceptual
  "1.4":
    name: interface vs type
    module: 类型基础
    status: mastered
    attempts: 1
    last_tested: 2026-08-07
    mastery_level: 1
    failure_type: detail
  "1.5":
    name: 类型断言 & 类型声明
    module: 类型基础
    status: mastered
    attempts: 2
    last_tested: 2026-08-07
    mastery_level: 1
    failure_type: detail
  "2.1":
    name: 联合类型 & 交叉类型
    module: 进阶类型
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    failure_type: null
  "2.2":
    name: 类型守卫 & 类型收窄
    module: 进阶类型
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    failure_type: null
  "2.3":
    name: 映射类型 & 索引签名
    module: 进阶类型
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    failure_type: null
  "3.1":
    name: 泛型基础（函数/接口/类）
    module: 泛型与类型体操
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    failure_type: null
  "3.2":
    name: 泛型约束（extends）
    module: 泛型与类型体操
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    failure_type: null
  "3.3":
    name: 条件类型 & infer
    module: 泛型与类型体操
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    failure_type: null
  "3.4":
    name: 内置工具类型实战
    module: 泛型与类型体操
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
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
