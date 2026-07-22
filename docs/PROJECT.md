# Miaoma VBuilder - 项目文档

> 基于 Vue3 + TypeScript 的可视化低代码平台，采用 pnpm monorepo + Turborepo 架构。
> 妙码学院官方出品，作者 @Heyi

---

## 一、技术栈

| 层面 | 技术 |
|------|------|
| 框架 | Vue 3.4 + TypeScript 5.5 |
| 构建 | Vite 5.3 + Turborepo 2.0 |
| 包管理 | pnpm 9.5 (workspace) |
| 状态管理 | Pinia 2.1 |
| 路由 | Vue Router 4.4 |
| 图表 | ECharts 5.5 + vue-echarts + D3.js 7.9 |
| 富文本 | Tiptap 2.5 |
| 流程编辑 | Vue Flow 1.39 |
| 拖拽 | smooth-dnd |
| 表单验证 | vee-validate 4.13 |
| 跨框架渲染 | veaury 2.4 (Vue 中嵌入 React) |
| 大数据表格 | @glideapps/glide-data-grid 6.0 (React 组件) |
| 代码规范 | ESLint 9 + Stylelint + Prettier + cspell |
| 提交规范 | commitlint + commitizen + cz-git + husky + lint-staged |
| E2E 测试 | Playwright |
| 监控 | Sentry |

---

## 二、Monorepo 结构

```
miaoma-vbuilder/
├── apps/
│   ├── builder/          # 主应用 - 可视化编辑器
│   └── playwright/       # E2E 测试子包
├── packages/
│   ├── blocks/           # 共享 Block 组件/逻辑
│   └── eslint/           # 共享 ESLint 配置
├── turbo.json            # Turborepo 构建编排
├── pnpm-workspace.yaml   # pnpm workspace 配置
└── package.json          # 根配置 (scripts, devDeps, lint-staged)
```

---

## 三、功能模块梳理

### 3.1 页面布局编辑引擎 (Page Layout Editor)

**对应 commit:** `6f339d0` - page layout editing engine

核心的可视化拖拽编辑能力，用户可通过拖拽组件到画布上进行页面搭建。

- **Block 系统**: 采用 `BlockSuite` 类统一管理所有可拖拽的 Block 组件，支持插件化扩展
- **拖拽排序**: 基于 smooth-dnd 实现 Block 的拖拽排列
- **Block 渲染**: `BlockRenderer` / `BlocksRenderer` 根据类型动态渲染对应组件
- **右侧面板**: 选中 Block 后可在右侧属性面板中编辑其属性

### 3.2 Block 组件体系

**对应 commit:** `6f339d0`, `77f75f9`, `feaadc9`, `3a989f4`, `fabeaf3`

分为 **Basic Blocks** 和 **External Blocks** 两类：

#### Basic Blocks

| Block 类型 | 说明 | 文件 |
|-----------|------|------|
| `heroTitle` | 英雄标题，支持左/中/右对齐 | `HeroTitleBlock.vue` |
| `view` | 数据视图/表格，支持字段配置与数据展示 | `ViewBlock.vue` |
| `chart` | 图表组件，支持多渲染引擎切换 | `ChartBlock.vue` |
| `quote` | 引用/提示块，支持 success/warning/error 状态 | `QuoteBlock.vue` |
| `image` | 图片块 | `ImageBlock.vue` |

#### External Blocks (外部插件)

| Block 类型 | 说明 | 文件 |
|-----------|------|------|
| `button` | 按钮组件 | `ButtonBlock.vue` |
| `form` | 表单组件，支持多字段类型 | `FormBlock.vue` |
| `notes` | 富文本笔记，基于 Tiptap 编辑器 | `NotesBlock.vue` |

#### 类型定义

所有 Block 统一由 `BlockInfo` 联合类型约束 (`types/block.ts`)，每个 Block 包含 `id`、`label`、`type`、`props` 字段。

### 3.3 多渲染器图表系统

**对应 commit:** `feaadc9` - add chart multitype renderer

ChartBlock 支持三种图表渲染引擎，可在编辑面板中切换：

| 渲染器 | 实现 | 说明 |
|--------|------|------|
| `echarts` | `EchartsRenderer.vue` | 基于 ECharts，功能最全 |
| `canvas` | `CanvasChartRenderer.vue` | Canvas 2D 原生绘制 |
| `svg` | `SVGChartRenderer.vue` | SVG 矢量渲染 (基于 D3.js) |

### 3.4 富文本编辑器 (Notes Editor)

**对应 commit:** `77f75f9` - add a rich text editor to support note block

- 基于 Tiptap 2.5 实现的富文本编辑能力
- 自定义扩展 `ColorHighlighter` 支持文本颜色高亮
- 作为 `NotesBlock` 的核心编辑体验

### 3.5 Action Flow (计算流程)

**对应 commit:** `fabeaf3` - add action flow to support calculations

- 基于 Vue Flow 实现的可视化流程编辑器
- 支持自定义节点类型：`ValueNode`（输入值）、`OperatorNode`（运算符）、`ResultNode`（结果）
- 左侧面板展示 Action 列表，支持查看和编辑

### 3.6 大数据表格方案

**对应 commit:** `3a989f4` - huge data table solution, connected to React components

- 使用 `@glideapps/glide-data-grid` (React 组件) 解决大数据量表格渲染性能问题
- 通过 `veaury` 在 Vue 项目中嵌入 React 组件，实现跨框架集成

