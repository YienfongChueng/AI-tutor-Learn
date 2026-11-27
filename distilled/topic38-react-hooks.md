# Hooks 实现——状态存哪，为什么不能写在 if 里

**它解决什么问题**
函数组件没有 this 实例，函数体每次重跑局部变量全灭——state 要跨渲染存活，必须有一块"活着的"内存。

**核心机制**
- Hook 链表挂 fiber.memoizedState，节点身份 = **调用顺序**（无名字无 key）
- renderWithHooks 每调一个 hook 拨一格指针；mount/update 是两套 dispatcher
- setX 只贴单：update 挂 hook.queue（循环链表 O(1) 尾插）+ 调度 render
- 结算在下一轮 render：updateReducer 遍历 queue，函数式 action 结算时才执行
- 条件调用 = 跳格，后续全部结构错位（读到别人的 queue），判空 throw 兜不住所有场景

**最容易被追问的点**
- "state 为什么活过渲染？" → 挂在 fiber（堆上跨渲染存活），不在函数局部变量里
- "setX 后 state 变了吗？" → 没变，只是贴单；新值在下轮 render 结算才诞生（快照语义根因）
- "函数式更新好在哪？" → 指令单结算时才执行、吃上一单输出；成品单同基旧值互相覆盖（1.3 更新丢失根因）
- "自定义 hook 有自己的存储吗？" → 没有，展开后占**调用者**链表节点（2.7 非单例根因）
- "为什么不用 key 找格子？" → key 冲突 + API 啰嗦；顺序可在编译期枚举，rules-of-hooks 才能静态拦截

**一句话类比**
fiber 是临时工的固定工位，Hooks 链表是抽屉里**按登记顺序编号**的文件袋；setX 是往袋里塞快递单，真正拆包算钱在下一次 render——柜子只认单号顺序，跳号取到的就是别人的包裹。
