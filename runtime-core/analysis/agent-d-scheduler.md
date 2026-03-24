# Vue3 Scheduler 调度器深度分析报告

> 分析文件：`packages/runtime-core/src/scheduler.ts`
> 版本：Vue3 Runtime-Core

---

## 一、概述

Vue3 的调度器（Scheduler）是响应式系统的核心基础设施，负责管理和执行组件更新、watch 回调、生命周期钩子等任务。它确保：

1. **批处理**：将多次数据变更合并为一次 DOM 更新
2. **优先级排序**：父组件先于子组件更新
3. **异步执行**：在微任务（microtask）中批量执行，避免阻塞主线程
4. **防无限递归**：检测并阻止组件的无限更新循环

---

## 二、核心数据结构

### 2.1 任务队列

```typescript
const queue: SchedulerJob[] = []           // 主任务队列（组件更新）
let flushIndex = -1                         // 当前执行位置

const pendingPostFlushCbs: SchedulerJob[] = []  // 后置回调队列
let activePostFlushCbs: SchedulerJob[] | null = null  // 正在执行的后置回调
let postFlushIndex = 0                      // 后置回调执行位置
```

| 队列 | 用途 | 执行时机 |
|------|------|----------|
| `queue` | 组件更新任务 (update) | 微任务中按优先级顺序执行 |
| `pendingPostFlushCbs` | 后置回调（mounted, updated） | 组件更新完成后执行 |

### 2.2 Promise 基础

```typescript
const resolvedPromise = /*@__PURE__*/ Promise.resolve() as Promise<any>
let currentFlushPromise: Promise<void> | null = null
```

- `resolvedPromise`：已解决的 Promise，用于调度微任务
- `currentFlushPromise`：当前正在执行的 flush promise

### 2.3 递归限制

```typescript
const RECURSION_LIMIT = 100
type CountMap = Map<SchedulerJob, number>
```

允许同一任务在单次 flush 周期内最多执行 100 次，超过则抛出警告。

---

## 三、任务标记 (SchedulerJobFlags)

```typescript
export enum SchedulerJobFlags {
  QUEUED = 1 << 0,        // 0001 - 已加入队列
  PRE = 1 << 1,           // 0010 - 预 flush 标记（如 pre watcher）
  ALLOW_RECURSE = 1 << 2, // 0100 - 允许递归触发
  DISPOSED = 1 << 3,      // 1000 - 已销毁/跳过
}
```

### 标记说明

| 标记 | 含义 | 使用场景 |
|------|------|----------|
| `QUEUED` | 任务已在队列中 | 防止重复入队 |
| `PRE` | 预执行任务 | pre watcher，需要在组件更新前执行 |
| `ALLOW_RECURSE` | 允许递归 | 组件更新函数、watch 回调 |
| `DISPOSED` | 已废弃 | 组件已卸载，跳过执行 |

### 为什么需要 ALLOW_RECURSE？

> 代码注释解释：
> - `Array.prototype.push` 实际上会触发读取操作（#1740），可能导致无限循环
> - 组件更新函数可能修改子组件的 props，触发 parent 依赖的 watch（#1801）
> - watch 回调不追踪依赖，如果递归触发通常是用户有意为之（#1727）

---

## 四、任务接口 (SchedulerJob)

```typescript
export interface SchedulerJob extends Function {
  id?: number              // 任务唯一标识（决定优先级）
  flags?: SchedulerJobFlags // 任务标记
  i?: ComponentInternalInstance // 关联的组件实例（用于错误报告）
}
```

---

## 五、核心函数详解

### 5.1 nextTick() - 下一个微任务

```typescript
export function nextTick(): Promise<void>
export function nextTick<T, R>(
  this: T,
  fn?: (this: T) => R | Promise<R>,
): Promise<void | R>
```

**作用**：返回一个 Promise，在调度器完成当前/下一次 flush 后 resolve。

**实现逻辑**：

```typescript
export function nextTick<T, R>(
  this: T,
  fn?: (this: T) => R | Promise<R>,
): Promise<void | R> {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}
```

- 如果当前有 flush 正在进行，返回 `currentFlushPromise`
- 否则返回已解决的 `resolvedPromise`
- 如果传入回调函数 `fn`，则在其后执行

**使用场景**：

```javascript
// 等待 DOM 更新完成后获取最新值
nextTick(() => {
  console.log(domElement.textContent) // 已更新
})
```

---

### 5.2 findInsertionIndex() - 二分查找插入位置

