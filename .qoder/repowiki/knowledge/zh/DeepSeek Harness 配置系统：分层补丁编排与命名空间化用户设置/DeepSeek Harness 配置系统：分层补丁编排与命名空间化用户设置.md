---
kind: configuration_system
name: DeepSeek Harness 配置系统：分层补丁编排与命名空间化用户设置
category: configuration_system
scope:
    - '**'
source_files:
    - apps/cli/src/args.ts
    - apps/cli/src/profile-boot.ts
    - apps/cli/src/bin.ts
    - apps/cli/src/dump-config.ts
    - packages/settings/settings/src/index.ts
    - packages/settings/settings/src/types.ts
    - packages/settings/settings/src/redact.ts
    - packages/settings/settings-file/src/index.ts
    - docs/subsystems/settings.md
    - apps/cli/config/agent-presets/code/agent.cordis.yml
    - examples/headless-agent/cordis.yml
---

## 1. 使用的系统与方案

仓库采用**两套互补的配置机制**，分别面向「插件组合编排」和「用户运行时设置」：

- **Cordis 插件组合配置（profile + patch）**：通过 `dsh` CLI 的 profile/patch 体系加载 Cordis 插件树。核心由 `apps/cli/src/profile-boot.ts` 实现，使用 `@deepseek-ai/dsh-app-boot` 提供的 `loadProfile`、`composeEntries`、`loadOverlayPatches`、`watchUserPatches` 等能力，将多个 YAML patch 文件按严格顺序叠加。
- **用户设置服务（settings seam）**：通过 `packages/settings/settings` 抽象出 `SettingsProvider` 基类，以命名空间（namespace）为单位注册 schema + base + user 三层合并；`packages/settings/settings-file` 提供基于 `settings.yaml`/`.json` 的文件实现，支持 chokidar 热重载、原子写入、跨进程文件锁与注释保留的 YAML diff 写回。

两者都遵循 Cordis 框架的服务注入模型（`ctx.settings`、`ctx.events`），并通过 schemastery 做 schema 校验。

## 2. 关键文件与包

| 路径 | 职责 |
|---|---|
| `apps/cli/src/args.ts` | dsh 命令行解析：`--profile`、`--patch`、`--dump-config`、`web`/`plugin` 子命令 |
| `apps/cli/src/profile-boot.ts` | profile 装载、patch 层组装、HMR 监听、`DSH_TELEMETRY_DISABLED` 开关 |
| `apps/cli/src/bin.ts` | 入口，调用 `parseDshArgs` → `runProfile` |
| `apps/cli/src/dump-config.ts` | 打印已组合的 profile 树（调试用） |
| `packages/settings/settings/src/index.ts` | `SettingsProvider` 抽象基类、命名空间注册、三层合并（schema defaults → base → user）、变更事件、revision 冲突检测、`installSettingsSection` 辅助 |
| `packages/settings/settings/src/types.ts` | 客户端安全类型面（`SettingsNamespace`、`SettingsUpdateSource`、Cordis Events 声明） |
| `packages/settings/settings/src/redact.ts` | 对 `role('secret')` 字段做脱敏枚举 |
| `packages/settings/settings-file/src/index.ts` | 文件型 `SettingsProvider`：YAML/JSON 文档、chokidar watch、原子写、comment-preserving diff、`$DSH_HOME/settings.yaml` 默认位置 |
| `docs/subsystems/settings.md` | 官方子系统文档，描述命名空间、scope、descriptor、事件契约 |
| `apps/cli/config/agent-presets/*` | 内置 agent preset 的 cordis.yml/preset.yml 示例 |
| `examples/*/cordis.yml` | 各示例工程的 profile 组合入口 |

## 3. 架构与设计约定

### 3.1 Cordis Profile/Patch 编排（启动期配置）

- **层次顺序（从低到高，后者优先）**：
  1. bundle 层：`package.json` 中 `dsh.profile.bundles` 列出的 bundle 各自声明的 patch
  2. profile 层：`$DSH_HOME/profiles/<name>/cordis.patch.yml`
  3. home 层：`$DSH_HOME/cordis.patch.yml`（机器级偏好，覆盖所有 profile）
  4. overlay 层：`--patch <path>` 指定的额外 patch 文件（argv 顺序）
  5. 遥测开关：若存在 `session-telemetry-otel` 行且 `DSH_TELEMETRY_DISABLED` 非空，则追加一个禁用该行的 patch

