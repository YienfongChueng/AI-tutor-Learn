# 4.8 搜索防抖（useDebounce 接入 Query）

**它解决什么问题**
搜索框每敲一键就发一次请求——请求风暴 + 缓存垃圾条目，防抖把"输入"和"请求"解耦。

**核心机制**
- value 每变一次重跑 effect，cleanup 清掉旧定时器
- 只有停手超过 delay 的输入活到 setDebounced
- 防抖值参与 queryKey，每个搜索词独立缓存条目
- 空串不传 q，条件 spread `...(kw ? {q} : {})`
- `placeholderData: prev=>prev`：换 key 借旧数据，isLoading 不闪

**最容易被追问的点**
- "为什么放防抖值不放原始输入？" → 原始输入每键新 key 新请求，且每个中间串留一条垃圾缓存
- "搜索框文字为什么会消失？" → loading 早退卸载了非受控 SearchBar，内部 state 随实例销毁；受控化（value=rawKeyword）从 props 恢复
- "Hook 为什么不能在 onClick 里调？" → Rules of Hooks 要求顶层同步调用；hook 应返回回调，回调里只调普通函数
- "useDebounce 需要 extends 约束吗？" → 体内只透传 value 不做形状操作，无需约束
- "placeholderData 借数据后 isLoading 还会变 true 吗？" → 不会，新 key 立即 success（借来的），首次加载无旧数据可借才会 loading

**一句话类比**
防抖像电梯关门：有人进（value 变化）就重新计时，一直没人进才真正关门发车（setDebounced）；queryKey 是按目的地分配的轿厢，每个搜索词各坐各的梯。
