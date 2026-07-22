# Miaoma VBuilder 功能亮点、实现思路与数据流分析

---

## 一、功能亮点总览

| # | 亮点 | 核心技术 | 解决的问题 |
|---|------|----------|------------|
| 1 | 插件化 Block 注册体系 | `BlockSuite` 类 + `provide/inject` + `Symbol` | 新增组件零侵入渲染层 |
| 2 | 双层拖拽系统 | `smooth-dnd` Vue 3 封装 + `behaviour="copy"` | 从面板拖入 + 画布内排序 |
| 3 | 策略模式多引擎渲染 | `<component :is>` + `computed` 分发 | 图表/Setting/预览模式统一切换 |
| 4 | vee-validate 表单状态管理 | `useForm` + `useField` + `useFieldArray` + `watch` | 属性面板独立表单态，变更实时同步画布 |
| 5 | 编辑态/运行态双模式 | `provide/inject('editable')` + `v-memo` | 同一套组件服务搭建和预览两个场景 |
| 6 | 跨框架桥接 Vue+React | `veaury` 的 `applyPureReactInVue` | Vue 中嵌入 React 高性能数据表格 |
| 7 | Tiptap 自定义扩展 | ProseMirror `Plugin` + `Decoration.inline` | 富文本中十六进制颜色实时渲染色块 |
| 8 | SegmentControl 滑动指示器 | DOM 测量 `getBoundingClientRect` + CSS transform | 通用分段控制器，指示器跟随动画 |
| 9 | Vue Flow 自定义计算节点 | `useHandleConnections` + `useNodesData` | 可视化流程图中节点间数据传递与计算 |
| 10 | iPhone 模拟器还原 | SVG 状态栏 + 实时 `setInterval` 时钟 | 编辑器内高度还原真机预览效果 |

---

## 二、亮点 1：插件化 Block 注册体系

### 2.1 实现思路

核心思想：**将组件的「注册」与「渲染」解耦**。渲染层不关心有哪些组件，只通过 type 查表获取对应的 Vue 组件。

### 2.2 类型系统设计

```
types/block.ts
    │
    ├── BaseBlockInfo { id, label }         ← 所有 Block 的公共字段
    │
    ├── HeroTitleBlockInfo extends BaseBlockInfo
    │   └── type: 'heroTitle', props: { align, content, description? }
    ├── QuoteBlockInfo extends BaseBlockInfo
    │   └── type: 'quote', props: { content, status }
    ├── ChartBlockInfo extends BaseBlockInfo
    │   └── type: 'chart', props: { chartType }
    ├── ... (8 个具体类型)
    │
    └── BlockInfo = 联合类型 (Discriminated Union)
        └── 通过 `type` 字段做类型区分
```

**关键设计：** 使用 TypeScript 的**可辨识联合类型**（Discriminated Union），每个 Block 的 `type` 字段是字面量类型，配合 `switch(type)` 可以实现类型收窄（type narrowing），在 Setting 面板中拿到精确的 props 类型。

### 2.3 BlockSuite 注册机制

```ts
// blocks.ts 核心逻辑
class BlockSuite {
    private blocks = baseBlocks        // 内置 5 个基础 Block

    getBlocksMap() {
        return Object.fromEntries(
            this.blocks.map(block => [block.type, block])
        )                              // → { quote: {type, material}, heroTitle: {...}, ... }
    }

    addBlock(block) {
        this.blocks.push(block)        // 插件扩展入口
    }
}

const blockSuite = new BlockSuite()
blockSuite.addBlock({ type: 'button', material: ButtonBlock })   // 外部注册
blockSuite.addBlock({ type: 'form', material: FormBlock })
blockSuite.addBlock({ type: 'notes', material: NotesBlock })
```

### 2.4 Vue 插件注入

```ts
// initBlocks() 返回一个 Vue 插件
const initBlocks = () => ({
    install(app) {
        app.provide(blocksMapSymbol, blocksMap)         // provide/inject 链
        app.config.globalProperties.$blocksMap = blocksMap  // 全局属性
    }
})

// main.ts 中使用
app.use(initBlocks())
```

**两条注入路径：**
- `provide/inject` — 推荐方式，类型安全，适合 Composition API
- `globalProperties` — 备用方式，适合 Options API 或模板中直接使用 `$blocksMap`

### 2.5 渲染层消费

```vue
<!-- BlockRenderer.vue -->
<component :is="$blocksMap[block.type].material" :blockInfo="block" />
```

