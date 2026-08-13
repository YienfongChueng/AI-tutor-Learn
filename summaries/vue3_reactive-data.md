# Vue3 响应式 · 主题块 1：响应式数据 总结

> vue@3.4 · 深度讲解模式（conceptual）· 2026-08-11 通过
> 含子节 1.1-1.6 + 补充串讲。本文包含 ASCII 图示，纯文本环境可读。

## 1. 机制总结图（端到端数据流）

```
  reactive(obj)                              ref(value)
       │                                         │
  createReactiveObject                          RefImpl
  ①isObject ②IS_REACTIVE ③缓存                  │  _value / _rawValue
  ④targetType ⑤new Proxy                        │  .value 访问器(不走Proxy)
       │                                         │
       ▼                                         │
   proxy ──get──> Reflect.get + track            │
                  ├ shallow? no                  │
                  ├ isRef(res)? -> .value 解包   │
                  └ isObject(res)? -> reactive(res) 惰性递归
                                                  │
        ──set──> Reflect.set                     │
                  ├ target===toRaw(receiver)? 防原型链重复
                  ├ hadKey? ADD : SET            │
                  └ hasChanged? -> trigger      │
                                                  │
  四变体: isReadonly × isShallow → createGetter/createSetter
   reactive(深层可改) / readonly(深层只读)
   shallowReactive(浅可改) / shallowReadonly(浅只读)
```

## 2. 源码地图

| 文件 | 关键函数 |
|------|---------|
| `reactivity/src/reactive.ts` | `createReactiveObject` / `reactive` / `readonly` / `shallowReactive` / `getTargetType` / `reactiveMap` |
| `reactivity/src/baseHandlers.ts` | `createGetter` / `createSetter` / `mutableHandlers` / `readonlyHandlers` / `shallowReactiveHandlers` / `shallowReadonlyHandlers` |
| `reactivity/src/ref.ts` | `ref` / `createRef` / `RefImpl` / `toReactive` |

## 3. 设计模式归纳

- **参数化差异**：`createGetter(isReadonly, shallow)` 用两个布尔参数生成四变体 handler，避免四份重复代码。
- **缓存模式**：四个独立 WeakMap（reactiveMap/readonlyMap/...）保证同对象同变体返回同一代理，且不阻 GC。
- **惰性求值**：子对象访问时才 `reactive(res)`，初始化 O(1)，未用深层永不代理。
- **访问器模式**：`RefImpl` 的 `.value` getter/setter 实现 track/trigger，绕过"原始值不能 Proxy"。
- **标记接口**：`ReactiveFlags`（`__v_isReactive` 等）用特殊 key + get trap 返回标记，不污染原始对象。

## 4. 跨主题块关联

- **主题块 1（数据层）的输出**：proxy 的 get/set trap 在读写时调用 `track(target, key)` / `trigger(target, op, key)`。
- **主题块 2（依赖收集与触发）的输入**：`track` 把"当前 effect"收进依赖结构；`trigger` 按 `(target, key)` 找到依赖并执行 effects。
- **接口点**：handler 只负责"何时调 track/trigger"，track/trigger 的内部（依赖存哪、如何找、effect 如何栈式管理）是主题块 2 的核心。
- **预告**：主题块 2 将讲 `effect()` / `ReactiveEffect` / `activeEffect` / `targetMap` 三层结构 / `track` / `trigger` / 嵌套 effect 栈 / `cleanupEffect`。

## 5. 复习方法

看 `distilled/topic13a` 和 `topic13b` 的标题 + "它解决什么问题"后**合上文档默想**核心机制与追问点，想不出再翻 `raw-document/topic13`。
