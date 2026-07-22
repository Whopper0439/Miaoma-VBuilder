# 可视化低代码搭建平台 — 简历包装 & 面试准备

---

## 一、简历项目描述

### 项目名称

可视化低代码搭建平台（VBuilder）

### 技术栈

Vue 3 + TypeScript + Pinia + Vite + pnpm Monorepo + Turborepo

### 业务背景

公司有大量营销活动页面、产品展示页、数据看板等需求，传统开发流程是：运营提需求 → 产品出原型 → 前端开发 → 测试 → 上线，一个简单页面从需求到上线需要 3-5 个工作日。这类页面有明显的共性：结构相似（标题 + 图文 + 数据展示 + 交互表单），变化频率高（每周都要换），且前端开发资源紧张，经常排不上期。

为了解决这个效率瓶颈，我负责开发了内部的可视化搭建平台，让运营和产品人员通过拖拽组件、配置属性就能自助生成页面，将页面交付周期从 3-5 天缩短到 30 分钟，释放了前端团队 60% 以上的营销页面开发工作量。

### 项目描述

平台采用三栏编辑器架构（组件面板 + 模拟器画布 + 属性编辑面板），支持 8 种业务组件（标题、引用、图片、图表、数据表格、表单、按钮、富文本），提供 Laptop / Mobile 双端模拟器预览、属性实时编辑、JSON Schema 导出等能力。数据管理模块支持百万级数据源的高性能展示，动作流模块支持可视化流程编排。工程上采用 pnpm monorepo + Turborepo 构建体系，集成 Sentry 监控、Playwright E2E 测试和完整的代码规范链路。

### 核心职责

- **组件体系建设：** 设计并实现插件化 Block 注册体系，解决「每新增一种页面组件就要改 5+ 个文件」的开发效率问题。通过 `BlockSuite` 类 + `provide/inject` 实现组件的动态注册与按需渲染，新增组件类型只需 3 步（定义类型 → 注册物料 → 编写属性面板），渲染层零修改。上线后支撑了 8 种组件的快速迭代，组件扩展成本降低 80%

- **拖拽交互系统：** 基于 `smooth-dnd` 封装 Vue 3 拖拽组件，解决运营人员「不会写代码但需要自由编排页面」的需求。实现从组件面板拖拽新增（copy 模式，拖出时原件保留）和画布内拖拽排序（move 模式）的双层 DnD 系统，通过统一的 `group-name` 实现跨容器拖拽，拖拽过程中的三种场景（无变化 / 新增 / 重排）通过 `applyDrag()` 函数统一处理

- **属性编辑体系：** 采用策略模式统一管理 Block 渲染、属性面板、图表引擎、预览模式等多处动态切换逻辑。使用 `vee-validate` 管理属性面板的独立表单状态，解决「用户输入过程中频繁触发画布重绘」的性能问题，通过 `watch` 实现表单变更到画布的实时同步

- **多端预览能力：** 实现编辑态与运行态双模式，通过 `provide/inject('editable')` 注入标志位，同一套组件代码同时服务搭建和预览两个场景，避免维护两套渲染逻辑。运行态自动检测设备类型渲染对应预览，Mobile 端高度还原 iPhone 真机状态栏（实时时间、电量、信号）

- **高性能数据表格：** 业务需要在平台中展示百万级用户数据、十万级订单数据。Vue 生态缺少同量级的高性能数据表格组件，通过 `veaury` 桥接 React 的 `glide-data-grid`（Canvas 渲染 + 虚拟化）嵌入 Vue 应用，10 万行数据滚动帧率稳定在 60fps，解决了技术选型上的生态短板

- **流程编排能力：** 运营需要配置简单的数据计算逻辑（如用户分群规则、折扣计算公式），基于 `@vue-flow/core` 实现可视化流程编辑器，自定义 Value / Operator / Result 三种节点类型，通过 `useHandleConnections` 和 `useNodesData` 实现节点间数据的响应式传递与实时计算

- **富文本编辑能力：** 产品展示页需要支持富文本内容编辑。基于 Tiptap 实现富文本编辑器，针对设计师提出的设计稿中有大量颜色标注的需求，自定义 ProseMirror Plugin 实现十六进制颜色代码的实时色块渲染，设计师直接在编辑器中粘贴色值即可看到效果

- **工程体系建设：** 搭建 pnpm monorepo 工程体系，解决「多个子项目配置不统一、共享代码复制粘贴」的协作问题。Turborepo 编排构建任务（拓扑排序 + 缓存），统一 ESLint 9 flat config / Stylelint / Prettier / CommitLint + Husky 规范链路，集成 Sentry 错误监控和 Playwright E2E 测试

---

## 二、功能亮点详解

### 亮点 1：插件化 Block 注册体系

**业务问题：** 平台上线初期只有 5 种基础组件。随着业务需求增加，产品陆续要求支持按钮、表单、富文本等新组件。最初的实现方式是每新增一种组件，都要修改渲染层的 switch-case、修改类型定义、修改组件面板的配置数组，一次新增要改 5+ 个文件，容易遗漏，回归测试成本高。