渲染层只做一件事：根据 `block.type` 查 `$blocksMap`，拿到 Vue 组件，通过 `<component :is>` 动态渲染。新增 Block 类型时，这个文件**不需要任何修改**。

### 2.6 数据流向图

```
┌─────────────────────────────────────────────────────────────┐
│                   Block 注册 & 渲染数据流                     │
│                                                             │
│  blocks.ts                                                  │
│  ┌──────────────┐     ┌─────────────┐                      │
│  │ baseBlocks[] │     │ addBlock()  │                      │
│  │ (5 个内置)   │     │ (3 个外部)  │                      │
│  └──────┬───────┘     └──────┬──────┘                      │
│         │                    │                              │
│         └────────┬───────────┘                              │
│                  ▼                                           │
│         BlockSuite.blocks = 8 个 Block                      │
│                  │                                           │
│         getBlocksMap()                                      │
│                  │                                           │
│                  ▼                                           │
│         blocksMap = { type → { type, material } }           │
│                  │                                           │
│    ┌─────────────┼─────────────────┐                        │
│    ▼                               ▼                        │
│  provide(blocksMapSymbol)    globalProperties.$blocksMap    │
│    │                               │                        │
│    ▼                               ▼                        │
│  inject(blocksMapSymbol)     this.$blocksMap                │
│    │                               │                        │
│    └───────────┬───────────────────┘                        │
│                ▼                                             │
│  BlockRenderer: <component :is="$blocksMap[type].material"> │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、亮点 2：双层拖拽系统

### 3.1 实现思路

项目有**两种拖拽场景**，共用同一套 `smooth-dnd` 库，通过 `behaviour` 和 `group-name` 配置区分：

| 场景 | 组件 | behaviour | 效果 |
|------|------|-----------|------|
| 从面板拖入画布 | `BlocksDrawer` | `copy` | 原始项保留，画布中新增副本 |
| 画布内排序 | `BlocksRenderer` | 默认 (move) | 元素在画布内移动位置 |

### 3.2 SmoothDnd Vue 3 封装

`smooth-dnd` 原生是 DOM 操作库，直接在 Vue 中使用会破坏响应式。项目用 `defineComponent` + render function 做了一层薄封装：

```ts
// SmoothDndContainer.ts
export const SmoothDndContainer = defineComponent({
    mounted() {
        // 将 Vue emit 映射为 smooth-dnd 回调
        const options = Object.assign({}, this.$props)
        for (const key in eventEmitterMap) {
            options[eventEmitterMap[key]] = (props) => {
                this.$emit(key, props)    // DOM 事件 → Vue 事件
            }
        }
        // 在 DOM 元素上初始化 smooth-dnd 容器
        this.container = smoothDnD(this.$el, options)
    },
    unmounted() {
        this.container?.dispose()         // 清理，防止内存泄漏
    },
    render() {
        return h(tagProps.value, { ref: 'container' }, this.$slots.default?.())
    }
})
```

**封装要点：**
- 生命周期绑定：`mounted` 初始化，`unmounted` 销毁
- 事件桥接：smooth-dnd 的回调函数映射为 Vue 的 `$emit`
- Props 透传：所有 smooth-dnd 配置（`orientation`、`behaviour`、`groupName` 等）作为组件 props

### 3.3 BlocksDrawer — 拖拽源（copy 模式）

```vue
<!-- BlocksDrawer.vue -->
<smooth-dnd-container
    behaviour="copy"              ← 拖出时保留原件
    group-name="blocks"           ← 与 BlocksRenderer 同组，允许跨容器拖拽
    :get-child-payload="index => getBlocksDefaultData(type)"  ← 拖拽时携带的数据
>
```

**`getChildPayload` 是关键：** 当用户从面板拖出一个 Block 时，`smooth-dnd` 调用这个函数获取 payload。`getBlocksDefaultData(type)` 返回一个带 `nanoid()` 的默认 Block 数据，这就是新 Block 的「出生证明」。

### 3.4 BlocksRenderer — 画布容器（move 模式）

```vue
<!-- BlocksRenderer.vue -->
<smooth-dnd-container
    drag-handle-selector=".handle"   ← 只有拖拽手柄可以触发拖拽
    group-name="blocks"              ← 与 BlocksDrawer 同组
    @drop="updateBlocks(applyDrag(toRaw(blocks), $event))"
