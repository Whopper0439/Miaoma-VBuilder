# Miaoma VBuilder - 代码执行路径文档

> 按功能模块梳理完整的代码执行路径，从入口到渲染的每一步调用链。

---

## 一、应用启动路径

### 1.1 入口初始化

```
main.ts
  ├── createApp(App)                        // 创建 Vue 应用实例
  ├── createPinia()                         // 创建 Pinia 状态管理
  ├── Sentry.init({ ... })                  // 初始化 Sentry 监控 (browserTracing + replay)
  ├── app.use(pinia)                        // 注册 Pinia
  ├── app.use(router)                       // 注册 Vue Router
  ├── app.use(initBlocks())                 // 注册 Block 系统插件
  │   └── blocks.ts → initBlocks()
  │       └── install(app)
  │           ├── app.provide(blocksMapSymbol, blocksMap)  // provide 注入
  │           └── app.config.globalProperties.$blocksMap = blocksMap  // 全局属性
  └── app.mount('#app')                     // 挂载到 DOM
```

### 1.2 BlockSuite 注册链

```
blocks.ts
  ├── baseBlocks = [quote, heroTitle, view, chart, image]   // 基础 Block 定义
  ├── new BlockSuite()                                       // 创建 Block 管理器
  │   ├── getBlocksMap()    → { quote: {...}, heroTitle: {...}, ... }
  │   ├── getBlocks()       → [...]
  │   ├── addBlock(block)   → this.blocks.push(block)
  │   └── hasBlock(type)    → boolean
  ├── blockSuite.addBlock({ type: 'button', material: ButtonBlock })   // 外部 Block
  ├── blockSuite.addBlock({ type: 'form', material: FormBlock })
  ├── blockSuite.addBlock({ type: 'notes', material: NotesBlock })
  └── blocksMap = blockSuite.getBlocksMap()                  // 最终 Block 映射表
```

---

## 二、路由与页面结构

### 2.1 路由配置

```
router/index.ts
  ├── /                     → redirect → /app/layout
  ├── /app                  → AppView (主布局壳)
  │   ├── /app/layout       → PageLayoutView (页面编辑器，默认页)
  │   ├── /app/dataSource   → DataSourceView
  │   │   └── /app/dataSource/:id → DataSourceDetailView → DataSourceContent
  │   └── /app/actions      → ActionsView
  │       └── /app/actions/:id → ActionDetailView → FlowEditor
  └── /runner               → RunnerView → AppRunnerRenderer (运行态)
```

### 2.2 页面组装路径

```
App.vue
  └── <RouterView />

AppView.vue  (/app)
  ├── header → AppNavigator         // 顶部导航栏
  └── main   → <RouterView />       // 子路由出口

PageLayoutView.vue  (/app/layout)   ← 默认首页
  ├── AppLeftPanel                  // 左侧面板
  ├── AppEditorRenderer             // 中间模拟器
  └── AppRightPanel                 // 右侧属性面板
```

---

## 三、页面布局编辑器 (核心功能)

### 3.1 三栏布局执行路径

```
PageLayoutView.vue
  │
  ├── AppLeftPanel.vue
  │   ├── Navigation.vue            // 上半区: 页面导航列表 (静态数据)
  │   ├── Components.vue            // 下半区: 已有 Block 组件大纲
  │   │   └── 从 appEditorStore.blocks 读取 → 用 blocksBaseMeta 映射图标/名称
  │   └── BlocksDrawer.vue          // 弹出抽屉: 可拖拽的 Block 素材库
  │       └── smooth-dnd-container (behaviour="copy", group-name="blocks")
  │           └── getChildPayload(index) → getBlocksDefaultData(type) → 返回带 nanoid 的默认数据
  │
  ├── AppEditorRenderer.vue
  │   ├── previewMode = ref('laptop')       // 默认桌面模式
  │   ├── KeepAlive + dynamic component     // 切换模拟器
  │   │   ├── LaptopPreviewer               // 桌面模拟器
  │   │   └── MobilePreviewer               // 手机模拟器
  │   └── @preview-mode-change → previewMode.value = mode
  │
  └── AppRightPanel.vue
      ├── useAppEditorStore() → blocks, currentBlockId
      ├── computed: currentBlockInfo → blocksMap[currentBlockId]
      ├── computed: blockSetting → switch(currentBlockInfo.type)
      │   ├── 'quote'     → QuoteSetting
      │   ├── 'chart'     → ChartSetting
      │   ├── 'image'     → ImageSetting
      │   ├── 'heroTitle' → HeroTitleSetting
      │   └── default     → ''
      ├── <component :is="blockSetting" @change="updateBlock" />   // 策略模式
      └── SchemaExporter → vue-json-pretty + copy-to-clipboard
```

