# 2.5 useContext 与状态提升的取舍

**它解决什么问题**
props 一层层手工传递（prop drilling）让中间组件沦为人肉传送带；Context 提供跨层直达通道，但广播式的重渲染让高频数据全局陪跑。

**核心机制**
- 三步：createContext -> Provider value -> useContext 任意深度取
- Provider value 变化 -> 所有消费者重渲染（无按需筛选）
- 放大器1：value 内联对象每次渲染都是新引用（Object.is 判变）
- 放大器2：消费者重渲染带走整棵子树
- 修法：useMemo 包住 value，依赖不变则引用复用

**最容易被追问的点**
- "App 无关 state 变了，消费者为啥也渲染？" -> 两层：父渲染子必渲染（通则）+ value 新引用触发广播（Context 特有）
- "setUser 要进 useMemo 依赖吗？" -> 不用，setState 函数引用恒定
- "搜索关键词能放 Context 吗？" -> 不能，高频变化全员陪跑；上状态库订阅切片
- "和 Vue provide/inject 的本质差异？" -> inject 靠依赖收集按需更新；Context 是广播
- "决策原则一句话？" -> Context 的价值=变化频率低，解决"传得远"不解决"变得快"

**一句话类比**
Context 像小区广播喇叭：传话确实不用爬楼了，但喊一次全小区都得醒--所以只适合播"今晚停水"（低频），不能拿来聊天（高频）。