>
```

**`applyDrag` 函数处理三种 drop 场景：**

```ts
const applyDrag = (arr, { removedIndex, addedIndex, payload }) => {
    const result = [...arr]

    // 场景 1：没有实际移动（拖出去又拖回来）
    if (addedIndex === null) return result

    // 场景 2：从 BlocksDrawer 拖入（removedIndex=null，有 payload）
    if (addedIndex !== null && removedIndex === null) {
        result.splice(addedIndex, 0, { id: `${Math.random()}`, ...payload })
    }

    // 场景 3：画布内重排（removedIndex 和 addedIndex 都有值）
    if (addedIndex !== null && removedIndex !== null) {
        return arrayMove(result, removedIndex, addedIndex)
    }

    return result
}
```

### 3.5 完整拖拽数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                    拖拽系统完整数据流                              │
│                                                                 │
│  ┌─────────────────┐                    ┌──────────────────┐    │
│  │  BlocksDrawer   │                    │  BlocksRenderer  │    │
│  │  (左侧组件面板)  │                    │  (中央画布)       │    │
│  │                 │   drag & drop      │                  │    │
│  │  behaviour=copy │ ─────────────────→ │  behaviour=move  │    │
│  │  group=blocks   │                    │  group=blocks    │    │
│  └────────┬────────┘                    └────────┬─────────┘    │
│           │                                      │              │
│  getChildPayload()                       @drop 事件触发         │
│  → getBlocksDefaultData(type)            → applyDrag()          │
│  → { id: nanoid(), type, props }         → 判断场景              │
│           │                                      │              │
│           │              payload                 │              │
│           └──────────────────────────────────────┘              │
│                                          │                      │
│                              ┌───────────┴───────────┐          │
│                              ▼                       ▼          │
│                    removedIndex=null         removedIndex有值    │
│                    addedIndex有值            addedIndex有值       │
│                    → splice 新增             → arrayMove 重排    │
│                              │                       │          │
│                              └───────────┬───────────┘          │
│                                          ▼                      │
│                              store.updateBlocks(newBlocks)      │
│                                          │                      │
│                                          ▼                      │
│                              Pinia blocks[] 更新                 │
│                                          │                      │
│                              ┌───────────┼───────────┐          │
│                              ▼           ▼           ▼          │
│                        BlockRenderer  Components   AppRightPanel│
│                        (画布重渲染)    (大纲更新)    (属性面板)   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、亮点 3：策略模式多处应用

项目中有三处使用了策略模式，核心都是 `<component :is>` + `computed` 分发：

### 4.1 BlockRenderer — 组件渲染策略

```vue
<component :is="$blocksMap[block.type].material" :blockInfo="block" />
```

根据 `block.type` 查表获取对应的 Vue 组件，8 种 Block 共用同一个渲染入口。

### 4.2 AppRightPanel — 属性面板策略

```ts
const blockSetting = computed(() => {
    switch (currentBlockInfo.value?.type) {
        case 'quote':    return QuoteSetting
        case 'chart':    return ChartSetting
        case 'image':    return ImageSetting
        case 'heroTitle': return HeroTitleSetting
        default:         return ''
    }
})
```

```vue
<component :is="blockSetting" :blockInfo="currentBlockInfo" @change="..." />
```

选中不同类型的 Block，右侧面板自动切换为对应的 Setting 组件。

### 4.3 ChartRenderer — 图表引擎策略

```ts
const renderer = computed(() => {
    switch (props.blockInfo.props.chartType) {
        case 'echarts': return EchartsRenderer
        case 'canvas':  return CanvasChartRenderer
        case 'svg':     return SVGChartRenderer
    }
})
```

```vue
<component :is="renderer" :block-info="blockInfo" />
```

### 4.4 AppEditorRenderer — 预览模式策略

```vue
<KeepAlive>
    <component
        :is="previewMode === 'laptop' ? LaptopPreviewer : MobilePreviewer"
    />
</KeepAlive>
```

`KeepAlive` 保证切换模式时保留组件状态（滚动位置、已输入内容等）。

### 4.5 策略模式统一架构

```
┌─────────────────────────────────────────────────────────┐
│                   策略模式统一架构                         │
│                                                         │
│  输入 (type/chartType/previewMode)                      │
│         │                                               │
│         ▼                                               │
│  computed 分发逻辑                                       │
│         │                                               │
│         ▼                                               │
│  返回对应的 Vue 组件                                      │
│         │                                               │
│         ▼                                               │
│  <component :is="strategy"> 渲染                        │
│         │                                               │
│         ▼                                               │
│  统一的 props 接口 (:blockInfo / :preview-mode)          │
└─────────────────────────────────────────────────────────┘
```

**优势：** 新增策略只需两步 —— 编写新组件 + 在 `switch` 中加一个 `case`。调用方代码零修改。

---

## 五、亮点 4：vee-validate 表单状态管理

### 5.1 问题背景

属性面板需要：
1. 独立的表单状态（不影响画布的 Block 数据）
2. 表单验证能力（必填、格式校验等）
3. 变更时实时同步到画布

### 5.2 实现方案（以 HeroTitleSetting 为例）

```ts
// ① 初始化表单，从 Block props 读取初始值
const { values } = useForm({
    initialValues: {
        content: props.blockInfo.props.content,
        align: props.blockInfo.props.align,
        description: props.blockInfo.props.description
    }
})

