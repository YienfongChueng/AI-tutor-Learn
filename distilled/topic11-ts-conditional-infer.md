# 3.3 条件类型 & infer

**它解决什么问题**
约束只能要求 T「至少有某形状」，无法按 T 形态分支或取出内部类型；条件类型 + infer 给类型层加 if-else 和提取能力。

**核心机制**
- T extends U ? X : Y 做类型层三元判断
- 裸联合触发分布性：拆开逐个判断再合并
- [T] extends [U] 包元组可关闭分布性
- infer R 在 extends 子句提取类型一部分（数组元素/Promise内部/函数返回）
- 嵌套条件类型做多分支；自造 ReturnType = (...args:any[])=>infer R

**最容易被追问的点**
- "T extends string ? true : false 对 string|number 得什么？" -> true|false = boolean（分布性拆开再合并）
- "怎么关掉分布性？" -> [T] extends [string]，包成元组不再裸
- "infer 提取什么？" -> 匹配成功时把类型一部分绑给 R，如 Promise<infer R> 取内部类型
- "ToArray<string|number> 等于 [] 吗？" -> 不等于；分布得 string[]|number[]（联合，满足其一即可），不坍缩成 []

**一句话类比**
条件类型像类型层问诊台，像数组就抽出元素类型(infer)，不像走 else；分布性像群体体检，联合里每个人分开问诊再合并结果。
