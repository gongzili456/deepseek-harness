---
kind: dependency_management
name: pnpm 工作区 + vendor 源码内嵌 + Dependabot 与脚本化版本治理
category: dependency_management
scope:
    - '**'
source_files:
    - package.json
    - pnpm-workspace.yaml
    - .github/dependabot.yml
    - scripts/rescope-vendor.ts
    - patches/node-pty@1.1.0.patch
    - python/sdk/pyproject.toml
    - python/sdk-runtime/pyproject.toml
    - native/landlock-run/package.json
    - examples/package.json
    - apps/cli/package.json
---

## 1. 使用的系统/工具

- **包管理器**：pnpm（`packageManager: "pnpm@11.7.0"`，通过 `.npmrc`/`pnpm-workspace.yaml` 声明），Node.js 引擎约束为 `^22.19.0 || >=24.0.0`。
- **工作区**：根级 `pnpm-workspace.yaml` 将 `vendor/*`、`packages/*/*`、`native/landlock-run` 及其子包、`apps/*`、`website`、`examples`、`python/sdk-runtime` 统一纳入同一依赖图；`linkWorkspacePackages: true` 使同仓库包之间以符号链接解析。
- **Python 生态**：`python/sdk` 使用 `pyproject.toml` + `uv.lock`（由 `uv` 管理），构建后端为 Hatchling；`python/sdk-runtime` 同样用 Hatchling 打包原生二进制产物。
- **更新自动化**：`.github/dependabot.yml` 同时监控 npm、uv 和 GitHub Actions 三个生态，默认每 30 天冷却一次，标签 `kind/dependency`、`area/infra`，并显式排除 `vendor/**`。
- **补丁机制**：`patches/node-pty@1.1.0.patch` 配合 pnpm 的 `patchedDependencies` 字段对第三方包进行精确修补。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `package.json`（根） | 定义 workspace 成员、顶层脚本（build/test/release/hygiene）、devDependencies |
| `pnpm-workspace.yaml` | 工作区目录、`overrides`、`peerDependencyRules`、`allowBuilds`、`minimumReleaseAgeExclude`、`patchedDependencies` |
| `.github/dependabot.yml` | 自动 PR 策略（npm / uv / actions） |
| `scripts/rescope-vendor.ts` | 将 vendor 下的 Cordis 上游包名重写为 `@deepseek-ai/*` 的 codemod，维护映射表与断言 |
| `patches/node-pty@1.1.0.patch` | 针对 node-pty 的官方补丁 |
| `python/sdk/pyproject.toml` + `uv.lock` | Python SDK 依赖声明与锁定 |
| `python/sdk-runtime/pyproject.toml` | 运行时二进制 wheel 的打包清单 |
| `native/landlock-run/package.json` | 原生启动器子工作区的独立 pnpm 配置 |

## 3. 架构与约定

### 3.1 三层依赖模型

1. **产品包层**：`packages/*/*` 下每个 `@deepseek-ai/dsh-*` 包通过 `workspace:^` 引用其他内部包，形成可组合的插件运行时。CLI（`apps/cli`）是装配入口，Web（`apps/web`）是前端产物。
2. **框架 vendor 层**：`vendor/` 存放 Cordis 框架源码副本（cosmokit、schemastery、plugin-loader/include/group/timer/hmr/logger-console 等）。这些包在 pnpm workspace 中作为普通成员存在，但通过 `scripts/rescope-vendor.ts` 将其 `name` 从 `cordis`/`@cordisjs/*` 重写为 `@deepseek-ai/cordis`/`@deepseek-ai/cordis-plugin-*`，从而避免污染公共 npm 命名空间。
3. **外部依赖层**：通过 `package.json` 的 `dependencies`/`devDependencies` 声明，并由 pnpm 解析到 `node_modules`。

### 3.2 版本锁定与升级策略