// ② 绑定单个字段
const { value: content, handleChange: handleContentChange } = useField('content')
const { value: align, handleChange: handleAlignChange } = useField('align')

// ③ 绑定动态数组字段（description 列表）
const { fields, push } = useFieldArray('description')

// ④ 监听表单变更，emit 给父组件
watch([values], ([newValues]) => {
    emit('change', {
        ...props.blockInfo,
        props: { ...props.blockInfo.props, ...newValues }
    })
})
```

### 5.3 表单数据流

```
┌───────────────────────────────────────────────────────────┐
│                属性面板表单数据流                            │
│                                                           │
│  Block 数据 (Pinia)                                       │
│       │                                                   │
│       ▼  props 传入                                       │
│  Setting 组件                                              │
│       │                                                   │
│       ▼  initialValues                                     │
│  useForm() 创建独立表单状态                                 │
│       │                                                   │
│       ├── useField('content') → v-model 绑定 <input>      │
│       ├── useField('align')   → SegmentedControl 绑定      │
│       └── useFieldArray('description') → 动态列表          │
│       │                                                   │
│       ▼  用户交互                                          │
│  表单值变化                                                │
│       │                                                   │
│       ▼  watch(values)                                    │
│  emit('change', newBlockInfo)                              │
│       │                                                   │
│       ▼                                                   │
│  AppRightPanel: @change → store.updateBlock(id, block)    │
│       │                                                   │
│       ▼                                                   │
│  Pinia blocks[] 更新 → BlockRenderer 响应式重渲染          │
└───────────────────────────────────────────────────────────┘
```

### 5.4 为什么不用直接 watch Block 数据？

| 方案 | 优点 | 缺点 |
|------|------|------|
| 直接 watch Block | 简单 | 用户输入过程中频繁触发画布重绘；无表单验证能力 |
| vee-validate | 独立表单态、验证能力、脏值检测 | 多一层抽象 |

---

## 六、亮点 5：编辑态/运行态双模式

### 6.1 provide/inject 控制

```ts
// AppRunnerRenderer.vue — 运行模式
provide('editable', false)

// BlockRenderer.vue — 消费端
const editable = inject('editable', true)   // 默认 true（编辑模式）

// NotesEditor.vue — 消费端
const editable = inject('editable', true)
```

### 6.2 editable 控制的 UI 元素

| 组件 | editable=true | editable=false |
|------|---------------|----------------|
| BlockRenderer | 显示选中边框、拖拽手柄、删除按钮 | 纯内容展示 |
| NotesEditor | 显示富文本工具栏（加粗/斜体/删除线） | 只读渲染 |
| SmoothDndContainer | 允许拖拽排序 | 不可拖拽 |

### 6.3 Runner 自动检测设备

```ts
// AppRunnerRenderer.vue
const device = ref<'laptop' | 'mobile'>('laptop')

const resize = () => {
    device.value = isMobileTablet() ? 'mobile' : 'laptop'
}

onMounted(() => {
    resize()  // 初始检测
    window.addEventListener('resize', resize)
})

