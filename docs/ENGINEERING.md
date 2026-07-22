# Miaoma VBuilder 工程化流程深度解析

## 一、Monorepo 架构总览

```
miaoma-vbuilder/                    # 根 workspace
├── apps/
│   ├── builder/                    # 主应用 (Vue 3 SPA)
│   └── playwright/                 # E2E 测试
├── packages/
│   ├── blocks/                     # 共享 composable (@miaoma/blocks)
│   └── eslint/                     # 共享 ESLint 配置 (eslint-config-miaoma)
├── pnpm-workspace.yaml            # workspace 声明
├── turbo.json                      # 任务编排
└── package.json                    # 根 scripts + 全局 devDeps
```

### pnpm-workspace.yaml

```yaml
packages:
  - 'apps/*'       # 所有应用包
  - 'packages/*'   # 所有共享库包
```

pnpm 通过这个文件识别 monorepo 结构。`apps/builder` 和 `packages/eslint` 等目录各自有独立的 `package.json`，但共享同一个 `node_modules`（通过 pnpm 的 content-addressable store 实现去重）。

**workspace 协议引用：** 根 `package.json` 中 `"eslint-config-miaoma": "workspace:*"` 表示引用本地包，pnpm 会自动建立软链接而不是从 npm 下载。

---

## 二、Turborepo 任务编排

### turbo.json 逐行解读

```json
{
    "tasks": {
        "build": {
            "dependsOn": ["^build"],    // ① 先构建所有上游依赖包
            "outputs": ["dist/**"]       // ② 缓存 dist 目录
        },
        "test": {
            "outputs": ["coverage/**"],  // ③ 缓存 coverage 目录
            "dependsOn": ["build"]       // ④ 等当前包 build 完再测
        },
        "dev": {
            "cache": false,              // ⑤ dev 不缓存
            "persistent": true           // ⑥ dev 是长驻进程，不会退出
        },
        "preview": {
            "cache": false,
            "persistent": true
        }
    },
    "ui": "tui"                          // ⑦ 使用终端 UI 展示任务状态
}
```

**关键点：**

| 字段 | 含义 |
|------|------|
| `"^build"` | `^` 前缀表示**依赖包的 build 先执行**。即 `packages/eslint` 和 `packages/blocks` 的 build 先跑完，`apps/builder` 的 build 才开始 |
| `"build"` (无 ^) | 表示**当前包自己的 build**。`test` 依赖当前包的 `build` 完成 |
| `"cache": false` | dev 模式每次都要重新启动，不走缓存 |
| `"persistent": true` | 标记为常驻进程，Turborepo 不会等待它退出 |

### 运行 `pnpm dev` 时的执行过程

```
turbo run dev
    │
    ├── 扫描所有 workspace 包的 package.json 中的 "dev" script
    │
    ├── 检查依赖关系图：
    │   ├── packages/eslint  → 无 "dev" script → 跳过
    │   ├── packages/blocks  → 无 "dev" script → 跳过
    │   └── apps/builder     → "dev": "vite"
    │
    └── 执行 apps/builder 的 dev
        └── vite 启动开发服务器 (localhost:3000)
```

### 运行 `pnpm build` 时的执行过程

```
turbo run build
    │
    ├── ① 拓扑排序依赖关系：
    │   packages/eslint  → packages/blocks → apps/builder
    │   (并行)              (并行)            (等待上游)
    │
    ├── ② 执行 packages/eslint 的 "build"
    │   └── 无 build script → Turborepo 跳过（不算失败）
    │
    ├── ③ 执行 packages/blocks 的 "build"
    │   └── 无 build script → 跳过
    │
    ├── ④ 上游全部完成，执行 apps/builder 的 "build"
    │   └── vite build → 输出 dist/
    │       ├── 检查缓存：若 dist/ 未变化 → 命中缓存，跳过
    │       └── 未命中 → 执行构建，缓存 dist/** 到 .turbo/
    │
    └── ⑤ 结果写入 .turbo/ 缓存目录
        下次运行若源码未变化 → 直接命中缓存，0 秒完成
```

