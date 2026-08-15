---
kind: configuration_system
name: DeepSeek Harness 配置系统：分层组合、Schemastery 校验与热重载文件存储
category: configuration_system
scope:
    - '**'
source_files:
    - packages/settings/settings/src/index.ts
    - packages/settings/settings-file/src/index.ts
    - apps/cli/config/agent-presets/standard/agent.cordis.yml
    - apps/cli/config/agent-presets/standard/preset.yml
    - examples/headless-agent/cordis.yml
    - docs/cordis-tutorial/05-config.md
    - apps/cli/src/dump-config.ts
    - scripts/gen-config-catalog.ts
    - scripts/verify-config-source-ownership.ts
    - scripts/verify-cordis-config.ts
---

## 1. 使用的系统与框架

DeepSeek Harness 的配置系统由三层组成：
- **Cordis 插件组合层**：每个应用通过 `cordis.yml`（YAML，支持 `!!js` 标签）声明插件清单与每行的 `config`。加载时按行顺序解析，将 `config` 交给插件导出的 Schemastery schema 校验后注入 `apply(ctx, config)`。
- **用户设置服务层 (`@deepseek-ai/dsh-settings`)**：抽象出 `SettingsProvider`，以“命名空间 + schema”为单位注册配置段；提供 `get/watch/update/replace/mutate` API、revision 并发冲突检测、以及 `settings/document-updated` / `settings/updated` 事件。
- **文件存储实现 (`@deepseek-ai/dsh-settings-file`)**：基于 YAML/JSON 的单一文档（默认 `<DSH_HOME>/settings.yaml`），使用 chokidar 监听外部编辑并热重载，写入时使用原子写 + 文件锁，并以注释保留的方式对 YAML 做最小 diff patch。

此外，CLI 还通过 `--dump-config` 输出 profile 的 patch 组合层（包内置 patch → 用户 profile patch → home patch → `--patch` overlay），不启动进程即可诊断组合结果。

## 2. 关键文件与包

- `packages/settings/settings/src/index.ts` — `SettingsProvider` 抽象基类、namespace 注册、合并层（schema defaults → base → user）、watcher 串行化、revision 冲突、`installSettingsSection` 辅助。
- `packages/settings/settings-file/src/index.ts` — `FileSettingsProvider`，YAML/JSON 文档读写、chokidar 热重载、`withFileLock` 互斥、`writeFileAtomic` 原子写、`patchNode` 注释保留 diff。
- `apps/cli/config/agent-presets/*/agent.cordis.yml` & `preset.yml` — 预置 Agent 组合与元数据（标准/最小/cordis/code 等 preset）。
- `examples/headless-agent/cordis.yml`、`examples/acp-agent/*.cordis.yml` — 示例应用的完整 Cordis 组合，展示如何挂载 `dsh-settings-file`、`dsh-credentials-local` 及业务插件。
- `docs/cordis-tutorial/05-config.md` — 官方教程，说明 `config` block、Schemastery schema、`!!js` 标签、fail-loud 行为。
- `apps/cli/src/dump-config.ts` — `dsh --profile <name> --dump-config` 的实现，打印各 patch 层。
- `scripts/gen-config-catalog.ts`、`scripts/verify-config-source-ownership.ts`、`scripts/verify-cordis-config.ts` — 生成配置目录文档、校验配置来源所有权与 cordis 格式。

## 3. 架构与设计约定

### 3.1 配置分层模型
`SettingsProvider` 的 resolved value 按固定顺序合并：
1. Schema 默认值
2. 注册的 `base`（即 Cordis entry 的 `config`）
3. 用户文档中该 namespace 的 section
4. 可选的 owner `validate(value)` 跨字段约束

任意一层失败都会拒绝整个值，保证 `apply` 永远拿到完整、已校验的配置。

### 3.2 命名空间与 schema
- 命名空间必须匹配 `/^[a-z][a-z0-9-]*$/`，通过 `settingsNamespace()` 工厂创建。
- 每个插件导出一个同名 `Config: Schema<T>`（Schemastery schema），既是 TypeScript 类型也是运行时校验器；Cordis 要求显式 schema，不接受裸对象。
- 通过 `role('secret')` 标记敏感字段，`describe({ redactSecrets: true })` 会剥离 secret 并在 descriptor 中列出位置，所有 wire 表面必须开启。

