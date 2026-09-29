# 3.1 Vite + React 项目结构与脚手架

## 讲解原文

> 你是从 Vue3 过来的，Vite 本身不陌生（Vue 脚手架同款内核）。本节真正的新知识点只有一个，但它是所有 React 新手的第一个"见鬼时刻"：**StrictMode 的双调用**。

### 一、脚手架：一个命令

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install && npm run dev
```

- CRA（create-react-app）**已停止维护**（2023 年起），新项目不要再用，面试提一句就知道你跟进社区
- Vite 内核你已熟悉：dev 时 esbuild 预构建 + 原生 ESM 按需加载，生产 Rollup 打包。Vue 和 React 只是换模板，工程体验完全一致

### 二、和 Vue 项目逐文件对照

```
my-react-app/                 my-vue-app/
├─ index.html                 ├─ index.html
│   └ <script src=/src/main.tsx>   │  └ <script src=/src/main.js>
├─ vite.config.ts             ├─ vite.config.ts
│   └ @vitejs/plugin-react         │   └ @vitejs/plugin-vue
└─ src/                       └─ src/
   ├─ main.tsx  ←入口             ├─ main.js  ←入口
   ├─ App.tsx                     ├─ App.vue
   └─ ...                         └─ ...
```

两个关键差异：

1. **`index.html` 在根目录**（Vite 的约定，入口 HTML 在外面往里引），Vue 项目同款，无迁移成本
2. 插件从 `plugin-vue` 换成 `plugin-react`（负责 JSX/Fast Refresh 转换）

### 三、main.tsx：挂载方式的对照

```tsx
// React
import { createRoot } from 'react-dom/client'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>
)
```

```ts
// Vue
createApp(App).mount('#app')
```

| | Vue | React |
|---|---|---|
| 挂载 | `createApp(App).mount()` | `createRoot(el).render(<App/>)` |
| 插件体系 | `app.use(router).use(pinia)` | 无全局 use，各库自带 Provider 组件或 `hydrateRoot` 类 API |
| 全局错误处理 | `app.config.errorHandler` | ErrorBoundary 组件（s3 再讲） |

注意 React 18 起 `ReactDOM.render` 已废弃，必须 `createRoot`（并发特性的入口，s5 源码篇会回来讲它）。

### 四、本节主角：StrictMode 双调用

装上 `<StrictMode>` 后，**仅开发模式**下 React 会故意：

1. **每个组件函数渲染两次**（同输入调两遍）
2. **每个 effect 走一遍 mount -> unmount -> 再 mount**（cleanup 被执行后再重跑）

生产构建**完全没有**这个行为。

**为什么？** 这是一次"疫苗"：React 在帮你排查两类病--

- 渲染不纯：两次渲染结果不一样（渲染里改了外部变量/`ref.current`、用了 `Math.random()`）-> 界面闪烁警告你
- effect 缺 cleanup：mount/unmount 走两遍，没写 cleanup 的副作用（重复请求、重复监听）**当场翻倍暴露**

**"见鬼时刻"**：你写了个 `useFetch('/api/user')`，Network 面板里**同一个请求发了两遍**。新手的反应是删掉 StrictMode（错误答案！）。正确反应：检查 effect 是否该有 cleanup/ignore 旗标（2.2/2.3 的 `let ignore = false` 正是为此）。删 StrictMode = 把体检报告撕了，病还在，只是生产环境才爆。

```tsx
// StrictMode 友好的请求（模块 2 写过）：
useEffect(() => {
  let ignore = false
  fetch(url).then(j => { if (!ignore) setData(j) })
  return () => { ignore = true }   // 模拟卸载时作废 -> 双调用无害
}, [url])
```

**Vue 对照**：Vue 没有这套机制（`app.config` 的 strict 只是 props 校验警告），因为 Vue 的响应式依赖追踪不需要调用顺序保证。React 的双调用本质是给"靠顺序索引的 Hooks 体系"做的压力测试。

### 五、生产项目结构参考（中型应用）

```
src/
├─ main.tsx            入口
├─ App.tsx             路由/Provider 组装层
├─ routes/             路由级页面
├─ components/         通用组件
├─ hooks/              自定义 Hook（2.7 的产物住这里）
├─ api/                请求封装 + 类型
└─ stores/             状态库（阶段 3 后）
```

约定俗成的分层信号：页面找 `routes/`，复用逻辑找 `hooks/`，别在一个 `App.tsx` 里堆完。

## 考核过程

### 题目 1：四问（辨析 / 输出序列 / 找病+预测 / Vue 对照）

**用户回答**：
1. 甲错（撕体检报告，问题还在）、乙对、丙错 ✅
2. connect、disconnect、connect ✅
3. 开发显示 1、生产显示 0 ❌；违反"渲染期间写 ref"铁律 ✅
4. Vue 依赖追踪不需要调用顺序保证 ✅

**点评**：第 3 问预测错，错误模式被反推定位：把 JSX 显示值当成 `++` 表达式的返回值（旧值）。正解：`count.current++` 是语句，`{count.current}` 是稍后单独的读取--开发 2、生产 1。

### 补充对话

- 确认题 1（改成前置 `++count.current`）：答对--前后置只改表达式返回值，JSX 单独读取，结果不变
- 确认题 2（移进 useEffect）：答"1"❌。正解显示 0 且永远停 0：effect 在渲染之后执行 + 改 ref 不触发重渲染（Vue"改 ref 视图自动更新"直觉的翻车现场）
- 最终确认（点无关按钮触发重渲染后显示几）：答对"2，重渲染读盒子最新值" ✅
- 用户主动提问："StrictMode 不是 render→effect→render→effect 吗？"--补讲双渲染是同一次提交的连续两次函数调用（纯度体检，第一份结果丢弃），effect 双跑是提交后另一场体检（cleanup 体检），两次 render 之间没有 effect

---
通过时间：2026-09-01
