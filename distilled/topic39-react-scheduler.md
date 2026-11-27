# 调度与时间切片——并发没有魔法

**它解决什么问题**
shouldYield 背后凭什么让？"谁先渲染"怎么裁决？——把可中断骨架变成用户看得见的并发特性。

**核心机制**
- 时间切片 = 长任务切成 N 个宏任务：MessageChannel 接力（零延迟），每片 ≈5ms 后让浏览器喘息
- 任务排序 = 两个小顶堆（taskQueue/timerQueue），堆顶最紧急
- lane = 31 位灯控面板（集合语义）：`pendingLanes |= lane` 开灯、`& renderLanes===0` 灯灭跳过 diff（bailout）
- 插队 = 亮更急的灯 → wip 作废重跑，队列可重放兜底
- useTransition 全部魔法 = 降一档灯（Default→Transition），让 InputContinuous 插队打断

**最容易被追问的点**
- "切片是暂停 JS 吗？" → 不是，是宏任务接力；间隙浏览器自由绘制/处理输入
- "为什么用 MessageChannel？" → setTimeout 最小 4ms+嵌套 clamp 节奏失控；rAF 在绘制前触发等于不让出
- "shouldYield 去掉哪些失效？" → 响应性失效（输入饿死/Transition 打断失效）；正确性都在——批量更新和 bailout 不依赖它
- "自动批处理怎么实现的？" → 18 起合并下沉到调度层：所有 set 进同一调度器，同宏任务一次结算
- "isPending 什么时候 true？" → 绿单已结算未 commit 的整个窗口（含作废重跑期间）

**一句话类比**
Scheduler 是咖啡师每做完一杯看一眼表（shouldYield），订单架自动把加急单顶上来（堆）；lane 是灯控面板——亮几盏灯就是有多少待办，插队就是点亮更红的灯；useTransition 只是给你的单子贴绿色标签：宁可列表晚到，不让输入卡顿。
