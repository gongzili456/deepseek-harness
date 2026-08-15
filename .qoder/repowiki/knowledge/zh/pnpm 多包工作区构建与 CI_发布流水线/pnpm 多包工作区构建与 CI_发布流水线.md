---
kind: build_system
name: pnpm 多包工作区构建与 CI/发布流水线
category: build_system
scope:
    - '**'
source_files:
    - package.json
    - pnpm-workspace.yaml
    - tsconfig.base.json
    - vitest.config.ts
    - .github/workflows/ci.yml
    - .github/workflows/release.yml
    - scripts/run-gates.ts
    - apps/cli/package.json
    - apps/web/package.json
    - native/landlock-run/package.json
    - python/sdk/pyproject.toml
    - python/sdk-runtime/pyproject.toml
---

## 1. 使用的系统与工具

- **包管理器**：pnpm 11（`packageManager: pnpm@11.7.0`），通过 `pnpm-workspace.yaml` 声明工作区成员，包括 `vendor/*`、`packages/*/*`、`native/landlock-run`、`apps/*`、`website`、`examples`、`python/sdk-runtime`。
- **TypeScript 编译**：`tsc -b`（Project References）+ `tsdown` 生成宿主/客户端产物；根级 `tsconfig.base.json` 提供跨包路径映射（`paths` 将 `@deepseek-ai/dsh-*` 指向源码目录），并通过 `composite` + `incremental` 启用增量构建。
- **Web 前端构建**：`apps/web` 使用 Vite 6 + React，独立 `vite.config.ts` 构建静态资源，由 CLI 的 `dsh web` 命令服务。
- **测试框架**：Vitest 4，根级 `vitest.config.ts` 定义两个项目（`thread-safe` 与 `process-bound`），按平台排除 Windows 不支持套件，覆盖率阈值强制 100%（perFile）。另有 `vitest.e2e.config.ts`、`vitest.snapshot.config.ts`、`vitest.web.config.ts`、`vitest.web-stress.config.ts` 等专用配置。
- **质量门禁**：自定义 Node 脚本 `scripts/run-gates.ts` 聚合所有 gate（typecheck、lint、coverage、snapshot、artifacts、consumers、Windows 阻塞/完整/观测集），通过 `DSH_GATE_CONCURRENCY` 控制并发，CI 中统一以 `pnpm run check:ci:*` 调用。
- **原生构建**：`native/landlock-run` 子仓库用 TypeScript + C（musl 静态链接）构建 Landlock 自限制执行器，通过 `scripts/build.ts` 和 `scripts/pack-release.mjs` 产出按平台分发的 npm optional 依赖。
- **Python SDK**：`python/sdk` 使用 Hatchling (`hatchling==1.30.1`) 打包 wheel，依赖 `deepseek-harness-runtime-bin`（同仓库 `python/sdk-runtime` 通过 `uv` editable 源注入）；运行时二进制由 Node 侧构建后拷贝进 wheel。
- **文档站点**：`website/` 使用 VitePress，通过 `pnpm --filter @deepseek-ai/website run build` 构建，并附带 `verify-doc-site-fragments` 校验。

## 2. 关键文件

- `package.json`：顶层脚本入口（`build`、`test`、`check:ci:*`、`release:*`、`docs:*`）。
- `pnpm-workspace.yaml`：工作区成员、`linkWorkspacePackages`、`overrides`、`allowBuilds`（白名单式允许 esbuild/lefthook/node-pty/koffi 等带 install script 的包）、`patchedDependencies`（`node-pty@1.1.0.patch`）。
- `tsconfig.base.json`：全局 TS 选项与 `paths` 别名映射（含 `@deepseek-ai/dsh-*` 通配映射到 `packages/*/src`）。
- `vitest.config.ts`：测试项目划分、覆盖率阈值 100%、平台/进程边界排除规则。
- `.github/workflows/ci.yml`：主 CI，包含 static、coverage、consumers、node-compat (22.19/26)、python-sdk、python-runtime、windows (Wine + native) 等多矩阵任务，以及 failover 到 self-hosted pool 的机制。
- `.github/workflows/release.yml`：dsh 发布流程——pack → verify-packed-install → 手动 dispatch 从 `dsh-v*` tag 触发 publish。
- `scripts/run-gates.ts`：gate 调度器，定义 `ci-primary` / `ci-static` / `ci-coverage` / `ci-snapshot` / `ci-consumers` / `ci-windows-blocking|complete|observational` / `node-compat` / `check-all` / `doc-sync` 等模式。
- `native/landlock-run/package.json`：原生子仓库构建与 release 脚本（`build:native`、`release:pack`、`release:publish`）。
- `python/sdk/pyproject.toml` + `python/sdk-runtime/pyproject.toml`：Python wheel 构建配置，`artifacts` 仅打包注入的二进制，排除开发期 node 闭包。