### 3.2 拖拽添加 Block 执行路径

```
用户从 BlocksDrawer 拖出一个 Block
  │
  ├── BlocksDrawer.vue
  │   └── smooth-dnd-container (behaviour="copy")
  │       └── getChildPayload(index)
  │           └── getBlocksDefaultData(type)    // constants/blocksBaseMeta.ts
  │               └── switch(type) → { id: nanoid(), type, label, props: {...默认值} }
  │
  ├── smooth-dnd 自动处理跨容器拖拽 (group-name="blocks" 匹配)
  │
  ├── BlocksRenderer.vue (目标容器)
  │   └── @drop → updateBlocks(applyDrag(toRaw(blocks), $event))
  │       └── applyDrag()
  │           ├── removedIndex === null → 添加模式
  │           │   └── result.splice(addedIndex, 0, { id: random, ...payload })
  │           └── removedIndex !== null → 移动模式
  │               └── arrayMove(result, removedIndex, addedIndex)
  │
  └── appEditorStore.updateBlocks(newBlocks)    // Pinia 状态更新
      └── blocks.value = newBlocks              // 响应式触发重渲染
```

### 3.3 Block 选中与编辑执行路径

```
用户点击画布中的某个 Block
  │
  ├── BlockRenderer.vue
  │   └── @click.stop="selectBlock(block.id)"
  │       └── appEditorStore.selectBlock(id)
  │           └── currentBlockId.value = id     // Pinia 状态更新
  │
  ├── BlockRenderer 响应式变化
  │   └── :class="{ selected: currentBlockId === block.id }"  // 显示选中边框
  │       └── 显示 block-toolbar (拖拽手柄 + 删除按钮)
  │
  ├── AppRightPanel 响应式变化
  │   ├── currentBlockInfo = computed → blocksMap[currentBlockId]
  │   ├── blockSetting = computed → switch(type) → 对应 Setting 组件
  │   └── <component :is="blockSetting" :blockInfo="currentBlockInfo" @change="..." />
  │
  └── Setting 组件内部 (以 HeroTitleSetting 为例)
      ├── useForm({ initialValues: { content, align, description } })
      ├── useField('content'), useField('align'), useFieldArray('description')
      ├── watch(values) → emit('change', { ...blockInfo, props: { ...newValues } })
      └── AppRightPanel @change → appEditorStore.updateBlock(id, newBlock)
          └── Object.assign(block, newBlock)    // 原地更新，触发响应式
```

### 3.4 Block 拖拽排序执行路径

```
用户在画布内拖拽 Block 排序
  │
  ├── BlockRenderer.vue
  │   └── .handle (drag-handle-selector=".handle")   // 只有拖拽手柄可触发
  │
  ├── BlocksRenderer.vue
  │   └── smooth-dnd-container (group-name="blocks")
  │       └── @drop → updateBlocks(applyDrag(toRaw(blocks), $event))
  │           └── applyDrag() → removedIndex !== null && addedIndex !== null
  │               └── arrayMove(result, removedIndex, addedIndex)  // utils/array.ts
  │                   └── newArray.splice(to, 0, newArray.splice(from, 1)[0])
  │
  └── appEditorStore.updateBlocks(newBlocks) → blocks.value 更新 → UI 重渲染
```

### 3.5 Block 删除执行路径

```
用户点击 Block 上方工具栏的删除按钮
  │
  └── BlockRenderer.vue
      └── @click="blocks.splice(i, 1)"    // 直接操作 Pinia storeToRefs 的响应式引用
          └── blocks 数组减少一项 → BlocksRenderer v-for 重渲染
```

---

## 四、模拟器系统

### 4.1 模式切换执行路径

```
AppEditorRenderer.vue
  ├── previewMode = ref<PreviewType>('laptop')
  ├── KeepAlive + <component :is="previewMode === 'laptop' ? LaptopPreviewer : MobilePreviewer" />
  │
  ├── PreviewModeSwitcher.vue
  │   ├── icons: [{ type: 'mobile', icon: Iphone }, { type: 'laptop', icon: LaptopOne }]
  │   └── @click="greet(icon.type)" → emit('preview-mode-change', mode)
  │       └── 逐层冒泡: MobilePreviewer/LaptopPreviewer → AppEditorRenderer
  │           └── handleModeChange(mode) → previewMode.value = mode
  │
  └── KeepAlive 缓存已渲染的组件实例，切换时保留状态
```

