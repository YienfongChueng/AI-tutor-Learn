# 4.2 react-redux：Provider + useSelector/useDispatch vs connect

**它解决什么问题**
原版 Redux 的 store 是体外对象——React 组件怎么拿它、state 变了谁重渲染？react-redux 只干两件事：注入 + 接神经。

**核心机制**
- Provider：Context 注入 store 实例（与 AuthProvider/StoreProvider 同款接线）
- useSelector：dispatch → reducer 新 state → 通知**所有**订阅者 → **所有 selector 都重跑** → Object.is 比较返回值 → 决定渲染（跑与渲染是两步）
- useDispatch + actionCreator（模块普通函数，**不在 store 树里**，import 直用）
- configureStore({reducer: {...}}) 把多 reducer 合并成树状 state
- 旧 connect：HOC + props 隐式注入 + mapState 对象**默认浅比较**；useSelector 严格 Object.is 要手动 shallowEqual

**最容易被追问的点**
- "selector 返回新对象？" → 3.1 同款反模式：引用恒变全量渲染；修法拆 selector / shallowEqual 第二参
- "没订阅的切片 selector 会跑吗？" → 会！所有 selector 都重跑，只是比较结果决定渲染
- "selector 全重跑算性能问题吗？" → 读对象返回字段是纳秒级；真凶是昂贵计算（大数组 filter/sort）→ reselect createSelector 记忆化，或算好放 store
- "connect 返回对象为什么没事？" → 内置默认浅比较 mapState 结果；useSelector 要手动加
- "actionCreator 能 selector 出来吗？" → 不能，store 树里只有 state

**一句话类比**
Provider 是配电箱进线，useSelector 是各屋自己装的分表（selector 决定看哪路电），dispatch 是提交用电申请——每块表都会抄一次表（selector 全重跑），但只有读数真变了的屋子才开灯（渲染）。