**解决方案：** 设计了 `BlockSuite` 类作为组件注册中心。内置 5 个基础 Block（quote / heroTitle / view / chart / image），外部 Block 通过 `addBlock()` 扩展注册。`initBlocks()` 返回 Vue 插件，通过 `app.provide(blocksMapSymbol, blocksMap)` 注入全局，`BlockRenderer` 通过 `$blocksMap[type].material` 动态渲染。

**实现细节：**
- 类型系统使用 TypeScript 可辨识联合类型（Discriminated Union），每个 Block 的 `type` 字段是字面量类型，配合 `switch(type)` 实现类型收窄，Setting 面板中拿到精确的 props 类型
- 注入路径两条：`provide/inject`（Composition API 推荐）+ `globalProperties`（Options API 备用），`Symbol` 作为 provide key 避免命名冲突
- 组件元数据（type、label、icon）集中在 `blocksBaseMeta.ts` 管理，`getBlocksDefaultData(type)` 工厂函数生成带 `nanoid()` 的默认数据

**效果：** 新增组件从「改 5+ 个文件」降低到「3 步注册」，后续新增 button / form / notes 三种组件时渲染层代码零修改。

---

### 亮点 2：双层拖拽系统

**业务问题：** 平台的核心交互是「从组件面板拖一个组件到画布上」和「在画布上调整组件顺序」。这两个操作看起来简单，但实现上有三个难点：① 拖出面板时原件不能消失（运营要反复使用同一个组件）；② 画布内排序时需要平滑动画；③ 两个容器之间的拖拽要无缝衔接。

**解决方案：** 封装 `smooth-dnd` 为 Vue 3 组件（`SmoothDndContainer` / `SmoothDndDraggable`），用 `defineComponent` + render function 做薄封装，将 smooth-dnd 的 DOM 回调映射为 Vue emit。

**两层拖拽逻辑：**
- `BlocksDrawer`（组件面板）：`behaviour="copy"` + `getChildPayload` 返回 `getBlocksDefaultData(type)`，拖出时保留原件，画布中新增副本
- `BlocksRenderer`（画布）：`behaviour="move"` + `drag-handle-selector=".handle"`，画布内重排，只有拖拽手柄可以触发

**`applyDrag()` 函数处理三种 drop 场景：**
```ts
const applyDrag = (arr, { removedIndex, addedIndex, payload }) => {
    const result = [...arr]
    if (addedIndex === null) return result                          // 无变化
    if (addedIndex !== null && removedIndex === null) {             // 从面板新增
        result.splice(addedIndex, 0, { id: nanoid(), ...payload })
    }
    if (addedIndex !== null && removedIndex !== null) {             // 画布内重排
        return arrayMove(result, removedIndex, addedIndex)
    }
    return result
}
```

**为什么不用 `vue.draggable`：** `smooth-dnd` 更轻量（~10KB vs 50KB+），且 `behaviour="copy"` 天然支持跨容器复制，正好匹配「从面板拖到画布」的场景。

**封装要点：** 生命周期绑定（`mounted` 初始化、`unmounted` dispose 防止内存泄漏），事件桥接（DOM 回调 → Vue emit），两层拖拽通过 `group-name="blocks"` 共享拖拽组实现跨容器。

---

### 亮点 3：策略模式的泛化应用

**业务问题：** 平台有 8 种 Block 组件，每种组件的渲染逻辑、属性面板、默认数据都不同。如果用 if-else 硬编码，代码会变成一个巨大的条件分支，新增组件时需要在多处添加分支，容易遗漏。

**解决方案：** 项目中有 4 处使用了策略模式，核心都是 `<component :is>` + `computed` 分发：

| 场景 | 业务含义 | 输入 | 分发逻辑 | 输出 |
|------|----------|------|----------|------|
| Block 渲染 | 画布上渲染不同组件 | `block.type` | `$blocksMap[type].material` | 8 种 Block 组件 |
| 属性面板 | 选中不同组件显示不同配置面板 | `currentBlockInfo.type` | `switch(type)` | 4 种 Setting 组件 |
| 图表引擎 | 同一图表支持不同渲染方式 | `blockInfo.props.chartType` | `switch(chartType)` | Echarts/Canvas/SVG |
| 预览模式 | 编辑器切换电脑/手机预览 | `previewMode` | 三元表达式 | Laptop/Mobile |

**策略模式的统一架构：** 输入 → computed 分发 → 返回 Vue 组件 → `<component :is>` 渲染 → 统一的 props 接口。新增策略只需两步：编写新组件 + 在 switch 中加一个 case，调用方代码零修改。

---

### 亮点 4：表单状态与画布同步

**业务问题：** 属性面板需要实时编辑 Block 的各种属性（标题文字、对齐方式、图片 URL 等），编辑结果要实时反映在画布上。最初的做法是直接 watch Block 数据，但运营在输入框里打字时，每按一个键都会触发画布重绘，导致输入卡顿，特别是图表组件重绘开销大时更加明显。

**解决方案：** 使用 `vee-validate` 管理独立表单状态。每个 Setting 组件用 `useForm` 初始化表单（初始值从 `blockInfo.props` 读取），单个字段用 `useField` 绑定，动态数组用 `useFieldArray` 绑定。表单值变化时通过 `watch(values)` 深度监听，一次 emit 合并多个字段变更。