---

## 三、包依赖分类解读

### 根 package.json — 全局工程工具

这些依赖**不参与业务代码打包**，只在开发流程中使用：

```
┌─────────────────────────────────────────────────────────┐
│                    全局工程依赖                           │
├──────────────┬──────────────────────────────────────────┤
│ 代码质量      │ eslint 9.7                              │
│              │ stylelint 16.7                           │
│              │ prettier 3.3                             │
│              │ cspell 8.10 (拼写检查)                    │
├──────────────┼──────────────────────────────────────────┤
│ Git 规范      │ husky 9.0 (git hooks 管理)              │
│              │ lint-staged 15.2 (暂存区文件检查)         │
│              │ @commitlint/cli 19.3 (commit msg 校验)   │
│              │ @commitlint/config-conventional 19.2     │
│              │ commitizen 4.3 + cz-git 1.9 (交互式提交) │
├──────────────┼──────────────────────────────────────────┤
│ 构建工具      │ turbo 2.0 (monorepo 任务编排)            │
│              │ rimraf 6.0 (跨平台 rm -rf)               │
└──────────────┴──────────────────────────────────────────┘
```

### apps/builder — 运行时依赖（打入产物包）

```
┌───────────────────────────────────────────────────────────────┐
│                    运行时依赖 (dependencies)                    │
├───────────────┬───────────────────────────────────────────────┤
│ 框架核心       │ vue 3.4.31                                    │
│               │ vue-router 4.4.0                              │
│               │ pinia 2.1.7                                   │
├───────────────┼───────────────────────────────────────────────┤
│ 拖拽系统       │ smooth-dnd 0.12.1                             │
├───────────────┼───────────────────────────────────────────────┤
│ 富文本编辑     │ @tiptap/core 2.5.1                            │
│               │ @tiptap/starter-kit 2.5.1                     │
│               │ @tiptap/vue-3 2.5.1                           │
│               │ @tiptap/pm 2.5.1 (ProseMirror 绑定)           │
├───────────────┼───────────────────────────────────────────────┤
│ 图表渲染       │ echarts 5.5.1 (标准图表)                      │
│               │ vue-echarts 6.7.3 (Vue 封装)                  │
│               │ zrender 5.6.0 (Canvas 2D 引擎，echarts 底层)   │
│               │ d3 7.9.0 (SVG 数据可视化)                     │
├───────────────┼───────────────────────────────────────────────┤
│ 表单验证       │ vee-validate 4.13.2                           │
├───────────────┼───────────────────────────────────────────────┤
│ 流程编辑       │ @vue-flow/core 1.39.0                         │
│               │ @vue-flow/additional-components 1.3.3         │
├───────────────┼───────────────────────────────────────────────┤
│ React 桥接     │ veaury 2.4.2 (Vue↔React 桥接)                │
│               │ react 18.3.1                                  │
│               │ react-dom 18.3.1                              │
│               │ @glideapps/glide-data-grid 6.0.3 (React 表格) │
├───────────────┼───────────────────────────────────────────────┤
│ 工具库         │ nanoid 5.0.7 (ID 生成)                       │
│               │ @icon-park/vue-next 1.4.2 (图标库)            │
│               │ copy-text-to-clipboard 3.2.0                  │
│               │ vue-json-pretty 2.4.0 (JSON 格式化展示)       │
└───────────────┴───────────────────────────────────────────────┘
```

### apps/builder — 开发依赖（不打入产物包）

