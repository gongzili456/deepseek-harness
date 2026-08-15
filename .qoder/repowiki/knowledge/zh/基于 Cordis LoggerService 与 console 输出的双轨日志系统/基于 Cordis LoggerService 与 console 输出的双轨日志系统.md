---
kind: logging_system
name: 基于 Cordis LoggerService 与 console 输出的双轨日志系统
category: logging_system
scope:
    - '**'
source_files:
    - packages/context/tmux-context/src/index.ts
    - packages/core/agent-loop/src/index.ts
    - packages/core/session/src/index.ts
    - packages/compaction/compaction-basic/src/index.ts
    - packages/client/hmr/src/index.ts
    - packages/client/connection/src/client/connection.ts
    - packages/client/runtime/src/client/sessions/session.ts
    - packages/api/gateway/src/client/index.ts
    - apps/cli/src/bin.ts
    - examples/package.json
    - tsconfig.base.json
---

## 1. 使用的系统与框架

仓库没有引入独立的第三方日志库（如 winston、pino、bunyan、signale 等）。日志输出采用**双轨模式**：
- **业务代码路径**：通过 Cordis 框架注入的 `ctx.logger`（类型 `LoggerService`，来自 `@deepseek-ai/cordis`）调用 `info/warn/error` 等方法。这是插件/服务内部的标准日志通道。
- **运行时/客户端路径**：在 Web 运行时、连接层、UI 组件等浏览器或宿主环境中，直接使用 Node/browser 原生的 `console.log/info/warn/error` 进行输出。

示例工作区通过依赖 `@deepseek-ai/cordis-plugin-logger-console`（由 `tsconfig.base.json` 中 `paths` 映射到 `./vendor/logger-console/src`）将 Cordis 的 logger 桥接到 `console`，作为开发/示例环境的 sink。

## 2. 关键文件与位置

- **Cordis 日志接口消费点**：`packages/context/tmux-context/src/index.ts` 通过 `import type { Context, LoggerService } from '@deepseek-ai/cordis'` 获取 `LoggerService`，并在查询失败时调用 `logger.warn(...)`。
- **核心业务中的 ctx.logger 使用**：大量 core 包（`agent-loop`、`session`、`compaction-basic`、`client/hmr`、`client/modules`、`client/ui-commands` 等）通过 `this.ctx.logger` / `ctx.logger` 调用 `warn`、`error`、`info`。
- **Console 直写路径**：`packages/client/connection`、`packages/client/runtime`、`packages/api/gateway`、`native/landlock-run/scripts/build.ts` 等直接调用 `console.error/warn/log`。
- **CLI 入口**：`apps/cli/src/bin.ts` 本身不直接输出日志，仅解析参数并动态加载 profile/plugin/dump-config 子模块；帮助/版本/错误由 Commander 处理。
- **示例 logger 桥接**：`examples/package.json` 声明 `"@deepseek-ai/cordis-plugin-logger-console": "workspace:*"`，`tsconfig.base.json` 将其 path 映射到 `./vendor/logger-console/src`。

## 3. 架构与约定

- **插件化日志 sink**：业务代码只依赖 `LoggerService` 抽象，不关心最终 sink 是 `console`、文件还是远程收集器。sink 由 Cordis 插件装配决定——示例工作区用 `cordis-plugin-logger-console` 把 logger 打到控制台。
- **结构化字段**：日志消息以模板字符串形式组织，包含上下文标识（如 `[web-runtime]`、`[client-connection]`、`tmux location query failed:`），但未见统一的 JSON 结构化字段 schema；结构化信息更多体现在 session 事件持久化（会话 log 作为不可变事件流）而非单条日志记录。
- **日志级别策略**：业务代码主要使用 `warn` 和 `error`；`info` 用于 compaction 步骤等进度性输出；`debug` 未见广泛使用。异常被捕获后降级为 warn 继续执行（如 tmux 查询失败不影响 turn 继续），体现“fail-soft”的健壮性约定。
- **测试中的 logger 替换**：测试通过 `vi.spyOn(ctx.logger, 'warn')` 或 `ctx.logger.warn = (...) => warnings.push(message)` 拦截日志，验证警告是否按预期触发，说明 logger 是可插拔且可观测的。

## 4. 约定与约束

- **业务代码禁止直接依赖具体日志实现**：通过 `LoggerService` 类型从 `@deepseek-ai/cordis` 注入，sink 由外部装配（见 `examples/package.json` + `tsconfig.base.json` 的 paths 映射）。
- **Web/客户端环境回退到 console**：当不存在 Cordis 上下文（如浏览器端连接层、host 脚本）时，直接调用 `console.*`，并以方括号前缀（如 `[web-runtime]`、`[client-connection]`）区分来源。
- **可选能力失败降级为 warn**：非关键路径（如 tmux 位置探测、HMR 重建、命令 listener 抛出）捕获异常后调用 `logger.warn` 并返回默认值，保证主流程不被中断。
- **CLI 不自行打印日志**：`bin.ts` 仅做参数分发，所有用户可见输出由被动态导入的 profile/plugin/dump-config 模块负责，保持 launcher 最小化。
- **无全局日志配置开关**：未发现环境变量或配置文件控制日志级别/格式；级别与 sink 由 Cordis 插件组合决定，当前仓库内仅定义了 console 桥接。
- **会话日志（session log）与运行日志分离**：`packages/core/session` 中的“log”指不可变的 session 事件序列（JSONL 持久化），用于推导对话历史与请求重建，不属于本类所指的运行时诊断日志。

综上，该仓库的日志系统是一个**轻量、插件化的方案**：业务代码通过 Cordis 的 `LoggerService` 抽象输出，sink 由 `cordis-plugin-logger-console` 在示例/开发环境中挂载到 `console`；浏览器/宿主侧则直接回退到原生 `console.*`。没有统一的结构化 schema、没有集中式日志级别开关，但通过 `ctx.logger` 的可插拔性与测试中的 spy 替换，实现了可观测性与可测试性。