## 3. 架构与约定

- **分层构建**：先 `build:lib:host`（`tsc -b tsconfig.host.json && tsdown --env.DSH_BUILD_FACE host`）再 `build:lib:client`（同理 client），最后 `build:web`（Vite）。CLI 产物为 `apps/cli/lib/bin.js`，Web 产物为 `apps/web/dist/`。
- **Vendor 隔离**：`vendor/*` 作为 workspace 成员被 `linkWorkspacePackages` 解析，但 `tsdown` 构建时通过显式 glob 排除，避免 vendor 源码进入产品包。
- **依赖安全策略**：`pnpm-workspace.yaml` 的 `allowBuilds` 默认拒绝所有带 lifecycle script 的依赖，仅白名单放行 esbuild、lefthook、node-pty、koffi 等明确审查过的包；`minimumReleaseAgeExclude` 对特定依赖豁免发布冷却期。
- **测试覆盖门控**：`vitest.config.ts` 中 `thresholds.perFile: true` 且 statements/branches/functions/lines 均为 100%，缺失必须附 v8 ignore 注释说明原因（见注释引用 quality-gates Agent Note）。
- **CI 可恢复性**：Linux 与 Windows 均支持通过仓库变量 `DSH_CI_FAILOVER_LINUX` / `DSH_CI_FAILOVER_WINDOWS` 切换至 self-hosted pool；master push 上运行 serial 自检 job 持续验证备用池就绪。
- **发布解耦**：`release.yml` 将 pack 与 publish 分离——pack 在 PR/master 上无凭据运行以证明可打包，publish 需手动 dispatch 且受 `environment: npm-publish` 保护，消费 pack 产出的 tarball 而非重新构建。
- **Python 绑定**：`python/sdk-runtime` 通过 hatch custom hook 将 Node 侧构建的 `dsh-jsonrpc-agent-*` 二进制注入 wheel，SDK 通过 `uv` editable source 指向该 runtime 包，实现 Python 与 Node 构建链的松耦合集成。

## 4. 约定与约束

- **Node 版本**：根 `engines.node` 要求 `^22.19.0 || >=24.0.0`，CI 主版本 `PRIMARY_NODE_VERSION=24`，兼容矩阵额外验证 22.19 与 26。
- **锁文件锁定**：所有安装步骤使用 `pnpm install --frozen-lockfile`，禁止隐式更新。
- **工作区协议**：包间依赖一律使用 `workspace:^` 或 `workspace:*`，禁止直接指定版本号。
- **构建产物路径**：TS 产物输出到各包 `lib/`，Web 产物输出到 `dist/`，CLI bin 指向 `lib/bin.js`。
- **覆盖率不可降级**：100% per-file 覆盖率是合并门槛；任何 `// c8 ignore next` 注释必须附带原因说明。
- **Gate 并发**：本地与 CI 通过 `DSH_GATE_CONCURRENCY` 环境变量统一控制 gate 并行度，`run-gates.ts` 会记录来源（自动推导 vs 显式设置）。
- **Windows 构建**：PR 阶段先在 Linux 上用 Wine 运行 `scripts/wine-windows-gates.sh` 做阻塞检查，再由独立的 `windows-native` job 在真实 Windows runner 上跑完整 inventory；`serial-windows` 作为 master push 上的 self-hosted 热备。
- **Playwright 缓存**：Chromium 系统依赖通过 `~/.cache/ms-playwright` 缓存，hosted 环境首次安装 `--with-deps`，self-hosted VM 镜像已预装系统包故跳过 apt 安装。
- **Bubblewrap 沙箱**：CI 在安装依赖的同时并行执行 `scripts/prepare-ci-bubblewrap.sh` 准备容器化沙箱环境。
- **发布标签约束**：`release.yml` 的 publish 输入描述明确要求“Must run from a dsh-v* tag”，这是发布流程的硬性约束。