### 4.2 手机模拟器渲染路径

```
MobilePreviewer.vue
  ├── <PreviewModeSwitcher />           // 顶部切换按钮
  ├── <StatusBar />                     // 模拟状态栏 (时间/电量)
  ├── <AppMobilePreviewer />            // 内容区
  │   └── AppPreviewer/MobilePreviewer.vue
  │       ├── 导航栏: "MiaoMa vBuilder"
  │       └── <BlocksRenderer />        // 复用同一个 Block 渲染器
  └── <TabBar />                        // 底部 Tab 栏
```

### 4.3 桌面模拟器渲染路径

```
LaptopPreviewer.vue
  ├── <div class="address-wrapper">https://miaomaedu.com/path/to/yoursite</div>  // 地址栏
  ├── <PreviewModeSwitcher @full-screen="toggle" />
  │   └── toggle() → runner.value.requestFullscreen() / document.exitFullscreen()
  └── <AppLaptopPreviewer />
      └── AppPreviewer/LaptopPreviewer.vue
          ├── 导航栏: Logo + "MiaoMa vBuilder"
          └── <BlocksRenderer />        // 复用同一个 Block 渲染器
```

---

## 五、Block 渲染引擎

### 5.1 BlocksRenderer → BlockRenderer 渲染链

```
BlocksRenderer.vue
  ├── storeToRefs(appEditorStore) → blocks
  ├── smooth-dnd-container (group-name="blocks", orientation="vertical")
  │   └── @drop → updateBlocks(applyDrag(...))
  └── v-for="(block, i) in blocks"
      └── <BlockRenderer :block="block" :i="i" />

BlockRenderer.vue
  ├── inject('editable', true)              // 默认可编辑; Runner 模式下为 false
  ├── storeToRefs(appEditorStore) → currentBlockId, blocks
  ├── @click.stop → selectBlock(block.id)
  ├── <component :is="$blocksMap[block.type].material" :blockInfo="block" />
  │   └── 从全局 $blocksMap 查找对应组件:
  │       ├── 'heroTitle' → HeroTitleBlock.vue
  │       ├── 'quote'     → QuoteBlock.vue
  │       ├── 'view'      → ViewBlock.vue
  │       ├── 'chart'     → ChartBlock.vue
  │       ├── 'image'     → ImageBlock.vue
  │       ├── 'button'    → ButtonBlock.vue
  │       ├── 'form'      → FormBlock.vue
  │       └── 'notes'     → NotesBlock.vue
  └── v-if="editable"
      └── 选中态 UI: 蓝色边框 + 工具栏 (拖拽/删除)
```

### 5.2 各 Block 组件渲染细节

```
HeroTitleBlock.vue
  ├── props: { blockInfo: HeroTitleBlockInfo }
  ├── computed: style → { 'text-align': props.blockInfo.props.align }
  └── <h1 :style="style">{{ content }}</h1> + <li> description 列表

QuoteBlock.vue
  ├── props: { blockInfo: QuoteBlockInfo }
  ├── STATUS_MAP: { success/warning/error → color, bgColor, icon }
  ├── computed: style → STATUS_MAP[status]
  └── <div :style="style"> <icon/> <span>{{ content }}</span> </div>

ImageBlock.vue
  ├── props: { blockInfo: ImageBlockInfo }
  └── <img :src="blockInfo.props.url" />

ViewBlock.vue
  ├── props: { blockInfo: ViewBlockInfo }
  └── 数据表格展示 (fields + data)

ButtonBlock.vue
  ├── props: { blockInfo: ButtonBlockInfo }
  └── <button>{{ content }}</button>

FormBlock.vue
  ├── props: { blockInfo: FormBlockInfo }
  └── 动态表单字段渲染 (fields 数组)

ChartBlock.vue → ChartRenderer.vue
  └── 详见「图表多渲染器」章节

NotesBlock.vue → NotesEditor.vue
  └── 详见「富文本编辑器」章节
```

---

## 六、图表多渲染器系统

### 6.1 渲染器选择执行路径

