# Vue3 Runtime-Core 核心原理深度解析

## 前言

Vue3 的 runtime-core 模块是整个框架的心脏——没了它，Vue 就是一堆死代码。这个模块管的事情很直接：组件怎么 render、怎么更新、怎么调度。听起来简单，但内部弯弯绕绕还挺多的。

这篇文章不搞花架子，只讲事实。

## 组件实例创建

当你写 `createApp().mount('#app')` 这行代码时，Vue 内部其实跑了三个关键步骤：

1. **createComponentInstance()** —— new 了一个组件实例对象，里面装满了 Vue 需要的状态
2. **setupStatefulComponent()** —— 执行你的 setup() 函数，把返回值塞进组件实例
3. **setupRenderEffect()** —— 这才是重头戏，创建了响应式渲染的副作用

简单说，mount 的过程就是：new 实例 → 执行 setup → 启动响应式渲染。

## 虚拟 DOM 与 Diff

Vue3 用虚拟 DOM 来描述真实 DOM，这个做法不算新鲜，但 Vue3 的实现有自己的特点。

### VNode 结构

VNode 是个普通对象，包含 type（标签类型）、props（属性）、children（子节点）等信息。没有魔法，就是个数据结构。

### Diff 算法

Vue3 的 Diff 算法做了三件事：

- 同层比较：只比较同一层级的节点，不跨层级
- key 匹配：通过 key 精准找到对应节点
- LIS 优化：移动节点时用最长递增子序列算法，减少 DOM 操作次数

这套组合拳打下来，比 Vue2 快了不少。

## 调度器

Vue3 用微任务（Promise）实现异步更新。这是故意的——同步更新的话，一个组件变了会触发连锁反应，一个大页面可能直接卡死。

核心流程很简单：

- **queueJob()** 把更新任务推进队列
- **flushJobs()** 在下一个微任务里执行队列里的所有更新
- **递归检查** 确保不会死循环（Vue 内部有保护机制）

## 总结

runtime-core 确实是 Vue3 最核心的部分。理解它怎么跑，debug 时心里有底，写业务时也更清楚自己在做什么。

以上。