```
┌───────────────────────────────────────────────────────────────┐
│                    开发依赖 (devDependencies)                   │
├───────────────┬───────────────────────────────────────────────┤
│ 构建工具       │ vite 5.3.3                                    │
│               │ @vitejs/plugin-vue 5.0.5                      │
│               │ @vitejs/plugin-vue-jsx 4.0.0                  │
│               │ @vitejs/plugin-react 4.3.1 (veaury 需要)      │
│               │ vite-plugin-vue-devtools 7.3.6                │
│               │ rollup-plugin-visualizer 5.12.0 (包分析)       │
├───────────────┼───────────────────────────────────────────────┤
│ TypeScript    │ typescript 5.5.3                              │
│               │ vue-tsc 2.0.26 (.vue 文件类型检查)            │
│               │ @vue/tsconfig 0.5.1                           │
│               │ @tsconfig/node20 20.1.4                       │
├───────────────┼───────────────────────────────────────────────┤
│ 测试           │ vitest 2.0.3 (单元测试)                       │
│               │ @vue/test-utils 2.4.6                         │
│               │ @playwright/test 1.45.1 (E2E)                 │
│               │ jsdom 24.1.0                                  │
├───────────────┼───────────────────────────────────────────────┤
│ 监控           │ @sentry/vue 8.26.0                            │
│               │ @sentry/vite-plugin 2.22.2                    │
└───────────────┴───────────────────────────────────────────────┘
```

### packages/eslint — 共享 ESLint 配置包

```
eslint-config-miaoma
├── @eslint/js 9.7.0               # ESLint 推荐规则
├── eslint-plugin-vue 9.27.0       # Vue 专用规则
├── vue-eslint-parser 9.4.3        # 解析 .vue 文件
├── @typescript-eslint/parser 7.16  # 解析 TypeScript
├── eslint-plugin-simple-import-sort 12.1  # import 排序
└── globals 15.8.0                  # 全局变量定义
```

根 `eslint.config.js` 只有一行：`export default [...eslintConfigMiaoma]`，所有规则都在这个包里集中定义。

### packages/blocks — 共享 composable 包

```
@miaoma/blocks
└── vue-demi 0.14.7    # Vue 2/3 兼容层
```

`vue-demi` 让同一个 composable 同时兼容 Vue 2 和 Vue 3，适合发布为公共 npm 包。目前只导出了 `useToggle`。

---

## 四、Vite 配置深度解读

### vite.config.ts

```ts
export default defineConfig({
    plugins: [
        vueDevTools(),                           // ① Vue DevTools 浏览器插件
        veauryVitePlugins({ type: 'vue' }),      // ② Vue+React 双框架编译
        visualizer({ open: true }),              // ③ 打包后自动打开体积分析图
        sentryVitePlugin({ ... })                // ④ 上传 sourcemap 到 Sentry
    ],
    resolve: {
        alias: { '@': './src' }                   // ⑤ 路径别名
    },
    server: {
        port: 3000,                               // ⑥ 开发服务器端口
        proxy: {                                  // ⑦ API 代理
            '/charts': {
                target: 'https://echarts.apache.org',
                rewrite: (path) => path.replace(/^\/charts/, '')
            }
        }
    },
    build: {
        sourcemap: true,                          // ⑧ 生产构建生成 sourcemap
        rollupOptions: {
            output: {
                manualChunks(id) {                 // ⑨ 代码分割策略
                    if (/(echarts|d3|zrender)/.test(id)) {
                        return 'charts'            // 图表库单独拆 chunk
                    }
                }
            }
        }
    }
})
```

**逐项解读：**

**① vueDevTools()** — 在浏览器中嵌入 Vue DevTools 面板，支持组件树检查、Pinia 状态查看、性能分析。

**② veauryVitePlugins({ type: 'vue' })** — 这是关键配置。`veaury` 同时注册了 `@vitejs/plugin-vue` 和 `@vitejs/plugin-react`（注意原始代码中 `vue()` 和 `vueJsx()` 被注释掉了，因为 veaury 内部已处理）。这使得 `.vue` 和 `.jsx/.tsx` 文件可以在同一个项目中混用。

**③ visualizer({ open: true })** — 构建完成后自动打开 `stats.html`，可视化展示各模块的打包体积，用于发现过大的依赖。