- **pnpm lockfile**：根级 `pnpm-lock.yaml` 锁定所有 workspace 成员的完整依赖树。
- **vendored 包不走 registry**：Dependabot 显式 `exclude-paths: ["vendor/**"]`，vendor 包的版本/变更由 `rescope-vendor.ts` 中的 `RENAMES` 映射与 `postinstall`/`hygiene` 流程控制，不依赖 npm 升级。
- **严格 build 脚本白名单**：`pnpm-workspace.yaml` 的 `allowBuilds` 仅允许 `esbuild`、`lefthook`、`node-pty`、`koffi`、`@deepseek-ai/dsh-subprocess-local@file:...` 等少数包执行安装/构建脚本；其余带 lifecycle script 的依赖（如 `@google/genai`、`protobufjs`）被显式拒绝，安装仍成功但不执行脚本，防止不可信代码在 CI 或开发者机器上运行任意命令。
- **peer dependency 收敛**：`peerDependencyRules.allowedVersions.typescript: ">=5 <7"` 强制全仓 TypeScript 主版本一致。
- **release age 豁免**：`minimumReleaseAgeExclude` 允许 `@earendil-works/pi-ai` 及一组 `node-addon-require-builtin-*` 平台包绕过 pnpm 的发布冷却期，因为它们是功能更新的必要依赖。

### 3.3 多语言依赖边界

- **Node 侧**：全部通过 pnpm workspace 统一管理，包括 `native/landlock-run` 这个单独维护 native 编译链的子工作区。
- **Python 侧**：`python/sdk` 与 `python/sdk-runtime` 各自维护独立的 `pyproject.toml`，并通过 `[tool.uv.sources]` 的 `editable = true` 指向本地 runtime 包，实现开发时解耦、发布时冻结。
- **示例与工作区聚合**：`examples/package.json` 把几乎所有 dsh 包以 `workspace:*` 形式声明，目的是让 `plain-node` 启动的 cordis 配置能直接解析真实 `exports→lib`，而非只作为构建目标。

### 3.4 发布与分发

- 根脚本提供 `release:dsh`、`release:vendor`、`release:pack`、`release:publish` 等流水线命令，分别处理产品包与 vendor 包的 bump/pack/publish。
- `scripts/gen-third-party-notices.ts` 生成 `THIRD_PARTY_NOTICES.md`，集中记录第三方许可证信息。
- Python SDK 通过 Hatchling 将注入的原生可执行文件（`dsh-jsonrpc-agent-*`）打包进 wheel，并在 sdist/wheel 构建钩子中完成。

## 4. 约定与约束

- **内部包一律通过 `workspace:^` 或 `workspace:*` 引用**：`apps/cli`、`examples` 等消费方从不写死版本号，保证开发期与发布期依赖图一致。
- **vendor 包禁止通过 npm 升级**：Dependabot 排除 `vendor/**`，任何 vendor 变更需走 `rescope-vendor.ts` 的映射表与 `hygiene` 校验，确保重命名幂等且无残留。
- **不允许未审查的安装脚本**：新增依赖若带有 lifecycle script，必须在 `pnpm-workspace.yaml` 的 `allowBuilds` 中显式声明 `true`/`false`，否则 pnpm install 失败。
- **TypeScript 主版本收敛**：通过 `peerDependencyRules` 强制所有包使用 `>=5 <7` 的 TS 版本。
- **node-pty 必须通过 patch 应用**：版本与补丁路径在 `patchedDependencies` 中固定，禁止随意升级而不同步补丁。
- **Python 依赖锁定**：`python/sdk` 使用 `uv.lock` 锁定，`python/sdk-runtime` 通过 Hatchling artifacts 列表精确控制 wheel 内容，排除 `node/` 等开发目录。
- **Hygiene 门禁**：根 `package.json` 的 `hygiene` 脚本串联 `rescope-vendor:check`、`knip`、`publint`、`constraints`、`verify-dsh-package-licenses`、`verify-package-invariants`、`verify-built-package-invariants`、`verify-cordis-config`、`verify-node-next-types`、`verify-runtime-closure`、`verify-vendored-links` 等检查，作为提交前/CI 的统一依赖契约入口。

## 5. 总结

该仓库采用 **pnpm workspace + vendor 源码内嵌 + Dependabot 半自动升级** 的组合策略：核心业务包通过 workspace 互相引用，Cordis 框架源码以 vendor 形式驻留并通过脚本重命名为 `@deepseek-ai/*` 以避免命名冲突，外部依赖通过 lockfile 与严格的 `allowBuilds` 白名单管控，Python 侧独立使用 uv/Hatchling 管理。整个体系由大量 `scripts/*.ts` 校验脚本与根 `hygiene` 命令统一约束，形成“可审计、可回滚、可自动化”的依赖治理闭环。