```typescript
function findInsertionIndex(id: number) {
  let start = flushIndex + 1
  let end = queue.length

  while (start < end) {
    const middle = (start + end) >>> 1
    const middleJob = queue[middle]
    const middleJobId = getId(middleJob)
    if (
      middleJobId < id ||
      (middleJobId === id && middleJob.flags! & SchedulerJobFlags.PRE)
    ) {
      start = middle + 1
    } else {
      end = middle
    }
  }

  return start
}
```

**作用**：使用二分查找在队列中找到正确的插入位置，保证**升序排列**。

**排序规则**：

1. `id` 小的在前（父组件 id < 子组件 id）
2. 同 id 时，带 `PRE` 标记的在前

**为什么需要排序？**

- **父先子后**：父组件在子组件之前创建，id 更小
- **跳过已卸载**：如果父组件更新时子组件已卸载，可以跳过子组件更新
- **pre watcher 优先**：pre watcher 需要在组件更新前执行

**getId 实现**：

```typescript
const getId = (job: SchedulerJob): number =>
  job.id == null 
    ? (job.flags! & SchedulerJobFlags.PRE ? -1 : Infinity) 
    : job.id
```

- 无 id 的 pre 任务：id = -1（最优先）
- 无 id 的普通任务：id = Infinity（最后执行）

---

### 5.3 queueJob() - 添加任务到队列

```typescript
export function queueJob(job: SchedulerJob): void {
  // 1. 检查是否已在队列中
  if (!(job.flags! & SchedulerJobFlags.QUEUED)) {
    const jobId = getId(job)
    const lastJob = queue[queue.length - 1]
    
    // 2. 快速路径：id 比队列最后一个大且不是 PRE 任务
    if (
      !lastJob ||
      (!(job.flags! & SchedulerJobFlags.PRE) && jobId >= getId(lastJob))
    ) {
      queue.push(job)
    } else {
      // 3. 慢速路径：需要二分查找插入位置
      queue.splice(findInsertionIndex(jobId), 0, job)
    }

    // 4. 标记为已入队
    job.flags! |= SchedulerJobFlags.QUEUED

    // 5. 触发 flush
    queueFlush()
  }
}
```

**优化策略**：

- **快速路径**：如果新任务的 id 比队列最后一个大，直接 push（O(1)）
- **慢速路径**：否则进行二分查找插入（O(log n)）
- **去重**：通过 `QUEUED` 标记防止重复入队

---

### 5.4 queueFlush() - 触发 flush

```typescript
function queueFlush() {
  if (!currentFlushPromise) {
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```

**关键点**：

- 使用 `Promise.then()` 将 `flushJobs` 放入**微任务队列**
- 只在第一次调用时创建 Promise（`currentFlushPromise` 为空时）
- 多次 `queueJob` 只会触发一次 microtask

---

### 5.5 queuePostFlushCb() - 添加后置回调

```typescript
export function queuePostFlushCb(cb: SchedulerJobs): void {
  if (!isArray(cb)) {
    // 单个回调
    if (activePostFlushCbs && cb.id === -1) {
      // 嵌套调用：插入到当前执行位置的后面
      activePostFlushCbs.splice(postFlushIndex + 1, 0, cb)
    } else if (!(cb.flags! & SchedulerJobFlags.QUEUED)) {
      pendingPostFlushCbs.push(cb)
      cb.flags! |= SchedulerJobFlags.QUEUED
    }
  } else {
    // 数组：生命周期钩子（已去重，直接添加）
    pendingPostFlushCbs.push(...cb)
  }
  queueFlush()
}
```

**去重逻辑**：

- 生命周期钩子数组在主队列中已去重，这里跳过检查以提高性能
- 单个回调通过 `QUEUED` 标记去重

---

### 5.6 flushPreFlushCbs() - 执行预置回调

```typescript
export function flushPreFlushCbs(
  instance?: ComponentInternalInstance,
  seen?: CountMap,
  i: number = flushIndex + 1,
): void {
  for (; i < queue.length; i++) {
    const cb = queue[i]
    // 1. 筛选 PRE 标记的任务
    if (cb && cb.flags! & SchedulerJobFlags.PRE) {
      // 2. 如果指定了 instance，只执行该组件的任务
      if (instance && cb.id !== instance.uid) {
        continue
      }
      // 3. 检查递归限制
      if (__DEV__ && checkRecursiveUpdates(seen!, cb)) {
        continue
      }
      // 4. 从队列移除并执行
      queue.splice(i, 1)
      i--
      // 5. 处理 ALLOW_RECURSE 标记
      if (cb.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
      cb()
      if (!(cb.flags! & SchedulerJobFlags.ALLOW_RECURSE)) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
    }
  }
}
```

**作用**：执行带有 `PRE` 标记的任务（如 pre watcher）

---

### 5.7 flushPostFlushCbs() - 执行后置回调