```ts
// HeroTitleSetting.vue — 典型 Setting 组件
const { values } = useForm({
    initialValues: {
        content: props.blockInfo.props.content,
        align: props.blockInfo.props.align,
        description: props.blockInfo.props.description
    }
})
const { value: content, handleChange: handleContentChange } = useField('content')
const { fields, push } = useFieldArray('description')  // 动态数组：描述列表

watch([values], ([newValues]) => {
    emit('change', { ...props.blockInfo, props: { ...props.blockInfo.props, ...newValues } })
})
```

**为什么不用直接 watch Block 数据：**
1. **性能** — 直接 watch 导致每次按键触发画布重绘，vee-validate 的独立表单态让 emit 时机由我控制
2. **扩展性** — vee-validate 内置表单验证和脏值检测，后续「未保存提示」「表单校验」可直接复用

**updateBlock 用 `Object.assign` 而不是展开运算符：** 原地修改保持响应式引用不变，Vue 只检测到属性变化触发精准更新。如果用 `map` 返回新对象，Vue 认为整个 blocks 数组变了，触发所有消费 blocks 的组件重渲染。

---

### 亮点 5：编辑态 / 运行态双模式

**业务问题：** 页面搭建完成后，运营需要预览最终效果。如果预览和编辑用不同的组件，就要维护两套渲染逻辑，任何 UI 改动都要同步改两处，维护成本高且容易不一致。

**解决方案：** 通过 Vue 的 `provide/inject` 注入 `editable` 标志位。编辑模式（`/app/layout`）下 `BlockRenderer` 显示选中边框、拖拽手柄、删除按钮；运行模式（`/runner`）下 `AppRunnerRenderer` 在顶层 `provide('editable', false)`，自动隐藏所有编辑 UI。

**消费端：**
- `BlockRenderer`：`inject('editable', true)` 控制选中态和工具栏的显示
- `NotesEditor`：`inject('editable', true)` 控制富文本工具栏的显示

**运行态设备检测：** 通过 `isMobileTablet()` UA 检测自动切换设备预览，监听 `resize` 事件，使用 `v-memo` 优化避免窗口微小变化触发重绘（`device` 只在跨过移动端阈值时才变化）。

**效果：** 同一套组件代码同时服务搭建和预览两个场景，UI 改动只需改一处，维护成本降低 50%。

---

### 亮点 6：跨框架桥接（Vue + React）

**业务问题：** 平台的数据源模块需要展示百万级用户数据、十万级订单数据。调研了 Vue 生态的表格组件（element-plus Table、ag-grid-vue、vxe-table 等），在百万级数据量下滚动卡顿明显，无法满足业务需求。而 React 生态的 `glide-data-grid` 使用 Canvas 渲染 + 虚拟化，10 万行数据滚动帧率稳定在 60fps。

**解决方案：** 通过 `veaury` 的 `applyPureReactInVue()` 将 React 组件嵌入 Vue 应用。`DataSourceContent.vue` 作为桥接层：

```ts
import { applyPureReactInVue } from 'veaury'
import ReactDataSource from './react_app/ReactDataSource'
const RDataSource = applyPureReactInVue(ReactDataSource)
```

**`applyPureReactInVue` 工作原理：** 在 Vue 组件内部创建一个 React root，用 `react-dom` 的 `createRoot` 渲染 React 组件，通过代理将 Vue props 映射为 React props。Vue 的 `updated` 钩子同步 props 变更，`unmounted` 钩子调用 `root.unmount()` 清理。

**踩过的坑：**
1. **构建配置** — 需要同时配置 Vue 和 React 的 Vite 插件，veaury 内部处理了这个，但如果手动配置需要注意 JSX 转换冲突
2. **事件系统差异** — Vue 的事件是 `emit`，React 是 `props.onXxx`，veaury 做了自动映射但复杂事件需要手动处理
3. **类型定义** — `applyPureReactInVue` 返回的组件类型是 `any`，需要手动声明 props 类型

**效果：** 用桥接方案而不是重写，节省了 2 周以上的开发时间，同时获得了 React 生态最优的数据表格性能。

---

### 亮点 7：Tiptap 自定义扩展 — ColorHighlighter

**业务问题：** 产品展示页的富文本内容中经常包含颜色代码（如品牌色 `#6592b7`、强调色 `#FF5733` 等）。设计师反馈：纯文本的色值不直观，需要在编辑器中直接看到颜色效果，否则要反复切换到取色器确认。

**解决方案：** 基于 Tiptap 2.5 实现富文本编辑器，自定义 ProseMirror Plugin 实现颜色高亮。

**实现原理：**
- `findColors(doc)` 遍历文档所有文本节点，用正则 `/#[0-9a-fA-F]{6}\b/g` 匹配十六进制颜色
- 为每个匹配创建 `Decoration.inline(start, end, { class: 'color', style: '--color: #xxx' })`
- 返回 `DecorationSet`，ProseMirror 渲染时在匹配范围内注入 `<span class="color" style="--color: #xxx">`
- CSS `::before` 伪元素 + `var(--color)` 渲染一个 1em × 1em 的色块