```
ChartBlock.vue
  └── <ChartRenderer :blockInfo="blockInfo" />

ChartRenderer.vue
  ├── props: { blockInfo: ChartBlockInfo }
  ├── computed: renderer → switch(blockInfo.props.chartType)
  │   ├── 'echarts' → EchartsRenderer
  │   ├── 'canvas'  → CanvasChartRenderer
  │   └── 'svg'     → SVGChartRenderer
  └── <component :is="renderer" :blockInfo="blockInfo" />
```

### 6.2 EchartsRenderer 执行路径

```
EchartsRenderer.vue
  ├── use([CanvasRenderer, GraphChart, TitleComponent, TooltipComponent, ...])  // 按需注册
  ├── chartContainer = ref<HTMLDivElement>()
  ├── onMounted:
  │   ├── chartInstance = echarts.init(chartContainer.value)
  │   ├── fetchChartData() → MOCK_DATA (关系图数据)
  │   │   └── chartInstance.setOption({
  │   │       series: [{ type: 'graph', layout: 'circular', data, links, categories }]
  │   │   })
  │   └── window.addEventListener('resize', resizeHandler)
  └── onBeforeUnmount:
      ├── window.removeEventListener('resize', resizeHandler)
      └── chartInstance.dispose()
```

### 6.3 CanvasChartRenderer 执行路径 (ZRender)

```
CanvasChartRenderer.vue
  ├── containerRef = ref<HTMLDivElement>()
  ├── onMounted:
  │   ├── zr = zrender.init(containerRef, { renderer: 'svg' })
  │   ├── 创建 Text ("妙码学院") + Arc + Text (叠加层) + 16 个随机 Rect
  │   └── setInterval (500ms):
  │       ├── 随机触发: t2 位移 + lines 闪烁
  │       └── setTimeout(100ms): 恢复原位 + 隐藏
  └── 效果: 赛博朋克风格的闪烁文字动画
```

### 6.4 SVGChartRenderer 执行路径 (D3.js)

```
SVGChartRenderer.vue
  ├── containerRef = ref<HTMLDivElement>()
  ├── onMounted:
  │   ├── svg = d3.select(container).append('svg').append('g')
  │   ├── projection = d3.geoMercator().center([107,38]).scale(500)  // 中国地图投影
  │   ├── d3.json('china_simplify.json') → 加载 GeoJSON
  │   ├── colors = d3.scaleLinear([渐变色域])
  │   ├── svg.selectAll('path').data(features).enter().append('path')
  │   │   ├── .attr('fill', colors(i))
  │   │   ├── .on('mouseover') → tooltip 显示省份名称
  │   │   └── .on('mouseout') → tooltip 隐藏
  └── 效果: 中国地图热力图 + hover 提示
```

---

## 七、图表属性设置面板

### 7.1 ChartSetting 执行路径

```
ChartSetting.vue
  ├── props: { blockInfo: ChartBlockInfo }
  ├── useForm({ initialValues: { chartType: blockInfo.props.chartType } })
  ├── useField<ChartType>('chartType')
  ├── watch(values) → emit('change', { ...blockInfo, props: { ...newValues } })
  └── <SegmentedControl :value="chartType" :data="[
        { label: 'Echarts', value: 'echarts' },
        { label: 'Canvas', value: 'canvas' },
        { label: 'SVG', value: 'svg' }
      ]" @change="handleChartTypeChange" />
      │
      └── SegmentedControl.vue (通用分段控制器)
          ├── props: { value, data }
          ├── emit: change
          └── 渲染为按钮组，高亮当前选中项
```

### 7.2 HeroTitleSetting 执行路径

```
HeroTitleSetting.vue
  ├── useForm({ initialValues: { content, align, description } })
  ├── useField('content') → <input> 绑定
  ├── useField<HeroTitleBlockAlign>('align') → <SegmentedControl> 绑定
  ├── useFieldArray('description') → 动态描述列表
  │   └── fields (响应式数组) + push() (添加新项)
  ├── watch(values) → emit('change', 合并后的 BlockInfo)
  └── 渲染: input(标题) + SegmentedControl(对齐) + 动态 input 列表 + 添加按钮
```

### 7.3 QuoteSetting 执行路径

```
QuoteSetting.vue
  ├── useForm({ initialValues: { content, status } })
  ├── useField('content') → <input> 绑定
  ├── useField<QuoteBlockStatusType>('status') → <SegmentedControl> 绑定
  ├── watch(values) → emit('change', 合并后的 BlockInfo)
  └── 渲染: SegmentedControl(成功/警告/错误) + input(内容)
```

