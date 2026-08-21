---
kind: dependency_management
name: pnpm monorepo 依赖管理：workspace、vendor、patch 与 Dependabot 协同
category: dependency_management
scope:
    - '**'
source_files:
    - package.json
    - pnpm-workspace.yaml
    - .github/dependabot.yml
    - patches/node-pty@1.1.0.patch
    - python/sdk/pyproject.toml
    - python/sdk-runtime/pyproject.toml
    - examples/package.json
    - apps/cli/package.json
---

## 1. 使用的系统与工具

- **包管理器**：pnpm（根 `package.json` 通过 `packageManager: "pnpm@11.7.0"` 锁定版本，Node 引擎要求 `^22.19.0 || >=24.0.0`）。
- **工作区**：`pnpm-workspace.yaml` 声明了 `vendor/*`、`packages/*/*`、`native/landlock-run` 及其子包、`apps/*`、`website`、`examples`、`python/sdk-runtime` 为 workspace members；其中 `examples` 仅用于依赖解析（注释明确“NOT build targets”），`python/sdk-runtime` 是单可执行发布物的部署根。
- **锁文件**：根目录存在 `pnpm-lock.yaml`，所有 npm 依赖由 pnpm 统一锁定。
- **Python 依赖**：`python/sdk` 使用 uv + `pyproject.toml` + `uv.lock`；`python/sdk-runtime` 同样用 `pyproject.toml`（hatchling 构建后端）。
- **自动更新**：`.github/dependabot.yml` 配置了三个生态的定时扫描：npm（排除 `vendor/**`）、uv（`python/sdk`）、GitHub Actions，均按 `Asia/Shanghai` 时区每日凌晨触发，带 30 天冷却期并打上 `kind/dependency` / `area/infra` 标签。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `package.json`（根） | 声明 workspace 成员、顶层脚本（build/test/release/hygiene）、devDependencies |
| `pnpm-workspace.yaml` | 工作区成员列表、`linkWorkspacePackages: true`、`overrides`、`peerDependencyRules`、`allowBuilds`、`minimumReleaseAgeExclude`、`patchedDependencies` |
| `.github/dependabot.yml` | npm / uv / GitHub Actions 依赖升级自动化 |
| `patches/node-pty@1.1.0.patch` | 针对 `node-pty` 的补丁，通过 `patchedDependencies` 应用 |
| `vendor/` | 存放 vendored 框架源码（当前为空，但 workspace 规则与 dependabot 排除表明其作为本地 fork 存在） |
| `python/sdk/pyproject.toml` + `uv.lock` | Python SDK 依赖声明与锁定 |
| `python/sdk-runtime/pyproject.toml` | Python 运行时 wheel 构建配置，打包预编译的 `dsh-jsonrpc-agent-*` 二进制 |
| `examples/package.json` | 聚合所有 leaf 插件的 `workspace:*` 依赖，使示例能直接以真实 package exports→lib 运行 |

## 3. 架构与约定

### 3.1 内部包通过 workspace 引用
仓库内所有 `@deepseek-ai/dsh-*` 包都位于 `packages/<domain>/<name>/package.json`，彼此之间一律以 `workspace:^` 或 `workspace:*` 引用。生产装配（如 `apps/cli`）大量使用 `workspace:^`，示例集合（`examples/package.json`）则使用 `workspace:*` 以精确匹配当前源码版本。这保证了开发时所有内部包始终指向本仓库源码，避免被 npm registry 上的旧版本污染。

### 3.2 上游框架以 vendor 形式驻留
`pnpm-workspace.yaml` 将 `vendor/*` 纳入工作区，并通过 `overrides` 把 `@deepseek-ai/cosmokit` 和 `@deepseek-ai/schemastery` 重定向到 `link:vendor/cosmokit` 与 `link:vendor/schemastery`，从而让本地构建解析到仓库内的 fork 版本。Dependabot 也显式 `exclude-paths: ["vendor/**"]`，说明 vendor 包的版本演进由人工维护而非自动 PR。