### 3.7 模拟器系统 (Simulator)

**对应 commit:** `49c1ae0`, `d3d4c1d`, `8b9b272`, `4c5e09f`

提供移动端和桌面端的实时预览：

| 模拟器 | 文件 | 说明 |
|--------|------|------|
| 手机端 | `MobilePreviewer.vue` | iPhone 样式模拟器，含 TabBar |
| 桌面端 | `LaptopPreviewer.vue` | 笔记本样式模拟器 |

- **PreviewModeSwitcher**: 切换编辑/预览模式
- **StatusBar / TabBar**: 模拟系统状态栏和底部导航栏
- 静态布局定位 (`d3d4c1d`)，修复 iPhone 模拟器 TabBar 分割线 (`8b9b272`)

### 3.8 App Runner (应用运行器)

**对应 commit:** `4c5e09f` - add app runner view

- 独立路由 `/runner`，用于运行态展示搭建好的应用
- `AppRunnerRenderer` 负责将编辑态的 Block 数据渲染为最终用户界面
- 地址栏样式优化 (`0f8791f`)

### 3.9 数据源管理

**对应 commit:** 存在于路由结构中

- 左侧面板 `DataSourceLeftPanel` 展示数据源列表
- `DataSourceContent` 展示数据源详情
- 支持按 ID 查看/编辑数据源 (`/app/dataSource/:id`)

### 3.10 Block 属性设置面板 (Right Panel)

**对应 commit:** `f14a9ec` - block setting right panel

选中 Block 后右侧弹出属性编辑面板：

| 设置面板 | 说明 |
|---------|------|
| `ChartSetting` | 图表类型、数据配置 |
| `HeroTitleSetting` | 标题文本、对齐方式 |
| `ImageSetting` | 图片 URL |
| `QuoteSetting` | 引用内容、状态类型 |
| `SchemaExporter` | 导出 Block 的 JSON Schema |
| `Breadcrumb` | 面包屑导航展示当前选中路径 |

### 3.11 导航系统

- `AppNavigator`: 顶部应用导航栏
- 左侧面板 `AppLeftPanel` 包含：组件抽屉 (`BlocksDrawer`)、组件列表 (`Components`)、导航菜单 (`Navigation`)
- `SegmentedControl`: 自定义分段控制器 UI 组件

---

## 四、路由结构

```
/                      → 重定向到 /app/layout
/app                   → AppView (主布局)
  ├── /app/layout      → 页面布局编辑器 (默认页)
  ├── /app/dataSource  → 数据源管理
  │   └── /app/dataSource/:id → 数据源详情
  └── /app/actions     → Action 管理
      └── /app/actions/:id → Action 详情
/runner                → 应用运行器 (独立页面)
```

---

## 五、Git Commit 历史与功能演进

| # | Commit | 类型 | 说明 |
|---|--------|------|------|
| 1 | `90d22cd` | feat | 项目基础 monorepo 架构搭建 |
| 2 | `49c1ae0` | feat | 应用布局与多模拟器系统 |
| 3 | `6f339d0` | feat | 页面布局编辑引擎核心 |
| 4 | `77f75f9` | feat | 富文本编辑器 (Tiptap)，支持 Notes Block |
| 5 | `feaadc9` | feat | 图表多渲染器 (ECharts / Canvas / SVG) |
| 6 | `d3d4c1d` | feat | 模拟器静态布局定位 |
| 7 | `f14a9ec` | feat | Block 属性设置右侧面板 |
| 8 | `fabeaf3` | feat | Action Flow 计算流程编辑器 |
| 9 | `3a989f4` | feat | 大数据表格方案 (React + veaury 集成) |
| 10 | `9f214c9` | refactor | ESLint 配置抽离为独立子包 |
| 11 | `255a4e1` | test | 添加 Playwright E2E 测试子包 |
| 12 | `4c5e09f` | feat | 应用运行器 (Runner) 视图 |
| 13 | `0f8791f` | style | 地址栏样式优化 |
| 14 | `8d6bdd2` | refactor | Runner 视图样式修复，添加 clean 脚本 |
| 15 | `233f716` | build | 接入 Turborepo 构建编排 |
| 16 | `8b9b272` | fix | iPhone 模拟器 TabBar 分割线修复 |
| 17 | `ffeaa5e` | feat | 性能优化与代码重构 |
| 18 | `1fbb8ea` | - | 最新提交 (miaoma-vbuilder) |

---

## 六、开发命令

```bash
# 安装依赖
pnpm install

# 启动所有子包开发服务
pnpm dev

# 仅启动 builder
pnpm dev:builder

# 构建所有子包
pnpm build

# 代码检查
pnpm lint          # ESLint + Stylelint
pnpm lint:vue      # ESLint (Vue/TS)
pnpm lint:style    # Stylelint (CSS/SCSS/Vue)
pnpm typecheck     # TypeScript 类型检查
pnpm spellcheck    # 拼写检查

# 清理构建产物
pnpm clean

# 规范化提交
pnpm commit
```

---

## 七、核心状态管理 (Pinia Store)

**`useAppEditorStore`** - 编辑器核心状态：

- `currentBlockId`: 当前选中的 Block ID
- `blocks`: 当前页面所有 Block 数据
- `selectBlock(id)`: 选中 Block
- `unSelectBlock()`: 取消选中
- `updateBlocks(newBlocks)`: 批量更新
- `updateBlock(id, newBlock)`: 更新单个 Block 属性
