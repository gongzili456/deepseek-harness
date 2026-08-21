---
kind: frontend_style
name: 基于 CSS Modules + --dsw-* 设计令牌的前端样式体系
category: frontend_style
scope:
    - '**'
source_files:
    - docs/web-styling.md
    - packages/client/ui-theme/README.md
    - packages/client/ui-theme/src/styles/base.css
    - packages/client/ui-theme/src/styles/scrollbar.css
    - packages/client/ui-theme/src/styles/design-platform.css
    - packages/client/ui-theme/src/styles/gradient-shadow-text.css
    - packages/client/ui-theme/src/styles/shiki.css
    - packages/client/ui-theme/src/boot-theme.ts
    - packages/client/ui-theme/src/theme-settings.ts
    - packages/client/ui-layout/src/index.ts
    - packages/client/ui-primitives/src/Button.module.css
    - packages/client/tsdown.client.ts
---

## 1. 系统与方法

DeepSeek Harness 的 Web UI 采用 **CSS Modules + 自定义设计令牌（--dsw-*）** 的轻量方案，**不使用任何第三方组件库或 Tailwind**。样式由 `packages/client/ui-theme` 集中管理主题令牌与全局样式，`packages/client/ui-layout` 负责将主题快照应用到 DOM，各功能包（ui-conversation、ui-primitives、ui-settings-* 等）通过 CSS Modules 编写组件级样式，并通过 `clsx` 组合类名。

构建工具链：`packages/client/tsdown.client.ts` 使用 tsdown 编译客户端产物，并启用 CSS Modules；根级 `tsconfig.base.client.json` / `tsconfig.client.json` 提供共享 TypeScript 配置。Web 应用入口位于 `apps/web/`，通过 Vite 构建。

## 2. 关键文件与包

- **主题令牌与全局样式**：`packages/client/ui-theme/src/styles/` 下五个 sheet：
  - `base.css`：定义基础字体栈与过渡变量（`--ds-font-family`、`--ds-ease-in-out` 等），依赖上游 deepsuite theme 提供的 `--dsw-*` 静态标量。
  - `design-platform.css`：声明语义别名 token（`--dsw-alias-*`）。
  - `scrollbar.css`：统一滚动条样式，绑定 `--dsh-scrollbar-thumb` / `--dsh-scrollbar-thumb-hover` 到 l1 表面 token，支持在容器上重绑定。
  - `gradient-shadow-text.css`、`shiki.css`：渐变/阴影/代码高亮。
- **主题运行时**：`packages/client/ui-theme/src/boot-theme.ts`、`index.ts`、`theme-settings.ts`，维护 light/dark/system 偏好，发布不可变 `ThemeSnapshot`，不直接操作 DOM。
- **主题应用层**：`packages/client/ui-layout/src/index.ts` 暴露宿主加载入口，presenter 将 snapshot 写入 `html { color-scheme }`、`body[data-ds-dark-theme]` 及内联 alias token。
- **组件样式示例**：`packages/client/ui-primitives/src/Button.module.css` 等大量 `.module.css` 文件，全部通过 `var(--dsw-alias-*)` 引用语义 token。
- **样式规范文档**：`docs/web-styling.md`（含中文镜像）。

## 3. 架构与约定

- **所有权分层**：`ui-theme` 独占 `--dsw-*` 静态标量、语义别名、排版、动效、渐变、阴影、滚动条样式与明暗偏好；`ui-layout` 负责把解析后的主题快照渲染到文档；功能包只消费语义别名，不再定义全局主题。
- **组件样式隔离**：每个组件的样式放在同目录的 `.module.css` 中，通过 CSS Modules 作用域化；组件可定义局部自定义属性以表达布局/呈现契约，但颜色、排版、层级、动效必须走主题包。
- **令牌命名空间**：
  - `--dsw-static-*`：静态色板值（如 `--dsw-static-blue-400`）。
  - `--dsw-alias-*`：语义别名（如 `--dsw-alias-label-primary`、`--dsw-alias-button-primary-fill`、`--dsw-alias-state-error-primary`）。
  - `--dsh-scrollbar-*`：滚动条专用变量，供 `scrollbar.css` 与消费者（如 ui-sidebar、ui-conversation）协作。
- **响应式策略**：通过 `prefers-color-scheme` 自动切换明暗主题；无独立断点系统，布局变化由组件自身处理。
- **无障碍**：规范要求保持键盘焦点可见性，并在添加过渡或 hover-only 控件时尊重 reduced-motion。

## 4. 约定与约束

以下规则来自 `docs/web-styling.md` 与 `packages/client/ui-theme/README.md` 的明确声明：

- **禁止引入组件库或 Tailwind**：组件样式必须使用 CSS Modules + `clsx`。
- **禁止硬编码颜色**：功能组件中只能使用 `--dsw-alias-*` 语义 token，不得复制静态调色板数值或直接写字面量颜色。
- **禁止在组件样式中放置主题选择器**：明暗覆盖必须由主题所有者（ui-theme）处理。
- **排版约定**：字号需搭配行高，已有角色匹配时使用主题排版变量。
- **文本不换行**：源码、终端输出、diff 行在组件契约要求列对齐时必须保持不换行；滚动条样式统一使用共享的 scrollbar 样式而非组件自定义选择器。
- **表现层归 CSS**：内联 React 样式可以传递组件级自定义属性，但不得编码主题分支。
- **滚动条重绑定契约**：`scrollbar.css` 绑定 `--dsh-scrollbar-thumb` / `--dsh-scrollbar-thumb-hover` 于 `body` 至 l1 表面 token；提升表面（菜单、气泡、对话框）需在自身容器重绑定为 l2；重绑到透明即隐藏滑块；`--dsh-scrollbar-width` 用于与 WebKit 滚动条宽度对齐的布局偏移。
- **变更流程**：新增/修改共享 token 必须在 `ui-theme` 中进行，再由功能包消费；公共样式契约变更需更新对应包的引用说明。
- **视觉回归测试**：遵循仓库 testing policy，变更需配合快照验证。
- **第三方主题扩展**：允许注册第三方主题 id 覆盖同名 alias 变量，但无完整性校验，属于扩展点而非产品特性。
- **token 唯一权威**：缺失的设计值不应随意追加，应优先使用最近的语义 token；经设计方批准的新增需同时提交静态步骤与语义别名。