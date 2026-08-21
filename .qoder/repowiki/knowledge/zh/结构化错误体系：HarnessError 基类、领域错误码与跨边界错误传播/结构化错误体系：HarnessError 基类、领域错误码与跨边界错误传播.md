---
kind: error_handling
name: 结构化错误体系：HarnessError 基类、领域错误码与跨边界错误传播
category: error_handling
scope:
    - '**'
source_files:
    - packages/llm/llm/src/error.ts
    - packages/fs/fs/src/types.ts
    - packages/core/session/src/index.ts
    - packages/compaction/compaction/src/index.ts
    - packages/core/tools/src/index.ts
    - packages/api/gateway/src/types.ts
    - packages/context/session-reference/src/config.ts
    - packages/attachment/attachment/src/error.ts
    - packages/host/webserver/src/index.ts
    - python/sdk/src/deepseek_harness/errors.py
---

## 1. 整体方案

仓库采用「统一基类 + 领域错误码 + 结构化传播」的错误处理模式，核心由 `@deepseek-ai/dsh-llm` 包中的 `HarnessError` 基类提供，各功能包（fs、session、compaction、tools、api-gateway、context/session-reference、attachment 等）各自定义一组稳定的字符串错误码枚举，并通过扩展 `HarnessError` 或实现相同 `{ code, message }` 形状的错误类向外暴露。Python SDK (`python/sdk/src/deepseek_harness/errors.py`) 也定义了独立的 `HarnessError` 基类及 `TransportClosedError`、`SdkProtocolError`、`JsonRpcError` 子类，用于 JSON-RPC over stdio 子进程通信。

## 2. 关键文件与位置

- **基类与工具**：`packages/llm/llm/src/error.ts` — 定义 `HarnessError`（携带 `code` 与标准 `cause` 链）、`errorChain()`（渲染完整 cause 链）、`isContextWindowExceededError()` / `isQuotaExceededError()`（从 provider 消息中识别上下文溢出/配额耗尽）、`isHarnessError()` 类型守卫。
- **文件系统错误**：`packages/fs/fs/src/types.ts` — `FsErrorCode` 联合类型（如 `FS_NOT_FOUND`、`FS_PERMISSION_DENIED`、`FS_STALE_VERSION`、`FS_ABORTED` 等），`FsError extends HarnessError`。
- **会话分叉错误**：`packages/core/session/src/index.ts` — `SessionForkErrorCode` 与 `SessionForkError extends Error`（该模块未继承 `HarnessError`，使用自有 `name`）。
- **压缩错误**：`packages/compaction/compaction/src/index.ts` — `ManualCompactionErrorCode` 与 `ManualCompactionError extends Error`。
- **工具执行错误**：`packages/core/tools/src/index.ts` — `ToolNotFoundError`、`ToolOutputError` 均继承 `HarnessError`；通过 `errorInfo()` 将 `HarnessError` 提取为 `{ name, code }` 供工具结果持久化。
- **API Gateway 错误**：`packages/api/gateway/src/types.ts` — `TypertGatewayErrorCode` 联合类型（如 `ambiguous-endpoint`、`arguments-invalid`、`service-unavailable` 等）。
- **Session Reference 错误**：`packages/context/session-reference/src/config.ts` — `SessionReferenceErrorCode` 与 `SessionReferenceError extends Error`。
- **Attachment 错误**：`packages/attachment/attachment/src/error.ts` — 刻意不继承 `HarnessError` 以避免循环依赖，但保持相同的 `{ code, message }` 形状以便 wire 层互换。
- **Web 服务器兜底**：`packages/host/webserver/src/index.ts` — 每个请求的 `.catch()` 记录日志并返回 400，绝不 process exit。
- **Python SDK 错误**：`python/sdk/src/deepseek_harness/errors.py` — `HarnessError` 基类及传输/协议/JSON-RPC 三类异常。

## 3. 架构与约定

- **错误分类原则**：每个能力域维护一份「稳定、机器可路由」的错误码联合类型（如 `FsErrorCode`、`ManualCompactionErrorCode`、`TypertGatewayErrorCode`、`SessionReferenceErrorCode`），调用方按 `code` 分支而非解析 `message`。
- **基类设计**：`HarnessError` 同时承载人类可读的 `message` 和机器可读的 `code`，并通过标准 `ErrorOptions.cause` 支持错误链；`errorChain()` 会递归渲染 `cause` 链与 `AggregateError.errors`，对 UI/日志输出做安全降级（遇到 hostile 对象时返回 `<unrenderable value>`）。
- **跨边界传播**：工具执行层通过 `errorInfo()` 把 `HarnessError` 抽取为 `{ name, code }` 写入工具结果，使下游重试/沙箱/回放逻辑可以区分未知工具、参数非法、策略拒绝等不同失败类别。
- **Wire 层兼容**：`dsh-attachment` 故意重新实现 `HarnessError` 的形状而不继承它，因为 `@deepseek-ai/dsh-llm` 反向依赖 attachment（`ImageBlock` 引用 `ImageAttachmentRef`），为避免循环依赖而采用「duck-typed 形状一致」的策略，注释明确说明「消费者按 `code` 路由，从不依赖原型链」。
- **Provider 错误归一化**：`isContextWindowExceededError()` / `isQuotaExceededError()` 通过正则匹配 OpenAI 等 provider 的错误消息文本，把第三方错误归类到 `CONTEXT_WINDOW_EXCEEDED` / `QUOTA` 两个 provider-neutral 代码，便于上层统一重试/提示策略。
- **Web 边界兜底**：`host/webserver` 在每条 HTTP 请求与 WebSocket upgrade 路径上包裹 `.catch()`，捕获后仅记录 warn 并返回 400/销毁 socket，确保单个畸形请求不会导致整个宿主进程退出。
- **Python 侧**：SDK 与运行时之间通过 JSON-RPC 传递错误，`JsonRpcError` 携带 `code/message/data`，`TransportClosedError` 表示子进程退出/管道关闭，`SdkProtocolError` 表示协议越界数据。

## 4. 约定与约束

- **禁止解析 message 做分支**：多处注释强调「route on `code`, never by parsing `message`」（见 `HarnessError` 注释、`FsError` 注释、`AttachmentError` 注释）。
- **错误码必须稳定且有限**：每个包的错误码都以字面量联合类型集中声明（如 `FsErrorCode` 的 12 个成员），新增失败需先加入联合类型再抛错。
- **业务异常优先用 `HarnessError` 子类**：工具注册表、fs、tools 等核心路径抛出 `ToolNotFoundError`、`ToolOutputError`、`FsError` 等具名子类，以便被 `instanceof HarnessError` 识别。
- **非 `HarnessError` 异常需包装**：内部 `core/agent` 中仍可见直接 `throw new Error(...)` 的情况（如重复 session id、无效 inbox splice），这些属于「内部不变量违反」，通常不会跨越包边界；跨边界处应转换为带 `code` 的结构化错误。
- **错误链必须保留 `cause`**：`HarnessError` 构造签名接受 `ErrorOptions`，`errorChain()` 专门处理 `cause` 链与 `AggregateError`，表明错误链是诊断与重放的基础设施。
- **UI/日志渲染必须防御 hostile 值**：`errorChain()` 与 `errorMessage()` 都包含 try/catch 保护，防止恶意 `toString`/`Symbol.toPrimitive` 破坏日志输出。
- **Python SDK 错误与 JS 错误解耦**：Python 侧独立定义 `HarnessError` 层次，不与 JS 共享基类，通过 JSON-RPC 的 `code/message/data` 字段映射到 JS 端错误模型。