### 7.4 ImageSetting 执行路径

```
ImageSetting.vue
  ├── useForm({ initialValues: { url } })
  ├── useField('url') → <input> 绑定
  ├── watch(values) → emit('change', 合并后的 BlockInfo)
  └── 渲染: input(图片URL)
```

### 7.5 SchemaExporter 执行路径

```
SchemaExporter.vue
  ├── props: { currentInfo: any }  // 单个 Block 或全部 blocks
  ├── <vue-json-pretty :data="currentInfo" showIcon showLineNumber editable />
  └── handleCopyText()
      └── copyText(JSON.stringify(currentInfo)) → 剪贴板
```

### 7.6 Breadcrumb 执行路径

```
Breadcrumb.vue
  ├── props: { blockInfo: BlockInfo | null }
  ├── emit: goHome
  └── 渲染: "页面" → @click="emit('goHome')" → appEditorStore.unSelectBlock()
      └── v-if="blockInfo" → "> blockInfo.label"
```

---

## 八、富文本编辑器 (Notes)

### 8.1 NotesEditor 执行路径

```
NotesBlock.vue
  └── <NotesEditor />

NotesEditor.vue
  ├── inject('editable', true)              // 编辑模式 or 只读模式
  ├── useEditor({
  │     editable,
  │     extensions: [
  │       StarterKit.configure({ bold: { HTMLAttributes: { class: 'custom-bold' } } }),
  │       ColorHighlighter                  // 自定义扩展: 十六进制颜色高亮
  │     ],
  │     content: '<p>...</p>'               // 初始 HTML 内容
  │   })
  ├── v-if="editable" → 工具栏:
  │   ├── @click → editor.chain().focus().toggleBold().run()
  │   ├── @click → editor.chain().focus().toggleItalic().run()
  │   └── @click → editor.chain().focus().toggleStrike().run()
  └── <EditorContent :editor="editor" />    // Tiptap 渲染的 ProseMirror 编辑器

ColorHighlighter.ts (自定义 Tiptap 扩展)
  └── 识别文本中的十六进制颜色值 (#FFF, #0D0D0D 等)
      └── 自动添加带颜色预览的 <span class="color" style="--color: #xxx">
```

---

## 九、Action Flow (计算流程)

### 9.1 页面入口路径

```
ActionsView.vue (/app/actions)
  ├── ActionsLeftPanel.vue
  │   └── ActionList.vue
  │       ├── actionList = ref([{ id: '1', type: 'math', title: 'Math' }, ...])
  │       └── <RouterLink :to="/app/actions/${action.id}"> → 路由切换
  │
  └── <RouterView v-if="route.params.id" />    // 有 ID 时显示详情
      └── ActionDetailView.vue
          └── <FlowEditor />
```

### 9.2 FlowEditor 执行路径

```
FlowEditor.vue
  ├── nodes = ref(initialNodes)             // 初始节点数据
  │   └── initial-elements.ts:
  │       ├── { id: '1', type: 'value', data: { value: 10 } }
  │       ├── { id: '2', type: 'value', data: { value: 30 } }
  │       ├── { id: '3', type: 'operator', data: { operator: '+' } }
  │       └── { id: '4', type: 'result' }
  │
  ├── edges = ref(initialEdges)             // 初始连线
  │   └── initial-elements.ts:
  │       ├── { id: 'e1-3', source: '1', target: '3', targetHandle: 'target-a' }
  │       ├── { id: 'e2-3', source: '2', target: '3', targetHandle: 'target-b' }
  │       └── { id: 'e3-4', source: '3', target: '4' }
  │
  ├── useVueFlow() → { onNodeDragStop, onConnect, addEdges, setTransform, toObject }
  │   ├── onConnect(params) → addEdges(params)    // 新连线自动添加
  │   └── 控制面板: resetTransform / updatePos / logToObject
  │
  └── <VueFlow :nodes="nodes" :edges="edges">
        ├── #node-value → <ValueNode />     // 数值输入节点
        ├── #node-operator → <OperatorNode /> // 运算符选择节点
        ├── #node-result → <ResultNode />    // 结果展示节点
        ├── <Background />
        ├── <MiniMap />
        └── <Controls />
```

### 9.3 各节点执行路径