onUnmounted(() => {
    window.removeEventListener('resize', resize)
})
```

```vue
<LaptopPreviewer v-memo="[device === 'laptop']" v-if="device === 'laptop'" />
<MobilePreviewer v-memo="[device === 'mobile']" v-if="device === 'mobile'" />
```

`v-memo` 优化：只有当 `device` 变化时才重新渲染，窗口大小的微小变化不会触发重绘。

### 6.4 双模式数据流

```
┌──────────────────────────────────────────────────────────┐
│                 编辑态 vs 运行态                           │
│                                                          │
│  编辑模式 (/app/layout)                                  │
│  ┌──────────────────────────────────────────────┐        │
│  │  PageLayoutView                              │        │
│  │  ├── AppLeftPanel (组件面板 + 大纲)           │        │
│  │  ├── AppEditorRenderer                       │        │
│  │  │   └── KeepAlive → LaptopPreviewer         │        │
│  │  │       └── BlocksRenderer                  │        │
│  │  │           └── BlockRenderer (editable=true)│       │
│  │  │               ├── 选中边框                  │        │
│  │  │               ├── 拖拽手柄                  │        │
│  │  │               └── 删除按钮                  │        │
│  │  └── AppRightPanel (属性编辑)                  │        │
│  └──────────────────────────────────────────────┘        │
│                                                          │
│  运行模式 (/runner)                                      │
│  ┌──────────────────────────────────────────────┐        │
│  │  RunnerView                                  │        │
│  │  └── AppRunnerRenderer                       │        │
│  │      provide('editable', false)              │        │
│  │      ├── isMobileTablet() 检测设备            │        │
│  │      └── LaptopPreviewer / MobilePreviewer   │        │
│  │          └── BlocksRenderer                  │        │
│  │              └── BlockRenderer (editable=false)│       │
│  │                  └── 纯内容展示，无编辑 UI     │        │
│  └──────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────┘
```

---

## 七、亮点 6：跨框架桥接（Vue + React）

### 7.1 实现原理

```ts
// DataSourceContent.vue
import { applyPureReactInVue } from 'veaury'
import ReactDataSource from './react_app/ReactDataSource'

const RDataSource = applyPureReactInVue(ReactDataSource)
```

```vue
<r-dataSource :id="id" />
```

`applyPureReactInVue` 的工作原理：
1. 创建一个 Vue 组件
2. 在该组件的 `mounted` 钩子中，用 `ReactDOM.createRoot()` 渲染 React 组件
3. 将 Vue 的 props 映射为 React 的 props
4. 在 `updated` 钩子中同步 props 变更
5. 在 `unmounted` 钩子中调用 `root.unmount()` 清理

### 7.2 构建配置

```ts
// vite.config.ts
import veauryVitePlugins from 'veaury/vite/index.js'

plugins: [
    veauryVitePlugins({ type: 'vue' })  // 同时注册 vue + react 编译器
]
```

`veaury` 内部会同时注册 `@vitejs/plugin-vue` 和 `@vitejs/plugin-react`，使 `.vue` 和 `.jsx/.tsx` 文件可以在同一项目中混用。

### 7.3 数据流

```
Vue 侧                          React 侧
─────────                       ──────────
DataSourceContent.vue
    │
    ├── props: { id }
    │
    ▼
applyPureReactInVue()
    │
    ├── createRoot(container)
    │
    ▼
ReactDataSource.jsx
    ├── props.id
    └── @glideapps/glide-data-grid
        └── 10 万行数据表格渲染
```

---

## 八、亮点 7：Tiptap 自定义扩展 — ColorHighlighter

### 8.1 实现原理

```ts
// ColorHighlighter.ts
export const ColorHighlighter = Extension.create({
    name: 'colorHighlighter',
    addProseMirrorPlugins() {
        return [
            new Plugin({
                state: {
                    init(_, { doc }) {
                        return findColors(doc)        // 初始化时扫描
                    },
                    apply(transaction, oldState) {
                        return transaction.docChanged
                            ? findColors(transaction.doc)  // 文档变化时重新扫描
                            : oldState
                    }
                },
                props: {
                    decorations(state) {
                        return this.getState(state)    // 返回 Decoration 集合
                    }
                }
            })
        ]
    }
})
```

### 8.2 findColors 函数

```ts
// helper.ts
const regex = /#[([0-9a-fA-F]{6})\b/g    // 匹配十六进制颜色

export const findColors = (doc) => {
    const decorations: Decoration[] = []

    doc.descendants((node, pos) => {
        // 遍历文档所有文本节点
        if (!node.isText) return

        let match
        while ((match = regex.exec(node.text))) {
            const color = match[1]
            const start = pos + match.index
            const end = start + match[0].length

            decorations.push(
                Decoration.inline(start, end, {
                    class: 'color',                    // CSS 类名
                    style: `--color: #${color}`        // CSS 自定义属性
                })
            )
        }
    })

    return DecorationSet.create(doc, decorations)
}
```

### 8.3 CSS 渲染

```css
.color::before {
    content: ' ';
    display: inline-block;
    width: 1em;
    height: 1em;
    background-color: var(--color);    ← 使用 CSS 自定义属性
    border: 1px solid rgba(128, 128, 128, 0.3);
    border-radius: 2px;
}
```

### 8.4 数据流

```
用户输入文本 "#FF5733"
    │
    ▼
