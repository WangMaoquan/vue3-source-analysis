# Vue3 Runtime-Core renderer.ts 深度分析报告

> 分析文件: `~/project/source-code/core/packages/runtime-core/src/renderer.ts`
> 总行数: 2593 行

---

## 目录

1. [文件概述](#1-文件概述)
2. [核心类型定义](#2-核心类型定义)
3. [渲染器选项结构 RendererOptions](#3-渲染器选项结构-rendereroptions)
4. [createRenderer() - 渲染器创建](#4-createrenderer---渲染器创建)
5. [patch() - VNode 对比算法](#5-patch---vnode-对比算法)
6. [processElement - 元素处理](#6-processelement---元素处理)
7. [processComponent - 组件处理](#7-processcomponent---组件处理)
8. [mountComponent - 组件挂载](#8-mountcomponent---组件挂载)
9. [updateComponent - 组件更新](#9-updatecomponent---组件更新)
10. [setupRenderEffect - 渲染副作用设置](#10-setuprendereffect---渲染副作用设置)
11. [Children Patch 算法](#11-children-patch-算法)
12. [移动与卸载](#12-移动与卸载)
13. [涉及的模块/依赖](#13-涉及的模块依赖)

---

## 1. 文件概述

`renderer.ts` 是 Vue3 runtime-core 最核心的文件，负责：
- **VNode 渲染**：将虚拟 DOM 转换为真实 DOM
- **组件生命周期**：挂载、更新、卸载
- **Diff 算法**：高效的子节点对比
- **平台抽象**：支持 DOM、SSR 等不同渲染目标

### 导入的关键模块

| 模块 | 用途 |
|------|------|
| `./vnode` | VNode 创建、克隆、类型判断 |
| `./component` | 组件实例创建、setup |
| `./componentRenderUtils` | 组件渲染工具函数 |
| `@vue/shared` | 共享工具常量 |
| `./scheduler` | 任务调度、队列管理 |
| `@vue/reactivity` | 响应式系统、副作用管理 |
| `./componentProps` | 属性更新处理 |
| `./componentSlots` | 插槽更新处理 |
| `./apiCreateApp` | 应用创建 API |
| `./components/Suspense` | 异步组件挂起机制 |
| `./components/Teleport` |  teleport 传送门 |
| `./components/KeepAlive` | 缓存组件 |
| `./hydration` | SSR 水合函数 |
| `./hmr` | 热更新支持 |
| `./devtools` | 开发者工具集成 |

---

## 2. 核心类型定义

### 2.1 Renderer 接口

```typescript
export interface Renderer<HostElement = RendererElement> {
  render: RootRenderFunction<HostElement>
  createApp: CreateAppFunction<HostElement>
}

export interface HydrationRenderer extends Renderer<Element | ShadowRoot> {
  hydrate: RootHydrateFunction
}
```

### 2.2 根渲染函数

```typescript
export type RootRenderFunction<HostElement = RendererElement> = (
  vnode: VNode | null,
  container: HostElement,
  namespace?: ElementNamespace,
) => void
```

### 2.3 元素命名空间

```typescript
export type ElementNamespace = 'svg' | 'mathml' | undefined
```

---

## 3. 渲染器选项结构 RendererOptions

`RendererOptions` 是渲染器的核心抽象，定义了平台特定的操作：

```typescript
export interface RendererOptions<
  HostNode = RendererNode,
  HostElement = RendererElement,
> {
  // 属性补丁
  patchProp(el, key, prevValue, nextValue, namespace?, parentComponent?): void
  
  // DOM 插入
  insert(el, parent, anchor?): void
  
  // DOM 删除
  remove(el): void
  
  // 创建元素
  createElement(type, namespace?, isCustomizedBuiltIn?, vnodeProps?): HostElement
  
  // 创建文本节点
  createText(text: string): HostNode
  
  // 创建注释节点
  createComment(text: string): HostNode
  
  // 设置文本内容
  setText(node: HostNode, text: string): void
  
  // 设置元素文本
  setElementText(node: HostElement, text: string): void
  
  // 获取父节点
  parentNode(node: HostNode): HostElement | null
  
  // 获取下一个兄弟节点
  nextSibling(node: HostNode): HostNode | null
  
  // 可选：查询选择器
  querySelector?(selector: string): HostElement | null
  
  // 可选：设置 scopeId
  setScopeId?(el: HostElement, id: string): void
  
  // 可选：克隆节点
  cloneNode?(node: HostNode): HostNode
  
  // 可选：插入静态内容
  insertStaticContent?(content, parent, anchor, namespace, start?, end?): [HostNode, HostNode]
}
```

### 解构出的主机函数

在 `baseCreateRenderer` 中解构：

```typescript
const {
  insert: hostInsert,
  remove: hostRemove,
  patchProp: hostPatchProp,
  createElement: hostCreateElement,
  createText: hostCreateText,
  createComment: hostCreateComment,
  setText: hostSetText,
  setElementText: hostSetElementText,
  parentNode: hostParentNode,
  nextSibling: hostNextSibling,
  setScopeId: hostSetScopeId = NOOP,
  insertStaticContent: hostInsertStaticContent,
} = options
```

---

## 4. createRenderer() - 渲染器创建

### 4.1 函数签名

```typescript
export function createRenderer<
  HostNode = RendererNode,
  HostElement = RendererElement,
>(options: RendererOptions<HostNode, HostElement>): Renderer<HostElement>
```

### 4.2 创建流程

```
createRenderer(options)
       │
       ▼
baseCreateRenderer(options)
       │
       ├── 初始化全局状态 (__VUE__ = true)
       ├── 解构 options 为主机函数
       │
       ├── 定义核心函数 (闭包内):
       │    ├── patch: PatchFn
       │    ├── processText: ProcessTextOrCommentFn
       │    ├── processCommentNode
       │    ├── mountStaticNode
       │    ├── patchStaticNode
       │    ├── moveStaticNode
       │    ├── removeStaticNode
       │    ├── processElement
       │    ├── mountElement
       │    ├── patchElement
       │    ├── patchBlockChildren
       │    ├── patchProps
       │    ├── processFragment
       │    ├── processComponent
       │    ├── mountComponent
       │    ├── updateComponent
       │    ├── setupRenderEffect
       │    ├── updateComponentPreRender
       │    ├── patchChildren
       │    ├── patchUnkeyedChildren
       │    ├── patchKeyedChildren
       │    ├── move
       │    ├── unmount
       │    ├── remove
       │    ├── removeFragment
       │    ├── unmountComponent
       │    ├── unmountChildren
       │    └── getNextHostNode
       │
       ├── 创建 internals 对象
       ├── 可选：创建 hydration 函数
       │
       └── 返回 { render, hydrate?, createApp }
```

### 4.3 相关函数

**createHydrationRenderer()** - 创建支持 SSR 水合的渲染器：

```typescript
export function createHydrationRenderer(
  options: RendererOptions<Node, Element>,
): HydrationRenderer {
  return baseCreateRenderer(options, createHydrationFunctions)
}
```

---

## 5. patch() - VNode 对比算法

### 5.1 函数签名

```typescript
type PatchFn = (
  n1: VNode | null,      // 旧 VNode，null 表示挂载
  n2: VNode,             // 新 VNode
  container: RendererElement,
  anchor?: RendererNode | null,
  parentComponent?: ComponentInternalInstance | null,
  parentSuspense?: SuspenseBoundary | null,
  namespace?: ElementNamespace,
  slotScopeIds?: string[] | null,
  optimized?: boolean,
) => void
```

### 5.2 patch 完整流程图

```
patch(n1, n2, container, anchor, parentComponent, ...)
         │
         ▼
    ┌─────────────────┐
    │ n1 === n2?      │──Yes──▶ return (相同节点，跳过)
    └────────┬────────┘
             │ No
             ▼
    ┌─────────────────────────────┐
    │ isSameVNodeType(n1, n2)?     │
    └─────────────┬───────────────┘
                  │
       ┌──────────┴──────────┐
       │ No                 │ Yes
       ▼                    ▼
  ┌──────────────┐    ┌─────────────┐
  │ 获取下一个   │    │ 检查 patchFlag │
  │ 锚点 anchor │    │ = BAIL?     │
  │ unmount(n1) │    └──────┬──────┘
  │ n1 = null   │           │
  └──────────────┘    ┌──────┴──────┐
                      │ 是         │ 否
                      ▼            ▼
               ┌──────────┐  ┌─────────────┐
               │ 取消优化 │  │ switch(type)│
               │ optimized│  └──────┬──────┘
               │ = false  │         │
               └──────────┘    ┌─────┴──────┐
                              ▼            ▼
                         Text        Comment
                              │            │
                              ▼            ▼
                         Static    Fragment
                              │            │
                              ▼            ▼
                   ┌─────────────────────┐
                   │ default:            │
                   │  - ELEMENT          │
                   │  - COMPONENT        │
                   │  - TELEPORT          │
                   │  - SUSPENSE         │
                   └─────────────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ 处理 ref            │
                   └─────────────────────┘
```

### 5.3 VNode 类型处理

```typescript
switch (type) {
  case Text:
    processText(n1, n2, container, anchor)
    break
  case Comment:
    processCommentNode(n1, n2, container, anchor)
    break
  case Static:
    // 静态节点处理
    if (n1 == null) {
      mountStaticNode(n2, container, anchor, namespace)
    }
    break
  case Fragment:
    processFragment(...)
    break
  default:
    if (shapeFlag & ShapeFlags.ELEMENT) {
      processElement(...)
    } else if (shapeFlag & ShapeFlags.COMPONENT) {
      processComponent(...)
    } else if (shapeFlag & ShapeFlags.TELEPORT) {
      TeleportImpl.process(...)
    } else if (shapeFlag & ShapeFlags.SUSPENSE) {
      SuspenseImpl.process(...)
    }
}
```

### 5.4 ref 处理

```typescript
// 设置 ref
if (ref != null && parentComponent) {
  setRef(ref, n1 && n1.ref, parentSuspense, n2 || n1, !n2)
} 
// 清除旧 ref
else if (ref == null && n1 && n1.ref != null) {
  setRef(n1.ref, null, parentSuspense, n1, true)
}
```

---

## 6. processElement - 元素处理

### 6.1 函数签名

```typescript
const processElement = (
  n1: VNode | null,
  n2: VNode,
  container: RendererElement,
  anchor: RendererNode | null,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  namespace: ElementNamespace,
  slotScopeIds: string[] | null,
  optimized: boolean,
) => void
```

### 6.2 处理流程

```
processElement(n1, n2, ...)
         │
         ▼
   ┌─────────────┐
   │ 设置命名空间  │── svg ──▶ namespace = 'svg'
   │ (n2.type)   │── math ──▶ namespace = 'mathml'
   └──────┬──────┘
          │
          ▼
   ┌──────────────┐
   │ n1 == null?  │──Yes──▶ mountElement(...)
   └──────┬───────┘
          │ No
          ▼
   ┌──────────────────────────┐
   │ 检查自定义元素           │
   │ n1.el._isVueCE          │
   └────────────┬────────────┘
               │
               ▼
   ┌──────────────────────────┐
   │ patchElement(...)        │
   └──────────────────────────┘
```

### 6.3 mountElement - 元素挂载

```typescript
const mountElement = (
  vnode,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  namespace,
  slotScopeIds,
  optimized,
) => {
  // 1. 创建元素
  el = vnode.el = hostCreateElement(vnode.type, namespace, props?.is, props)
  
  // 2. 处理子节点
  if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
    hostSetElementText(el, vnode.children)
  } else if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
    mountChildren(...)
  }
  
  // 3. 指令处理
  if (dirs) invokeDirectiveHook(vnode, null, parentComponent, 'created')
  
  // 4. 设置 scopeId
  setScopeId(el, vnode, vnode.scopeId, slotScopeIds, parentComponent)
  
  // 5. 设置 props
  for (key in props) {
    if (key !== 'value' && !isReservedProp(key)) {
      hostPatchProp(el, key, null, props[key], namespace, parentComponent)
    }
  }
  // 特殊处理 value 属性
  if ('value' in props) {
    hostPatchProp(el, 'value', null, props.value, namespace)
  }
  
  // 6. 插入 DOM
  hostInsert(el, container, anchor)
  
  // 7. 触发生命周期钩子
  queuePostRenderEffect(() => {
    // onVnodeMounted
    // transition.enter
    // dirs mounted
  }, parentSuspense)
}
```

### 6.4 patchElement - 元素更新

```typescript
const patchElement = (
  n1: VNode,
  n2: VNode,
  parentComponent,
  parentSuspense,
  namespace,
  slotScopeIds,
  optimized,
) => {
  const el = (n2.el = n1.el)
  
  // 1. 触发 beforeUpdate 钩子
  if (newProps.onVnodeBeforeUpdate) {
    invokeVNodeHook(vnodeHook, parentComponent, n2, n1)
  }
  
  // 2. 处理动态子节点
  if (dynamicChildren) {
    patchBlockChildren(n1.dynamicChildren!, dynamicChildren, ...)
  } else if (!optimized) {
    patchChildren(...) // full diff
  }
  
  // 3. 根据 patchFlag 处理 props
  if (patchFlag > 0) {
    if (patchFlag & PatchFlags.FULL_PROPS) {
      patchProps(el, oldProps, newProps, ...)
    } else {
      // CLASS - 动态 class
      if (patchFlag & PatchFlags.CLASS) { ... }
      
      // STYLE - 动态 style
      if (patchFlag & PatchFlags.STYLE) { ... }
      
      // PROPS - 动态 props
      if (patchFlag & PatchFlags.PROPS) { ... }
    }
    
    // TEXT - 动态文本
    if (patchFlag & PatchFlags.TEXT) { ... }
  } else if (!optimized) {
    patchProps(...) // full diff
  }
  
  // 4. 触发 updated 钩子
  queuePostRenderEffect(() => {
    // onVnodeUpdated
    // dirs updated
  }, parentSuspense)
}
```

### 6.5 patchBlockChildren - 块级子节点对比

用于优化模式的快速对比：

```typescript
const patchBlockChildren = (
  oldChildren,
  newChildren,
  fallbackContainer,
  parentComponent,
  parentSuspense,
  namespace,
  slotScopeIds,
) => {
  for (let i = 0; i < newChildren.length; i++) {
    const oldVNode = oldChildren[i]
    const newVNode = newChildren[i]
    
    // 确定容器
    const container = oldVNode.el && (
      oldVNode.type === Fragment ||
      !isSameVNodeType(oldVNode, newVNode) ||
      oldVNode.shapeFlag & (ShapeFlags.COMPONENT | ShapeFlags.TELEPORT | ShapeFlags.SUSPENSE)
    ) ? hostParentNode(oldVNode.el) : fallbackContainer
    
    patch(oldVNode, newVNode, container, ...)
  }
}
```

### 6.6 patchProps - 属性对比

```typescript
const patchProps = (
  el: RendererElement,
  oldProps: Data,
  newProps: Data,
  parentComponent,
  namespace,
) => {
  // 1. 移除旧属性
  for (key in oldProps) {
    if (!isReservedProp(key) && !(key in newProps)) {
      hostPatchProp(el, key, oldProps[key], null, namespace, parentComponent)
    }
  }
  
  // 2. 更新/添加新属性
  for (key in newProps) {
    if (isReservedProp(key)) continue
    const next = newProps[key]
    const prev = oldProps[key]
    if (next !== prev && key !== 'value') {
      hostPatchProp(el, key, prev, next, namespace, parentComponent)
    }
  }
  
  // 3. 特殊处理 value
  if ('value' in newProps) {
    hostPatchProp(el, 'value', oldProps.value, newProps.value, namespace)
  }
}
```

---

## 7. processComponent - 组件处理

### 7.1 函数签名

```typescript
const processComponent = (
  n1: VNode | null,
  n2: VNode,
  container: RendererElement,
  anchor: RendererNode | null,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  namespace: ElementNamespace,
  slotScopeIds: string[] | null,
  optimized: boolean,
) => void
```

### 7.2 处理流程

```
processComponent(n1, n2, ...)
         │
         ▼
   ┌─────────────────┐
   │ n2.slotScopeIds │
   │ = slotScopeIds  │
   └────────┬────────┘
            │
            ▼
   ┌──────────────────┐
   │ n1 == null?      │
   └────────┬─────────┘
            │
     ┌──────┴──────┐
     │ Yes         │ No
     ▼             ▼
┌────────────┐  ┌─────────────┐
│ n2.shapeFlag│  │updateComponent│
│ COMPONENT  │  │  (n1, n2)   │
│ _KEPT_ALIVE│  └─────────────┘
│?           │
└─────┬──────┘
      │
      ▼
┌────────────────┐
│ KeepAlive      │
│ .activate()    │
└───────┬────────┘
        │
        ▼
┌────────────────┐
│ mountComponent │
└────────────────┘
```

---

## 8. mountComponent - 组件挂载

### 8.1 函数签名

```typescript
export type MountComponentFn = (
  initialVNode: VNode,
  container: RendererElement,
  anchor: RendererNode | null,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  namespace: ElementNamespace,
  optimized: boolean,
) => void
```

### 8.2 挂载流程图

```
mountComponent(initialVNode, container, anchor, ...)
         │
         ▼
   ┌─────────────────────────────┐
   │ 创建组件实例                │
   │ createComponentInstance() │
   └─────────────┬───────────────┘
                 │
                 ▼
   ┌─────────────────────────────┐
   │ HMR 注册 (dev only)        │
   └─────────────┬───────────────┘
                 │
                 ▼
   ┌─────────────────────────────┐
   │ 注入 KeepAlive 渲染器       │
   │ (如果是 KeepAlive)         │
   └─────────────┬───────────────┘
                 │
                 ▼
   ┌─────────────────────────────┐
   │ setupComponent()           │
   │  - 处理 props              │
   │  - 处理 slots              │
   │  - 执行 setup()            │
   └─────────────┬───────────────┘
                 │
                 ▼
   ┌─────────────────────────────┐
   │ 检查异步依赖                │
   │ instance.asyncDep?         │
   └─────────────┬───────────────┘
                 │
     ┌───────────┴───────────┐
     │ 是                    │ 否
     ▼                       ▼
┌────────────┐     ┌─────────────────────┐
│ 注册到     │     │ setupRenderEffect()│
│ Suspense  │     │ - 创建响应式 effect │
│ 创建占位符 │     │ - 执行首次渲染     │
└────────────┘     └─────────────────────┘
```

### 8.3 详细步骤

```typescript
const mountComponent = (
  initialVNode,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  namespace,
  optimized,
) => {
  // 1. 创建组件实例
  const instance = compatMountInstance || 
    (initialVNode.component = createComponentInstance(
      initialVNode,
      parentComponent,
      parentSuspense,
    ))

  // 2. HMR 注册
  if (__DEV__ && instance.type.__hmrId) {
    registerHMR(instance)
  }

  // 3. KeepAlive 注入渲染器
  if (isKeepAlive(initialVNode)) {
    instance.ctx.renderer = internals
  }

  // 4. Setup 组件
  setupComponent(instance, false, optimized)

  // 5. 异步组件处理
  if (__FEATURE_SUSPENSE__ && instance.asyncDep) {
    parentSuspense.registerDep(instance, setupRenderEffect, optimized)
    // 创建占位符
    if (!initialVNode.el) {
      const placeholder = createVNode(Comment)
      processCommentNode(null, placeholder, container, anchor)
    }
  } else {
    // 6. 设置渲染 effect
    setupRenderEffect(instance, initialVNode, container, anchor, ...)
  }
}
```

---

## 9. updateComponent - 组件更新

### 9.1 函数签名

```typescript
const updateComponent = (n1: VNode, n2: VNode, optimized: boolean) => void
```

### 9.2 更新流程

```
updateComponent(n1, n2, optimized)
         │
         ▼
   ┌──────────────────────────┐
   │ instance = n2.component  │
   │ = n1.component          │
   └────────────┬─────────────┘
                │
                ▼
   ┌──────────────────────────┐
   │ shouldUpdateComponent()  │
   │ 检查是否需要更新          │
   └────────────┬─────────────┘
                │
        ┌───────┴───────┐
        │ 是            │ 否
        ▼               ▼
┌────────────────┐  ┌────────────────┐
│ 检查异步组件    │  │ n2.el = n1.el  │
│ instance.      │  │ instance.vnode │
│ asyncDep?      │  │ = n2           │
└───────┬────────┘  └────────────────┘
        │
        ▼
┌───────────────────────────┐
│ 异步组件未解析?           │
│ (asyncDep && !resolved)  │
└────────────┬──────────────┘
      ┌─────┴─────┐
      │ 是        │ 否
      ▼           ▼
┌────────────┐  ┌──────────────────┐
│ 更新 props │  │ instance.next=n2 │
│ 和 slots   │  │ instance.update()│
│ (不渲染)   │  │ 触发响应式 effect│
└────────────┘  └──────────────────┘
```

### 9.3 详细代码

```typescript
const updateComponent = (n1: VNode, n2: VNode, optimized: boolean) => {
  const instance = (n2.component = n1.component)!
  
  if (shouldUpdateComponent(n1, n2, optimized)) {
    // 异步组件处理
    if (__FEATURE_SUSPENSE__ && instance.asyncDep && !instance.asyncResolved) {
      updateComponentPreRender(instance, n2, optimized)
      return
    }
    
    // 正常更新
    instance.next = n2
    instance.update() // 触发响应式 effect
  } else {
    // 不需要更新，仅复制属性
    n2.el = n1.el
    instance.vnode = n2
  }
}
```

### 9.4 updateComponentPreRender

```typescript
const updateComponentPreRender = (
  instance: ComponentInternalInstance,
  nextVNode: VNode,
  optimized: boolean,
) => {
  nextVNode.component = instance
  instance.vnode = nextVNode
  instance.next = null
  
  // 更新 props
  updateProps(instance, nextVNode.props, prevProps, optimized)
  
  // 更新 slots
  updateSlots(instance, nextVNode.children, optimized)
  
  // 刷新 pre watchers
  flushPreFlushCbs(instance)
}
```

---

## 10. setupRenderEffect - 渲染副作用设置

### 10.1 函数签名

```typescript
export type SetupRenderEffectFn = (
  instance: ComponentInternalInstance,
  initialVNode: VNode,
  container: RendererElement,
  anchor: RendererNode | null,
  parentSuspense: SuspenseBoundary | null,
  namespace: ElementNamespace,
  optimized: boolean,
) => void
```

### 10.2 核心逻辑

这是 Vue3 响应式渲染的核心，创建了组件的渲染 effect：

```typescript
const setupRenderEffect = (
  instance,
  initialVNode,
  container,
  anchor,
  parentSuspense,
  namespace,
  optimized,
) => {
  // 创建组件更新函数
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      // === 首次挂载 ===
      
      // 1. beforeMount 钩子
      if (bm) invokeArrayFns(bm)
      
      // 2. onVnodeBeforeMount
      if (props.onVnodeBeforeMount) { ... }
      
      // 3. 渲染组件根
      const subTree = renderComponentRoot(instance)
      
      // 4. patch (挂载子树的 DOM)
      patch(null, subTree, container, anchor, instance, ...)
      
      // 5. 触发 mounted 钩子
      if (m) queuePostRenderEffect(m, parentSuspense)
      
      // 6. onVnodeMounted
      if (props.onVnodeMounted) { ... }
      
      // 7. activated 钩子 (KeepAlive)
      if (shapeFlag & COMPONENT_SHOULD_KEEP_ALIVE) { ... }
      
      instance.isMounted = true
      
    } else {
      // === 更新阶段 ===
      
      // 1. 获取 next (新的 vnode)
      let { next, bu, u, parent, vnode } = instance
      
      // 2. 更新 props/slots
      if (next) {
        next.el = vnode.el
        updateComponentPreRender(instance, next, optimized)
      } else {
        next = vnode
      }
      
      // 3. beforeUpdate 钩子
      if (bu) invokeArrayFns(bu)
      
      // 4. onVnodeBeforeUpdate
      if (next.props.onVnodeBeforeUpdate) { ... }
      
      // 5. 渲染新子树
      const nextTree = renderComponentRoot(instance)
      const prevTree = instance.subTree
      
      // 6. patch 对比新旧子树
      patch(prevTree, nextTree, hostParentNode(prevTree.el!), ...)
      
      // 7. 触发 updated 钩子
      if (u) queuePostRenderEffect(u, parentSuspense)
      
      // 8. onVnodeUpdated
      if (next.props.onVnodeUpdated) { ... }
    }
  }

  // 创建响应式 effect
  instance.scope.on()
  const effect = new ReactiveEffect(componentUpdateFn)
  instance.scope.off()

  // 创建更新函数
  const update = (instance.update = effect.run.bind(effect))
  
  // 设置调度器
  effect.scheduler = () => queueJob(job)
  
  // 首次执行
  update()
}
```

### 10.3 流程图

```
setupRenderEffect(...)
         │
         ▼
   ┌────────────────────────┐
   │ 创建 componentUpdateFn │
   │ (组件更新函数)          │
   └───────────┬────────────┘
                │
                ▼
   ┌────────────────────────┐
   │ 创建 ReactiveEffect    │
   │ instance.effect        │
   └───────────┬────────────┘
                │
                ▼
   ┌────────────────────────┐
   │ 创建 update 函数        │
   │ instance.update        │
   └───────────┬────────────┘
                │
                ▼
   ┌────────────────────────┐
   │ 设置调度器             │
   │ effect.scheduler       │
   └───────────┬────────────┘
                │
                ▼
   ┌────────────────────────┐
   │ 执行 update()          │
   │ 首次渲染                │
   └────────────────────────┘
```

---

## 11. Children Patch 算法

### 11.1 patchChildren - 子节点对比入口

```typescript
const patchChildren = (
  n1,
  n2,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  namespace,
  slotScopeIds,
  optimized = false,
) => {
  const c1 = n1.children
  const c2 = n2.children
  const { patchFlag, shapeFlag } = n2

  // 快速路径：有 patchFlag
  if (patchFlag > 0) {
    if (patchFlag & PatchFlags.KEYED_FRAGMENT) {
      patchKeyedChildren(...)
      return
    } else if (patchFlag & PatchFlags.UNKEYED_FRAGMENT) {
      patchUnkeyedChildren(...)
      return
    }
  }

  // 普通对比
  if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
    // 文本 vs 数组
    if (prevShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
      unmountChildren(c1, ...)
    }
    if (c2 !== c1) {
      hostSetElementText(container, c2)
    }
  } else {
    if (prevShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
      if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
        patchKeyedChildren(...)
      } else {
        unmountChildren(c1, ...)
      }
    } else {
      // text -> array 或 null
      if (prevShapeFlag & ShapeFlags.TEXT_CHILDREN) {
        hostSetElementText(container, '')
      }
      if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
        mountChildren(c2, ...)
      }
    }
  }
}
```

### 11.2 patchUnkeyedChildren - 无 key 子节点对比

简单的一一对应对比：

```typescript
const patchUnkeyedChildren = (
  c1: VNode[],
  c2: VNodeArrayChildren,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  namespace,
  slotScopeIds,
  optimized,
) => {
  const oldLength = c1.length
  const newLength = c2.length
  const commonLength = Math.min(oldLength, newLength)

  // 1. 对比共同部分
  for (i = 0; i < commonLength; i++) {
    patch(c1[i], c2[i], ...)
  }

  // 2. 移除多余的旧节点
  if (oldLength > newLength) {
    unmountChildren(c1, ..., commonLength)
  } 
  // 3. 挂载多余的新节点
  else {
    mountChildren(c2, ..., commonLength)
  }
}
```

### 11.3 patchKeyedChildren - 有 key 子节点对比（核心 Diff 算法）

这是 Vue3 高效 Diff 算法的核心，使用了四步算法：

```typescript
const patchKeyedChildren = (
  c1: VNode[],    // 旧子节点
  c2: VNodeArrayChildren, // 新子节点
  container,
  parentAnchor,
  parentComponent,
  parentSuspense,
  namespace,
  slotScopeIds,
  optimized,
) => {
  let i = 0
  const l2 = c2.length
  let e1 = c1.length - 1  // 旧末尾索引
  let e2 = l2 - 1         // 新末尾索引

  // === 步骤 1: 从头部同步 ===
  // (a b) c  →  (a b) d e
  while (i <= e1 && i <= e2) {
    if (isSameVNodeType(n1, n2)) {
      patch(n1, n2, ...)
    } else {
      break
    }
    i++
  }

  // === 步骤 2: 从尾部同步 ===
  // a (b c) → d e (b c)
  while (i <= e1 && i <= e2) {
    if (isSameVNodeType(n1, n2)) {
      patch(n1, n2, ...)
    } else {
      break
    }
    e1--
    e2--
  }

  // === 步骤 3: 同步后新增 ===
  // (a b) → (a b) c
  if (i > e1) {
    if (i <= e2) {
      // 挂载新节点
      while (i <= e2) {
        patch(null, c2[i], ...)
        i++
      }
    }
  }

  // === 步骤 4: 同步后删除 ===
  // (a b) c → (a b)
  else if (i > e2) {
    while (i <= e1) {
      unmount(c1[i], ...)
      i++
    }
  }

  // === 步骤 5: 未知序列 ===
  // a b [c d e] f g → a b [e d c h] f g
  else {
    const s1 = i  // 旧开始
    const s2 = i  // 新开始

    // 5.1 构建 key → newIndex 映射
    const keyToNewIndexMap = new Map()
    for (i = s2; i <= e2; i++) {
      keyToNewIndexMap.set(c2[i].key, i)
    }

    // 5.2 遍历旧节点，尝试匹配
    const newIndexToOldIndexMap = new Array(toBePatched)
    for (i = 0; i < toBePatched; i++) newIndexToOldIndexMap[i] = 0

    for (i = s1; i <= e1; i++) {
      // 查找新位置
      newIndex = keyToNewIndexMap.get(prevChild.key) || 找到同类型无key节点
      
      if (newIndex === undefined) {
        unmount(prevChild, ...) // 移除
      } else {
        // 标记映射
        newIndexToOldIndexMap[newIndex - s2] = i + 1
        // patch
        patch(prevChild, c2[newIndex], ...)
        patched++
      }
    }

    // 5.3 移动和挂载
    // 计算最长递增子序列
    const increasingNewIndexSequence = moved 
      ? getSequence(newIndexToOldIndexMap) 
      : EMPTY_ARR
    
    // 倒序遍历，从后往前插入
    for (i = toBePatched - 1; i >= 0; i--) {
      const nextIndex = s2 + i
      const nextChild = c2[nextIndex]
      
      if (newIndexToOldIndexMap[i] === 0) {
        // 挂载新节点
        patch(null, nextChild, ...)
      } else if (moved) {
        // 移动
        if (j < 0 || i !== increasingNewIndexSequence[j]) {
          move(nextChild, container, anchor, MoveType.REORDER)
        } else {
          j--
        }
      }
    }
  }
}
```

### 11.4 Diff 算法图解

```
旧: [A, B, C, D, E]
新: [A, B, F, C, D, E]

步骤1 - 头部同步:
  A vs A ✓ → patch
  B vs B ✓ → patch
  C vs F ✗ → 停止
  
  i=2, e1=4, e2=4

步骤2 - 尾部同步:
  E vs E ✓ → patch
  D vs D ✓ → patch
  C vs C ✓ → patch
  
  i=2, e1=1, e2=1

步骤3 - 头部新增:
  i > e1? No (2 > 1)
  
步骤4 - 尾部删除:
  i > e2? No (2 > 1)

步骤5 - 未知序列:
  s1=2, e1=1 (无节点)
  s2=2, e2=1 (无节点)
  → 不执行

结果: 保持 A, B 不动，C,D,E 重新挂载（实际是 patch）

---

更复杂例子:
旧: [A, B, C, D, E]
新: [A, F, B, C, E, D]

步骤1: A vs A ✓ → patch, i=1
步骤2: E vs E ✓, D vs D ✓ → patch, e1=2, e2=2

剩余:
旧: [B, C, D] (索引 1-3)
新: [F, B, C] (索引 1-3)

步骤5:
- 构建 keyToNewIndexMap: {F:1, B:2, C:3}
- 遍历旧节点:
  B -> newIndex=2, patch, newIndexToOldIndexMap[0]=2
  C -> newIndex=3, patch, newIndexToOldIndexMap[1]=3  
  D -> newIndex=undefined, unmount

移动计算:
newIndexToOldIndexMap = [2, 3, 0]
LIS = [1, 2] (对应索引 1,2，即 B,C)

从后往前:
- i=2: C, 需要移动到 D 之前? C 在索引3，新位置是3
- i=1: B, 需要移动? B 在索引2，新位置是2
- i=0: F, newIndexToOldIndexMap[0]=0, 挂载新节点
```

### 11.5 getSequence - 最长递增子序列

用于最小化 DOM 移动：

```typescript
function getSequence(arr: number[]): number[] {
  const p = arr.slice()
  const result = [0]
  // 二分查找找递增序列
  for (let i = 0; i < len; i++) {
    if (arrI !== 0) {
      if (arr[result[last]] < arrI) {
        p[i] = result[last]
        result.push(i)
        continue
      }
      // 二分查找
      u = 0, v = result.length - 1
      while (u < v) {
        c = (u + v) >> 1
        if (arr[result[c]] < arrI) {
          u = c + 1
        } else {
          v = c
        }
      }
      if (arrI < arr[result[u]]) {
        if (u > 0) p[i] = result[u - 1]
        result[u] = i
      }
    }
  }
  // 回溯
  u = result.length
  while (u-- > 0) {
    result[u] = v
    v = p[v]
  }
  return result
}
```

---

## 12. 移动与卸载

### 12.1 move - 移动 VNode

```typescript
const move = (
  vnode,
  container,
  anchor,
  moveType,
  parentSuspense = null,
) => {
  const { el, type, transition, children, shapeFlag } = vnode

  // 组件
  if (shapeFlag & ShapeFlags.COMPONENT) {
    move(vnode.component.subTree, container, anchor, moveType)
    return
  }

  // Suspense
  if (shapeFlag & ShapeFlags.SUSPENSE) {
    vnode.suspense.move(container, anchor, moveType)
    return
  }

  // Teleport
  if (shapeFlag & ShapeFlags.TELEPORT) {
    TeleportImpl.move(vnode, container, anchor, internals)
    return
  }

  // Fragment
  if (type === Fragment) {
    // 移动所有子节点
    for (let i = 0; i < children.length; i++) {
      move(children[i], container, anchor, moveType)
    }
    return
  }

  // Static
  if (type === Static) {
    moveStaticNode(vnode, container, anchor)
    return
  }

  // Transition
  const needTransition = moveType !== MoveType.REORDER && 
                         shapeFlag & ShapeFlags.ELEMENT && transition

  if (needTransition) {
    if (moveType === MoveType.ENTER) {
      transition.beforeEnter(el)
      hostInsert(el, container, anchor)
      queuePostRenderEffect(() => transition.enter(el), parentSuspense)
    } else {
      // LEAVE
      transition.leave(el, () => {
        hostRemove(el)
      })
    }
  } else {
    hostInsert(el, container, anchor)
  }
}
```

### 12.2 unmount - 卸载 VNode

```typescript
const unmount = (
  vnode,
  parentComponent,
  parentSuspense,
  doRemove = false,
  optimized = false,
) => {
  const { type, props, ref, children, dynamicChildren, shapeFlag, dirs, cacheIndex } = vnode

  // 1. 清除 ref
  if (ref != null) {
    setRef(ref, null, parentSuspense, vnode, true)
  }

  // 2. 清除缓存
  if (cacheIndex != null) {
    parentComponent.renderCache[cacheIndex] = undefined
  }

  // 3. KeepAlive 组件
  if (shapeFlag & ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE) {
    KeepAliveContext.deactivate(vnode)
    return
  }

  // 4. 调用 beforeUnmount 钩子
  if (props.onVnodeBeforeUnmount) {
    invokeVNodeHook(props.onVnodeBeforeUnmount, parentComponent, vnode)
  }

  // 5. 卸载
  if (shapeFlag & ShapeFlags.COMPONENT) {
    unmountComponent(vnode.component, parentSuspense, doRemove)
  } else if (shapeFlag & ShapeFlags.SUSPENSE) {
    vnode.suspense.unmount(...)
  } else if (shouldInvokeDirs) {
    invokeDirectiveHook(vnode, null, parentComponent, 'beforeUnmount')
  } else if (shapeFlag & ShapeFlags.TELEPORT) {
    TeleportImpl.remove(...)
  } else if (dynamicChildren) {
    // 快速路径：只卸载动态子节点
    unmountChildren(dynamicChildren, ...)
  } else if (type === Fragment || shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
    unmountChildren(children, ...)
  }

  // 6. 移除 DOM
  if (doRemove) {
    remove(vnode)
  }

  // 7. 触发 unmounted 钩子
  queuePostRenderEffect(() => {
    // onVnodeUnmounted
    // dirs unmounted
  }, parentSuspense)
}
```

### 12.3 unmountComponent - 卸载组件

```typescript
const unmountComponent = (
  instance: ComponentInternalInstance,
  parentSuspense,
  doRemove,
) => {
  // HMR 注销
  if (__DEV__ && instance.type.__hmrId) {
    unregisterHMR(instance)
  }

  // beforeUnmount 钩子
  if (bum) invokeArrayFns(bum)

  // 停止响应式 scope
  scope.stop()

  // 调度任务标记为已处理
  if (job) {
    job.flags |= SchedulerJobFlags.DISPOSED
    // 卸载子树
    unmount(subTree, instance, parentSuspense, doRemove)
  }

  // unmounted 钩子
  if (um) queuePostRenderEffect(um, parentSuspense)

  // 标记已卸载
  queuePostRenderEffect(() => {
    instance.isUnmounted = true
  }, parentSuspense)

  // devtools
  devtoolsComponentRemoved(instance)
}
```

---

## 13. 涉及的模块/依赖

### 13.1 内部模块

| 模块 | 关键导出 |
|------|----------|
| `./vnode` | `VNode`, `createVNode`, `isSameVNodeType`, `normalizeVNode`, `cloneIfMounted`, `invokeVNodeHook` |
| `./component` | `ComponentInternalInstance`, `createComponentInstance`, `setupComponent` |
| `./componentRenderUtils` | `renderComponentRoot`, `shouldUpdateComponent`, `updateHOCHostEl` |
| `./componentProps` | `updateProps` |
| `./componentSlots` | `updateSlots` |
| `./scheduler` | `queueJob`, `queuePostFlushCb`, `flushPostFlushCbs`, `flushPreFlushCbs` |
| `./apiCreateApp` | `createAppAPI` |
| `./components/Suspense` | `queueEffectWithSuspense`, `isSuspense`, `SuspenseBoundary` |
| `./components/Teleport` | `TeleportImpl`, `TeleportVNode`, `TeleportEndKey` |
| `./components/KeepAlive` | `isKeepAlive`, `KeepAliveContext` |
| `./components/BaseTransition` | `TransitionHooks`, `leaveCbKey` |
| `./hydration` | `createHydrationFunctions`, `RootHydrateFunction` |
| `./directives` | `invokeDirectiveHook` |
| `./hmr` | `isHmrUpdating`, `registerHMR`, `unregisterHMR` |
| `./devtools` | `setDevtoolsHook`, `devtoolsComponentAdded/Removed/Updated` |
| `./warning` | `warn`, `pushWarningContext`, `popWarningContext` |
| `./profiling` | `startMeasure`, `endMeasure` |
| `./featureFlags` | `initFeatureFlags` |
| `./apiAsyncComponent` | `isAsyncWrapper` |
| `./rendererTemplateRef` | `setRef` |
| `./compat/compatConfig` | `isCompatEnabled`, `DeprecationTypes` |

### 13.2 外部模块 (@vue/shared)

| 导出 | 用途 |
|------|------|
| `EMPTY_ARR` | 空数组常量 |
| `EMPTY_OBJ` | 空对象常量 |
| `NOOP` | 空函数 |
| `PatchFlags` | Patch 标志枚举 |
| `ShapeFlags` | VNode 形状标志 |
| `def` | 定义响应式属性 |
| `getGlobalThis` | 获取全局对象 |
| `invokeArrayFns` | 执行函数数组 |
| `isArray` | 数组检查 |
| `isReservedProp` | 保留属性检查 |

### 13.3 外部模块 (@vue/reactivity)

| 导出 | 用途 |
|------|------|
| `ReactiveEffect` | 响应式副作用类 |
| `EffectFlags` | 副作用标志 |
| `pauseTracking` | 暂停依赖追踪 |
| `resetTracking` | 重置追踪 |

---

## 附录：PatchFlags 详解

| 标志 | 值 | 含义 |
|------|-----|------|
| `TEXT` | 1 | 只有文本子节点变化 |
| `CLASS` | 2 | 只有 class 变化 |
| `STYLE` | 4 | 只有 style 变化 |
| `PROPS` | 8 | 非 class/style 的 props 变化 |
| `FULL_PROPS` | 16 | 有动态 key 的 props 变化 |
| `KEYED_FRAGMENT` | 32 | 带 key 的 fragment |
| `UNKEYED_FRAGMENT` | 64 | 不带 key 的 fragment |
| `NEED_HYDRATION` | 128 | 需要水合 |
| `STABLE_FRAGMENT` | 256 | 稳定的 fragment |
| `DEV_ROOT_FRAGMENT` | 512 | 开发根 fragment |
| `BAIL` | -2 | 降级到完整 diff |

---

## 总结

`renderer.ts` 是 Vue3 响应式系统的核心实现，包含了：

1. **渲染器创建**：`createRenderer()` 工厂函数，创建平台特定的渲染器
2. **VNode 对比**：`patch()` 函数是整个渲染流程的核心
3. **组件生命周期**：`mountComponent` → `setupComponent` → `setupRenderEffect`
4. **高效 Diff**：四步算法 + LIS 最长递增子序列优化
5. **平台抽象**：通过 `RendererOptions` 适配不同渲染目标（DOM、SSR等）

理解这个文件是掌握 Vue3 响应式原理的关键。
