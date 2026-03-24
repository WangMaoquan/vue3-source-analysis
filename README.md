# Vue3 源码分析仓库

> 系统性分析 Vue3 核心模块源码，构建知识图谱与流程图。

## 目录结构

```
vue3-source-analysis/
├── reactivity/        # 响应式模块 ✅ 已完成
│   ├── analysis/       # 分析文档
│   ├── ontology/       # 知识图谱
│   └── diagrams/      # 流程图/示意图
├── runtime-core/      # 运行时核心 ✅ 已完成
├── compiler-sfc/     # 单文件组件编译（待分析）
├── compiler-dom/     # DOM 编译（待分析）
├── runtime-dom/      # DOM 运行时（待分析）
├── shared/           # 共享工具（待分析）
├── compiler-core/    # 核心编译（待分析）
└── server-renderer/ # 服务端渲染（待分析）
```

## 已完成模块

### Reactivity（响应式模块）
- 分析文档: `reactivity/analysis/`
- 知识图谱: `reactivity/ontology/index.html`
- 流程图: `reactivity/diagrams/`

### Runtime-Core（运行时核心）
- 分析文档: `runtime-core/analysis/README.md`

## 分析方法

每个模块分析流程：
1. 目录结构与入口
2. 核心 API 实现
3. 关键机制流程
4. 测试用例验证
5. 构建知识图谱

## 查看知识图谱

```bash
cd reactivity/ontology
python -m http.server 8080
# 访问 http://localhost:8080
```
