# Opencode ACP `ext_method` 适配改造方案（阶段一：Opencode 侧）

日期：2026-03-16  
范围：`opencode-refactor/opencode` + `python-sdk`（仅研究与方案，不含代码改动）

## 1. 问题背景

当前目标是实现这样一条链路：

1. 大模型在 Opencode 会话中**自主调用工具**（不是人工触发协议调用）。
2. 工具在 Opencode 内部通过 ACP 扩展请求（`ext_method`）向对端请求系统级信息。
3. 对端（后续 nanobot）返回结构化结果给 ACP 调用方，而不是经由聊天 channel 输出。

这意味着需要先完成 **Opencode 侧桥接能力**，把「模型工具层」与「ACP 扩展传输层」打通。

---

## 2. 现状结论（基于源码）

### 2.1 ACP 扩展语义（python-sdk）

- 扩展方法以 `_` 前缀走 JSON-RPC 扩展路由（如 `_vendor/method`）。
- `ext_method("x/y", params)` 在连接层会发送 `"_x/y"` 请求。
- 未识别的扩展请求按 JSON-RPC 语义返回 `-32601`（Method not found）。

对应参考：

- `python-sdk/src/acp/router.py`
- `python-sdk/src/acp/client/connection.py`
- `python-sdk/src/acp/agent/connection.py`
- `python-sdk/tests/test_rpc.py`

### 2.2 Opencode ACP 层现状

- ACP 入口：`opencode/packages/opencode/src/cli/cmd/acp.ts`（创建 `AgentSideConnection` + `ACP.Agent`）。
- ACP 会话管理：`opencode/packages/opencode/src/acp/session.ts`（`ACPSessionManager`）。
- ACP Agent 主要实现的是标准 ACP 方法（`initialize/newSession/loadSession/prompt/...`），尚未看到扩展方法桥接暴露。

### 2.3 Opencode 工具与插件现状

- 模型可用工具汇总与注册在 `ToolRegistry`（`tool/registry.ts`）。
- 工具执行入口在 `session/prompt.ts`，会把工具作为 model-callable tools 注入 LLM。
- Plugin 可以注入 `tool` 定义（`packages/plugin/src/index.ts` 的 `Hooks.tool`）。

**关键限制**：插件/工具上下文默认不直接持有 ACP 连接对象；因此“仅写插件”无法直接发送 ACP 扩展请求，需要 Opencode ACP 层提供桥接 API。

---

## 3. 改造目标（Opencode 阶段）

在不引入 nanobot 侧耦合的前提下，先在 Opencode 内实现：

1. 一个可被工具调用的 ACP 扩展请求 Broker（桥接器）。
2. 一个模型可调用的业务工具（示例：`ask_nanobot_session`），通过 Broker 间接调用 `ext_method`。
3. 明确的参数校验、错误分类与超时语义，避免把原始传输细节直接暴露给模型。

---

## 4. 总体方案（推荐）

### 4.1 分层边界

- **工具层（Tool）**：面向模型，输入应是业务语义（operation + typed args）。
- **桥接层（ACP Ext Broker）**：负责 operation -> ACP 扩展方法名映射，统一发起 `ext_method`。
- **传输层（ACP Connection）**：只做协议传输，不承载业务策略。

### 4.2 设计原则

1. 不向模型暴露原始 `{ method, params }` 透传能力。
2. 采用 operation 白名单（allowlist），每个 operation 绑定输入/输出 schema。
3. 一次工具调用最多触发一次 ext request，避免隐式 fan-out。
4. 失败语义标准化：
   - `preflight_reject`
   - `unsupported_operation`
   - `remote_invalid_params`
   - `unknown_outcome`（超时/链路中断后无法确认远端是否执行）

### 4.3 最小可行能力

以 `ask_nanobot_session` 为目标能力，Opencode 侧先定义固定 operation：

- `operation = "session_info.get"`

输入建议（示例）：

- `selectorType`: `"acp_session_id" | "nanobot_session_id" | "channel_session_key"`
- `selectorValue`: `string`

输出建议（示例）：

- `found: boolean`
- `resolvedBy: string`
- `session?: { nanobotSessionId, acpSessionId, channelSessionKey, endpoint { host, port, url } }`
- `error?: { code, message }`

---

## 5. 改造大致内容（只列 Opencode）

### 5.1 新增 ACP 扩展桥接模块

- 在 `opencode/packages/opencode/src/acp/` 下新增 broker 模块（命名待实现时确定）。
- 职责：
  - 根据 `sessionID` 定位会话上下文。
  - 将 operation 映射到扩展方法名（例如 `nanobot/session_info`）。
  - 调用 ACP 连接扩展请求。
  - 统一做超时、错误映射、结果校验。

### 5.2 新增业务工具（可先用 plugin tool）

- 工具 ID：`ask_nanobot_session`（建议名，可在实现时最终定名）。
- 工具描述强调“系统元信息查询，不会向聊天频道发消息”。
- 工具执行函数只调用 broker，不触及原始 ACP wire 细节。

### 5.3 会话上下文对齐

- 需要把工具执行时的 `sessionID` 与 ACP 会话上下文对齐。
- 若当前 session 无可用 ACP 扩展通道，返回 `preflight_reject` 而非 silent fail。

### 5.4 可观测性与审计

- 记录 operation、sessionID、耗时、结果类别。
- 对 selector 与返回值做安全脱敏（避免日志泄漏敏感数据）。

---

## 6. 风险与规避

1. **协议漂移风险**：远端扩展 payload 变更导致解析失败。  
   规避：operation 版本化（例如 `session_info.get.v1`）。

2. **幂等性风险**：超时重试可能造成远端重复执行。  
   规避：默认不自动重试，除非 operation 明确标记为幂等。

3. **模型误用风险**：模型尝试调用未开放 operation。  
   规避：严格 allowlist + 清晰错误返回。

4. **职责污染风险**：把传输细节泄露到工具层。  
   规避：强制工具只面向业务 schema，传输细节由 broker 封装。

---

## 7. 与 nanobot 阶段的边界

本阶段只保证 Opencode 具备“发起扩展调用”的能力与契约，不依赖 nanobot 先行改造。

后续 nanobot 侧再实现对应扩展方法处理器与返回数据源即可对接。

---

## 8. 交付物定义（本文件对应）

本文件是架构与改造范围说明，回答三个问题：

1. Opencode 现状下为什么不能只靠插件直连 ext_method。
2. Opencode 侧应该先补哪一层能力。
3. 改造分层、风险与阶段边界如何划分。
