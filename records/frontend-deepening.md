---
type: roadmap
created: 2026-08-10
plan: 前端深化
blocks:
  vue3:
    name: Vue3 深入（响应式原理/源码）
    mode: deep-dive
    order: 1
    status: paused
    paused_at: 2026-08-17
    paused_note: 主题块1已通过；主题块2讲到2.4（未考核），恢复时先补读再讲2.5
    record: vue3.md
  react:
    name: React（基础复盘/hooks深入/工程化/实战/源码原理）
    mode: mixed
    order: 2
    status: learning
    started_at: 2026-08-17
    record: react.md
  state:
    name: 数据流（Pinia/Mobx/Zustand）
    mode: knowledge
    order: 3
    status: pending
    record: state-management.md
  build:
    name: 构建工具（Vite/Webpack 原理）
    mode: deep-dive
    order: 4
    status: pending
    record: build-tools.md
---

# 前端深化路线图

> 创建于 2026-08-10。TypeScript 主题已 100% 完成（见 records/typescript.md），进入前端深化阶段。

## 学习顺序与状态

| 序 | 主题 | 模式 | 状态 | 进度记录 |
|----|------|------|------|---------|
| 1 | Vue3 深入（响应式原理/源码） | 深度讲解 | 🔄 学习中 | vue3.md |
| 2 | React（基础复盘/hooks/工程化/实战/源码） | 混合 | ⬜ 待开始 | react.md |
| 3 | 数据流（Pinia/Mobx/Zustand） | 知识 | ⬜ 待开始 | state-management.md |
| 4 | 构建工具（Vite/Webpack 原理） | 深度讲解 | ⬜ 待开始 | build-tools.md |

## 自动推进规则（给未来的我）

1. 启动 `/ai-tutor` 时，先读本文件 `blocks`，找到第一个 `status != mastered` 的块。
2. 若该块 `status: learning`，读取对应 `record` 文件恢复进度，继续教学。
3. 若该块 `status: pending`，开始新块：建对应 record 文件，status 改 learning。
4. 每块完成（该块所有节点 mastered）后：本块 status -> mastered，下一块 status -> learning。
5. 全部 4 块 mastered = 前端深化阶段完成，可进入「后端补足」阶段（见全栈学习路径）。
6. 间隔复习（records/typescript.md 等）仍按 mastery_level 独立运行，到期时穿插复习。

## 模式说明
- **深度讲解模式**（Vue3源码 / 构建工具原理）：源码/原理级，讲透机制，单节长输出，按「最小可闭环机制」推进，统一考核。
- **混合模式**（React）：基础复盘/hooks/工程化/实战用知识模式，源码原理用深度讲解模式。
- **知识模式**（数据流）：学会用 + 理解机制，理论精简重实践。

## 备注
- 用户背景：中级前端/Vue3/有后端思维/目标全栈。
- 用户要求：路线图持久化，学完一块自动开始下一块，避免跨 session 丢上下文。
