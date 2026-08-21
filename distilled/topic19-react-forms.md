# 受控组件与表单（vs v-model）

**它解决什么问题**
把 v-model 的糖剥掉，手动维持"state 是唯一真相源"的受控闭环。

**核心机制**
- v-model = value + onChange 两行手写，敲键->setState->重渲染回填
- checkbox 用 checked，select 的 value 挂父级，textarea 用 value 属性
- 一个 handler 管全表单：name + 计算属性名 + 函数式更新
- 文件上传只能非受控（defaultValue + ref）
- 修饰符手工等价：number/trim 在 onChange 处理，lazy=onBlur

**最容易被追问的点**
- "只写 value 不写 onChange？" -> 输入完全没反应 + 警告；显式 readOnly 才合法
- "v-model.number 怎么还原？" -> 按字段分派 `Number(v)`+isNaN 兜底，不能全局按数据形状猜（"007"会被变 7）
- "非受控什么时候用？" -> 文件、性能敏感的超长表单、一次性读取
- "受控的代价？" -> 每键一次重渲染（Vue 的 v-model 同样有，只是糖藏起来了）

**一句话类比**
v-model 是自动挡，React 表单是手动挡--挡位（value）和离合（onChange）都自己踩，但车是同一辆。
