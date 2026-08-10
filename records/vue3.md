---
mode: deep-dive
start_date: 2026-08-10
target_depth: conceptual
source_version: "vue@3.4"
nodes:
  "1.1":
    name: 为什么用 Proxy（vs Vue2 defineProperty）
    module: 响应式数据
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: Vue2 defineProperty 的根本缺陷是什么，Proxy 如何解决
  "1.2":
    name: reactive()：Proxy 创建 + handler 总览
    module: 响应式数据
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: reactive 如何用 Proxy 包裹对象（createReactiveObject 流程）
  "1.3":
    name: baseHandlers：get 触发 track、set 触发 trigger
    module: 响应式数据
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: get/set 拦截器如何与依赖系统衔接
  "1.4":
    name: ref()：为什么原始值不能 Proxy + 实现
    module: 响应式数据
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: ref 为何存在，原始值响应式如何实现
  "1.5":
    name: 深层响应式：惰性递归代理
    module: 响应式数据
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 子对象何时被代理，惰性代理的性能好处
  "1.6":
    name: readonly / shallow 变体
    module: 响应式数据
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 不同响应式变体的 handler 差异
  "2.1":
    name: effect()：ReactiveEffect 与 activeEffect
    module: 依赖收集与触发
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: effect 与 ReactiveEffect 类、activeEffect 的关系
  "2.2":
    name: targetMap 三层结构
    module: 依赖收集与触发
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: WeakMap->Map->Set 三层结构为什么这样设计
  "2.3":
    name: track()：当前 effect 如何进 dep
    module: 依赖收集与触发
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: track 如何把当前 activeEffect 收进 dep
  "2.4":
    name: trigger()：找到 dep 并执行 effects
    module: 依赖收集与触发
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: trigger 如何按 key 找到 dep 并执行 effects
  "2.5":
    name: 嵌套 effect 与 activeEffect 栈
    module: 依赖收集与触发
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: 嵌套 effect 如何靠 activeEffect 栈正确收集
  "2.6":
    name: cleanupEffect 与避免无限循环
    module: 依赖收集与触发
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: cleanupEffect 的作用，如何避免重复/无限执行
  "3.1":
    name: computed()：懒执行 + dirty flag + 缓存
    module: 派生状态与调度
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: computed 的懒执行 + dirty + 缓存如何配合
  "3.2":
    name: computed 既是 effect 又是 dep
    module: 派生状态与调度
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: computed 为何双向收集（作为 effect 订阅，作为 dep 被订阅）
  "3.3":
    name: watch()：effect + scheduler 拿新旧值
    module: 派生状态与调度
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: watch 如何用 effect + scheduler 拿新旧值
  "3.4":
    name: scheduler 与 queueJob 批处理
    module: 派生状态与调度
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: scheduler 与 queueJob 如何合并多次更新
  "3.5":
    name: nextTick：Promise.resolve.then 微任务
    module: 派生状态与调度
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: nextTick 用微任务实现的意义
  "3.6":
    name: flush 时机与组件渲染衔接
    module: 派生状态与调度
    status: pending
    attempts: 0
    last_tested: null
    mastery_level: 0
    core_question: pre/post/sync 三种 flush 时机与组件渲染如何衔接
---

# Vue3 响应式原理/源码 学习记录

> 深度讲解模式 · target_depth: conceptual（设计思想+核心机制，关键处给源码定位）· 参考 vue@3.4

## 学习日志

| 日期 | 节点 | 结果 | 备注 |
|------|------|------|------|