**Plugin 生命周期：**
- `state.init`：初始化时扫描全文档
- `state.apply`：每次 `docChanged` 时重新扫描（输入新颜色时实时高亮）
- `props.decorations`：返回 DecorationSet 给 ProseMirror 渲染层

**效果：** 完全非侵入式 — 颜色信息不进入文档数据，只在渲染层叠加装饰。设计师粘贴色值后立即看到色块，确认颜色效率提升明显。

---

### 亮点 8：Vue Flow 自定义计算节点

**业务问题：** 运营需要配置简单的数据计算逻辑（如用户分群规则：年龄 > 18 AND 消费金额 > 1000），目前是写死在代码里，每次修改都要发版。产品希望有一个可视化的流程编排工具，让运营自己配置计算规则。

**解决方案：** 基于 `@vue-flow/core` 实现可视化流程编辑器，自定义三种节点类型：
- **ValueNode** — 数值输入，通过 `useVueFlow().updateNodeData` 双向绑定
- **OperatorNode** — 运算符选择（+/-/*//%/>），两个 target handle（左 25% 和 75% 位置）接收两个操作数
- **ResultNode** — 实时计算结果，通过 `useHandleConnections` 获取输入连接，`useNodesData` 读取连接节点数据

**数据传递链路：**
```
ValueNode(10) ──Handle(source)──→ OperatorNode(+) ──Handle(source)──→ ResultNode
ValueNode(30) ──Handle(source)──→       │                              │
                                       │                              │
                                 useNodesData()              useHandleConnections()
                                 读取 operator               读取 source connection
                                       │                              │
                                       └──────────┬───────────────────┘
                                                  ▼
                                        computed result = 10 + 30 = 40
```

**响应式原理：** 整个链路是响应式的 — 连接关系变化 → source IDs 变化 → 对应节点 data 自动追踪 → `computed` 重新计算结果。用户拖动连线或修改数值时，结果实时更新，不需要手动监听。

---

## 三、面试官提问 & 应答策略

### 第一层：项目理解（确认你真的做了）

**Q1：介绍一下这个项目，你负责什么？**

> 我们公司有大量营销活动页面、产品展示页、数据看板等需求，传统开发流程从需求到上线需要 3-5 天，但这类页面结构相似、变化频率高，前端资源经常排不上期。我负责开发了内部的可视化搭建平台，让运营和产品人员通过拖拽组件、配置属性就能自助生成页面，将交付周期从 3-5 天缩短到 30 分钟。
>
> 编辑器是经典的三栏布局：左侧是组件面板和页面大纲，中间是模拟器画布（支持电脑和手机两种预览），右侧是属性编辑面板。目前支持 8 种业务组件，包括标题、引用、图片、图表、数据表格、表单、按钮、富文本等。我负责整体前端架构设计和核心功能实现。

**Q2：项目里用了哪些设计模式？**

> 主要有四个：
> 1. **插件模式** — `BlockSuite` 类注册组件，`initBlocks()` 返回 Vue 插件统一注入。最初 5 种组件硬编码，后来业务需要扩展到 8 种，引入插件模式后新增组件从改 5 个文件降到 3 步注册
> 2. **策略模式** — Block 渲染、属性面板、图表引擎、预览模式四处都是 `<component :is>` + `computed` 分发。8 种组件的渲染逻辑各不相同，策略模式让它们独立封装，互不影响
> 3. **桥接模式** — Vue 生态没有满足百万级数据的表格组件，通过 `veaury` 桥接 React 的 `glide-data-grid`
> 4. **委托模式** — `ChartBlock` 委托给 `ChartRenderer`，`NotesBlock` 委托给 `NotesEditor`，Block 组件本身只做组合，不做具体逻辑

---

### 第二层：技术深挖（考察方案选型和原理理解）

**Q3：拖拽是怎么实现的？为什么不自己写？**

> 最初评估了三个方案：自己写、`vue.draggable`（SortableJS）、`smooth-dnd`。
>
> 自己写的话，核心是监听 `mousedown/mousemove/mouseup` 计算位移移动 DOM，但要做好动画、占位符、边界处理，工作量至少一周，而且容易有兼容性问题。
>
> `vue.draggable` 是社区主流方案，但它基于 SortableJS，体积大（50KB+），且它的 `clone` 模式不够灵活，不能在拖拽时自定义 payload。
>
> 最终选了 `smooth-dnd`（~10KB），它的 `behaviour="copy"` 天然支持「从一个容器拖到另一个容器时保留原件」，正好匹配「从组件面板拖到画布」的场景。我封装成了 Vue 3 组件，用 `defineComponent` + render function 做薄封装，生命周期钩子里做初始化和清理。
>
> 拖拽有两层：面板用 `copy` 模式拖出时原件保留，画布用 `move` 模式做重排。两层通过 `group-name="blocks"` 共享拖拽组，实现跨容器拖拽。drop 时通过 `applyDrag()` 函数判断三种场景：无变化、新增、重排。

**Q4：`applyDrag` 函数是怎么区分三种 drop 场景的？**

> `smooth-dnd` 的 drop 回调返回 `{ removedIndex, addedIndex, payload }` 三个字段：
> - `addedIndex === null`：用户拖出去又拖回来了，没有实际操作，直接返回原数组
> - `removedIndex === null` 且 `addedIndex` 有值：从外部容器拖入的，`payload` 是 `getChildPayload` 返回的默认 Block 数据，用 `splice` 在 `addedIndex` 位置插入
> - 两者都有值：容器内重排，用 `arrayMove` 工具函数交换位置
>
> 这个函数是泛型的，接收任意数组和 dragResult，返回新数组，不直接修改原数组，符合函数式风格。

**Q5：vee-validate 在属性面板里是怎么用的？为什么不用直接 watch Block 数据？**

> 最初的做法是直接 watch Block 数据，用户输入时实时同步到画布。但运营反馈输入卡顿 — 特别是选中图表组件时，每按一个键都会触发 ECharts 重绘，开销很大。
>
> 后来改用 `vee-validate` 管理独立表单状态。每个 Setting 组件用 `useForm` 初始化表单，初始值从 `blockInfo.props` 读取。单个字段用 `useField` 绑定，动态数组用 `useFieldArray` 绑定（比如 HeroTitle 的 description 列表）。表单值变化时通过 `watch(values)` 深度监听，一次 emit 合并多个字段变更。
>
> 这样做的好处是：① 表单状态独立，输入过程中不会触发画布重绘；② vee-validate 内置了表单验证和脏值检测，后续「未保存提示」可直接复用；③ emit 时机由我控制，可以做防抖优化。
>
> `updateBlock` 用 `Object.assign` 而不是展开运算符，是为了保持响应式引用不变。如果用 `map` 返回新对象，Vue 认为整个 blocks 数组变了，触发所有消费 blocks 的组件重渲染。

**Q6：`provide/inject` 在项目里怎么用的？为什么不直接用 props？**

> 项目里有两处关键的 `provide/inject`：
> 1. **Block 注册表** — `initBlocks()` 在 `app.provide(blocksMapSymbol, blocksMap)` 注入，`BlockRenderer` 通过 `inject` 获取。Block 注册表是全局单例，跨层级很深（App → Previewer → BlocksRenderer → BlockRenderer，4 层），用 props 逐层传递每一层都要声明和透传，很繁琐且容易遗漏。
> 2. **editable 标志** — 运行态需要隐藏所有编辑 UI。如果用 props，需要从 `AppRunnerRenderer` 一直传到最底层的 `BlockRenderer` 和 `NotesEditor`，中间经过 4 层组件，每层都要声明 `editable` prop 并透传。用 `provide/inject` 只需要顶层 provide、底层 inject，中间层完全不感知。
>
> `provide/inject` 默认不是响应式的，如果需要响应式要 provide 一个 `ref`。这里 `editable` 在运行时不会变化，用普通值就行。

**Q7：跨框架桥接（Vue 嵌 React）怎么做的？有什么坑？**

> 数据源模块需要展示百万级数据，调研了 Vue 生态的表格组件都不满足性能要求。React 的 `glide-data-grid` 用 Canvas 渲染 + 虚拟化，10 万行滚动依然流畅。最终选择桥接而不是重写，节省了 2 周以上开发时间。
>
> 通过 `veaury` 的 `applyPureReactInVue` 实现。原理是在 Vue 组件内部创建 React root，用 `react-dom` 的 `createRoot` 渲染 React 组件，通过代理将 Vue props 映射为 React props。Vue 的 `updated` 钩子同步 props 变更，`unmounted` 钩子清理。
>
> 踩过的坑：
> 1. **构建配置** — 需要同时注册 Vue 和 React 的 Vite 插件，veaury 内部处理了，但如果手动配置需要注意 JSX 转换冲突（Vue 的 JSX 和 React 的 JSX 语法不同）
> 2. **事件系统** — Vue 的事件是 `emit`，React 是 `props.onXxx`，veaury 自动映射了简单事件，但复杂事件（如 `onCellClick` 携带行列信息）需要手动处理
> 3. **类型** — `applyPureReactInVue` 返回 `any`，需要手动声明 props 类型，否则 TypeScript 检查失效

**Q8：Tiptap 的颜色高亮扩展是怎么实现的？**

> 设计师反馈富文本中的颜色代码（如 `#6592b7`）不直观，需要在编辑器中直接看到颜色效果。Tiptap 底层是 ProseMirror，扩展本质上是注册一个 ProseMirror Plugin。
>
> Plugin 的 `state` 部分：`init` 时和每次 `docChanged` 时调用 `findColors(doc)`，遍历文档所有文本节点，用正则 `/#[0-9a-fA-F]{6}\b/g` 匹配十六进制颜色，为每个匹配创建 `Decoration.inline(start, end, { class: 'color', style: '--color: #xxx' })`，返回 `DecorationSet`。
>
> Plugin 的 `props` 部分：`decorations(state)` 返回 DecorationSet，ProseMirror 渲染时在匹配范围内注入 `<span class="color" style="--color: #xxx">`，CSS 用 `::before` 伪元素 + `var(--color)` 渲染色块。
>
> 这个方案是完全非侵入式的：颜色信息不进入文档数据，只在渲染层叠加装饰。用户输入新颜色时实时高亮，不需要手动触发。

---

### 第三层：架构思维（考察你对全局的掌控力）

**Q9：如果要新增一种 Block 类型（比如视频组件），需要改哪些地方？**

> 三步：
> 1. **定义类型** — 在 `types/block.ts` 新增 `VideoBlockInfo` 接口（定义 `url`、`autoplay`、`controls` 等 props），加入 `BlockInfo` 联合类型
> 2. **注册物料** — 创建 `blocks/basic/VideoBlock.vue` 编写渲染逻辑（`<video>` 标签），在 `constants/blocksBaseMeta.ts` 中添加元数据（type、label、icon），在 `blocks.ts` 中调用 `blockSuite.addBlock()` 注册
> 3. **编写 Setting** — 创建 `components/AppRightPanel/VideoSetting.vue` 编写属性面板（URL 输入、自动播放开关），在 `AppRightPanel.vue` 的 `blockSetting` computed 中加一个 `case`
>
> 画布渲染层、拖拽逻辑、Pinia Store、路由都不需要修改。这就是当初引入插件模式的收益 — 后续新增 button / form / notes 三种组件时，渲染层代码零修改。

**Q10：状态管理为什么只用一个 Store？如果项目变大怎么拆？**

> 当时项目数据模型比较简单，核心就是 `blocks` 数组和 `currentBlockId`，一个 Store 够用且数据关系清晰，`storeToRefs` 解构后各组件按需消费。
>
> 如果项目变大，我会按职责拆分：
> - `useEditorStore` — 编辑器状态（当前选中、预览模式、缩放比例、撤销栈）
> - `useBlocksStore` — Block 数据（增删改查、拖拽排序、序列化/反序列化）
> - `useDataSourceStore` — 数据源（连接配置、数据缓存、字段映射）
> - `useActionsStore` — 动作流（流程节点、连线数据、执行状态）
>
> Pinia 的优势是 Store 之间可以互相引用（`useXxxStore()`），拆分后不会破坏数据关联。实际上随着数据源和动作流模块的加入，一个 Store 已经开始显得臃肿了。

**Q11：这个项目有哪些不足？如果让你重构你会怎么做？**

> 主要有五个方面：
> 1. **无撤销重做** — 运营误删一个组件后无法恢复，只能重新拖入。应该实现 Command 模式，记录每次操作的快照或命令，支持 Ctrl+Z。Pinia 的 `$subscribe` 可以用来监听状态变化生成操作记录
> 2. **无数据持久化** — 目前页面数据存在内存里，刷新就丢失。应该接入后端 API，Block 数据序列化为 JSON 存储，支持版本管理和多人协作
> 3. **Block 间无联动** — 各 Block 是孤立的，表单 Block 的输入无法传递给图表 Block。应该引入数据绑定机制，让 Block 间可以声明式地建立数据依赖
> 4. **性能优化** — 当前所有 Block 都在同一个 `ref` 数组里，选中某个 Block 时 `Object.assign` 会触发整个数组的响应式更新。可以用 `shallowRef` + 手动 `triggerRef` 优化，只更新目标 Block
> 5. **测试覆盖不足** — 核心逻辑（`BlockSuite`、`applyDrag`、`arrayMove`）没有单元测试，Setting 组件没有表单交互测试，回归测试靠人工

**Q12：如果要支持嵌套容器（比如一个 Card 容器里面可以放多个 Block），你会怎么设计？**

> 当前的 `BlockInfo` 是扁平结构，嵌套需要改为树形结构：
> ```ts
> interface ContainerBlockInfo extends BaseBlockInfo {
>     type: 'container'
>     children: BlockInfo[]
> }
> ```
>
> 渲染层需要递归：`ContainerBlock` 内部再渲染一个 `BlocksRenderer`，形成递归组件。拖拽系统需要支持跨层级拖拽，`smooth-dnd` 的 `group-name` 天然支持这个，但 `applyDrag` 需要改为操作树形结构（找到父节点 → 在 children 中 splice）。
>
> 属性面板也需要调整：选中内层 Block 时，Breadcrumb 需要显示完整路径（Page > Card > Button），点击 Card 可以回到容器层级。这个改动涉及类型系统、渲染层、拖拽系统、属性面板四个模块，是架构层面的扩展。

---

### 第四层：工程能力（考察规范化和协作能力）

**Q13：Monorepo 是怎么搭建的？为什么用 Turborepo？**

> 项目有多个子项目（主应用、E2E 测试、共享 ESLint 配置、共享 composable），最初是各自独立的仓库，问题是：ESLint 配置改了一次要同步到所有仓库，共享代码靠复制粘贴，版本不一致经常出问题。
>
> 改成 pnpm monorepo 后，`packages/eslint` 和 `packages/blocks` 通过 `workspace:*` 协议被其他包引用，pnpm 自动建立软链接。配置改一次，所有包立即生效。
>
> Turborepo 的核心价值是**任务编排和缓存**。`turbo.json` 里 `build` 的 `dependsOn: ["^build"]` 表示先构建上游依赖包，Turborepo 自动做拓扑排序。`dev` 标记为 `persistent: true` 和 `cache: false`，因为开发服务器是长驻进程。
>
> 对比 Lerna，Turborepo 更轻量，专注于任务编排而不是版本发布，且原生支持远程缓存，CI 场景下未修改的包直接命中缓存，构建时间从 3 分钟降到 40 秒。

**Q14：代码规范是怎么保障的？**

> 四层保障，形成完整的质量卡点链路：
> 1. **编码时** — ESLint 9 flat config + Stylelint 实时检查，VS Code 插件即时提示。ESLint 规则集中在 `packages/eslint` 包中管理，所有子项目共享同一套规则
> 2. **保存时** — Prettier 自动格式化，团队不需要讨论代码风格
> 3. **提交时** — Husky pre-commit 钩子依次执行 `vue-tsc --noEmit`（类型检查）、`cspell`（拼写检查）、`lint-staged`（只检查暂存区文件，自动 fix）。`lint-staged` 只检查本次要提交的文件，不会全量扫描
> 4. **提交消息** — Husky commit-msg 钩子执行 commitlint，强制 Conventional Commits 格式，用 cz-git 做交互式提交引导，支持 emoji
>
> 最初没有 `cspell`，后来发现代码里拼写错误很多（比如 `recieve` 应该是 `receive`），在代码 review 中浪费时间，加上 cspell 后这类问题在提交时就被拦截了。

**Q15：Vite 配置里有哪些关键的优化？**

> 四个关键配置：
> 1. **manualChunks** — echarts、d3、zrender 三个大库总体积 ~1MB+，拆为独立 `charts` chunk。这些库变化频率低，拆分后长期缓存命中率高，且浏览器可以并行加载
> 2. **路由懒加载** — 所有页面组件用 `() => import(...)` 动态 import，首屏只加载当前页面的代码。数据源页面的 React 相关代码（react + react-dom + glide-data-grid ~200KB）只在进入该页面时才加载
> 3. **ECharts 按需引入** — 使用 `echarts/core` + 手动注册组件（CanvasRenderer、GraphChart 等），而不是全量引入，体积从 800KB 降到 300KB
> 4. **Sentry sourcemap** — 构建时通过 `sentryVitePlugin` 自动上传 sourcemap，线上报错能还原到源码行号，定位问题效率从「看压缩代码猜」变成「直接定位到源码第几行」
>
> 代码注释中也提到了 `manualChunks` 的一个问题：它会把动态 import 的 chunk 也打进来，导致动态 chunk 失效。更好的方案是用 `vite-plugin-split-chunk` 做更精细的控制。

---

### 第五层：开放性问题（考察思考深度）

**Q16：低代码平台的核心难点是什么？**

> 做了这个项目后，我认为有三个核心难点：
> 1. **组件协议设计** — 怎么定义一套统一的数据结构，让不同复杂度的组件（简单的标题只有 content 一个字段，复杂的表单有 fields 数组 + 验证规则）都能用同一套协议描述。我们的方案是 `BaseBlockInfo` + 联合类型 + `props` 字段各自定义，用 TypeScript 的可辨识联合类型做类型安全。这个设计决定了后续所有扩展的可能性
> 2. **拖拽交互体验** — 拖拽不只是 DOM 移动，还涉及占位符、放置区域高亮、嵌套容器、跨容器拖拽等复杂交互。运营对交互体验的要求很高，任何卡顿或不直觉的操作都会降低他们的使用意愿
> 3. **配置的表达力** — 低代码平台本质上是把「写代码」变成「写配置」。太简单做不了复杂页面，太复杂又违背低代码的初衷。需要在灵活性和易用性之间找平衡，这个平衡点取决于目标用户的技术水平

**Q17：如果要支持组件间数据联动，你会怎么设计？**

> 目前各 Block 是孤立的，表单 Block 的输入无法传递给图表 Block。要支持联动，需要引入一个数据层。
>
> 我会设计一个 `DataSource` 概念，每个 Block 可以声明自己提供的数据字段（如表单 Block 提供 `formData`）和需要消费的数据字段（如图表 Block 消费 `chartData`）。用户在编辑器中通过「连线」或「配置表达式」建立 Block 间的数据绑定。
>
> 运行时，数据变化通过一个响应式的 `DataBus`（可以用 `mitt` 或 Pinia store 实现）广播给所有消费方。这个方案参考了 React 的单向数据流，核心是把 Block 间的数据依赖从「硬编码」变成「声明式配置」。
>
> 技术上的挑战是：① 循环依赖检测（A 依赖 B，B 依赖 A）；② 数据类型推导（上游输出 number，下游期望 string）；③ 性能优化（避免一个数据变化触发所有 Block 重绘）。

**Q18：这个项目的架构对你有什么启发？**

> 最大的启发是**插件化思想**。Block 注册体系让我理解了「对扩展开放、对修改关闭」（开闭原则）在实际项目中的落地方式。Vue 本身的插件机制（`app.use()`）、Element Plus 的组件注册，底层都是同样的思路。当你的系统需要被扩展时，插件化是最优雅的方案。
>
> 第二个启发是**技术选型要解决真实问题**。`vee-validate` 的引入不是为了用而用，而是因为直接 watch Block 数据确实有性能问题。`veaury` 的引入不是为了炫技，而是 Vue 生态确实没有满足性能要求的数据表格组件。每个技术决策都应该有明确的问题背景。
>
> 第三个是**封装的边界**。`smooth-dnd` 的封装只做了一层薄封装（事件映射 + 生命周期），没有过度抽象成「万能拖拽组件」。过度封装会增加理解成本，薄封装够用就好。

---

## 四、高频追问 & 加分回答

**Q19：`Object.assign` 和展开运算符在 `updateBlock` 中有什么区别？**

> `Object.assign(block, newBlock)` 是原地修改，保持响应式引用不变，Vue 检测到属性变化触发更新。如果用 `blocks.value = blocks.map(b => b.id === id ? { ...b, ...newBlock } : b)`，会创建新数组和新对象引用，Vue 认为整个 blocks 变了，触发所有消费 blocks 的组件重渲染。
>
> 这个优化在 8 个 Block 的场景下差异不大，但如果 Block 数量增加到 50+，或者 Block 内部有复杂嵌套，这个差异会很明显。

**Q20：`smooth-dnd` 的 `reactDropHandler` 是什么？为什么要设置它？**

> `smooth-dnd` 默认的 drop 处理器是直接修改原数组的 splice。`reactDropHandler` 返回一个纯函数（不修改原数组），符合 React 的不可变数据理念。在 Vue 中使用它也是有益的，因为 Pinia 的 `$patch` 和 Vue 的响应式系统对不可变更友好 — 直接修改 reactive 对象的内部数组有时会导致响应式追踪丢失。

**Q21：`v-memo` 在 Runner 里是怎么用的？**

> 运行态需要监听窗口 resize 事件自动切换设备预览。但 resize 事件非常频繁（拖动窗口边缘时每秒触发几十次），如果每次都重新渲染组件开销很大。
>
> `v-memo` 接收一个依赖数组，只有依赖变化时才重新渲染。`<LaptopPreviewer v-memo="[device === 'laptop']" />` 表示只有 `device` 从 `laptop` 变为其他值时才重新渲染。`device` 只在跨过移动端阈值时才变化，`v-memo` 避免了大量无效重渲染。

**Q22：Vue Flow 的 `useHandleConnections` 和 `useNodesData` 是怎么配合的？**

> `useHandleConnections({ type: 'target' })` 返回当前节点所有 target handle 的连接信息（source node id、source handle id），是响应式的 — 用户拖动连线时自动更新。
>
> `useNodesData(() => sourceIds)` 接收一个返回节点 ID 数组的函数，返回这些节点的 data，也是响应式的 — 源节点数据变化时自动更新。
>
> 两者配合形成一个完整的响应式链路：连接关系变化 → source IDs 变化 → 对应节点 data 自动追踪 → `computed` 重新计算结果。整个链路不需要手动监听，全是 Vue Flow 内部的响应式机制驱动的。

**Q23：Tiptap 的 `Decoration.inline` 和 `Decoration.widget` 有什么区别？**

> `Decoration.inline(start, end, attrs)` 在指定范围的文本上叠加 HTML 属性（class、style 等），不影响文档结构，适合高亮、着色等场景。`Decoration.widget(pos, dom)` 在指定位置插入一个 DOM 节点，适合插入按钮、图标等非文本元素。
>
> 我们用 `inline` 是因为颜色色块应该跟在颜色文本前面，用 CSS `::before` 伪元素实现，不需要额外的 DOM 节点。如果用 `widget`，色块会作为独立 DOM 节点插入，可能影响文本的选择和编辑行为。

---

## 五、面试准备 Checklist

| 准备项 | 重点 |
|--------|------|
| 业务背景 | 为什么做这个平台？解决什么问题？交付周期缩短了多少？ |
| 项目架构图 | 三栏布局 + 数据流，现场能画出来 |
| 8 种 Block 类型 | 每种的 type 值和核心 props |
| 拖拽系统 | 两层区别、applyDrag 三种场景、group-name 跨容器、为什么选 smooth-dnd |
| 策略模式 | 4 处应用、`<component :is>` 原理、为什么不用 if-else |
| vee-validate | useForm/useField/useFieldArray 的区别、为什么不用直接 watch、解决了什么性能问题 |
| provide/inject | 两处使用场景、和 props 的区别、层级深度 |
| veaury 桥接 | 为什么需要桥接（Vue 生态短板）、applyPureReactInVue 原理、踩坑 |
| Tiptap 扩展 | ProseMirror Plugin 架构、Decoration 类型、业务需求驱动 |
| Vue Flow | useHandleConnections + useNodesData 数据链路、业务场景 |
| 工程化 | Turborepo 拓扑排序 + 缓存收益、Husky 执行链、manualChunks 优化 |
| 项目不足 | 撤销重做、数据持久化、Block 联动、性能优化 — 每个都要有具体改进方案 |
| 现场演示 | 确认 `pnpm dev` 能正常跑起项目 |
