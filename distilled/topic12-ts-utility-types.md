# 3.4 内置工具类型实战

**它解决什么问题**
常用类型变换（变可选/挑字段/删字段/拿返回类型）反复手写太累；TS 内置工具类型用泛型+映射+条件类型+infer 造好现成轮子。

**核心机制**
- Partial/Required/Readonly：映射类型加减 ? 和 readonly
- Pick/Omit：映射遍历指定 key 留/删字段（Omit=Pick+Exclude）
- Record<K,V>：映射遍历 K 批量造同 value 对象
- ReturnType/Parameters：条件类型+infer 提取返回/参数类型
- Exclude/Extract：分布性条件类型剔除/提取联合成员

**最容易被追问的点**
- "Partial 底层怎么实现？" -> { [K in keyof T]?: T[K] }，映射类型加 ?
- "Omit 底层？" -> Pick<T, Exclude<keyof T, K>>，不是单独映射
- "ReturnType 底层？" -> T extends (...args:any[])=>infer R ? R : never
- "Record 的 K 约束？" -> K extends keyof any（string|number|symbol）
- "忘了工具类型怎么办？" -> 用映射+条件类型+infer 现推

**一句话类比**
工具类型像类型版 Lodash，对象变换/函数探针/批量构造各司其职，底层全是你学的零件拼的，忘了能现推。
