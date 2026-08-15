---
kind: error_handling
name: 错误处理体系：HarnessError 语义化错误码、InvariantError 运行时不变式与 Python SDK 异常分层
category: error_handling
scope:
    - '**'
source_files:
    - python/sdk/src/deepseek_harness/errors.py
    - packages/core/tools/src/index.ts
    - packages/core/tools/src/code-mode.ts
    - packages/core/tools/src/json-schema.ts
    - packages/attachment/attachment/src/error.ts
    - packages/runtime-diagnostics/invariants/src/index.ts
    - packages/core/agent-loop/src/agent.ts
    - packages/acp/acp/src/index.ts
    - packages/api/gateway/src/client/index.ts
    - packages/code-runtime/code-runtime-worker-thread/src/bootstrap.ts
---

## 1. 整体方案

仓库采用**分层错误模型**：
- **Python SDK**（`python/sdk/src/deepseek_harness/errors.py`）定义 `HarnessError` 基类，派生 `TransportClosedError`、`SdkProtocolError`、`JsonRpcError`（携带 `code/message/data`），用于 JSON-RPC over stdio 的运行时通信失败。
- **TypeScript 核心层**通过 `@deepseek-ai/dsh-llm` 提供的 `HarnessError`（带 `name` + `code` 双字段）作为跨包可路由的错误基类；各业务包再继承它表达领域错误。
- **不变式检查**由 `packages/runtime-diagnostics/invariants` 提供 `InvariantRegistry`，注册后触发时抛出 `InvariantError(packageName, message)`，用于在运行期断言被破坏时立即中止当前 fiber。
- **LLM 层**使用 `LlmError`（含 `CONTEXT_WINDOW_EXCEEDED_CODE` 等常量）以及 `errorChain(error)` 将任意 `unknown` 值归一化为链式文本，供会话事件持久化。

## 2. 关键文件与位置

| 层次 | 文件 | 职责 |
|---|---|---|
| Python SDK | `python/sdk/src/deepseek_harness/errors.py` | `HarnessError` / `TransportClosedError` / `SdkProtocolError` / `JsonRpcError` |
| LLM 抽象 | `packages/llm`（通过 `@deepseek-ai/dsh-llm` 暴露） | `HarnessError`、`LlmError`、`errorChain`、`CONTEXT_WINDOW_EXCEEDED_CODE` |
| 工具执行 | `packages/core/tools/src/index.ts` | `ToolNotFoundError`、`ToolOutputError`、`CodeRunFailedError`（`code-mode.ts`）、`JsonSchemaError` |
| 附件模块 | `packages/attachment/attachment/src/error.ts` | `AttachmentError`（独立实现 `HarnessError` 形状以避免循环依赖） |
| 不变式 | `packages/runtime-diagnostics/invariants/src/index.ts` | `InvariantRegistry.register` → 抛 `InvariantError` |
| Agent 循环 | `packages/core/agent-loop/src/agent.ts` | 统一把 `LlmError` 原样转发、其它错误用 `errorChain` 包装为 `{ message, code: 'UNKNOWN' }` |
| ACP 适配层 | `packages/acp/acp/src/index.ts` | `catch (error: unknown)` 后用 `errorChain` 生成诊断字符串 |
| Client API | `packages/api/gateway/src/client/index.ts` | 配置阶段重复/冲突直接 `throw new Error(...)` 快速失败 |
| Code Runtime | `packages/code-runtime/code-runtime-worker-thread/src/bootstrap.ts` | 子进程绑定调用封装为 `BindingCallError extends CapturedError` |

## 3. 架构与约定

### 3.1 结构化错误码优先于类型判断
所有面向协议/策略的可路由错误都遵循 `{ name, code }` 结构：
- `ToolNotFoundError.code === 'UNKNOWN_TOOL'`
- `ToolOutputError.code === 'INVALID_TOOL_OUTPUT'`
- `CodeRunFailedError.code === 'CODE_RUN_FAILED'`
- `JsonSchemaError` 继承自同一基类
- `LlmError` 使用 `CONTEXT_WINDOW_EXCEEDED_CODE` 等常量
- `AttachmentError` 刻意重实现该形状以规避 `dsh-llm` 与其相互依赖造成的循环

消费端通过 `instanceof HarnessError` 或读取 `.code` 做分支（见 `core/tools/src/index.ts` 中 `errorInfo()` 提取 `{ name, code }`），而非依赖原型链。

### 3.2 未知错误归一化
`errorMessage(error)`（`core/tools/src/index.ts`）对任意 `unknown` 安全提取消息：优先 `Error.message`，其次对象上的 `message` 属性，最后 `String(error)`；若 `instanceof` / 属性访问本身抛错则回退到 `<unprintable thrown value>`。`errorChain` 在 agent-loop 中把非 `LlmError` 的异常统一包装为 `{ message: errorChain(error), code: 'UNKNOWN' }`，确保会话事件永远携带结构化错误。

### 3.3 不变式即致命错误
`invariants.register(packageName)(installer)` 会在选中时把 `fail(message)` 注入为 `throw new InvariantError(packageName, message)`。这是一种“开发期/可选开启”的断言机制：正常路径不触发，一旦触发即中断当前 fiber 并向上冒泡，用于保证如“轮次必须闭合”、“不可序列化数据不得进入日志”等契约。

### 3.4 子进程/Worker 边界错误隔离
`code-runtime-worker-thread` 中的 `CapturedError`（别名 `Error`）包裹所有跨 worker 调用的异常，并在 `BindingCallError` 中保留 `stack`/`message`，避免结构化克隆丢失堆栈信息。

### 3.5 启动/配置阶段快速失败
CLI 参数解析、API gateway 客户端挂载、Cordis 插件加载等基础设施层遇到非法状态直接 `throw new Error(...)`，因为此时没有上层恢复逻辑，应让进程退出（CLI 通过 Commander 的 `exitOverride` 转为 `process.exit`）。

## 4. 约定与约束

- **领域错误必须继承 `HarnessError` 并设置稳定 `code`**：`ToolNotFoundError`、`ToolOutputError`、`CodeRunFailedError`、`JsonSchemaError` 均如此，以便重试/沙箱/回放代码能区分错误来源。
- **面向协议的错误禁止携带原始字节或主机路径**：`AttachmentError` 构造注释明确要求 `message` 不含 raw bytes/host paths。
- **不可序列化的值必须在入口处拒绝**：`materializePresentation` / `snapshotJsonValue` 对 lossless JSON 校验失败时抛 `TypeError` 或 `ToolOutputError`，防止下游持久化崩溃。
- **不变式检查是可选开关**：通过 `ctx.invariants` 注册后才生效，未选中时不产生额外开销。
- **Python SDK 异常分层清晰**：`HarnessError` 为根，`TransportClosedError`（子进程退出）、`SdkProtocolError`（协议违规）、`JsonRpcError`（远程返回错误码）各自描述不同故障面，调用方可按层级 catch。
- **Agent 循环统一错误出口**：`agent.ts` 把所有 LLM 失败归一为 `LlmError` 或 `{ code: 'UNKNOWN', message }`，保证 `session.append('turn/end', { error })` 始终有结构。

## 5. 适用性说明

本仓库存在完整的错误处理体系：Python SDK 异常分层、TypeScript 侧基于 `HarnessError` 的结构化错误码、`InvariantError` 运行时不变式、`errorChain` 未知错误归一化、以及各子系统（tools、compaction、agent-loop、client API）对错误的捕获与传播约定。因此本类别完全适用。
