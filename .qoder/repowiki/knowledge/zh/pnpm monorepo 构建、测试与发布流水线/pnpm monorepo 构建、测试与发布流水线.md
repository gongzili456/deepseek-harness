---
kind: build_system
name: pnpm monorepo 构建、测试与发布流水线
category: build_system
scope:
    - '**'
source_files:
    - package.json
    - pnpm-workspace.yaml
    - tsconfig.base.json
    - tsconfig.host.json
    - tsconfig.client.json
    - tsdown.config.ts
    - vitest.config.ts
    - vitest.e2e.config.ts
    - vitest.web.config.ts
    - vitest.snapshot.config.ts
    - .github/workflows/ci.yml
    - .github/workflows/release.yml
    - .github/workflows/build-exe-for-python-sdk.yml
    - apps/cli/package.json
    - scripts/run-gates.ts
    - scripts/release/bump.ts
    - scripts/release/verify.ts
    - scripts/release/pack.ts
    - scripts/release/publish.ts
    - patches/node-pty@1.1.0.patch
---

## 1. 使用的系统与工具

- **包管理**：pnpm 11（`packageManager: "pnpm@11.7.0"`），通过 `pnpm-workspace.yaml` 声明 workspace，启用 `linkWorkspacePackages` 将本地源码链接到依赖解析图。
- **TypeScript 编译**：`tsc -b`（Project References + incremental）配合 `tsconfig.base.json` 作为全局路径映射门面；宿主/客户端分别由 `tsconfig.host.json` / `tsconfig.client.json` 聚合引用。`tsdown.config.ts` 负责按 `DSH_BUILD_FACE=host|client` 选择入口并运行 Typert 代码生成。
- **Web 前端构建**：`apps/web` 使用 Vite（`vite.config.ts`），通过 `pnpm --filter @deepseek-ai/dsh-web-frontend run build` 触发。
- **测试框架**：Vitest 4（`vitest.config.ts`），配置两个 project：`thread-safe` 与 `process-bound`，统一 v8 覆盖率门限为每文件 100%（语句/分支/函数/行）。
- **质量门禁**：oxlint（`.oxlintrc.json`）、knip、publint、jscpd、lefthook 钩子，全部通过根 `scripts/run-gates.ts` 编排，并以 `check:ci:*` npm scripts 暴露给 CI。
- **CI/CD**：GitHub Actions（`.github/workflows/*.yml`），主流程 `ci.yml` 拆分 static / coverage / consumers / node-compat / python-sdk / windows 等并行 job，并通过 `all-checks-passed` 汇总 required check。
- **发布**：`release.yml` 分 pack 与 publish 两阶段，仅从 tag 手动 dispatch 发布；版本由 `scripts/release/bump.ts` 管理（`--family dsh|vendor`）。
- **原生组件**：`native/landlock-run` 独立维护 musl 静态二进制，通过 `scripts/build-exe-for-python-sdk.ts` 与 Python wheel 集成。

## 2. 关键文件

- `package.json` — 根脚本入口（build/test/lint/release/hygiene 等）
- `pnpm-workspace.yaml` — workspace 成员、`allowBuilds` 白名单、`patchedDependencies`
- `tsconfig.base.json` — 全局 TS 选项与 `@deepseek-ai/*` 路径映射门面
- `tsconfig.host.json` / `tsconfig.client.json` — 宿主/客户端聚合项目引用
- `tsdown.config.ts` — tsdown 工作区打包配置，区分 host/client 构建面
- `vitest.config.ts` / `vitest.e2e.config.ts` / `vitest.web.config.ts` / `vitest.snapshot.config.ts` — 多套测试配置
- `.github/workflows/ci.yml` — 主 CI 流水线（Linux/Windows/Python/兼容性矩阵）
- `.github/workflows/release.yml` — dsh npm 包打包与发布
- `.github/workflows/build-exe-for-python-sdk.yml` — Python SDK 可执行产物构建
- `apps/cli/package.json` — `dsh` CLI 的 `bin` 入口与依赖装配
- `scripts/run-gates.ts` — 质量门禁编排器
- `scripts/release/{bump,verify,pack,publish}.ts` — 发布脚本集
- `patches/node-pty@1.1.0.patch` — 对第三方包的 patch 应用

