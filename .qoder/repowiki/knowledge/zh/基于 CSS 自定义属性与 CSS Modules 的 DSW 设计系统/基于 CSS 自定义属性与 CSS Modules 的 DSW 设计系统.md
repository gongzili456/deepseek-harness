---
kind: frontend_style
name: 基于 CSS 自定义属性与 CSS Modules 的 DSW 设计系统
category: frontend_style
scope:
    - '**'
source_files:
    - docs/web-styling.md
    - packages/client/ui-theme/README.md
    - packages/client/ui-theme/src/styles/base.css
    - packages/client/ui-theme/src/styles/design-platform.css
    - packages/client/ui-theme/src/styles/scrollbar.css
    - packages/client/ui-theme/src/styles/gradient-shadow-text.css
    - packages/client/ui-theme/src/styles/shiki.css
    - apps/web/vite.config.ts
    - apps/web/package.json
    - packages/client/tsdown.client.ts
---

## 1. 体系概览

DeepSeek Harness 的前端样式采用「设计令牌（Design Tokens）+ CSS Modules + 主题快照」的分层架构，不使用 Tailwind、styled-components、Emotion、MUI、Chakra 等第三方 UI 框架或原子化 CSS 方案。所有视觉表现通过 `--dsw-*` CSS 自定义属性在运行时注入，组件仅消费语义化的 `--dsw-alias-*` 别名。

构建工具链：
- Web 应用入口位于 `apps/web`，使用 Vite + React 构建，通过 `vite.config.ts` 中的 alias 将 `@deepseek-ai/dsh-client-web` 等包直接指向源码目录，使 `.module.css` 经 Vite 的 lightningcss 管线编译后内联进 bundle。
- 各 client 子包通过 `tsdown.client.ts` 打包时识别 `*.module.css` 并内联到产物中；测试断言也依赖 CSS Modules 生成的 `[hash]_[local]` 类名约定（见 e2e 注释）。
- 文档站点 `website/` 单独使用 VitePress，与产品前端样式无关。

## 2. 核心文件与职责

| 文件 / 包 | 职责 |
|---|---|
| `packages/client/ui-theme/src/styles/base.css` | 定义字体族、缓动曲线、过渡时长等基础变量，被 token 表引用 |
| `packages/client/ui-theme/src/styles/design-platform.css` | 声明全部 `--dsw-static-*` 静态色板与 `--dsw-alias-*` 语义别名，提供 light / dark 两套覆盖（`body[data-ds-dark-theme]`） |
| `packages/client/ui-theme/src/styles/scrollbar.css` | 唯一消费 `--dsw-alias-scrollbar-*` 的滚动条样式，绑定 `--dsh-scrollbar-thumb*` 实现可重绑定的滚动条主题 |
| `packages/client/ui-theme/src/styles/gradient-shadow-text.css` | 渐变、阴影、文本效果 |
| `packages/client/ui-theme/src/styles/shiki.css` | 语法高亮主题适配 |
| `packages/client/ui-layout` | 主题呈现者：把 `ThemeSnapshot` 应用到 DOM（设置 `html { color-scheme }`、`body[data-ds-dark-theme]`、内联别名 token），不持有状态 |
| `packages/client/ui-theme` | 主题服务：管理 `light/dark/system` 偏好，通过 Host settings API 持久化，发布不可变 `ThemeSnapshot` |
| `docs/web-styling.md` | 官方风格规范：规定所有权、组件规则、变更流程 |

## 3. 架构与约定

### 令牌分层
- **静态层**：`--dsw-static-*`（如 `--dsw-static-neutral-bluish-950`、`--dsw-static-deepseek-500`）是 Figma 导出的固定色值，按明/暗模式分别声明。
- **语义别名层**：`--dsw-alias-*`（如 `--dsw-alias-bg-base`、`--dsw-alias-label-primary`、`--dsw-alias-state-error-primary`）映射到静态层，feature 组件只消费这一层。
- **运行时注入层**：`ui-layout` 的 presenter 把 `ThemeSnapshot` 以 inline style 形式写入 `document.body.style`，从而切换主题。

### 主题切换机制
- `ui-theme` 维护当前偏好（`light` / `dark` / `system`），`system` 通过 `prefers-color-scheme` 解析。
- 当宿主包含 HTTP server 时，会在 `<body>` 后立即注入已保存的设置，确保 shell 加载页渲染前 `color-scheme` 和 `data-ds-dark-theme` 已就绪。
- 远程浏览器无法访问特权 settings API，偏好保持进程本地。

### 组件样式约定（来自 `docs/web-styling.md`）
- 必须使用 CSS Modules + `clsx`，禁止引入 Tailwind 或任何组件库。
- feature 组件只能使用 `--dsw-alias-*` 语义 token，不得复制静态色值或直接写字面量颜色。
- 组件内不得出现主题选择器（如 `body[data-ds-dark-theme]`），明/暗覆盖由主题包负责。
- 字体大小需搭配行高，优先复用主题排版变量。
- 源码文本、终端输出、diff 行必须保留换行（column preservation），统一使用共享滚动条样式而非组件内自定义滚动条选择器。
- 演示逻辑放在 CSS 中；React inline style 只能传递组件级 custom property，不能编码主题分支。
- 新增过渡或 hover-only 控制时必须保留键盘焦点可见性与 reduced-motion 行为。

### 滚动条重绑定契约
`scrollbar.css` 在 `body` 上绑定 `--dsh-scrollbar-thumb` / `--dsh-scrollbar-thumb-hover`，默认指向 l1 表面 token；菜单、popover、dialog 等提升层级容器可重新绑定为 l2 或 `transparent`。标准属性 `scrollbar-width` / `scrollbar-color` 与 `::-webkit-scrollbar*` 伪元素路径互斥，Firefox 走标准路径，WebKit 引擎走伪元素路径。

## 4. 约束与强制检查

- `apps/web/vite.config.ts` 内置 `rejectStandaloneServe` 插件：直接在仓库 checkout 下运行 `vite serve` 会抛出错误，强制通过 `pnpm dsh web` 启动，避免裸 Vite 暴露无 boot manifest 的 shell。
- 构建产物按约定组织：`assets/[name]-[hash].js`、`assets/langs/`（shiki 语法）、`assets/fonts/`（KaTeX 字体），vendor chunk 仅包含明确列出的 React-free 包（katex、shiki、micromark 系列）。
- 每个 client 包自带 `css-modules.d.ts` 声明 `*.module.css` 模块类型，保证 TypeScript 对 CSS Modules 的类型支持。
- 视觉回归遵循仓库 testing policy，token 变更需同步更新 owning package reference。

## 5. 关键依赖

- 构建：Vite + `@vitejs/plugin-react` + lightningcss（CSS Modules 编译）
- 运行时：React 18
- 样式：原生 CSS 自定义属性 + CSS Modules + clsx 类名拼接
- 无 Tailwind、无 CSS-in-JS、无 UI 组件库