**④ sentryVitePlugin** — 构建时自动将 sourcemap 上传到 Sentry，这样线上报错时能还原到源码行号。authToken 硬编码在配置中（生产环境应走环境变量）。

**⑦ proxy 配置** — ECharts 的示例数据托管在 `echarts.apache.org`，开发环境通过 Vite 代理避免跨域问题。`/charts/xxx` → `https://echarts.apache.org/xxx`。

**⑨ manualChunks** — 将 echarts、d3、zrender 三个大库拆到独立的 `charts` chunk。这三个库总体积 ~1MB+，拆分后可以利用浏览器并行加载，且这些库变化频率低，长期缓存命中率高。

代码中的注释也提到了一个问题：`manualChunks` 函数会把动态 import 的 chunk 也打进来，导致动态 chunk 失效。推荐了解 `vite-plugin-split-chunk` 插件做更精细的控制。

---

## 五、TypeScript 配置解读

### 配置层级关系

```
tsconfig.json (根)                    # 基础配置，所有包共享
    │
    ├── apps/builder/tsconfig.json    # Project References 入口
    │   ├── tsconfig.app.json         # 应用代码配置
    │   ├── tsconfig.node.json        # Node 端配置 (vite.config, playwright.config)
    │   └── tsconfig.vitest.json      # 测试配置
    │
    └── tsconfig.eslint.json          # ESLint 专用（被注释掉了）
```

### 根 tsconfig.json

```json
{
    "compilerOptions": {
        "target": "ES2015",           // 编译目标：ES6 语法
        "module": "ESNext",           // 模块系统：ESM
        "moduleResolution": "node",   // 模块解析：Node 风格
        "strict": true,               // 严格模式全开
        "lib": ["DOM", "ESNext"],     // 类型库：浏览器 DOM + 最新 ES
        "jsx": "preserve",            // JSX 不转换，交给 Vite 处理
        "esModuleInterop": true,      // 允许 CJS/ESM 互操作
        "skipLibCheck": true,         // 跳过 .d.ts 类型检查（加速）
        "resolveJsonModule": true,    // 允许 import JSON 文件
        "experimentalDecorators": true // 装饰器语法支持
    }
}
```

### tsconfig.app.json（应用代码）

```json
{
    "extends": "@vue/tsconfig/tsconfig.dom.json",  // Vue 官方 DOM 类型预设
    "include": ["env.d.ts", "src/**/*", "src/**/*.vue"],
    "exclude": ["src/**/__tests__/*"],
    "compilerOptions": {
        "composite": true,           // 启用 Project References
        "baseUrl": ".",
        "paths": {
            "@/*": ["./src/*"]        // @ 别名映射
        }
    }
}
```

`composite: true` 是 Project References 的要求，让 `vue-tsc` 可以增量编译。`@vue/tsconfig/tsconfig.dom.json` 提供了 Vue 3 的全局类型定义（`ref`、`computed` 等的类型）。

### tsconfig.node.json（Node 端配置）

```json
{
    "extends": "@tsconfig/node20/tsconfig.json",
    "include": ["vite.config.*", "vitest.config.*", "playwright.config.*"],
    "compilerOptions": {
        "module": "ESNext",
        "moduleResolution": "Bundler",  // Bundler 模式：允许无后缀 import
        "types": ["node"]
    }
}
```

专门用于 `vite.config.ts`、`vitest.config.ts` 等在 Node 环境运行的配置文件。`moduleResolution: "Bundler"` 是 Vite 推荐的解析模式。

### tsconfig.vitest.json（测试配置）

```json
{
    "extends": "./tsconfig.app.json",
    "exclude": [],                    // 不排除 __tests__ 目录
    "compilerOptions": {
        "types": ["node", "jsdom"]    // 添加 jsdom 类型 (window, document 等)
    }
}
```