### 3.3 严格的构建脚本白名单
`pnpm-workspace.yaml` 的 `allowBuilds` 字段采用“默认拒绝”策略：只有 `esbuild`、`lefthook`、`node-pty`、`koffi`、以及通过 `file:` 引入的 `@deepseek-ai/dsh-subprocess-local` 被显式允许；`@google/genai`、`protobufjs`、`node-addon-require-builtin` 等传递依赖中的生命周期脚本被标记为 `false`（pnpm 仍会安装它们，但跳过脚本）。该机制在 pnpm 10+ 下成为硬性门禁，任何新引入的含 install/build 脚本的依赖都必须在此登记。

### 3.4 Peer 依赖约束
`peerDependencyRules.allowedVersions.typescript: ">=5 <7"` 强制整个工作区的 TypeScript 版本落在 5.x 范围内，防止某个包引入不兼容的 TS 版本。

### 3.5 补丁机制
`node-pty@1.1.0` 通过 `patchedDependencies` 映射到 `patches/node-pty@1.1.0.patch`，由 pnpm 在安装阶段自动应用。这是仓库中唯一记录的补丁，用于修复 PTY 跨平台边界问题。

### 3.6 Python 侧依赖
`python/sdk` 通过 `pyproject.toml` 声明对 `deepseek-harness-runtime-bin==0.0.0.dev0` 的依赖，后者由 `python/sdk-runtime` 以 hatchling 构建 wheel，并在 wheel 中嵌入预编译的 `dsh-jsonrpc-agent-*` 平台二进制（见 `artifacts` 配置）。编辑模式下通过 `[tool.uv.sources]` 将 `deepseek-harness-runtime-bin` 指向本地路径并启用 editable，实现 Python SDK 与本机 Node 构建产物联动。

### 3.7 发布与版本同步
根 `package.json` 提供 `release:dsh`、`release:vendor`、`release:verify`、`release:pack`、`release:publish` 等脚本，配合 `scripts/release/` 下的 bump/pack/publish 流程，统一管理 dsh 主包与 vendor 包的版本族（family）。

## 4. 约定与约束

- **内部包必须使用 workspace 引用**：所有 `@deepseek-ai/dsh-*` 包在相互依赖时一律使用 `workspace:^` 或 `workspace:*`，不得写死版本号。
- **vendor 包不走 Dependabot**：`vendor/**` 从 npm 依赖扫描中排除，版本变更由人工维护。
- **新增含生命周期脚本的依赖必须登记 allowBuilds**：pnpm 10+ 默认阻止未列出的 install/build 脚本，因此任何新依赖若携带脚本必须在 `pnpm-workspace.yaml` 的 `allowBuilds` 中显式声明（true/false）。
- **TypeScript 版本被 peer 约束限制在 5.x**：超出范围会在依赖解析时报错。
- **examples 仅用于依赖解析**：`examples` 被加入 workspace 的唯一目的是让每个 leaf 的 `cordis.yml` 能通过 `workspace:*` 解析到真实包导出，它不是构建目标（tsdown 显式排除）。
- **Python SDK 与 runtime 绑定**：SDK 通过 `==0.0.0.dev0` 精确绑定同仓库构建的 runtime wheel，确保 Python 用户拿到的是与 Node 构建一致的 dsh 二进制。
- **补丁需提交到 patches/ 并通过 patchedDependencies 关联**：目前仅 `node-pty@1.1.0` 遵循此模式。
- **hygiene 门禁串联依赖校验**：根 `hygiene` 脚本依次执行 `rescope-vendor:check`、`knip`、`publint`、`constraints`、`verify-dsh-package-licenses`、`verify-package-invariants`、`verify-built-package-invariants`、`verify-cordis-config`、`verify-node-next-types`、`verify-runtime-closure`、`verify-vendored-links`，覆盖依赖完整性、许可证、闭包、链接等维度。
