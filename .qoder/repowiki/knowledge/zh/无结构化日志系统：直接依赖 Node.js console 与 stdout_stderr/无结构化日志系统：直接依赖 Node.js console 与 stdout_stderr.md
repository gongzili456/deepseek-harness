---
kind: logging_system
name: 无结构化日志系统：直接依赖 Node.js console 与 stdout/stderr
category: logging_system
scope:
    - '**'
source_files:
    - packages/host/apiproxy/src/api-proxy.ts
    - packages/host/apiproxy/src/fetch/client.ts
    - packages/client/runtime/src/client/sessions/session.ts
    - packages/client/connection/src/client/connection.ts
    - packages/client/ui-commands/src/client/popup.ts
    - packages/client/ui-input-trigger/src/client/controller.ts
    - apps/cli/src/bin.ts
    - package.json
---

## 1. 使用的方案

仓库中**没有引入任何第三方日志框架**。根 `package.json`、各子包 `package.json` 以及 `pnpm-lock.yaml` 中均未出现 pino、winston、bunyan、log4js、tslog、signale、consola、debug（除 website 开发依赖外）等日志库。唯一发现的是 `website/package.json` 中的 `debug: "4.4.3"`，仅用于文档站点构建脚本，不属于运行时日志。

所有运行时输出均通过 **Node.js 原生 `console.log / console.warn / console.error / console.info / console.debug`** 直接写入标准输出/错误流，没有任何封装层、格式化器或可配置级别控制。

## 2. 关键文件与位置

- `packages/host/apiproxy/src/api-proxy.ts`：在 presenter 失败时 fallback 到通用处理前打印 `console.error('api-proxy: presenter failed for ...')`
- `packages/host/apiproxy/src/fetch/client.ts`：SSE envelope listener 抛错与 malformed frame 丢弃时打印 `console.error('[apiproxy] ...')`
- `apps/cli/src/bin.ts`：CLI 入口本身不直接调用 console，而是委托给 profile/plugin/dump-config 子模块；这些子模块同样未见 console 调用
- 大量客户端代码（`packages/client/runtime`、`packages/client/connection`、`packages/client/ui-*`）使用 `console.error('[web-runtime] ...' | '[ui-commands] ...' | '[ui-input-trigger] ...' | 'locale subscriber crashed'` 等带前缀的错误输出，作为浏览器端调试手段

## 3. 架构与约定

- **无集中式 logger 初始化**：不存在类似 `logger.configure({ level, format })` 的启动逻辑，也没有环境变量（如 `LOG_LEVEL`、`DSH_LOG_*`）驱动日志级别切换的代码路径
- **无结构化字段**：日志消息为模板字符串拼接，未定义统一的 JSON 结构、traceId、sessionId、spanId 等字段
- **无 sink 路由**：所有输出直接落到进程 stdout/stderr，无法按来源模块分流到文件、远程收集器或告警通道
- **前端 vs 宿主差异**：宿主侧（host/boot）几乎不输出日志（仅 apiproxy 的少量 error），而 Web 客户端侧大量使用 `console.error/warn` 作为调试输出，并统一以 `[web-runtime]`、`[ui-commands]` 等方括号前缀区分来源
- **工具执行输出**：Agent 工具代码通过 `return` 或 `console.log(...)` 返回结果（见 `packages/core/tools/src/ts-types.ts` 注释），这是“工具输出”而非“运行日志”，由会话 log 持久化机制承载，不属于应用日志子系统

## 4. 约定与约束

- **约定**：需要输出的错误信息直接使用 `console.error` 并以方括号模块前缀标注来源（如 `[apiproxy]`、`[web-runtime]`、`[ui-commands]`），便于在终端中快速定位
- **约束**：仓库刻意避免新增外部依赖（`.agents/notes/implemented/process/2026-07-26-dependencies-over-hand-rolling.md` 记录了这一倾向），因此未引入专用日志库；这意味着当前日志能力是“尽力而为”的原始输出，不具备级别过滤、采样、异步落盘、结构化查询等生产级特性
- **测试侧**：E2E 与快照测试通过捕获会话事件（session log）验证行为，而非断言控制台输出；因此日志缺失不会导致测试不稳定

总结：该仓库采用**零依赖、直接写 `console` 的极简日志方式**，没有统一的日志框架、级别策略、结构化格式或输出路由；日志主要用于开发期调试与问题排查，不适合作为生产观测基础设施。