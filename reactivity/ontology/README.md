# Vue3 Reactivity 知识图谱

基于 Ontology 技能构建的 Vue3 响应式系统知识图谱。

## 📊 图谱概览

- **实体数量**: 14 个
- **关系数量**: 26 条
- **覆盖范围**: Vue3 `@vue/reactivity` 核心模块

## 🎯 实体类型

| 实体 | 类型 | 描述 |
|------|------|------|
| Ref | ReactivityPrimitive | 响应式引用，包装单个值的容器 |
| Reactive | ReactivityPrimitive | 创建对象的深层响应式代理 |
| Effect | ReactivityPrimitive | 副作用函数，响应数据变化自动执行 |
| Computed | ReactivityPrimitive | 计算属性，基于依赖缓存的派生值 |
| EffectScope | ReactivityPrimitive | effect 作用域，批量收集和清理 effects |
| RefImpl | ImplementationClass | Ref 的具体实现类 |
| Proxy | ProxyHandler | JS Proxy 对象，拦截属性访问 |
| Dep | InternalClass | 依赖管理器，管理订阅当前数据的 effects |
| Link | InternalClass | 连接 Dep 和 Effect 的双向链表节点 |
| track | Operation | 追踪依赖 - 在 getter 中收集当前 effect |
| trigger | Operation | 触发更新 - 在 setter 中通知所有订阅的 effects |
| CollectionHandlers | HandlerSet | 处理 Map/Set/WeakMap/WeakSet 的 Proxy handlers |
| ArrayInstrumentations | HandlerSet | 数组方法的特殊处理（如 includes, indexOf 等） |
| GlobalVersion | InternalState | 全局版本计数器，用于 dirty-check 优化 |

## 🔗 关系类型

- `implements` - 实现依赖（如 Ref 依赖 RefImpl）
- `calls` - 调用关系（如 effect 调用 track/trigger）
- `extends` - 扩展关系（如 reactive 扩展 Proxy handlers）
- `operates_on` - 操作关系（如 track/trigger 操作 Dep）
- `contains` - 包含关系（如 Dep/Effect 维护 Link 链表）
- `invokes` - 触发关系（如 Effect 执行时触发 track）
- `uses` - 使用关系（如 Computed 使用 effect 追踪依赖）
- `uses_for` - 专门用于（如 reactive 处理集合/数组时使用特定 handlers）
- `manages` - 管理关系（如 EffectScope 管理多个 Effect）
- `optimizes` - 优化关系（如 GlobalVersion 优化 Dep 的 dirty-check）
- `references` - 引用关系（如 Link 引用 Dep 和 Effect）

## 📁 文件结构

```
vue-reactivity-ontology/
├── graph.json      # GraphJSON 格式数据
├── index.html      # 可视化交互页面
└── README.md       # 本文件
```

## 🚀 使用方法

### 1. 查看 GraphJSON

直接打开 `graph.json` 查看结构化数据：
- `nodes` - 实体数组
- `edges` - 关系数组
- `metadata` - 元数据

### 2. 浏览器可视化

使用任意本地 HTTP 服务器打开 `index.html`：

```bash
# 使用 Python
python -m http.server 8080

# 或使用 Node.js
npx serve .

# 然后访问 http://localhost:8080
```

### 3. 可视化功能

- **交互式力导向图**: 自动布局，可拖拽节点
- **悬停提示**: 显示实体/关系的详细信息
- **缩放平移**: 支持鼠标滚轮缩放和拖拽平移
- **图例说明**: 按类型区分的颜色编码
- **动画控制**: 可暂停/继续布局动画

## 🏗️ 核心架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Reactivity Primitives                    │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────────┐  │
│  │   Ref   │  │ Reactive │  │ Effect  │  │   Computed   │  │
│  └────┬────┘  └────┬─────┘  └────┬────┘  └──────┬───────┘  │
│       │            │             │               │          │
│       ▼            ▼             ▼               ▼          │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────────┐  │
│  │ RefImpl │  │  Proxy   │  │  Link   │  │    Effect    │  │
│  └────┬────┘  └────┬─────┘  └────┬────┘  └──────┬───────┘  │
│       │            │             │               │          │
│       │            │             ▼               ▼          │
│       │            │      ┌──────────────────────────┐     │
│       │            │      │   track  ◄──────► trigger │     │
│       │            │      └──────────┬───────────────┘     │
│       │            │                 │                     │
│       │            │                 ▼                     │
│       │            │           ┌─────────┐                 │
│       │            │           │   Dep   │                 │
│       │            │           └────┬────┘                 │
│       │            │                │                      │
│       │            ▼                ▼                      │
│       │    ┌─────────────────────────────────────┐         │
│       │    │ CollectionHandlers / ArrayInst.    │         │
│       │    └─────────────────────────────────────┘         │
└───────┼────────────────────────────────────────────────────┘
        │
        ▼
  ┌─────────────┐
  │ GlobalVersion│
  └─────────────┘
```

## 📚 参考

- [Vue3 Reactivity 源码](https://github.com/vuejs/core/tree/main/packages/reactivity)
- [Vue3 响应式原理文档](https://vuejs.org/guide/extras/reactivity-in-depth.html)