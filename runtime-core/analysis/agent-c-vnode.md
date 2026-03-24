# Vue3 Runtime-Core VNode 深度分析报告

> 基于 `packages/runtime-core/src/vnode.ts` (913行) 源码分析

---

## 📑 目录

1. [概述](#1-概述)
2. [VNodeTypes - 节点类型定义](#2-vnodetypes---节点类型定义)
3. [VNode 结构完整字段](#3-vnode-结构完整字段)
4. [ShapeFlags - 形状标记](#4-shapeflags---形状标记)
5. [PatchFlags - 补丁标记](#5-patchflags---补丁标记)
6. [VNode 创建流程](#6-vnode-创建流程)
7. [子节点规范化 - normalizeChildren](#7-子节点规范化---normalizechildren)
8. [核心函数详解](#8-核心函数详解)
9. [块级优化系统](#9-块级优化系统)

---

## 1. 概述

`vnode.ts` 是 Vue3 `runtime-core` 的核心文件，定义了虚拟 DOM 的基础数据结构 VNode（Virtual Node）。VNode 是 Vue 渲染系统的核心抽象，它是一个 JavaScript 对象，用于描述真实的 DOM 节点。

### 文件位置
```
packages/runtime-core/src/vnode.ts
```

### 导出内容
```typescript
// 特殊节点类型
export const Fragment = Symbol.for('v-fgt')
export const Text: unique symbol = Symbol.for('v-txt')
export const Comment: unique symbol = Symbol.for('v-cmt')
export const Static: unique symbol = Symbol.for('v-stc')

// 核心函数
export function createVNode(...)      // 创建虚拟节点
export function createBaseVNode(...)  // 基础虚拟节点创建
export function createElementBlock()  // 创建块级元素节点
export function createBlock()          // 创建块级节点
export function createTextVNode()      // 创建文本节点
export function createStaticVNode()    // 创建静态节点
export function createCommentVNode()  // 创建注释节点
export function cloneVNode()          // 克隆虚拟节点
export function normalizeVNode()      // 规范化虚拟节点
export function normalizeChildren()  // 规范化子节点
export function mergeProps()          // 合并属性
export function isVNode()             // 类型判断
export function isSameVNodeType()     // 比较节点类型
```

---

## 2. VNodeTypes - 节点类型定义

### 类型联合定义 (第 57-68 行)

```typescript
export type VNodeTypes =
  | string                              // HTML 标签名: 'div', 'span' 等
  | VNode                               // 已有 VNode 对象
  | Component                           // 组件选项对象
  | typeof Text                         // 文本节点
  | typeof Static                       // 静态节点
  | typeof Comment                      // 注释节点
  | typeof Fragment                     // 片段节点
  | typeof Teleport                     // Teleport 组件
  | typeof TeleportImpl                 // Teleport 实现
  | typeof Suspense                    // Suspense 组件
  | typeof SuspenseImpl                 // Suspense 实现
```

### VNodeTypes 说明表

| 类型 | 符号 | 说明 |
|------|------|------|
| Element | `string` | HTML/SVG 元素，如 `'div'`, `'span'`, `'a'`, `'input'` |
| Component | `Object` | Vue 组件选项对象或导入的组件 |
| Fragment | `Symbol('v-fgt')` | 片段，用于多个根元素场景 |
| Text | `Symbol('v-txt')` | 文本节点 |
| Comment | `Symbol('v-cmt')` | 注释节点（v-if="false" 时使用） |
| Static | `Symbol('v-stc')` | 静态节点（经过编译优化的静态内容） |
| Teleport | `Symbol` | Teleport 传送门组件 |
| Suspense | `Symbol` | Suspense 异步组件占位 |

---

## 3. VNode 结构完整字段

### VNode 接口定义 (第 123-202 行)

```typescript
export interface VNode<
  HostNode = RendererNode,
  HostElement = RendererElement,
  ExtraProps = { [key: string]: any },
> {
  // ======== 标记属性 ========
  
  /**
   * VNode 标识，用于 isVNode() 判断
   * 值: true
   */
  __v_isVNode: true

  /**
   * 响应式跳过标记
   */
  [ReactiveFlags.SKIP]: true

  // ======== 核心属性 ========
  
  /**
   * 节点类型
   * 可以是字符串(HTML标签)、组件对象、或特殊符号
   */
  type: VNodeTypes

  /**
   * 节点属性 (props)
   * 包含 DOM 属性、事件监听器、指令等
   */
  props: (VNodeProps & ExtraProps) | null

  /**
   * 节点 key
   * 用于 diff 算法中的节点识别
   */
  key: PropertyKey | null

  /**
   * 节点 ref
   * 用于 ref 引用
   */
  ref: VNodeNormalizedRef | null

  /**
   * SFC 作用域 ID (仅 SFC 使用)
   */
  scopeId: string | null

  /**
   * Slot 作用域 ID 列表 (仅 SFC 使用)
   */
  slotScopeIds: string[] | null

  /**
   * 子节点
   */
  children: VNodeNormalizedChildren

  // ======== 组件相关 ========
  
  /**
   * 组件内部实例
   * 如果是组件节点，此属性指向组件实例
   */
  component: ComponentInternalInstance | null

  /**
   * 指令绑定列表
   */
  dirs: DirectiveBinding[] | null

  /**
   * 过渡动画钩子
   */
  transition: TransitionHooks<HostElement> | null

  // ======== DOM 相关 ========
  
  /**
   * 对应的真实 DOM 元素
   * 挂载后填充
   */
  el: HostNode | null

  /**
   * 异步组件的占位符
   */
  placeholder: HostNode | null

  /**
   * 片段锚点
   * 用于 Fragment 的插入位置标记
   */
  anchor: HostNode | null

  /**
   * Teleport 目标容器
   */
  target: HostElement | null

  /**
   * Teleport 目标起始锚点
   */
  targetStart: HostNode | null

  /**
   * Teleport 目标锚点
   */
  targetAnchor: HostElement | null

  /**
   * 静态节点包含的元素数量
   */
  staticCount: number

  // ======== Suspense 相关 ========
  
  /**
   * Suspense 边界
   */
  suspense: SuspenseBoundary | null

  /**
   * Suspense 内容 (异步加载的组件)
   */
  ssContent: VNode | null

  /**
   * Suspense 回退内容 (加载中显示)
   */
  ssFallback: VNode | null

  // ======== 优化标记 ========
  
  /**
   * 形状标记
   * 位运算组合，标识节点类型特征
   */
  shapeFlag: number

  /**
   * 补丁标记
   * 编译器生成的优化提示
   */
  patchFlag: number

  /**
   * 动态属性列表
   */
  dynamicProps: string[] | null

  /**
   * 动态子节点列表
   * 用于块级优化
   */
  dynamicChildren: (VNode[] & { hasOnce?: boolean }) | null

  // ======== 应用上下文 ========
  
  /**
   * 应用上下文
   * 根节点独有，包含 app 配置
   */
  appContext: AppContext | null

  // ======== 词法作用域 ========
  
  /**
   * 词法作用域所有者实例
   */
  ctx: ComponentInternalInstance | null

  // ======== v-memo 相关 ========
  
  /**
   * v-memo 依赖数组
   */
  memo?: any[]

  /**
   * v-memo 缓存索引
   */
  cacheIndex?: number

  // ======== 兼容性 ========
  
  /**
   * 兼容根节点标记
   */
  isCompatRoot?: true

  /**
   * 自定义元素拦截钩子
   */
  ce?: (instance: ComponentInternalInstance) => void
}
```

### VNode 完整字段表

| 字段 | 类型 | 说明 |
|------|------|------|
| `__v_isVNode` | `true` | VNode 标识常量 |
| `__v_skip` | `true` | 响应式跳过标记 |
| `type` | `VNodeTypes` | 节点类型 |
| `props` | `VNodeProps \| null` | 属性对象 |
| `key` | `PropertyKey \| null` | 唯一标识 key |
| `ref` | `VNodeNormalizedRef \| null` | ref 引用 |
| `scopeId` | `string \| null` | SFC 作用域 ID |
| `slotScopeIds` | `string[] \| null` | Slot 作用域 ID 列表 |
| `children` | `VNodeNormalizedChildren` | 子节点 |
| `component` | `ComponentInternalInstance \| null` | 组件实例 |
| `dirs` | `DirectiveBinding[] \| null` | 指令列表 |
| `transition` | `TransitionHooks \| null` | 过渡钩子 |
| `el` | `HostNode \| null` | 真实 DOM 元素 |
| `placeholder` | `HostNode \| null` | 异步占位符 |
| `anchor` | `HostNode \| null` | 片段锚点 |
| `target` | `HostElement \| null` | Teleport 目标 |
| `targetStart` | `HostNode \| null` | Teleport 起始锚点 |
| `targetAnchor` | `HostElement \| null` | Teleport 目标锚点 |
| `staticCount` | `number` | 静态元素计数 |
| `suspense` | `SuspenseBoundary \| null` | Suspense 边界 |
| `ssContent` | `VNode \| null` | Suspense 内容 |
| `ssFallback` | `VNode \| null` | Suspense 回退 |
| `shapeFlag` | `number` | 形状标记 |
| `patchFlag` | `number` | 补丁标记 |
| `dynamicProps` | `string[] \| null` | 动态属性 |
| `dynamicChildren` | `VNode[] \| null` | 动态子节点 |
| `appContext` | `AppContext \| null` | 应用上下文 |
| `ctx` | `ComponentInternalInstance \| null` | 词法作用域实例 |
| `memo` | `any[]` | v-memo 依赖 |
| `cacheIndex` | `number` | v-memo 缓存索引 |
| `isCompatRoot` | `true` | 兼容根标记 |
| `ce` | `Function` | 自定义元素钩子 |

---

## 4. ShapeFlags - 形状标记

`ShapeFlags` 使用**位运算**组合多个类型标识，每个标记占一个比特位。

### 定义 (来自 `packages/shared/src/shapeFlags.ts`)

```typescript
export enum ShapeFlags {
  // 基础类型 (1-4)
  ELEMENT = 1,                    // 0000 0001 = 1
  FUNCTIONAL_COMPONENT = 1 << 1,  // 0000 0010 = 2
  STATEFUL_COMPONENT = 1 << 2,   // 0000 0100 = 4
  TEXT_CHILDREN = 1 << 3,         // 0000 1000 = 8
  
  // 子节点类型 (16-32)
  ARRAY_CHILDREN = 1 << 4,        // 0001 0000 = 16
  SLOTS_CHILDREN = 1 << 5,        // 0010 0000 = 32
  
  // 特殊组件 (64-128)
  TELEPORT = 1 << 6,              // 0100 0000 = 64
  SUSPENSE = 1 << 7,              // 1000 0000 = 128
  
  // 组件生命周期标记 (256-512)
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8,   // 0001 0000 0000 = 256
  COMPONENT_KEPT_ALIVE = 1 << 9,          // 0010 0000 0000 = 512
  
  // 组合标记
  COMPONENT = STATEFUL_COMPONENT | FUNCTIONAL_COMPONENT  // 6
}
```

### ShapeFlags 完整说明表

| 标记 | 值 (十进制) | 二进制 | 说明 |
|------|-------------|--------|------|
| `ELEMENT` | 1 | `0000 0001` | HTML/SVG 元素节点 |
| `FUNCTIONAL_COMPONENT` | 2 | `0000 0010` | 函数式组件 (无状态) |
| `STATEFUL_COMPONENT` | 4 | `0000 0100` | 有状态组件 (标准组件) |
| `TEXT_CHILDREN` | 8 | `0000 1000` | 文本子节点 (字符串) |
| `ARRAY_CHILDREN` | 16 | `0001 0000` | 数组子节点 (VNode[]) |
| `SLOTS_CHILDREN` | 32 | `0010 0000` | Slot 子节点 |
| `TELEPORT` | 64 | `0100 0000` | Teleport 组件 |
| `SUSPENSE` | 128 | `1000 0000` | Suspense 组件 |
| `COMPONENT_SHOULD_KEEP_ALIVE` | 256 | `0001 0000 0000` | 需要 keep-alive 的组件 |
| `COMPONENT_KEPT_ALIVE` | 512 | `0010 0000 0000` | 已被 keep-alive 的组件 |
| `COMPONENT` | 6 | `0000 0110` | 任意组件 (组合) |

### ShapeFlag 使用场景

```typescript
// 判断是否为元素
if (shapeFlag & ShapeFlags.ELEMENT) { ... }

// 判断是否为组件
if (shapeFlag & ShapeFlags.COMPONENT) { ... }

// 判断子节点类型
if (shapeFlag & ShapeFlags.TEXT_CHILDREN) { ... }
if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) { ... }
if (shapeFlag & ShapeFlags.SLOTS_CHILDREN) { ... }

// 组合判断
const shapeFlag = ShapeFlags.ELEMENT | ShapeFlags.ARRAY_CHILDREN
// 表示: 这是一个有数组子节点的元素
```

---

## 5. PatchFlags - 补丁标记

`PatchFlags` 是**编译器生成**的优化提示，用于告诉运行时哪些部分需要更新。

### 定义 (来自 `packages/shared/src/patchFlags.ts`)

```typescript
export enum PatchFlags {
  // ======== 正数标记: 位运算组合 ========
  
  /** 动态文本内容 (children fast path) */
  TEXT = 1,                    // 0000 0001 = 1
  
  /** 动态 class 绑定 */
  CLASS = 1 << 1,              // 0000 0010 = 2
  
  /** 动态 style 绑定 */
  STYLE = 1 << 2,              // 0000 0100 = 4
  
  /** 非 class/style 的动态 props */
  PROPS = 1 << 3,              // 0000 1000 = 8
  
  /** 动态 props 键 (key 变化需完整 diff) */
  FULL_PROPS = 1 << 4,         // 0001 0000 = 16
  
  /** 需要 hydration 的 props */
  NEED_HYDRATION = 1 << 5,     // 0010 0000 = 32
  
  /** 子节点顺序稳定的 Fragment */
  STABLE_FRAGMENT = 1 << 6,    // 0100 0000 = 64
  
  /** 有 key 或部分有 key 的 Fragment */
  KEYED_FRAGMENT = 1 << 7,     // 1000 0000 = 128
  
  /** 无 key 子节点的 Fragment */
  UNKEYED_FRAGMENT = 1 << 8,   // 0001 0000 0000 = 256
  
  /** 仅需非 props 修补 (ref/指令) */
  NEED_PATCH = 1 << 9,         // 0010 0000 0000 = 512
  
  /** 动态 slot 的组件 */
  DYNAMIC_SLOTS = 1 << 10,     // 0100 0000 0000 = 1024
  
  /** 开发环境根片段 (模板根级注释) */
  DEV_ROOT_FRAGMENT = 1 << 11,  // 1000 0000 0000 = 2048
  
  // ======== 负数标记: 特殊标志 (互斥) ========
  
  /** 缓存的静态 vnode */
  CACHED = -1,
  
  /** 跳出优化模式 (完整 diff) */
  BAIL = -2
}
```

### PatchFlags 完整说明表

| 标记 | 值 | 说明 |
|------|-----|------|
| `TEXT` | `1` | 动态文本内容，children 的快速路径 |
| `CLASS` | `2` | 动态 class 绑定 |
| `CLASS` | `4` | 动态 style 绑定 |
| `PROPS` | `8` | 有动态 props（非 class/style） |
| `FULL_PROPS` | `16` | props 键也会变化，需完整 diff |
| `NEED_HYDRATION` | `32` | 需要 hydration 的元素 |
| `STABLE_FRAGMENT` | `64` | 子节点顺序不变的 Fragment |
| `KEYED_FRAGMENT` | `128` | 有 key 子节点的 Fragment |
| `UNKEYED_FRAGMENT` | `256` | 无 key 子节点的 Fragment |
| `NEED_PATCH` | `512` | 只需修补 ref/指令 |
| `DYNAMIC_SLOTS` | `1024` | 动态 slot 的组件 |
| `DEV_ROOT_FRAGMENT` | `2048` | 开发环境根片段 |
| `CACHED` | `-1` | 缓存的静态节点，跳过整个子树 |
| `BAIL` | `-2` | 跳出优化模式，执行完整 diff |

### PatchFlag 使用示例

```vue
<!-- 模板 -->
<div class="foo">{{ text }}</div>

<!-- 编译后的 render 函数 -->
// patchFlag = TEXT | CLASS = 1 | 2 = 3
render() {
  return h('div', { class: 'foo' }, text)
}
```

```typescript
// 运行时处理 patchFlag
if (patchFlag & PatchFlags.TEXT) {
  // 更新 textContent
}
if (patchFlag & PatchFlags.CLASS) {
  // 更新 class
}
if (patchFlag === PatchFlags.BAIL) {
  // 完整 diff
}
```

---

## 6. VNode 创建流程

### 6.1 入口函数: createVNode()

```typescript
// 第 370-372 行
export const createVNode = (
  __DEV__ ? createVNodeWithArgsTransform : _createVNode
) as typeof _createVNode
```

开发环境会使用 `createVNodeWithArgsTransform` 进行参数转换（用于测试工具），生产环境直接使用 `_createVNode`。

### 6.2 核心创建函数: _createVNode()

**位置**: 第 374-429 行

```typescript
function _createVNode(
  type: VNodeTypes | ClassComponent | typeof NULL_DYNAMIC_COMPONENT,
  props: (Data & VNodeProps) | null = null,
  children: unknown = null,
  patchFlag: number = 0,
  dynamicProps: string[] | null = null,
  isBlockNode: boolean = false,
): VNode
```

#### 创建流程详解

```
_createVNode() 执行流程:

┌─────────────────────────────────────────────────────────────┐
│ 1. 类型校验                                                  │
│    - type 为空/null → 警告并设为 Comment                    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. 克隆已有 VNode                                            │
│    - 如果 type 本身就是 VNode (如 <component :is="vnode"/>) │
│    - 调用 cloneVNode 克隆，并合并 props                     │
│    - 设置 patchFlag = BAIL (跳出优化)                       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. 类组件规范化                                              │
│    - 如果是类组件 → 使用 __vccOpts (Vue Class Component)    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. 兼容处理 (2.x)                                            │
│    - 转换遗留函数式/异步组件                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. Class & Style 规范化                                     │
│    - class: 转换为字符串                                      │
│    - style: 转换为字符串/对象，确保非响应式                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. 计算 ShapeFlag                                            │
│    - string → ELEMENT                                        │
│    - Suspense → SUSPENSE                                     │
│    - Teleport → TELEPORT                                     │
│    - Object → STATEFUL_COMPONENT                             │
│    - Function → FUNCTIONAL_COMPONENT                        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. 调用 createBaseVNode 完成创建                             │
│    - needFullChildrenNormalization = true                   │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 基础创建函数: createBaseVNode()

**位置**: 第 319-366 行

```typescript
function createBaseVNode(
  type: VNodeTypes | ClassComponent | typeof NULL_DYNAMIC_COMPONENT,
  props: (Data & VNodeProps) | null = null,
  children: unknown = null,
  patchFlag = 0,
  dynamicProps: string[] | null = null,
  shapeFlag: number = type === Fragment ? 0 : ShapeFlags.ELEMENT,
  isBlockNode = false,
  needFullChildrenNormalization = false,
): VNode
```

#### 创建 VNode 对象

```typescript
const vnode = {
  // 标记
  __v_isVNode: true,
  __v_skip: true,
  
  // 核心属性
  type,
  props,
  key: props && normalizeKey(props),
  ref: props && normalizeRef(props),
  
  // SFC
  scopeId: currentScopeId,
  slotScopeIds: null,
  
  // 子节点
  children,
  
  // 组件
  component: null,
  suspense: null,
  ssContent: null,
  ssFallback: null,
  dirs: null,
  transition: null,
  
  // DOM
  el: null,
  anchor: null,
  target: null,
  targetStart: null,
  targetAnchor: null,
  staticCount: 0,
  
  // 优化标记
  shapeFlag,
  patchFlag,
  dynamicProps,
  dynamicChildren: null,
  
  // 上下文
  appContext: null,
  ctx: currentRenderingInstance,
} as VNode
```

#### 块级跟踪 (Block Tracking)

```typescript
// 满足以下条件时，跟踪此 vnode 为动态节点：
// 1. isBlockTreeEnabled > 0
// 2. 不是块节点本身 (!isBlockNode)
// 3. 有父块 (currentBlock 存在)
// 4. patchFlag > 0 或 是组件节点
// 5. patchFlag 不是 NEED_HYDRATION
if (
  isBlockTreeEnabled > 0 &&
  !isBlockNode &&
  currentBlock &&
  (vnode.patchFlag > 0 || shapeFlag & ShapeFlags.COMPONENT) &&
  vnode.patchFlag !== PatchFlags.NEED_HYDRATION
) {
  currentBlock.push(vnode)  // 添加到动态节点列表
}
```

---

## 7. 子节点规范化 - normalizeChildren()

**位置**: 第 663-710 行

### 函数签名

```typescript
export function normalizeChildren(vnode: VNode, children: unknown): void
```

### 规范化流程

```
normalizeChildren() 执行流程:

┌─────────────────────────────────────────────────────────────┐
│ 输入: vnode, children                                        │
└─────────────────────────────────────────────────────────────┘
                              ↓
         ┌────────────────────────────────────┐
         │ children == null                    │
         │ → children = null                   │
         └────────────────────────────────────┘
                              ↓
         ┌────────────────────────────────────┐
         │ isArray(children)                  │
         │ → type = ARRAY_CHILDREN             │
         └────────────────────────────────────┘
                              ↓
         ┌────────────────────────────────────┐
         │ typeof children === 'object'        │
         │                                    │
         │ 如果是 Element/Teleport:          │
         │   → 处理 slot.default              │
         │ 否则:                              │
         │   → type = SLOTS_CHILDREN          │
         │   → 处理 slotFlag                  │
         └────────────────────────────────────┘
                              ↓
         ┌────────────────────────────────────┐
         │ isFunction(children)               │
         │ → 包装为 { default: fn, _ctx }     │
         │ → type = SLOTS_CHILDREN            │
         └────────────────────────────────────┘
                              ↓
         ┌────────────────────────────────────┐
         │ 其他 (string, number, boolean)     │
         │ → 转为 String(children)            │
         │ → 如果是 Teleport:                 │
         │     type = ARRAY_CHILDREN          │
         │     [createTextVNode()]            │
         │ → 否则:                            │
         │     type = TEXT_CHILDREN           │
         └────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 设置 vnode.children 和 vnode.shapeFlag                       │
└─────────────────────────────────────────────────────────────┘
```

### 子节点类型处理

```typescript
// 1. null/undefined
if (children == null) {
  children = null
}

// 2. 数组 → ARRAY_CHILDREN
else if (isArray(children)) {
  type = ShapeFlags.ARRAY_CHILDREN
}

// 3. 对象 → SLOTS_CHILDREN 或处理 slot
else if (typeof children === 'object') {
  if (shapeFlag & (ShapeFlags.ELEMENT | ShapeFlags.TELEPORT)) {
    // Element/Teleport: 处理 slot.default
    const slot = (children as any).default
    if (slot) {
      normalizeChildren(vnode, slot())
    }
  } else {
    // 组件: SLOTS_CHILDREN
    type = ShapeFlags.SLOTS_CHILDREN
  }
}

// 4. 函数 → SLOTS_CHILDREN
else if (isFunction(children)) {
  children = { default: children, _ctx: currentRenderingInstance }
  type = ShapeFlags.SLOTS_CHILDREN
}

// 5. 其他 (string, number) → TEXT_CHILDREN
else {
  children = String(children)
  if (shapeFlag & ShapeFlags.TELEPORT) {
    type = ShapeFlags.ARRAY_CHILDREN
    children = [createTextVNode(children as string)]
  } else {
    type = ShapeFlags.TEXT_CHILDREN
  }
}

vnode.children = children as VNodeNormalizedChildren
vnode.shapeFlag |= type
```

---

## 8. 核心函数详解

### 8.1 cloneVNode() - 克隆 VNode

**位置**: 第 489-551 行

```typescript
export function cloneVNode<T, U>(
  vnode: VNode<T, U>,
  extraProps?: (Data & VNodeProps) | null,
  mergeRef = false,
  cloneTransition = false,
): VNode<T, U>
```

**功能**:
- 克隆现有 VNode
- 可合并额外属性
- 处理 ref 合并（支持多个 ref）
- 克隆 Suspense 相关内容
- 特殊处理 CACHED 节点

### 8.2 normalizeVNode() - 规范化单个子节点

**位置**: 第 606-621 行

```typescript
export function normalizeVNode(child: VNodeChild): VNode
```

**返回值**:
- `null/undefined/boolean` → Comment 节点
- `Array` → Fragment（保留引用防止污染）
- 已是 VNode → cloneIfMounted()
- 其他 → Text 节点

### 8.3 mergeProps() - 合并属性

**位置**: 第 713-738 行

```typescript
export function mergeProps(...args: (Data & VNodeProps)[]): Data
```

**合并规则**:
- `class`: 数组合并
- `style`: 数组合并
- `onXxx`: 事件监听器数组合并（去重）
- 其他: 后者优先

### 8.4 isVNode() / isSameVNodeType()

**位置**: 第 298-317 行

```typescript
export function isVNode(value: any): value is VNode {
  return value ? value.__v_isVNode === true : false
}

export function isSameVNodeType(n1: VNode, n2: VNode): boolean {
  return n1.type === n2.type && n1.key === n2.key
}
```

---

## 9. 块级优化系统

Vue3 编译器和运行时配合的优化机制，用于减少不必要的 DOM diff。

### 9.1 核心概念

- **Block**: 一个包含动态子节点的代码块
- **Dynamic Children**: 块内的动态节点列表
- **Patch Flag**: 编译器标记的更新提示

### 9.2 块栈 (Block Stack)

```typescript
export const blockStack: VNode['dynamicChildren'][] = []
export let currentBlock: VNode['dynamicChildren'] = null
```

### 9.3 块操作函数

```typescript
// 打开块
export function openBlock(disableTracking = false): void {
  blockStack.push((currentBlock = disableTracking ? null : []))
}

// 关闭块
export function closeBlock(): void {
  blockStack.pop()
  currentBlock = blockStack[blockStack.length - 1] || null
}

// 设置块跟踪
export function setBlockTracking(value: number, inVOnce = false): void {
  isBlockTreeEnabled += value
  // 处理 v-once
}
```

### 9.4 创建块节点

```typescript
// 创建块级元素
export function createElementBlock(...) {
  return setupBlock(createBaseVNode(...))
}

// 创建块级节点
export function createBlock(...) {
  return setupBlock(createVNode(..., true))
}

// 设置块
function setupBlock(vnode: VNode) {
  // 1. 保存动态子节点
  vnode.dynamicChildren = isBlockTreeEnabled > 0 
    ? currentBlock || (EMPTY_ARR as any) 
    : null
  
  // 2. 关闭块
  closeBlock()
  
  // 3. 注册到父块
  if (isBlockTreeEnabled > 0 && currentBlock) {
    currentBlock.push(vnode)
  }
  
  return vnode
}
```

### 9.5 块优化工作流程

```
模板:
┌────────────────────────────────────────────┐
│ <div>                                      │
│   <span>{{ dynamic }}</span>   ← 动态     │
│   <static />                   ← 静态     │
│ </div>                                     │
└────────────────────────────────────────────┘

编译后的 render 函数:
┌────────────────────────────────────────────┐
│ function render() {                        │
│   openBlock()               // 打开块      │
│   return createBlock('div', {               │
│     children: [                             │
│       createVNode('span', null, dynamic,    │
│         patchFlag=1    // TEXT             │
│       ),                                    │
│       createVNode('static', ...)            │
│     ]                                       │
│   })                       // 创建块       │
│ }                                           │
└────────────────────────────────────────────┘

渲染时:
1. openBlock() → 创建新的 currentBlock = []
2. createBlock() → 
   - 创建 div VNode
   - 所有 VNode 都被 push 到 currentBlock
   - currentBlock 赋值给 div.dynamicChildren
3. closeBlock() → 弹出块栈
```

---

## 📊 总结

### VNode 核心要点

1. **VNode 是虚拟 DOM 的核心抽象**
   - JavaScript 对象描述真实 DOM
   - 支持多种节点类型

2. **ShapeFlags 使用位运算**
   - 组合多个类型标识
   - 高效判断节点特征

3. **PatchFlags 是编译优化提示**
   - 标记需要更新的部分
   - 减少运行时 diff 开销

4. **块级优化是 Vue3 核心优化**
   - 编译器生成结构化代码
   - 运行时跳过静态内容

### 文件关键行号

| 功能 | 行号 |
|------|------|
| VNode 接口定义 | 123-202 |
| createVNode 导出 | 370-372 |
| _createVNode 实现 | 374-429 |
| createBaseVNode 实现 | 319-366 |
| cloneVNode 实现 | 489-551 |
| normalizeChildren 实现 | 663-710 |
| isSameVNodeType | 305-317 |
| createBlock | 280-288 |
| openBlock/closeBlock | 214-225 |

---

*报告生成时间: 基于 Vue3 源码分析*
*源码版本: packages/runtime-core/src/vnode.ts*
