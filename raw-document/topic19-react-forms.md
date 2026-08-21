# 受控组件与表单（vs v-model）

## 讲解原文

### 一句话锚定

**v-model 是 `:value` + `@input` 的语法糖（你在 Vue 里早就知道），React 表单就是把这层糖剥掉后手写的那两行。**所谓"受控"= 表单的值完全由 state 驱动，输入框只是 state 的"显示器 + 遥控器"。

---

### 1. v-model 拆解对照

```html
<!-- Vue -->
<input v-model="text" />
<!-- 等价于 -->
<input :value="text" @input="text = $event.target.value" />
```

```jsx
// React：没有语法糖，两行都自己写
const [text, setText] = useState('')
<input value={text} onChange={e => setText(e.target.value)} />
```

**受控闭环**：敲键 -> onChange -> setText -> 重渲染 -> `value={新text}` 回填。state 是唯一真相源，输入框本身不持有状态。

**只写一半会怎样（高频坑）**：

```jsx
<input value={text} />                {/* 没有 onChange：输入完全没反应 + 控制台警告（React 以为你想做只读） */}
<input onChange={...} />              {/* 没有 value：非受控，输入框自己管自己，state 拿不到值 */}
<input value={text} readOnly />       {/* 显式只读：合法 */}
```

---

### 2. 各种表单控件的受控写法

| 控件 | Vue v-model | React 受控 |
|------|-------------|------------|
| text/number 输入框 | `v-model="text"` | `value={text}` + `onChange` 取 `e.target.value` |
| 多行文本 | `v-model="text"` | `<textarea value={text}>`（不是 children！Vue 的区别点） |
| 复选框 | `v-model="ok"` | `checked={ok}` + 取 `e.target.checked`（注意不是 value） |
| 单选组 | `v-model="gender"` | 每个 `<input type="radio" checked={g==='male'}>` 手动对齐 |
| 下拉单选 | `v-model="city"` | `<select value={city}>` 挂父级（不是 `<option selected>`） |
| 文件 | `v-model` 不支持 | **只能非受控**：`defaultValue` + ref（2.4），file 对象不可序列化不可受控 |

统一处理的惯用法（用计算属性名收敛 handler）：

```jsx
const [form, setForm] = useState({ user: '', pass: '' })
function handleChange(e) {
  const { name, value } = e.target
  setForm(prev => ({ ...prev, [name]: value }))   // 不可变更新 + 计算属性名，一个 handler 管全表单
}
<input name="user" value={form.user} onChange={handleChange} />
<input name="pass" value={form.pass} onChange={handleChange} />
```

---

### 3. v-model 修饰符的手工等价

| Vue 修饰符 | React 手写 |
|-----------|-----------|
| `v-model.number` | 提交或 onChange 时 `Number(e.target.value)` |
| `v-model.trim` | `e.target.value.trim()` |
| `v-model.lazy` | 把 onChange 换成 `onBlur`（失焦才同步） |

没有魔法，全是你自己的代码。

---

### 4. 非受控：让 DOM 自己管状态

```jsx
const inputRef = useRef(null)                    // 2.4 详讲 ref
<input defaultValue="初始值" ref={inputRef} />
// 要值时才去读：inputRef.current.value
```

取舍：

| | 受控 | 非受控 |
|---|------|--------|
| 真相源 | React state | DOM |
| 每键触发重渲染 | 是（大表单可感知） | 否 |
| 校验/联动/禁用按钮 | 天然支持（state 里都有） | 要手动读 DOM 对比 |
| 适用 | 绝大多数场景 | 文件上传、超长表单性能敏感、简单一次性读取 |

Vue 人的直觉校准：Vue 的 v-model 也是受控的（底层同步到响应式数据），所以你在 Vue 里的表单习惯基本可以直接迁移，只是糖没了，**每键一次重渲染在两边都存在**--只是 Vue 帮你把触发藏起来了。

## 考核过程

### 题目 1：编码题（Vue v-model 表单翻译成 React 受控组件，一个 handler 管全部字段）

**用户回答（第 1 轮）**：结构正确（一个 handler + 计算属性名 + 函数式更新）。三处硬伤：
- checkbox 用 `value={form.agree}`（应为 `checked`，本节表格原题），初始值 `agree: ''`（应为 false）
- `<form onClick={onSubmit}>`（对应 Vue `@submit.prevent`，应挂 `onSubmit`；1.3 内容回炉）
- `trim(e.target.value)` 把字符串方法当全局函数调用（与 1.3 同款错误）

**点评**：细节错误 ×3，给定位提示后修一版。

**用户回答（第 2 轮）**：三处全修对 ✅。但新引入逻辑问题：`isNaN(Number(v)) ? v.trim() : Number(v)` 全局套用所有字段--姓名 "007" 会被转成数字 7。修饰符是**字段语义**不是数据形状猜测。

**用户回答（第 3 轮）**：handler else 分支按 `name` 分派（age -> Number+isNaN 兜底、name -> trim、其余原值）✅

### 补充对话
- 提示过 `e.target.type === 'checkbox'` 判断可替代 type 参数方案；`onChange={e => handleChange(e)}` 可简写为 `onChange={handleChange}`（用户最终版采用 type 判断）
- JSX 尾部两次粘贴截断（button/form 闭合），未计入评判

---
通过时间：2026-08-17

考核题（编码题）：把这个 Vue 表单翻译成 React 受控组件，要求一个 handler 管全部字段：
 写完整的 React 组件（state 初始化 + handler + JSX），并保证：trim/number 语义保留、submit 不刷新页面。
```jsx
function Form() {
 const [form ,setForm] = useState({name: '',age: '',agree: false ,city: ''})
 function handleChange(e) {
  const {name,type} = e.target
  let value = ''
  if(type === 'checkbox') {
    value = e.target.checked
  }else {
    if(name === 'age') {
      const newVal = Number(e.target.value)
      value = isNaN(newVal) ? e.target.value : newVal
    }else if(name === 'name') {
      value = e.target.value.trim()
    }else {
      value = e.target.value
    }
    
  }
  setForm(prev => ({ ...prev, [name]: value }))
 }
 function onSubmit(e) {
    e.preventDefault()
    // submit form api
 }
  return (
    <form onSubmit={onSubmit}>
      <input name='name' value={form.name} placeholder="姓名" onChange={e => handleChange(e)} />
      <input name="age" value={form.age} onChange={e => handleChange(e)}  placeholder="年龄" />
      <input type="checkbox" name="agree" checked={form.agree} onChange={e => handleChange(e)} /> 同意协议
      <select name="city" value={form.city} onChange={e => handleChange(e)}>
        <option value="sh">上海</option>
        <option value="bj">北京</option>
      </select>
      <button>提交</button>
    </form>
  )
}
```


 

  
