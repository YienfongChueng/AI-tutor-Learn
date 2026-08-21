# JSX 本质与 Vue 模板对照

**它解决什么问题**
回答"JSX 到底是什么"，这是 React 一切心智模型的起点。

**核心机制**
- 构建时把 JSX 编译成 `_jsx(type, props)` 调用
- 运行时求值出普通对象 `{type, props, key}`（ReactElement）
- JSX 是表达式：v-if=三元、v-for=map、v-slot=children
- Vue 模板编译成 `h()` 调用，与 `_jsx()` 殊途同归
- 差异：Vue 用 DSL 约束换编译优化，React 全量 diff 留给运行时

**最容易被追问的点**
- "`typeof <p/>` 是什么？" -> `'object'`，element 是普通对象；"React Element"是身份不是 typeof 返回值
- "children 怎么切分？" -> 逐段：`Hi {name}<b/>` -> `['Hi ', 'Tom', <b/>]`，文本/表达式不合并、空格保留
- "哪些 child 会被包成 _jsx？" -> 只有元素（带标签）；文本和表达式求值结果直接进数组
- "首字母为什么要大写？" -> 小写被当成字符串 DOM 标签，大写才是组件函数引用
- "为什么没有 v-if 这类指令？" -> JSX 图灵完备，JS 本身的能力就是指令

**一句话类比**
JSX 之于 `_jsx()`，就像模板字符串之于字符串拼接--换件外衣，产物还是那个数据。
