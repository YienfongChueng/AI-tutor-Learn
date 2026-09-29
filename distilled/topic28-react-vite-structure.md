# 3.1 Vite + React 项目结构与 StrictMode

**它解决什么问题**
React 新项目怎么起步（CRA 已死），以及为什么开发模式下请求会发两遍、组件会渲染两次--StrictMode 是疫苗不是 bug。

**核心机制**
- 脚手架：`npm create vite@latest -- --template react-ts`
- 挂载：`createRoot(el).render(<App/>)`（React 18 必须用 createRoot）
- StrictMode（仅开发）：组件函数连续调用两次 + effect 走 setup→cleanup→setup
- 双渲染是同一次提交的纯度体检，第一份结果丢弃，两次 render 之间没有 effect
- 生产环境完全没有双调用行为

**最容易被追问的点**
- "请求双发怎么办？" -> 补 cleanup/ignore，删 StrictMode=撕体检报告
- "dev 和 prod 显示不一致说明什么？" -> 渲染不纯（渲染期写了 ref.current/外部变量）
- "改了 ref 为啥界面不动？" -> effect 里改 ref 不触发重渲染，屏幕停在渲染时的旧值
- "StrictMode 是 render→effect→render→effect 吗？" -> 不是，render×2 背靠背在提交前，effect×2 在提交后
- "Vue 为啥不需要？" -> 依赖追踪不依赖调用顺序

**一句话类比**
StrictMode 像入职体检的复检：同样的血样（props/state）抽两管各化验一次，结果不一致就说明身体（渲染函数）不干净。