继承 `tsconfig.app.json`，但把 `__tests__` 目录从 exclude 中移除，并添加 `jsdom` 类型支持。

---

## 六、ESLint 配置解读

### 扁平配置架构 (ESLint 9 Flat Config)

```
根 eslint.config.js
    └── import eslintConfigMiaoma from 'eslint-config-miaoma'
        └── packages/eslint/eslint.config.js
```

### packages/eslint/eslint.config.js 核心规则

```js
export default [
    // ① 全局变量声明
    {
        languageOptions: {
            globals: {
                ...globals.browser,
                // Vue 3 Composition API 全局宏（不需要 import）
                computed: 'readonly',
                ref: 'readonly',
                reactive: 'readonly',
                watch: 'readonly',
                onMounted: 'readonly',
                // ... 等
            }
        }
    },
    // ② 文件匹配 + 规则
    {
        files: ['**/*.{ts,tsx,vue}'],
        rules: {
            ...js.configs.recommended.rules,       // ESLint 推荐规则
            ...pluginVue.configs['flat/recommended'].rules,  // Vue 推荐规则
            'no-unused-vars': 'error',              // 未使用变量报错
            'no-console': 'error',                  // 禁止 console
            'simple-import-sort/imports': 'error',   // import 强制排序
            'simple-import-sort/exports': 'error',   // export 强制排序
            'vue/valid-define-emits': 'error'        // defineEmits 校验
        },
        languageOptions: {
            parser: vueEslintParser,                 // 用 vue-eslint-parser 解析 .vue
            parserOptions: {
                parser: tsParser,                    // <script> 内用 TS 解析器
                sourceType: 'module'
            }
        }
    }
]
```

**关键设计：**
- Vue 3 Composition API 的全局宏（`ref`、`computed` 等）声明为 `readonly` globals，这样不需要每个文件 import 就不会报 `no-undef`
- 解析器是**双层嵌套**：外层 `vue-eslint-parser` 解析 `.vue` 文件结构，内层 `@typescript-eslint/parser` 解析 `<script>` 中的 TypeScript

---

## 七、Git Hooks 完整流程

### 提交时的完整执行链

```
git commit -m "feat: add new block"
    │
    ├── ① .husky/commit-msg 触发
    │   └── commitlint --edit $1
    │       └── 校验 commit message 是否符合 Conventional Commits 格式
    │           ├── ✅ 通过 → 继续
    │           └── ❌ 失败 → 拒绝提交
    │
    ├── ② .husky/pre-commit 触发
    │   └── pnpm typecheck && pnpm spellcheck && pnpm lint-staged
    │       │
    │       ├── ②a pnpm typecheck
    │       │   └── pnpm --filter builder typecheck
    │       │       └── vue-tsc --noEmit
    │       │           └── 对整个 builder 项目做 TypeScript 类型检查
    │       │               ├── ✅ 通过
    │       │               └── ❌ 失败 → 拒绝提交
    │       │
    │       ├── ②b pnpm spellcheck
    │       │   └── cspell lint --dot --gitignore --cache
    │       │       └── 检查所有代码文件中的英文拼写错误
    │       │           ├── ✅ 通过
    │       │           └── ❌ 失败 → 拒绝提交
    │       │
    │       └── ②c pnpm lint-staged
    │           └── 只对 git add 暂存区的文件执行：
    │               ├── *.md, *.json  → prettier --write
    │               ├── *.css, *.less → stylelint --fix + prettier
    │               ├── *.js, *.jsx   → eslint --fix + prettier
    │               └── *.ts, *.tsx   → eslint --fix + prettier
    │
    └── ③ 全部通过 → commit 成功
```

### lint-staged 为什么重要？

不是每次保存都跑全量 lint（项目大了会很慢），而是**只检查本次要提交的文件**，且自动 `--fix` 修复能修的问题。

---

## 八、脚本运行过程详解

### `pnpm dev` — 开发模式