```
ValueNode.vue
  ├── props: { id, data: { value: number } }
  ├── useVueFlow() → updateNodeData
  ├── computed: value = { get: data.value, set: updateNodeData(id, { value }) }
  └── <input v-model="value" type="number" /> + <Handle type="source" />

OperatorNode.vue
  ├── props: { id, data: { operator: string } }
  ├── operators = ['+', '-', '*', '/', '%', '>']
  ├── useVueFlow() → updateNodeData
  └── v-for="operator" → <button :class="{ selected }" @click="updateNodeData(id, { operator })">
      + <Handle type="source" /> + <Handle id="target-a" type="target" /> + <Handle id="target-b" />

ResultNode.vue
  ├── useHandleConnections({ type: 'target' })         // 获取入边连接
  ├── operatorSourceConnections → 向前追溯到 Value 节点
  ├── useNodesData(源节点ID列表) → 响应式获取节点数据
  ├── mathFunctions: { '+', '-', '*', '/', '%', '>' }
  ├── computed: result → reduce → mathFunctions[operator](a, b) → Math.round(*100)/100
  └── 渲染: "valueA operator valueB = result" (结果为正绿色, 负橙色)
      + <Handle type="target" :style="{ background: result > 0 ? green : orange }" />
```

### 9.4 数据流图

```
  ValueNode(10) ──→ ┌─────────────┐
                    │ OperatorNode │ ──→ ResultNode
  ValueNode(30) ──→ │   (+)        │      (10+30=40)
                    └─────────────┘

  数据流向: ValueNode.data.value → OperatorNode.data.operator → ResultNode.computed.result
  响应式链: Vue Flow useNodesData → computed → 自动重渲染
```

---

## 十、大数据表格 (React 集成)

### 10.1 DataSource 页面路径

```
DataSourceView.vue (/app/dataSource)
  ├── DataSourceLeftPanel.vue
  │   └── DataSourceList.vue
  │       ├── dataSourceList = ref([
  │       │   { id: '1', type: 'sync', title: 'Users 100w Rows' },
  │       │   { id: '2', type: 'basic', title: 'Orders 10w Rows' },
  │       │   { id: '3', type: 'basic', title: 'Foods 10 Rows' }
  │       │ ])
  │       └── <RouterLink :to="/app/dataSource/${id}"> → 路由切换
  │
  └── <RouterView /> → DataSourceDetailView.vue
      ├── route.params.id → dsId
      └── <DataSourceContent :id="dsId" />
```

### 10.2 React 组件嵌入路径

```
DataSourceContent.vue
  ├── import { applyPureReactInVue } from 'veaury'   // 跨框架桥接
  ├── import ReactDataSource from './react_app/ReactDataSource'  // React 组件
  ├── RDataSource = applyPureReactInVue(ReactDataSource)         // Vue 封装
  └── <r-dataSource :id="id" />                      // 在 Vue 模板中使用

  执行链:
  Vue 组件 → veaury 桥接层 → React.createElement(ReactDataSource) → React 渲染
  └── React 组件内部使用 @glideapps/glide-data-grid 渲染高性能大数据表格
```

---

## 十一、应用运行器 (Runner)

### 11.1 Runner 路由路径

```
RunnerView.vue (/runner)
  └── <AppRunnerRenderer />
```

### 11.2 AppRunnerRenderer 执行路径

```
AppRunnerRenderer.vue
  ├── provide('editable', false)            // 关键: 注入 editable=false
  │   └── BlockRenderer.vue → inject('editable', true) → 收到 false
  │       └── v-if="editable" → false → 隐藏选中态 UI 和工具栏
  │
  ├── device = ref<'laptop' | 'mobile'>('laptop')
  ├── onMounted:
  │   ├── isMobileTablet() → UA 检测 (utils/detect.ts)
  │   └── window.addEventListener('resize', resize)
  │
  └── <LaptopPreviewer v-if="device === 'laptop'" />
      <MobilePreviewer v-if="device === 'mobile" />

  与编辑模式的区别:
  ├── 编辑模式: PageLayoutView → AppEditorRenderer → MobilePreviewer/LaptopPreviewer (editable=true)
  └── 运行模式: RunnerView → AppRunnerRenderer → MobilePreviewer/LaptopPreviewer (editable=false)
      ├── 无左侧面板、无右侧面板
      ├── 无 Block 选中/拖拽/删除能力
      └── 自动适配设备类型
```

---

## 十二、SmoothDnd 拖拽底层实现