ProseMirror 文档更新 (transaction.docChanged)
    │
    ▼
Plugin.state.apply() 触发
    │
    ▼
findColors(doc) 遍历所有文本节点
    │
    ├── regex.exec() 匹配 "#FF5733"
    │
    ▼
Decoration.inline(start, end, { class: 'color', style: '--color: #FF5733' })
    │
    ▼
DecorationSet 返回给 ProseMirror
    │
    ▼
ProseMirror 渲染时在 "#FF5733" 前插入 <span class="color" style="--color: #FF5733">
    │
    ▼
CSS .color::before 渲染一个 1em × 1em 的色块
```

---

## 九、亮点 8：SegmentedControl 滑动指示器

### 9.1 核心实现

```ts
// 监听选中值变化，计算指示器位置
watch([innerValue, observerRef], ([v]) => {
    const element = refs.value[v]                    // 获取选中项的 DOM 元素
    const elementRect = element.getBoundingClientRect()  // 测量尺寸

    const scaledValue = element.offsetWidth / elementRect.width
    activePosition.width = elementRect.width * scaledValue
    activePosition.height = elementRect.height
    activePosition.translate = [
        element.parentElement.offsetLeft,            // 相对于容器的偏移
        element.parentElement.offsetTop
    ]
})
```

```vue
<span class="segmented-control-indicator"
    :style="{
        width: `${width}px`,
        height: `${height}px`,
        transform: `translate(${translate[0]}px, ${translate[1]}px)`
    }"
/>
```

### 9.2 设计亮点

- **DOM 测量驱动**：不硬编码宽度，通过 `getBoundingClientRect()` 实时测量，自动适配不同文字长度
- **CSS transform 动画**：`transition: all 0.2s ease-in-out` 实现滑动效果
- **隐藏原生 radio**：`input[type=radio]` 设置 `width:0; height:0; overflow:hidden`，用自定义 label 替代

---

## 十、亮点 9：Vue Flow 自定义计算节点

### 10.1 ResultNode 数据流

```ts
// 1. 获取输入连接（从 ResultNode 往回找）
const sourceConnections = useHandleConnections({ type: 'target' })

// 2. 获取 Operator 节点的输入连接（从 OperatorNode 往回找 ValueNode）
const operatorSourceConnections = computed(() =>
    getConnectedEdges(sourceConnections.value[0].source)
        .filter(e => e.source !== sourceConnections.value[0].source)
)

// 3. 获取连接节点的数据
const operatorData = useNodesData(() =>
    sourceConnections.value.map(c => c.source)
)
const valueData = useNodesData(() =>
    operatorSourceConnections.value.map(c => c.source)
)

// 4. 计算结果
const result = computed(() => {
    return operatorData.value.reduce((acc, { data }) => {
        const operator = data?.operator
        if (operator) {
            const [a, b] = valueData.value.map(({ data }) => data?.value)
            if (a && b) return mathFunctions[operator](a, b)
        }
        return acc
    }, 0)
})
```

### 10.2 节点间数据传递架构

```
ValueNode (10)  ──→  OperatorNode (+)  ──→  ResultNode
ValueNode (30)  ──→        │                   │
                          │                   │
                    useNodesData()       useHandleConnections()
                    读取 operator        读取 source connection
                          │                   │
                          └───────┬───────────┘
                                  ▼
                         computed result = 10 + 30 = 40