```
pnpm dev
  → turbo run dev
    → apps/builder: vite
      │
      ├── 1. 读取 vite.config.ts
      ├── 2. 注册插件链：
      │   ├── vueDevTools() → 注入 HMR 增强
      │   ├── veauryVitePlugins() → 注册 vue + react 编译器
      │   ├── visualizer() → 构建后打开分析图
      │   └── sentryVitePlugin() → 上传 sourcemap
      ├── 3. 启动 dev server (localhost:3000)
      ├── 4. 监听文件变化 → HMR 热更新
      └── 5. 代理 /charts → echarts.apache.org
```

### `pnpm build` — 生产构建

```
pnpm build
  → turbo run build
    → 检查缓存 → 未命中
    → apps/builder: vite build
      │
      ├── 1. TypeScript 编译 (vue-tsc)
      ├── 2. Vite 打包：
      │   ├── 解析 import 依赖图
      │   ├── vue compiler 编译 .vue 文件 → render 函数
      │   ├── esbuild 编译 TS/JS
      │   ├── Rollup tree-shaking 移除死代码
      │   ├── manualChunks 分割：
      │   │   ├── vendor (vue, pinia, vue-router 等)
      │   │   ├── charts (echarts, d3, zrender)
      │   │   └── 其他按需分割
      │   └── terser 压缩
      ├── 3. 输出 dist/
      │   ├── index.html
      │   ├── assets/
      │   │   ├── index-[hash].js      (主包)
      │   │   ├── charts-[hash].js     (图表库)
      │   │   └── index-[hash].css
      │   └── favicon.ico
      ├── 4. Sentry 插件上传 sourcemap
      └── 5. Visualizer 生成 stats.html
```

### `pnpm lint` — 代码检查

```
pnpm lint
  → pnpm lint:ts && pnpm lint:style
    │
    ├── lint:ts: eslint --fix
    │   └── 读取根 eslint.config.js
    │       └── 加载 eslint-config-miaoma 规则
    │           └── 扫描 **/*.{ts,tsx,vue} 文件
    │               └── 自动修复 + 报错
    │
    └── lint:style: stylelint "{packages,apps}/**/*.{css,scss,vue}"
        └── 读取 stylelint.config.js
            ├── *.css  → stylelint-config-standard
            ├── *.scss → stylelint-config-standard-scss
            └── *.vue  → stylelint-config-standard-scss + stylelint-config-standard-vue/scss
```

### `pnpm commit` — 交互式提交

```
pnpm commit
  → git-cz (commitizen + cz-git)
    │
    ├── ? Select the type of change:
    │   feat ✨ / fix 🐛 / docs 📝 / style 💄 / refactor 📦️ ...
    ├── ? Scope (optional): component or file name
    ├── ? Short description: imperative tense
    ├── ? Breaking changes? (optional)
    ├── ? Issues affected? (optional)
    │
    └── 生成 commit message:
        "feat(blocks): ✨ add video block type"
        │
        └── 触发 commitlint 校验 → 通过 → commit
```

### `pnpm typecheck` — 类型检查

```
pnpm typecheck
  → pnpm --filter builder typecheck
    → vue-tsc --noEmit
      │
      ├── 读取 tsconfig.json → Project References
      ├── 加载 tsconfig.app.json
      │   ├── extends: @vue/tsconfig/tsconfig.dom.json
      │   ├── paths: @/* → ./src/*
      │   └── include: src/**/*, src/**/*.vue
      ├── 加载 tsconfig.node.json (配置文件)
      ├── 加载 tsconfig.vitest.json (测试文件)
      │
      └── 逐一类型检查所有文件
          ├── .vue 文件：vue-tsc 解析 <script lang="ts">
          ├── .ts 文件：tsc 标准检查
          └── --noEmit：不生成 .js 文件，只做检查
```

### `pnpm clean` — 清理

