# Vue3 Runtime-Core 源码分析

> 分析时间: 2026-03-24
> 源码版本: Vue 3.x

---

## 1. 模块概述

`runtime-core` 是 Vue3 核心运行时模块，负责组件渲染、虚拟 DOM、调度器等核心功能。

**源码目录**: `~/project/source-code/core/packages/runtime-core`

---

## 2. 核心文件分析

### 2.1 component.ts - 组件实例

**核心内容**:
- `createComponentInstance()` - 创建组件实例
- `setupStatefulComponent()` - 处理 Composition API
- `ComponentInternalInstance` - 组件实例数据结构（60+ 字段）
- 生命周期钩子定义
- 组件代理机制

**关键字段**:
```typescript
interface ComponentInternalInstance {
  uid: number
  vnode: VNode
  type: ConcreteComponent
  parent: ComponentInternalInstance | null
  root: ComponentInternalInstance
  render: InternalRenderFunction | null
  proxy: ComponentPublicInstance | null
  setupState: Data
  props: Data
  attrs: Data
  slots: Slots
  isMounted: boolean
  // ... 60+ 字段
}
```

### 2.2 renderer.ts - 渲染器

**核心内容**:
- `createRenderer()` - 创建渲染器
- `mountComponent()` - 组件挂载（6大步骤）
- `updateComponent()` - 组件更新
- `patch()` - VNode 对比算法（Diff）
- `processElement()` - 元素处理
- `processComponent()` - 组件处理

**Diff 算法核心**:
1. 同层比较
2. key 匹配
3. 最长递增子序列（LIS）优化移动
4. 跳过静态节点

### 2.3 vnode.ts - 虚拟节点

**VNode 结构**: 31 个字段

**VNode 类型**:
- `string` - 文本节点
- `VNode` - 元素/组件节点
- `Component` - 组件
- `Fragment` - 片段
- `Text` - 文本
- `Comment` - 注释

**ShapeFlags**:
```typescript
ELEMENT: 1
FUNCTIONAL_COMPONENT: 2
STATEFUL_COMPONENT: 4
TEXT_CHILDREN: 8
ARRAY_CHILDREN: 16
SLOTS_CHILDREN: 32
TELEPORT: 64
SUSPENSE: 128
```

**PatchFlags**:
```typescript
TEXT: 1     // 文本节点
CLASS: 2    // class
STYLE: 3    // style
PROPS: 4    // 非class/style属性
FULL: -1    // 完全diff
BAIL: -2    // 退出优化
```

### 2.4 scheduler.ts - 调度器

**核心队列**:
- `queue` - 主任务队列
- `pendingPostFlushCbs` - 后置回调队列

**调度流程**:
```
数据变化
  → queueJob()
    → 二分查找排序
    → queueFlush()
      → Promise.resolve().then(flushJobs)
        → 按 id 顺序执行
          → checkRecursiveUpdates() 防止无限递归
```

**递归限制**: `RECURSION_LIMIT = 100`

---

## 3. 组件挂载与更新流程

### 3.1 首次挂载

```
createApp()
  → app.mount('#app')
    → createRenderer().createApp()
      → mount()
        → createComponentInstance()
          → setupStatefulComponent()
            → setupRenderEffect()
              → componentUpdateFn()
                → renderComponentRoot()
                  → processComponent()
                    → mountElement()
```

### 3.2 更新流程

```
响应式数据变化
  → trigger effect
    → queueJob()
      → flushJobs()
        → componentUpdateFn()
          → renderComponentRoot()  // 重新执行 render
            → patch()               // Diff 算法
```

---

## 4. 知识图谱

交互式知识图谱请查看：
- `ontology/index.html`

---

## 5. 流程图

流程图请查看：
- `diagrams/flowcharts.html`

---

## 6. 参考资料

- Vue3 源码: `~/project/source-code/core/packages/runtime-core`
- 详细分析报告:
  - `analysis/agent-a-component.md`
  - `analysis/agent-b-renderer.md`
  - `analysis/agent-c-vnode.md`
  - `analysis/agent-d-scheduler.md`

---

*此分析由 AI Agent 团队完成*