```

---

## 十一、组件封装分析

### 11.1 封装层级总览

```
┌──────────────────────────────────────────────────────────────┐
│                      组件封装层级                              │
│                                                              │
│  第 1 层：UI 原子组件                                         │
│  ├── SegmentedControl    (通用分段控制器)                      │
│  └── SmoothDnd*          (拖拽原语封装)                       │
│                                                              │
│  第 2 层：Block 物料组件                                      │
│  ├── HeroTitleBlock      (标题渲染)                           │
│  ├── QuoteBlock          (引用渲染)                           │
│  ├── ImageBlock          (图片渲染)                           │
│  ├── ChartBlock          (图表渲染 → 委托给 ChartRenderer)    │
│  ├── ButtonBlock         (按钮渲染)                           │
│  ├── FormBlock           (表单渲染)                           │
│  └── NotesBlock          (富文本渲染 → 委托给 NotesEditor)    │
│                                                              │
│  第 3 层：编辑器复合组件                                       │
│  ├── BlockRenderer       (单个 Block 渲染 + 选中态 + 工具栏)  │
│  ├── BlocksRenderer      (Block 列表 + DnD 容器)             │
│  ├── *Setting            (属性编辑面板 × 4)                   │
│  └── SchemaExporter      (JSON 预览)                         │
│                                                              │
│  第 4 层：容器/布局组件                                       │
│  ├── AppLeftPanel        (左侧面板壳)                        │
│  ├── AppEditorRenderer   (中央画布壳 + 模式切换)              │
│  ├── AppRightPanel       (右侧面板壳 + 策略分发)              │
│  ├── LaptopPreviewer     (电脑模拟器壳)                      │
│  └── MobilePreviewer     (iPhone 模拟器壳)                   │
│                                                              │
│  第 5 层：页面级组件                                          │
│  ├── PageLayoutView      (三栏布局)                          │
│  ├── DataSourceView      (数据源两栏布局)                     │
│  ├── ActionsView         (动作流两栏布局)                     │
│  └── RunnerView          (运行器布局)                        │
└──────────────────────────────────────────────────────────────┘
```

### 11.2 各层封装要点

#### 第 1 层：UI 原子组件

**SegmentedControl** — 通用性设计：
- Props: `{ value?: string, data: { value, label }[] }`
- 内部管理指示器位置，外部只关心选中值
- 通过 `emit('change', value)` 向外通信
- 被 ChartSetting、QuoteSetting、HeroTitleSetting 复用

**SmoothDndContainer / SmoothDndDraggable** — 桥接封装：
- 将 DOM 库 `smooth-dnd` 封装为 Vue 组件
- Props 1:1 映射 smooth-dnd 配置
- 生命周期管理（mounted 初始化，unmounted 销毁）
- 事件桥接（DOM 回调 → Vue emit）

#### 第 2 层：Block 物料组件

**统一接口约定：**
```ts
// 所有 Block 组件接收相同的 prop
defineProps<{ blockInfo: BlockInfo }>()
```

**ChartBlock 委托模式：**
```vue
<!-- ChartBlock.vue — 只做一件事：委托给 ChartRenderer -->
<ChartRenderer :block-info="blockInfo" />
```

**NotesBlock 委托模式：**
```vue
<!-- NotesBlock.vue — 只做一件事：委托给 NotesEditor -->
<NotesEditor />
```

Block 组件本身保持简单，复杂逻辑委托给专门的渲染器组件。

#### 第 3 层：编辑器复合组件

**BlockRenderer — 职责组合：**
```
BlockRenderer = 动态组件渲染 + 选中态管理 + 编辑工具栏 + 点击选中事件
```

```vue
<div @click.stop="selectBlock(block.id)">
    <!-- 渲染层：动态组件 -->
    <component :is="$blocksMap[block.type].material" :blockInfo="block" />

    <!-- 编辑层：选中态 + 工具栏（仅 editable 模式） -->
    <div v-if="editable" :class="{ selected: currentBlockId === block.id }">
        <div v-if="currentBlockId === block.id" class="block-toolbar">
            <drag />      <!-- 拖拽手柄 -->
            <delete />    <!-- 删除按钮 -->
        </div>
    </div>
</div>
```

**关键设计：** 编辑层（indicator + toolbar）用 `position: absolute` 叠加在渲染层上方，`pointer-events: none` 让点击穿透到渲染层，但工具栏区域用 `pointer-events: visible` 恢复交互。

**BlocksRenderer — 容器职责：**
```
BlocksRenderer = SmoothDnd 容器 + drop 事件处理 + 数组操作
```

#### 第 4 层：容器/布局组件

**AppRightPanel — 策略分发容器：**
```
AppRightPanel = Breadcrumb + 策略分发 Setting + SchemaExporter
```

```vue
<Breadcrumb :blockInfo="currentBlockInfo" @goHome="unSelectBlock()" />

<template v-if="currentBlockInfo">
    <!-- 有选中 Block：策略分发 Setting -->
    <component :is="blockSetting" :blockInfo="currentBlockInfo" @change="..." />
    <SchemaExporter :currentInfo="currentBlockInfo" />
</template>
<template v-else>
    <!-- 无选中：展示全部 blocks JSON -->
    <SchemaExporter :currentInfo="appEditorStore.blocks" />