```typescript
export function flushPostFlushCbs(seen?: CountMap): void {
  if (pendingPostFlushCbs.length) {
    // 1. 去重并按 id 排序
    const deduped = [...new Set(pendingPostFlushCbs)].sort(
      (a, b) => getId(a) - getId(b),
    )
    pendingPostFlushCbs.length = 0

    // 2. 处理嵌套调用
    if (activePostFlushCbs) {
      activePostFlushCbs.push(...deduped)
      return
    }

    // 3. 执行回调
    activePostFlushCbs = deduped
    for (
      postFlushIndex = 0;
      postFlushIndex < activePostFlushCbs.length;
      postFlushIndex++
    ) {
      const cb = activePostFlushCbs[postFlushIndex]
      if (__DEV__ && checkRecursiveUpdates(seen!, cb)) {
        continue
      }
      if (cb.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
      // 跳过已 DISPOSED 的回调
      if (!(cb.flags! & SchedulerJobFlags.DISPOSED)) cb()
      cb.flags! &= ~SchedulerJobFlags.QUEUED
    }
    activePostFlushCbs = null
    postFlushIndex = 0
  }
}
```

**关键特性**：

1. **去重**：使用 `Set` 去除重复回调
2. **排序**：按 id 升序执行
3. **嵌套处理**：如果已经有 active 的后置回调，追加到末尾
4. **DISPOSED 跳过**：已卸载组件的钩子不执行

---

### 5.8 flushJobs() - 主 flush 函数

```typescript
function flushJobs(seen?: CountMap) {
  // 1. 开发模式：创建递归检查函数
  const check = __DEV__
    ? (job: SchedulerJob) => checkRecursiveUpdates(seen!, job)
    : NOOP

  try {
    // 2. 执行主队列中的所有任务
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job && !(job.flags! & SchedulerJobFlags.DISPOSED)) {
        if (__DEV__ && check(job)) {
          continue
        }
        // 处理 ALLOW_RECURSE
        if (job.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
          job.flags! &= ~SchedulerJobFlags.QUEUED
        }
        // 执行任务
        callWithErrorHandling(
          job,
          job.i,
          job.i ? ErrorCodes.COMPONENT_UPDATE : ErrorCodes.SCHEDULER,
        )
        // 清除 QUEUED 标记
        if (!(job.flags! & SchedulerJobFlags.ALLOW_RECURSE)) {
          job.flags! &= ~SchedulerJobFlags.QUEUED
        }
      }
    }
  } finally {
    // 3. 确保无论成功/失败都清理状态
    for (; flushIndex < queue.length; flushIndex++) {
      const job = queue[queue.length - 1]
      if (job) {
        job.flags! &= ~SchedulerJobFlags.QUEUED
      }
    }

    // 4. 重置队列
    flushIndex = -1
    queue.length = 0

    // 5. 执行后置回调
    flushPostFlushCbs(seen)

    // 6. 清理
    currentFlushPromise = null
    
    // 7. 递归检查：如果在 flush 过程中有新任务，加入下一轮
    if (queue.length || pendingPostFlushCbs.length) {
      flushJobs(seen)
    }
  }
}
```

**执行流程**：

```
┌─────────────────────────────────────────────────────────┐
│                     flushJobs()                         │
├─────────────────────────────────────────────────────────┤
│  1. for loop: 执行 queue 中所有任务                      │
│     ├── 检查 DISPOSED                                   │
│     ├── 检查递归限制 (DEV 模式)                          │
│     ├── 处理 ALLOW_RECURSE                              │
│     └── callWithErrorHandling(job)                      │
│                                                          │
│  2. finally 块:                                          │
│     ├── 清理所有任务的 QUEUED 标记                       │
│     ├── 清空 queue                                       │
│     ├── 执行 flushPostFlushCbs()                        │
│     └── 如果有新任务 → 递归调用 flushJobs()              │
└─────────────────────────────────────────────────────────┘
```

---

### 5.9 checkRecursiveUpdates() - 递归检测

```typescript
function checkRecursiveUpdates(seen: CountMap, fn: SchedulerJob) {
  const count = seen.get(fn) || 0
  
  // 超过限制
  if (count > RECURSION_LIMIT) {
    const instance = fn.i
    const componentName = instance && getComponentName(instance.type)
    handleError(
      `Maximum recursive updates exceeded${
        componentName ? ` in component <${componentName}>` : ``
      }. ` +
        `This means you have a reactive effect that is mutating its own ` +
        `dependencies and thus recursively triggering itself. Possible sources ` +
        `include component template, render function, updated hook or ` +
        `watcher source function.`,
      null,
      ErrorCodes.APP_ERROR_HANDLER,
    )
    return true  // 表示检测到问题，应跳过
  }
  
  seen.set(fn, count + 1)
  return false
}
```

