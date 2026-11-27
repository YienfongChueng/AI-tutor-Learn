# 2.2 Mobx React 接线（五步全链路）

**它解决什么问题**
心智模型会了但不知道文件怎么摆——Mobx 没有 CLI 约定，接线是"自由度换纪律"的重灾区。

**核心机制**
- 五步：类定义（makeAutoObservable）→ RootStore 组合（子 store 构造器传 this）→ Context 注入（useStore 判空 throw）→ Provider 包裹 → observer 消费
- 跨 store：子 store 持有 root 引用，派生值直接读兄弟 store 的 computed，零仪式
- makeAutoObservable(this, overrides, {autoBind}) 三参分工：target / 排除表 / 自动绑定
- observer 追踪 render 内属性访问——**没有 storeToRefs 问题**（与 Pinia 的模型差异）
- autoBind 的价值：解构 action 当 onClick 回调不丢 this

**最容易被追问的点**
- "makeObservable 和 makeAutoObservable 区别？" → 前者手动完整标注映射，后者自动标注+overrides 排除表；排除表语义只在 Auto 版
- "onClick 直传 action 拿参数行吗？" → 参数是 MouseEvent；业务快照从 store 内部取（toJS(this.root.records.records)）
- "异步 action 的纪律？" → await 前可直改（还在 action 作用域），await 后必须 runInAction
- "toJS 什么时候用？" → 发 API / JSON.stringify / 深拷贝前摘 Proxy 壳；boolean/number 上用是滥用
- "为什么不用 useState 同步 loading？" → observable 就是渲染状态，本地 state 会造双份真相

**一句话类比**
接线像给房子通水电：makeAutoObservable 是总闸、observer 是各屋开关、runInAction 是电工作业票、RootStore 是配电箱——三闸一箱，少一个灯就不亮。