### 3.3 持久化与热重载
- 默认文档路径为 `$DSH_HOME/settings.yaml`（或 `.json`），可通过 provider 配置覆盖。
- 写入走序列化队列：同一 namespace 的 `update/replace/mutate` 串行执行；不同 namespace 共享单操作链，避免 watcher reload 与 write 竞争。
- 外部编辑通过 chokidar 监听，debounce 后 reconcile：先读磁盘、再 publish，解析失败仅警告并保留 last good document，不会崩溃进程。
- YAML 写入使用 `yaml` 库的 `Document` 进行注释保留的最小 diff patch；JSON 直接重写。
- 文件权限：目录 `0o700`，文件 `0o600`，使用 `withFileLock` 防止多进程同时写入。

### 3.4 并发与一致性
- 每次 write 携带 `expectedRevision`，若 namespace 在读取后被修改则抛出 `SettingsConflictError`（含 expected/actual revision）。
- watcher 回调串行化：每个 watcher 维护 `tail` Promise 链，确保回调按 commit 顺序依次执行，disposer 后不再触发。
- `publish` 时先快照 before 状态，再 swap document，最后 bump revision 并 emit 事件；invariant 异常会被收集并在所有 listener 执行后统一抛出。

### 3.5 Cordis 组合配置
- 每个 `cordis.yml` 条目可带 `config` 块，由对应插件的 schema 校验；错误时 fiber 进入 FAILED 并终止加载。
- 支持 `disabled: !!js ...` 与 `config` 中的 `!!js` 表达式，在加载期求值，用于平台分支、环境变量注入等。
- Preset 是带 `agent.cordis.yml` 的目录，位于 shipped 根与用户 `~/.agent-presets/`，通过 `discoverPresets` 扫描并按 id 去重（先入优先）。

## 4. 约定与约束

- **插件必须导出 Schemastery schema**：教程明确要求 `Config` 必须是 Schemastery schema，普通对象不被接受（见 `docs/cordis-tutorial/05-config.md`）。
- **命名空间命名规范**：`settingsNamespace()` 强制 kebab-case 小写字母开头，否则抛错。
- **只允许 JSON 兼容数据写入**：`cloneJsonShaped` 拒绝 Date、Map、BigInt、循环引用、非有限数、非 plain object 等，确保 YAML/JSON 持久化不失真。
- **用户文档必须为 map 根**：`parse()` 要求顶层为对象，否则抛 `TypeError`。
- **无效 stored section 不崩溃进程**：reload 时解析失败仅 warn 并保留 last good value；但初始 load 失败会作为启动失败处理（fail loud）。
- **Preset 目录健康检查**：`scanRoot` 会尝试用 loader 的 schema 解析 `agent.cordis.yml`，缺失或不可解析的目录报告为 broken 而非跳过。
- **Wire 表面必须 redact secret**：`describe` 的 `redactSecrets` 选项被设计为 wire 面默认开启，避免敏感信息泄露到远程 UI。
- **Patch 组合顺序固定**：`--dump-config` 输出顺序为：包内 patch → profile patch → home patch → `--patch` overlay，便于诊断。
- **配置文件归属校验**：`scripts/verify-config-source-ownership.ts` 校验每个配置来源的归属权，防止越权覆盖。

## 5. 典型用法模式

- 插件通过 `installSettingsSection(ctx, ns, schema, entry, hooks)` 将自身 Cordis entry 的 `config` 作为 `base` 层注册到 settings 服务，并在 settings 可用时接管、不可用时回退到 composition entry。
- 业务插件（如 `llm-deepseek`、`agent-default-model`、`permission-presets`）均遵循此模式，使同一份 schema 既驱动 UI 表单又驱动运行时行为。
- 示例应用（如 `headless-agent`）在 `cordis.yml` 中先挂载 `dsh-settings-file`，再挂载依赖配置的插件，从而支持运行时切换模型、凭据等。