**限制机制**：

- 每个任务在单次 flush 中最多执行 **100 次**
- 超过限制抛出错误，帮助开发者定位无限循环

---

## 六、任务流程图

```
数据变更 (ref.value = x)
        │
        ▼
   触发 trigger()
        │
        ▼
   调度器 queueJob(job)
        │
        ├─ 检查 QUEUED 标记
        │
        ├─ 按 id 排序插入队列
        │
        └─ queueFlush()
              │
              ▼
        Promise.resolve().then(flushJobs)
              │
              ▼ (microtask)
         flushJobs()
              │
              ├─ 遍历执行 queue
              │     └── 组件更新
              │
              ├─ flushPostFlushCbs()
              │     └── mounted/updated 钩子
              │
              └─ 检查是否有新任务
                    │
                    ▼
              递归或结束

nextTick(() => {...})
        │
        ▼
   返回 currentFlushPromise
        │
        ▼ (等待 flushJobs 完成)
        Promise resolve
```

---

## 七、优先级机制

### 7.1 id 决定优先级

- **越小越先执行**
- 组件的 `uid` 是递增的，创建顺序决定更新顺序
- 父组件 → 子组件（因为父先创建）

### 7.2 特殊 id 处理

| 情况 | id 值 | 说明 |
|------|-------|------|
| 有 id | job.id | 使用实际 id |
| 无 id + PRE | -1 | 最优先 |
| 无 id | Infinity | 最后执行 |

---

## 八、微任务 vs 宏任务

### 为什么使用微任务？

```typescript
// queueFlush 内部
currentFlushPromise = resolvedPromise.then(flushJobs)
```

- **微任务（Promise.then）**：在当前调用栈结束后、下一个宏任务之前执行
- **优势**：
  1. 同一个事件循环内可以合并多次变更
  2. 不会阻塞 UI 渲染
  3. 比 `setTimeout`（宏任务）更高效

### 执行顺序示例

```javascript
ref1.value = 1  // 触发 queueJob
ref2.value = 2  // 触发 queueJob
console.log('同步代码')

// 调度器安排：Promise.resolve().then(flushJobs)
// 微任务队列: [flushJobs]

console.log('同步代码2')
// 同步代码全部执行完，开始执行微任务
// → 执行 flushJobs，DOM 更新
```

---

## 九、防无限递归机制

### 9.1 递归检测

```typescript
const RECURSION_LIMIT = 100

function checkRecursiveUpdates(seen, fn) {
  const count = seen.get(fn) || 0
  if (count > RECURSION_LIMIT) {
    // 抛出错误
    return true
  }
  seen.set(fn, count + 1)
  return false
}
```

### 9.2 触发场景

1. **watch 回调修改响应式数据**
2. **组件 render 函数触发自身更新**
3. **updated 钩子中修改状态**

### 9.3 ALLOW_RECURSE 标记

```typescript
// 可以递归的任务
- 组件更新函数 (componentUpdateFn)
- watch 回调 (watcher's callback)

// 不能递归的任务
- 其他由内部触发的 job
```

---

## 十、错误处理

```typescript
callWithErrorHandling(
  job,
  job.i,                           // 组件实例
  job.i 
    ? ErrorCodes.COMPONENT_UPDATE  // 组件更新错误
    : ErrorCodes.SCHEDULER          // 调度器错误
)
```

- 捕获执行过程中的错误
- 提供组件上下文信息

---

## 十一、关键要点总结

| 概念 | 说明 |
|------|------|
| **queue** | 主任务队列，存储组件更新任务 |
| **pendingPostFlushCbs** | 后置回调队列（mounted/updated） |
| **nextTick()** | 返回 Promise，在 flush 完成后 resolve |
| **queueJob()** | 添加任务到队列，自动排序和去重 |
| **flushJobs()** | 执行队列任务的主函数 |
| **SchedulerJobFlags** | QUEUED/PRE/ALLOW_RECURSE/DISPOSED |
| **RECURSION_LIMIT** | 100，防止无限递归 |
| **微任务调度** | 使用 Promise.then() 而非 setTimeout |

---

## 十二、与 Vue2 的区别

| 特性 | Vue2 | Vue3 |
|------|------|------|
| 调度方式 | setImmediate / MutationObserver / setTimeout | Promise.then (微任务) |
| 队列管理 | 单一队列 | 分离 queue + pendingPostFlushCbs |
| 递归检测 | 无 | RECURSION_LIMIT (100) |
| 任务标记 | 无 | SchedulerJobFlags 位运算 |
| id 处理 | 简单比较 | 二分查找插入 |

---

*分析完成*