</template>
```

**AppEditorRenderer — 模式切换容器：**
```
AppEditorRenderer = KeepAlive + 动态组件 (Laptop/Mobile)
```

#### 第 5 层：页面级组件

**PageLayoutView — 最终组装：**
```vue
<AppLeftPanel />           <!-- 20% 宽度 -->
<AppEditorRenderer />      <!-- 自适应宽度 -->
<AppRightPanel />          <!-- 360px 固定宽度 (var(--panel-width)) -->
```

### 11.3 组件通信方式汇总

| 通信方式 | 使用场景 | 示例 |
|----------|----------|------|
| **Props ↓ / Emit ↑** | 父子组件直接通信 | AppRightPanel → Setting 组件 |
| **Pinia Store** | 跨层级共享状态 | BlockRenderer ↔ AppRightPanel |
| **provide / inject** | 跨层级配置注入 | editable、blocksMap |
| **$blocksMap (globalProperties)** | 全局查表 | BlockRenderer 渲染 |
| **Router** | 页面级导航 | AppNavigator → 各 View |

### 11.4 组件复用关系图

```
┌───────────────────────────────────────────────────────────────┐
│                    组件复用关系                                 │
│                                                               │
│  SmoothDndContainer                                           │
│  ├── 被 BlocksDrawer 使用 (behaviour="copy")                  │
│  └── 被 BlocksRenderer 使用 (behaviour="move")                │
│                                                               │
│  SegmentedControl                                              │
│  ├── 被 ChartSetting 使用 (echarts/canvas/svg)                │
│  ├── 被 QuoteSetting 使用 (success/warning/error)             │
│  └── 被 HeroTitleSetting 使用 (left/center/right)             │
│                                                               │
│  BlocksRenderer                                                │
│  ├── 被 AppPreviewer/LaptopPreviewer 使用 (编辑模式)           │
│  ├── 被 AppPreviewer/MobilePreviewer 使用 (编辑模式)           │
│  └── 被 AppRunnerRenderer 内的 Previewer 使用 (运行模式)       │
│                                                               │
│  BlockRenderer                                                 │
│  └── 被 BlocksRenderer 内部循环渲染                            │
│                                                               │
│  LaptopPreviewer / MobilePreviewer                             │
│  ├── 被 AppEditorRenderer 使用 (编辑器模拟器)                  │
│  └── 被 AppRunnerRenderer 使用 (运行器预览)                    │
│                                                               │
│  SchemaExporter                                                │
│  └── 被 AppRightPanel 始终渲染 (选中时展示当前 Block JSON)     │
└───────────────────────────────────────────────────────────────┘
```

---

## 十二、核心数据流全景图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        核心数据流全景                                  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                    数据源                                  │       │
│  │                                                           │       │
│  │  mocks/blocks.ts (初始 8 个 Block)                        │       │
│  │       │                                                   │       │
│  │       ▼                                                   │       │
│  │  Pinia Store (appEditor)                                  │       │
│  │  ├── blocks: BlockInfo[]                                  │       │
│  │  └── currentBlockId: string | null                        │       │
│  └───────────────────────────┬───────────────────────────────┘       │
│                              │                                       │
│              ┌───────────────┼───────────────┐                       │
│              ▼               ▼               ▼                       │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐              │
│  │   左侧面板     │ │   中央画布     │ │   右侧面板     │              │
│  │               │ │               │ │               │              │
│  │ BlocksDrawer  │ │ BlocksRenderer│ │ AppRightPanel │              │
│  │ ──── DnD ────→│ │ ── 渲染 ────→ │ │ ── 编辑 ────→ │              │
│  │ (copy 模式)   │ │ BlockRenderer │ │ Setting 组件  │              │
│  │               │ │ ── 选中 ────→ │ │ ── emit ────→ │              │
│  │ getChildPayload│ │ selectBlock() │ │ updateBlock() │              │
│  │ → 默认数据     │ │               │ │               │              │
│  └───────────────┘ └───────────────┘ └───────────────┘              │
│                              │                                       │
│                              ▼                                       │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                    Pinia actions                           │       │
│  │                                                           │       │
│  │  selectBlock(id)     → currentBlockId = id                │       │
│  │  unSelectBlock()     → currentBlockId = null              │       │
│  │  updateBlocks(arr)   → blocks = arr (拖拽排序/新增)       │       │
│  │  updateBlock(id, b)  → Object.assign(block, b) (属性编辑) │       │
│  └───────────────────────────────────────────────────────────┘       │
│                              │                                       │
│                              ▼                                       │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                    响应式更新                               │       │
│  │                                                           │       │
│  │  storeToRefs(blocks) → 所有消费 blocks 的组件自动更新       │       │
│  │  storeToRefs(currentBlockId) → 右侧面板自动切换            │       │
│  └───────────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```