## 3. 架构与约定

### 构建面分离
- 宿主侧（Node）：`tsc -b tsconfig.host.json && tsdown --env.DSH_BUILD_FACE host`，产出 `lib/types/{index,invariant,startup}.js` 供运行时加载。
- 客户端侧（浏览器）：`tsc -b tsconfig.client.json && tsdown --env.DSH_BUILD_FACE client`，由各包自身 vite 配置产出浏览器 bundle。
- Web 前端单独构建：`pnpm --filter @deepseek-ai/dsh-web-frontend run build`。

### Workspace 依赖约束
- `pnpm-workspace.yaml` 的 `allowBuilds` 显式白名单允许 `esbuild`、`node-pty`、`koffi`、`lefthook` 等带 install/build 脚本的包，其余默认拒绝——这是强制约束，未列入的包安装直接失败。
- `overrides` 将 `@deepseek-ai/cosmokit`、`@deepseek-ai/schemastery` 指向 `vendor/` 下的本地源码，保证 vendored framework 与 harness 同步开发。
- `patchedDependencies` 对 `node-pty@1.1.0` 应用仓库级 patch。

### 测试与覆盖率
- Vitest 双 project 隔离进程绑定测试；覆盖率排除 vendor/examples/types-only/bin/worker 等目录，阈值 100% 每文件。
- E2E 使用 Playwright + Cordis 启动真实 Chromium（`vitest.web.config.ts`），快照驱动（`vitest.snapshot.config.ts`）。
- Windows 平台通过 Wine 在 Linux 上运行部分 gate（`scripts/wine-windows-gates.sh`），另有原生 Windows runner 跑完整 native 套件。

### CI 拓扑
- PR 触发：static / coverage / consumers / node-compat / python-sdk / python-runtime / windows（Wine + native）并行。
- Master 推送：额外运行 self-hosted 备用池的 serial 全量回归（Linux/macOS/Windows）。
- 所有阻塞 job 汇总到 `all-checks-passed`，用于 branch protection。
- 通过 `DSH_CI_FAILOVER_LINUX` / `DSH_CI_FAILOVER_WINDOWS` 仓库变量一键切换至自托管 pool。

### 发布流程
- `pnpm run release:dsh` → `release:verify` → `release:pack` → `release:publish`，仅在 `dsh-v*` tag 上手动触发 publish。
- 同时打包 vendored framework (`--family vendor`) 与 landlock entry，验证 packed install 后再发布。
- Python SDK 通过 `python/sdk/pyproject.toml`（uv）与 `python/sdk-runtime` 的 hatch 构建 wheel，CI 中调用 `build-exe-for-python-sdk.yml`。

## 4. 约定与约束

- **Node 版本**：要求 `^22.19.0 || >=24.0.0`，CI 以 Node 24 为主，另用 matrix 校验 22.19 与 26 兼容。
- **依赖锁定**：所有安装步骤使用 `pnpm install --frozen-lockfile`，禁止隐式更新。
- **构建面必须显式声明**：`tsdown.config.ts` 强制 `DSH_BUILD_FACE` 为 `host` 或 `client`，否则抛错。
- **覆盖率 100% 门控**：`vitest.config.ts` 中 `thresholds.perFile: true` 且四项指标均为 100%，任何新增未覆盖分支都会阻断合并。
- **install/build 脚本白名单**：`pnpm-workspace.yaml` 的 `allowBuilds` 是硬性约束，未列入的依赖无法安装。
- **workspace 成员固定**：`pnpm-workspace.yaml` 明确列出 `vendor/*`、`packages/*/*`、`apps/*`、`website`、`examples`、`python/sdk-runtime`，新增包需在此注册。
- **发布产物不可变**：`release.yml` 的 publish job 仅下载 pack job 产出的 artifact，不重新构建，确保发布物可追溯。
- **Telemetry 关闭**：CI 环境变量 `DSH_TELEMETRY_DISABLED=1` 阻止向生产端上报遥测。
- **Playwright 缓存**：PR 恢复 master 产生的 pnpm store 与 Playwright Chromium 缓存，避免每次重复下载。
- **Windows 符号链接**：原生 Windows runner 需先启用 Developer Mode 以支持 symlink（`reg add ... AllowDevelopmentWithoutDevLicense`）。