### 12.1 Vue 封装层

```
SmoothDndContainer.ts (defineComponent)
  ├── props: orientation, behaviour, groupName, dragHandleSelector, getChildPayload, ...
  ├── mounted():
  │   ├── 遍历事件映射 → 将 smooth-dnd 事件转为 Vue $emit
  │   └── smoothDnD(containerElement, options) → 初始化拖拽容器
  ├── unmounted():
  │   └── this.container.dispose() → 清理
  └── render(): h(tag, { ref: 'container' }, slots.default())

SmoothDndDraggable.ts (defineComponent)
  ├── props: tag (默认 'div')
  └── render(): h(tagProps, wrapperClass, slots.default())

utils.ts
  ├── getTagProps(component, wrapperClass?) → 解析 tag prop + 合并 class
  └── validateTagProp(value) → 验证 tag 类型 (string | component | object)
```

### 12.2 拖拽组通信

```
两个 smooth-dnd-container 通过 group-name="blocks" 连通:
  │
  ├── BlocksDrawer.vue (源容器)
  │   └── behaviour="copy" → 拖出时保留原件 (复制模式)
  │       └── getChildPayload(index) → getBlocksDefaultData(type)
  │
  └── BlocksRenderer.vue (目标容器)
      └── behaviour 默认 → 接收拖入项
          └── @drop → applyDrag() → updateBlocks()

  跨容器拖拽: smooth-dnd 内部通过 group-name 匹配，自动处理拖拽预览和放置逻辑
```

---

## 十三、状态管理 (Pinia Store)

### 13.1 useAppEditorStore 执行路径

```
stores/appEditor.ts
  ├── defineStore('appEditor', () => {
  │   ├── currentBlockId = ref<string | null>(null)    // 当前选中 Block
  │   ├── blocks = ref(blocksData)                     // 所有 Block 数据 (初始来自 mocks/blocks.ts)
  │   │
  │   ├── selectBlock(id)      → currentBlockId.value = id
  │   ├── unSelectBlock()      → currentBlockId.value = null
  │   ├── updateBlocks(new)    → blocks.value = new    // 批量更新 (拖拽排序/添加)
  │   └── updateBlock(id, new) → Object.assign(block, new)  // 单个更新 (属性编辑)
  │       └── 遍历 blocks 找到匹配 id → 原地替换 (保持响应式引用)
  │
  └── 返回所有状态和方法供组件使用
```

### 13.2 状态消费方

```
消费方                    使用的状态/方法
───────────────────────────────────────────────────
BlockRenderer.vue         currentBlockId, blocks, selectBlock()
BlocksRenderer.vue        blocks, updateBlocks()
AppRightPanel.vue         currentBlockId, blocks, unSelectBlock(), updateBlock()
Components.vue            blocks (只读, 展示大纲)
AppNavigator.vue          (无直接 store 依赖, 通过路由)
```

---

## 十四、全局 CSS 变量与主题

```
assets/variable.css       → CSS 自定义属性定义 (颜色/字号/字重)
assets/reset.css          → 浏览器样式重置
assets/base.css           → 基础样式
assets/main.css           → 主样式入口 (聚合以上)

关键 CSS 变量:
  --color-primary         // 主题色
  --color-gray-100~900    // 灰度色阶
  --color-white/black     // 黑白
  --color-text            // 文本色
  --panel-width           // 面板宽度
  --font-size-small/normal/large
  --font-weight-bold/bolder
```

---

## 十五、开发工具链执行路径

### 15.1 构建链

```
pnpm dev → turbo run dev
  ├── builder: vite dev → Vue + React (veaury) + HMR
  └── playwright: playwright test (独立)

pnpm build → turbo run build
  └── builder: vite build → rollup → dist/

turbo.json
  └── pipeline: { build: { dependsOn: ["^build"] }, dev: { cache: false } }
```

### 15.2 代码质量链

```
git commit → husky pre-commit hook
  └── lint-staged
      ├── *.{md,json}    → prettier --write
      ├── *.{css,less}   → stylelint --fix + prettier --write
      ├── *.{js,jsx}     → eslint --fix + prettier --write
      └── *.{ts,tsx}     → eslint --fix + prettier --parser=typescript --write

git commit-msg → commitlint
  └── commitlint.config.js → 校验 Conventional Commits 格式

pnpm commit → git-cz (cz-git)
  └── 交互式提交 → emoji + scope + subject 生成规范化 commit message
```