- 每个 profile 目录会重写一份空的 `cordis.yml` 作为 Loader 的 include 根锚点，实际组合全部来自上述 patch 层。
- 运行时通过 `watchUserPatches` 同时监听 profile 层与 home 层的 patch 文件变化，触发 recomposition；bundle 层在重启前保持不变。
- CLI 参数仅解析 launcher 自身需要的 flag（`--profile`、`--patch`、`--dump-config`、`--dump-default-config`），其余 argv 透传给被启动的应用插件。

### 3.2 用户设置（运行时配置）

- **三层合并**：最终值 = `mergeLayers(schema defaults, base, user section)`，其中 `base` 是插件 composition entry 的 subset，`user section` 是持久化文档中的同名 namespace 片段。
- **命名空间**：通过 `settingsNamespace(value)` 构造，必须匹配 `/^[a-z][a-z0-9-]*$/`，防止与其他 id 混淆。
- **注册接口**：插件通过 `ctx.settings.register(ns, schema, { base?, applies?, validate? })` 声明自己的配置段，返回 `SettingsScope<T>`（`get()` / `watch()` / `update()` / `replace()` / `mutate()`）。
- **变更传播**：每次提交（无论是 `update`/`replace`/`mutate` 还是外部编辑经 provider `publish`）都会 bump 单调 revision 并广播 `settings/updated`（resolved 值深相等才跳过）与 `settings/document-updated`（raw section 变化，即使 resolved 未变）。
- **并发安全**：每个 namespace 有独立的串行写队列；`expectedRevision` 用于乐观锁冲突检测（`SettingsConflictError`）；文件提供者使用 `withFileLock` + `writeFileAtomic` 保证多进程安全。
- **热重载**：`FileSettingsProvider` 默认启用 chokidar watch，debounce 窗口内多次写入只触发一次 reload；reload 失败时记录警告并保留 last good document，不中断进程。
- **秘密字段**：schema 中标记 `role('secret')` 的字段在 `describe({ redactSecrets: true })` 时被剥离，并在 descriptor 的 `secrets` 数组中列出 `{path, set}`，供 UI 渲染 write-only 输入而不泄露值。
- **可选消费者模式**：`installSettingsSection(ctx, ns, schema, entry, hooks)` 让消费方在 settings provider 存在时自动注册，provider 卸载时回退到 composition entry，避免强依赖。

### 3.3 配置文件格式与存储

- Cordis profile/patch：YAML 格式的 patch 列表，由 Loader 解析为 EntryOptions 树。
- 用户设置文档：`settings.yaml`（默认，位于 `$DSH_HOME`，可通过 `dshHome` 覆盖）或 `.json`；YAML 写回使用 yaml-js 的 `Document` 进行 comment-preserving 的节点级 diff，保持用户注释不被覆盖。
- 权限：目录创建使用 `0o700`，文件写入使用 `0o600`。

## 4. 约定与约束

- **CLI 参数边界**：`args.ts` 明确禁止父级选项出现在 `web`/`plugin` 子命令之前（`rejectParentOptions`），launcher 只认自己定义的 flag，其余透传。
- **telemetry 开关语义**：`DSH_TELEMETRY_DISABLED` 任何非空值即禁用（包括 `'0'`/`'false'`），设计原则是“隐私开关宁可误关也不误开”。
- **settings 写入数据限制**：`cloneJsonShaped` 拒绝非 JSON 兼容值（Date、Map、BigInt、循环引用、非有限数字、undefined 数组项等），确保持久化可逆。
- **命名空间命名规范**：必须是小写 kebab-case，由 `settingsNamespace` 在构造时强制校验，否则抛 `TypeError`。
- **wire 表面脱敏强制**：文档明确要求“every wire surface MUST pass `redactSecrets: true`”，违反会在配置 UI 中泄露 secret。
- **provider 初始化失败策略**：`FileSettingsProvider` 的 `load` 阶段解析失败视为 boot failure（fail loud），而 watcher reload 失败则 warn 并保留 last good document——区分启动期与运行期的容错策略。
- **patch 层不可变性**：`profile-boot.ts` 在每次 live recomposition 时对 patch 数组执行 `structuredClone`，避免 bundle 的 insert 引用被用户覆盖污染。
- **home patch 优先级高于 profile patch**：`$DSH_HOME/cordis.patch.yml` 始终在 profile 层之后应用，体现“机器级偏好 > 项目级配置”的约定。

## 5. 总结

该仓库的配置系统围绕两个维度组织：启动期的 Cordis profile/patch 编排负责把插件树拼装成可运行的宿主，运行期的 settings seam 负责以命名空间为单位管理用户可编辑的运行时配置。两者都通过严格的层次顺序、schema 校验、事件驱动的热更新以及并发安全的写路径来保证一致性。CLI 仅暴露最小化的 launcher flag，真正的业务配置下沉到各插件的 composition entry 与用户 settings 文档中。