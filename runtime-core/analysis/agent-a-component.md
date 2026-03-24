# Vue3 Runtime-Core component.ts 深度分析报告

> 分析文件：~/project/source-code/core/packages/runtime-core/src/component.ts
> 总行数：1297 行

---

## 📋 目录

1. [模块概览](#一模块概览)
2. [核心数据类型定义](#二核心数据类型定义)
3. [核心函数详解](#三核心函数详解)
4. [组件实例创建与渲染流程](#四组件实例创建与渲染流程)
5. [生命周期钩子](#五生命周期钩子)
6. [组件代理机制](#六组件代理机制)
7. [依赖模块关系图](#七依赖模块关系图)
8. [总结](#八总结)

---

## 一、模块概览

### 1.1 文件定位

`component.ts` 是 Vue3 `runtime-core` 包中最核心的文件之一，负责：
- **组件实例的创建** (`createComponentInstance`)
- **组件 setup 逻辑的处理** (`setupComponent`, `setupStatefulComponent`)
- **组件上下文的构建** (SetupContext, ComponentInternalInstance)
- **组件渲染的完成** (`finishComponentSetup`)

### 1.2 主要导出

| 导出项 | 用途 |
|--------|------|
| `ComponentInstance<T>` | 组件实例类型提取工具类型 |
| `ComponentInternalInstance` | 组件内部实例接口 |
| `FunctionalComponent` | 函数式组件类型定义 |
| `ClassComponent` | 类组件类型定义 |
| `ConcreteComponent` | 具体组件类型 |
| `LifecycleHook<T>` | 生命周期钩子类型 |
| `SetupContext` | Setup 上下文类型 |
| `InternalRenderFunction` | 内部渲染函数类型 |
| `createComponentInstance()` | 创建组件实例 |
| `setupComponent()` | 设置组件 |
| `setupStatefulComponent()` | 处理有状态组件 |
| `handleSetupResult()` | 处理 setup 返回值 |
| `finishComponentSetup()` | 完成组件设置 |
| `getCurrentInstance()` | 获取当前实例 |
| `setCurrentInstance()` | 设置当前实例 |
| `getComponentPublicInstance()` | 获取组件公开实例 |

---

## 二、核心数据类型定义

### 2.1 ComponentInternalInstance 接口

这是 Vue3 组件内部实例的核心数据结构，包含 60+ 个属性：

```typescript
export interface ComponentInternalInstance {
  // ============ 基础信息 ============
  uid: number                      // 唯一标识符
  type: ConcreteComponent          // 组件类型定义
  parent: ComponentInternalInstance | null  // 父实例
  root: ComponentInternalInstance  // 根实例
  appContext: AppContext           // 应用上下文
  vnode: VNode                     // 当前组件的 VNode
  next: VNode | null               // 待更新的新 VNode
  subTree: VNode                   // 组件的渲染子树

  // ============ 响应式与更新系统 ============
  effect: ReactiveEffect          // 渲染 effect 实例
  update: () => void               // 强制更新函数
  job: SchedulerJob                // 调度任务
  scope: EffectScope               // 响应式作用域

  // ============ 渲染相关 ============
  render: InternalRenderFunction | null  // 渲染函数
  ssrRender?: Function | null      // SSR 渲染函数
  renderCache: (Function | VNode | undefined)[]  // 渲染缓存

  // ============ 依赖注入系统 ============
  provides: Data                  // 提供的数据

  // ============ useId 相关 ============
  ids: [string, number, number]   // [边界前缀, useId调用索引, 备用]

  // ============ 代理与访问 ============
  accessCache: Data | null        // 属性访问缓存
  proxy: ComponentPublicInstance | null  // 公开实例代理 (this)
  withProxy: ComponentPublicInstance | null  // runtime-compiled 专用代理
  ctx: Data                       // 组件上下文（包含所有响应式属性）

  // ============ 状态数据 ============
  data: Data                      // 组件 data 选项
  props: Data                     // 解析后的 props
  attrs: Data                     // 属性
  slots: InternalSlots           // 插槽
  refs: Data                      // ref 引用
  setupState: Data                // setup 返回的状态
  setupContext: SetupContext | null  // setup 上下文

  // ============ 组件通信 ============
  emit: EmitFn                    // 事件发射函数
  emitted: Record<string, boolean> | null  // 已发射事件记录

  // ============ Props 默认值 ============
  propsDefaults: Data             // props 默认值

  // ============ 组件选项解析 ============
  propsOptions: NormalizedPropsOptions  // 解析后的 props 选项
  emitsOptions: ObjectEmitsOptions | null  // 解析后的 emits 选项
  inheritAttrs?: boolean          // 是否继承属性

  // ============ 局部资源注册 ============
  components: Record<string, ConcreteComponent> | null  // 局部组件
  directives: Record<string, Directive> | null  // 局部指令

  // ============ 暴露与公开 ============
  exposed: Record<string, any> | null   // 通过 expose() 暴露的内容
  exposeProxy: Record<string, any> | null  // 暴露内容的代理

  // ============ Suspense 异步组件 ============
  suspense: SuspenseBoundary | null    // Suspense 边界
  suspenseId: number                   // Suspense ID
  asyncDep: Promise<any> | null        // 异步依赖
  asyncResolved: boolean               // 是否已解析

  // ============ 生命周期状态 ============
  isMounted: boolean          // 是否已挂载
  isUnmounted: boolean        // 是否已卸载
  isDeactivated: boolean      // 是否已停用

  // ============ 生命周期钩子 ============
  bc: LifecycleHook           // beforeCreate
  c: LifecycleHook            // created
  bm: LifecycleHook           // beforeMount
  m: LifecycleHook            // mounted
  bu: LifecycleHook           // beforeUpdate
  u: LifecycleHook            // updated
  um: LifecycleHook          // unmounted
  bum: LifecycleHook          // beforeUnmount
  da: LifecycleHook          // deactivated
  a: LifecycleHook           // activated
  rtg: LifecycleHook          // renderTriggered
  rtc: LifecycleHook          // renderTracked
  ec: LifecycleHook           // errorCaptured
  sp: LifecycleHook           // serverPrefetch

  // ============ 性能优化缓存 ============
  f?: () => void              // $forceUpdate 缓存
  n?: () => Promise<void>    // $nextTick 缓存
  ut?: (vars?: Record<string, unknown>) => void  // teleport CSS 变量更新

  // ============ Custom Element 支持 ============
  ce?: ComponentCustomElementInterface  // 自定义元素实例
  isCE?: boolean              // 是否为自定义元素
  ceReload?: (newStyles?: string[]) => void  // 自定义元素热重载

  // ============ 开发工具 ============
  getCssVars?: () => Record<string, unknown>  // CSS 变量获取
  resolvedOptions?: MergedComponentOptions  // 已解析选项（v2 兼容）
}
```

### 2.2 功能组件类型定义

```typescript
export interface FunctionalComponent<
  P = {},                                    // Props 类型
  E extends EmitsOptions | Record<string, any[]> = {},  // Emits 类型
  S extends Record<string, any> = any,      // Slots 类型
  EE extends EmitsOptions = ShortEmitsToObject<E>,  // 简化的 emits
> extends ComponentInternalOptions {
  // 函数调用签名 - 函数式组件本质上是一个函数
  (
    props: P & EmitsToProps<EE>,
    ctx: Omit<SetupContext<EE, IfAny<S, {}, SlotsType<S>>>, 'expose'>
  ): any
  
  props?: ComponentPropsOptions<P>           // props 选项
  emits?: EE | (keyof EE)[]                 // emits 选项
  slots?: IfAny<S, Slots, SlotsType<S>>    // slots 选项
  inheritAttrs?: boolean                   // 继承属性
  displayName?: string                     // 显示名称
  compatConfig?: CompatConfig              // 兼容性配置
}
```

### 2.3 SetupContext 类型

```typescript
export type SetupContext<
  E = EmitsOptions,
  S extends SlotsType = {},
> = E extends any
  ? {
      attrs: Data                          // 属性
      slots: UnwrapSlotsType<S>           // 插槽
      emit: EmitFn<E>                     // 事件发射
      expose: <Exposed extends Record<string, any> = Record<string, any>>(
        exposed?: Exposed,
      ) => void                           // 暴露公共实例
    }
  : never
```

### 2.4 LifecycleHook 类型

```typescript
export type LifecycleHook<TFn = Function> = (TFn & SchedulerJob)[] | null
```

生命周期钩子是一个数组（存储多个同名钩子）或 null。

### 2.5 InternalRenderFunction 类型

```typescript
export type InternalRenderFunction = {
  (
    ctx: ComponentPublicInstance,
    cache: ComponentInternalInstance['renderCache'],
    $props: ComponentInternalInstance['props'],
    $setup: ComponentInternalInstance['setupState'],
    $data: ComponentInternalInstance['data'],
    $options: ComponentInternalInstance['ctx'],
  ): VNodeChild
  _rc?: boolean  // 是否为 runtime-compiled
  _compatChecked?: boolean  // v3 兼容检查（仅 compat）
  _compatWrapped?: boolean  // v2 兼容包装（仅 compat）
}
```

---

## 三、核心函数详解

### 3.1 createComponentInstance()

**文件位置**：第 605-687 行

**函数签名**：
```typescript
export function createComponentInstance(
  vnode: VNode,
  parent: ComponentInternalInstance | null,
  suspense: SuspenseBoundary | null,
): ComponentInternalInstance
```

**功能**：创建组件内部实例

**参数说明**：
| 参数 | 类型 | 说明 |
|------|------|------|
| vnode | VNode | 组件的虚拟节点 |
| parent | ComponentInternalInstance \| null | 父组件实例 |
| suspense | SuspenseBoundary \| null | Suspense 边界 |

**核心逻辑**：

1. **继承应用上下文**：
   ```typescript
   const appContext = (parent ? parent.appContext : vnode.appContext) || emptyAppContext
   ```

2. **创建实例对象**：初始化所有必要属性
   - uid：递增的唯一标识
   - vnode/type/parent：建立节点关系
   - provides：继承或创建新的 provides 对象（实现依赖注入）
   - ids：继承或初始化 useId 计数器

3. **创建开发模式上下文**：
   ```typescript
   if (__DEV__) {
     instance.ctx = createDevRenderContext(instance)
   } else {
     instance.ctx = { _: instance }
   }
   ```

4. **绑定 emit 函数**：
   ```typescript
   instance.emit = emit.bind(null, instance)
   ```

5. **处理自定义元素**：
   ```typescript
   if (vnode.ce) {
     vnode.ce(instance)
   }
   ```

**关键点**：
- 实例的 `root` 在创建时立即设置为父实例的 root 或自身
- `provides` 使用原型链实现 provide/inject（父 provides 作为原型）
- `ctx` 在开发模式使用特殊处理，生产模式使用 `{ _: instance }` 简洁形式

---

### 3.2 setupComponent()

**文件位置**：第 804-821 行

**函数签名**：
```typescript
export function setupComponent(
  instance: ComponentInternalInstance,
  isSSR = false,
  optimized = false,
): Promise<void> | undefined
```

**功能**：初始化组件的 props、slots，并调用 setup()

**参数说明**：
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| instance | ComponentInternalInstance | - | 组件实例 |
| isSSR | boolean | false | 是否为 SSR 模式 |
| optimized | boolean | false | 是否已优化 |

**核心逻辑**：

1. **设置 SSR 状态**（如果是 SSR）
2. **初始化 props**：`initProps(instance, props, isStateful, isSSR)`
3. **初始化 slots**：`initSlots(instance, children, optimized || isSSR)`
4. **调用 setup**：有状态组件调用 `setupStatefulComponent()`
5. **返回结果**：异步 setup 返回 Promise，同步返回 undefined

**流程图**：
```
setupComponent(instance)
    ↓
[isSSR] setInSSRSetupState(true)
    ↓
initProps() - 初始化 props
    ↓
initSlots() - 初始化 slots
    ↓
isStatefulComponent() - 判断是否是有状态组件
    ↓
    ├─ 是 → setupStatefulComponent(instance, isSSR)
    └─ 否 → undefined (函数组件)
    ↓
[isSSR] setInSSRSetupState(false)
    ↓
返回 setupResult (Promise | undefined)
```

---

### 3.3 setupStatefulComponent()

**文件位置**：第 823-909 行

**函数签名**：
```typescript
function setupStatefulComponent(
  instance: ComponentInternalInstance,
  isSSR: boolean,
): void | Promise<unknown>
```

**功能**：处理有状态组件的 setup() 函数调用

**核心逻辑**：

1. **验证组件名称**（开发模式）
   - 验证 name、components、directives、compilerOptions

2. **创建渲染代理访问缓存**
   ```typescript
   instance.accessCache = Object.create(null)
   ```

3. **创建公开实例代理**
   ```typescript
   instance.proxy = new Proxy(instance.ctx, PublicInstanceProxyHandlers)
   ```

4. **调用 setup() 函数**
   ```typescript
   const setupResult = callWithErrorHandling(
     setup,
     instance,
     ErrorCodes.SETUP_FUNCTION,
     [
       shallowReadonly(instance.props),
       setupContext,
     ],
   )
   ```

5. **处理 setup 返回值**
   - **异步 setup**：返回 Promise
     - SSR：等待 Promise 解析
     - CSR：标记 asyncDep，触发 Suspense
   - **同步 setup**：直接处理

**关键点**：
- setupContext 根据 setup 参数数量决定（1个参数无 context，2个参数有 context）
- 使用 `pauseTracking()` 和 `resetTracking()` 包装 setup 执行
- 异步 setup 会标记 `markAsyncBoundary(instance)`

---

### 3.4 handleSetupResult()

**文件位置**：第 912-943 行

**函数签名**：
```typescript
export function handleSetupResult(
  instance: ComponentInternalInstance,
  setupResult: unknown,
  isSSR: boolean,
): void
```

**功能**：处理 setup() 函数的返回值

**参数说明**：
| 参数 | 类型 | 说明 |
|------|------|------|
| instance | ComponentInternalInstance | 组件实例 |
| setupResult | unknown | setup() 的返回值 |
| isSSR | boolean | 是否为 SSR |

**返回值处理**：

| 返回值类型 | 处理方式 |
|-----------|----------|
| 函数 | 作为渲染函数 `instance.render = setupResult` |
| 对象 | 作为 setup 状态 `instance.setupState = proxyRefs(setupResult)` |
| undefined/null | 跳过（warn in dev） |
| VNode | 警告（不允许直接返回 VNode） |

**特殊处理**：
- SSR 模式下，如果 setup 函数名是 `ssrRender`，设置为 `instance.ssrRender`
- 开发模式保存原始结果到 `devtoolsRawSetupState`
- 使用 `proxyRefs()` 处理 ref（自动解包）

---

### 3.5 finishComponentSetup()

**文件位置**：第 976-1071 行

**函数签名**：
```typescript
export function finishComponentSetup(
  instance: ComponentInternalInstance,
  isSSR: boolean,
  skipOptions?: boolean,
): void
```

**功能**：完成组件设置，包括模板编译和应用选项

**核心逻辑**：

1. **兼容处理**（compat 模式）
   - 转换遗留渲染函数
   - 验证 compatConfig

2. **模板编译**
   - 如果没有 render 函数且有 template，进行运行时编译
   - 从 Component.template 或 inline-template 获取模板
   - 合并全局和组件级编译器选项

3. **安装运行时编译代理**
   ```typescript
   if (installWithProxy) {
     installWithProxy(instance)
   }
   ```

4. **应用选项 API**（如果启用）
   ```typescript
   if (__FEATURE_OPTIONS_API__ && !(__COMPAT__ && skipOptions)) {
     applyOptions(instance)
   }
   ```

5. **警告缺失模板/渲染函数**（开发模式）

---

### 3.6 createSetupContext()

**文件位置**：第 1107-1161 行

**函数签名**：
```typescript
export function createSetupContext(
  instance: ComponentInternalInstance,
): SetupContext
```

**功能**：创建 setup() 函数的第二个参数（上下文）

**返回对象**：
```typescript
{
  attrs: Data,           // 属性（响应式追踪）
  slots: Slots,          // 插槽
  emit: EmitFn,          // 事件发射
  expose: (exposed) => void  // 暴露公共实例
}
```

**关键点**：
- 开发模式使用 `Object.freeze()` 冻结上下文对象
- 开发模式使用 Proxy 追踪 attrs 和 slots 访问
- emit 绑定到实例的 emit 函数

---

### 3.7 getComponentPublicInstance()

**文件位置**：第 1163-1186 行

**函数签名**：
```typescript
export function getComponentPublicInstance(
  instance: ComponentInternalInstance,
): ComponentPublicInstance | ComponentInternalInstance['exposed'] | null
```

**功能**：获取组件的公开实例（用户通过 `this` 访问的对象）

**返回逻辑**：
```
instance.exposed 存在?
  ├─ 是 → 返回 exposeProxy (支持响应式追踪)
  └─ 否 → 返回 instance.proxy (this 代理)
```

**exposeProxy 特点**：
- 使用 `proxyRefs(markRaw(instance.exposed))` 创建
- 同时支持直接属性和公开属性映射（publicPropertiesMap）
- 实现了 get/has  traps 支持属性查找

---

## 四、组件实例创建与渲染流程

### 4.1 完整流程图

```
用户编写组件
       ↓
编译/渲染器创建 VNode(vnode)
       ↓
createComponentInstance(vnode, parent, suspense)
       ↓
┌─────────────────────────────────────────┐
│  初始化 ComponentInternalInstance       │
│  - uid, vnode, type, parent, root      │
│  - appContext, provides                │
│  - propsOptions, emitsOptions         │
│  - ctx (开发/生产模式不同)             │
└─────────────────────────────────────────┘
       ↓
setupComponent(instance, isSSR)
       ↓
┌─────────────────────────────────────────┐
│  initProps(instance, ...)              │  初始化 props
│  initSlots(instance, ...)              │  初始化 slots
└─────────────────────────────────────────┘
       ↓
isStatefulComponent(instance) ?
       ↓
┌──────────────────┐   ┌────────────────┐
│  setupStateful    │   │  函数组件      │
│  Component()      │   │  无 setup      │
└──────────────────┘   └────────────────┘
       ↓
┌─────────────────────────────────────────┐
│  创建 PublicInstanceProxyHandlers       │
│  调用 setup(props, setupContext)       │
│  handleSetupResult()                   │
│  finishComponentSetup()                │
└─────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────┐
│  最终得到完全初始化的组件实例           │
│  - instance.render                     │
│  - instance.proxy (this)                │
│  - instance.setupState                  │
│  - instance.ctx                         │
└─────────────────────────────────────────┘
```

### 4.2 关键阶段说明

**阶段 1：实例创建**
- 仅创建数据结构，不执行用户代码
- 设置基础属性和父子关系
- 创建空的 ctx 对象

**阶段 2：Props/Slots 初始化**
- 从 vnode 提取 props 和 children
- 标准化处理（normalizePropsOptions, normalizeEmitsOptions）
- 创建 slots 对象

**阶段 3：Setup 执行**（有状态组件）
- 创建 setupContext
- 调用 setup 函数
- 处理返回值（函数或对象）
- 设置 render 函数

**阶段 4：完成设置**
- 运行时编译（如需要）
- 应用 Options API（兼容模式）
- 验证配置

---

## 五、生命周期钩子

### 5.1 钩子类型定义

在 ComponentInternalInstance 中，使用简短属性名存储：

| 简写 | 完整名称 | 时机 |
|------|----------|------|
| `bc` | beforeCreate | 创建实例前 |
| `c` | created | 创建实例后 |
| `bm` | beforeMount | 挂载前 |
| `m` | mounted | 挂载后 |
| `bu` | beforeUpdate | 更新前 |
| `u` | updated | 更新后 |
| `bum` | beforeUnmount | 卸载前 |
| `um` | unmounted | 卸载后 |
| `da` | deactivated | 停用（KeepAlive） |
| `a` | activated | 激活（KeepAlive） |
| `rtc` | renderTracked | 渲染追踪 |
| `rtg` | renderTriggered | 渲染触发 |
| `ec` | errorCaptured | 错误捕获 |
| `sp` | serverPrefetch | SSR 预取 |

### 5.2 类型定义

```typescript
export type LifecycleHook<TFn = Function> = (TFn & SchedulerJob)[] | null
```

每个钩子是一个数组（存储多个同名钩子调用）或 null。

---

## 六、组件代理机制

### 6.1 代理结构

Vue3 组件使用多层代理机制：

```
用户访问 this.xxx
       ↓
instance.proxy (PublicInstanceProxyHandlers)
       ↓
instance.ctx (原始数据)
       ↓
instance.props / instance.setupState / instance.data / ...
```

### 6.2 PublicInstanceProxyHandlers

定义在 `componentPublicInstance.ts` 中，核心逻辑：

```typescript
export const PublicInstanceProxyHandlers = {
  get(target, key) {
    // 1. 检查 accessCache 缓存
    // 2. 检查公开属性映射 (publicPropertiesMap)
    // 3. 检查 setupState
    // 4. 检查 data
    // 5. 检查 props
    // 6. 检查 ctx
    // 7. 检查 $开头的内置属性
  },
  set(target, key, value) {
    // 设置到 ctx（用户自定义属性）
  },
  has(target, key) {
    // 检查是否有该属性
  }
}
```

### 6.3 访问缓存 (accessCache)

为了性能，使用 `accessCache` 缓存属性访问类型：

```typescript
// accessCache 值类型
// 0 = public, 1 = setupState, 2 = data, 3 = props, 4 = context
```

避免频繁使用 `hasOwnProperty` 检查。

### 6.4 withProxy（运行时编译）

对于使用 `with` 块的运行时编译渲染函数，使用不同的代理：

```typescript
instance.withProxy = new Proxy(
  instance.ctx,
  RuntimeCompiledPublicInstanceProxyHandlers
)
```

这个代理更高效，只允许白名单内的全局属性穿透。

---

## 七、依赖模块关系图

### 7.1 导入依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| vnode | `./vnode` | VNode 类型 |
| reactivity | `@vue/reactivity` | 响应式工具 |
| componentPublicInstance | `./componentPublicInstance` | 公开实例代理 |
| componentProps | `./componentProps` | Props 处理 |
| componentSlots | `./componentSlots` | Slots 处理 |
| warning | `./warning` | 警告函数 |
| errorHandling | `./errorHandling` | 错误处理 |
| apiCreateApp | `./apiCreateApp` | 应用上下文 |
| directives | `./directives` | 指令验证 |
| componentOptions | `./componentOptions` | 选项应用 |
| componentEmits | `./componentEmits` | 事件处理 |
| shared | `@vue/shared` | 工具函数 |
| components/Suspense | `./components/Suspense` | Suspense 类型 |
| compiler-core | `@vue/compiler-core` | 编译器类型 |
| componentRenderUtils | `./componentRenderUtils` | 渲染工具 |
| componentRenderContext | `./componentRenderContext` | 渲染上下文 |
| profiling | `./profiling` | 性能分析 |
| compat/renderFn | `./compat/renderFn` | 兼容渲染函数 |
| compat/compatConfig | `./compat/compatConfig` | 兼容配置 |
| scheduler | `./scheduler` | 调度器 |
| enums | `./enums` | 枚举定义 |

### 7.2 模块交互

```
component.ts
    ├─→ componentPublicInstance.ts (代理处理器)
    ├─→ componentProps.ts (props 初始化)
    ├─→ componentSlots.ts (slots 初始化)
    ├─→ componentEmits.ts (emit 处理)
    ├─→ componentOptions.ts (选项应用)
    ├─→ errorHandling.ts (错误处理)
    └─→ renderer.ts (使用实例进行渲染)
```

---

## 八、总结

### 8.1 核心要点

1. **组件实例是 Vue3 响应式系统的核心载体**：所有响应式数据、渲染逻辑、生命周期都围绕 ComponentInternalInstance 组织。

2. **Setup 函数是 Composition API 的入口**：通过 setupStatefulComponent 处理 setup 的执行和结果，是 Vue3 区别于 Vue2 的关键设计。

3. **多层代理机制保证性能**：accessCache 缓存、PublicInstanceProxyHandlers 路由、withProxy 优化，构建了高效的属性访问体系。

4. **Provide/Inject 通过原型链实现**：parent.provides 作为原型，子组件的 provides 是新对象，实现跨层级数据共享。

5. **异步组件通过 Suspense 处理**：异步 setup 返回 Promise 时，标记 asyncDep 并等待 Suspense 重新触发渲染。

### 8.2 关键设计模式

| 设计模式 | 应用场景 |
|----------|----------|
| 代理模式 | this 访问、exposeProxy |
| 原型链 | provide/inject |
| 工厂模式 | createComponentInstance |
| 策略模式 | PublicInstanceProxyHandlers vs RuntimeCompiledPublicInstanceProxyHandlers |
| 享元模式 | accessCache 缓存 |
| 责任链 | 属性查找顺序（setup → data → props → ctx） |

### 8.3 与 Vue2 的主要区别

| 特性 | Vue2 | Vue3 |
|------|------|------|
| 组件实例 | Vue.extend 创建类 | createComponentInstance 创建对象 |
| 响应式 | defineProperty Proxy | Proxy |
| Setup | 无 | setup() 函数 |
| 生命周期 | mixins 合并 | setup + 应用选项 |
| 渲染函数 | template 编译 | render 函数优先 |

---

> 本报告基于 Vue3.4+ runtime-core 源码分析
> 分析时间：2026-03-24