```
pnpm clean
  → rimraf "{apps,packages}/**/{node_modules,docs,lib,dist,stats.html}"
              node_modules pnpm-lock.yaml .eslintcache .cspellcache docs dist
    │
    └── 删除：
        ├── 所有子包的 node_modules, dist, docs, lib, stats.html
        ├── 根 node_modules
        ├── pnpm-lock.yaml
        └── 缓存文件 (.eslintcache, .cspellcache)
```

---

## 九、包体积分析 & 优化策略

### 构建产物 chunk 分割

```
dist/assets/
├── index-[hash].js        # 主包 (Vue + Pinia + Router + 业务代码)
├── charts-[hash].js       # 图表库 (echarts + d3 + zrender) ~1MB+
└── index-[hash].css       # 样式
```

### 已做的优化

| 优化手段 | 具体做法 |
|----------|----------|
| **代码分割** | `manualChunks` 将 echarts/d3/zrender 拆为独立 chunk |
| **动态 import** | 路由组件全部使用 `() => import(...)` 懒加载 |
| **Tree Shaking** | ECharts 使用按需引入（`echarts/core` + 手动注册组件） |
| **Sourcemap** | 生产构建开启 sourcemap，配合 Sentry 线上报错定位 |
| **缓存** | Turborepo 缓存构建产物，未修改的包不重新构建 |

### 可进一步优化的方向

| 方向 | 说明 |
|------|------|
| **React 按需加载** | `react` + `react-dom` + `glide-data-grid` (~200KB) 应该做动态 import，只在进入数据源页面时加载 |
| **veaury 替代** | 考虑用 Web Components 或 iframe 嵌入 React 组件，避免把 React 打入主包 |
| **D3 按需引入** | D3 全量 ~500KB，只用到了 `d3-geo` 和 `d3-scale`，应按模块引入 |
| **vite-plugin-split-chunk** | 替代 `manualChunks`，更精细地控制 chunk 分割策略 |

---

## 十、工程化全景图

```
开发者写代码
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                   代码编写阶段                            │
│                                                         │
│  ESLint (实时检查)  ←──  eslint-config-miaoma            │
│  TypeScript (类型)  ←──  vue-tsc + tsconfig.app.json    │
│  拼写检查           ←──  cspell                          │
└────────────────────────┬────────────────────────────────┘
                         │ 保存文件
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   提交阶段 (git commit)                   │
│                                                         │
│  .husky/pre-commit                                      │
│  ├── pnpm typecheck     → vue-tsc --noEmit              │
│  ├── pnpm spellcheck    → cspell lint                   │
│  └── pnpm lint-staged   → eslint --fix + prettier       │
│                                                         │
│  .husky/commit-msg                                      │
│  └── commitlint         → Conventional Commits 校验      │
└────────────────────────┬────────────────────────────────┘
                         │ push
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   构建阶段 (pnpm build)                   │
│                                                         │
│  Turborepo 调度                                          │
│  ├── packages/*  (先构建)                                │
│  └── apps/builder                                       │
│      └── vite build                                     │
│          ├── vue compiler → render 函数                  │
│          ├── esbuild → TS/JS 编译                        │
│          ├── Rollup → tree-shaking + chunk 分割          │
│          ├── manualChunks → charts 独立 chunk            │
│          ├── Sentry plugin → 上传 sourcemap              │
│          └── 输出 dist/                                  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   运行时                                 │
│                                                         │
│  main.ts 启动流程：                                      │
│  1. createApp(App)                                      │
│  2. Sentry.init() → 错误监控 + Session Replay            │
│  3. app.use(pinia) → 状态管理                            │
│  4. app.use(router) → 路由                               │
│  5. app.use(initBlocks()) → Block 插件注册               │
│  6. app.mount('#app')                                   │
└─────────────────────────────────────────────────────────┘
```

这套工程化体系覆盖了**编码 → 提交 → 构建 → 运行**的完整链路，每个环节都有质量卡点，可以作为 monorepo 项目的标准工程化模板。
