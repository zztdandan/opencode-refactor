# Opencode 插件工具方案：`ask_nanobot_session`（通过 ACP `ext_method`）

日期：2026-03-16  
状态：设计指南（仅研究，不含代码提交）

## 1. 目标

本指南描述如何在 Opencode 体系下规划一个“模型自主可调用”的工具，使其能够通过 Opencode ACP 兼容层调用扩展方法（`ext_method`）。

核心目标：

1. 模型调用工具（Tool Call）而不是人工输入命令。
2. 工具调用 Opencode 内部 Broker。
3. Broker 通过 ACP 扩展请求访问对端系统信息。

---

## 2. 先明确两件事

### 2.1 `ext_method` 不是自然语言接口

- `ext_method` 属于 ACP 扩展 JSON-RPC 请求机制。
- 需要代码层显式调用，协议层方法名走 `_` 前缀扩展路由。

### 2.2 模型“自主调用”要靠 Tool

- 大模型能自主的是调用它可见的工具。
- 因此应向模型暴露业务工具（如 `ask_nanobot_session`），再由该工具内部走 `ext_method`。

---

## 3. Opencode 内的推荐结构

## 3.1 工具定义层（Plugin/Tool）

使用 Opencode 已有插件能力定义工具：

- 插件接口：`Hooks.tool`
- 工具注册来源：
  - 插件导出的 `tool` 字段
  - 或 `{tool,tools}/*.ts` 自定义工具文件

建议工具定义：

- `id`: `ask_nanobot_session`
- `description`: 查询当前 ACP 会话关联的系统侧 session/endpoint 信息
- `args`（示例）：
  - `selectorType`
  - `selectorValue`

## 3.2 桥接层（Broker）

工具内部**不直接**构造/发送原始扩展 RPC；而是调用一个统一 broker：

- `broker.call(sessionID, operation, args)`

broker 负责：

1. operation 白名单检查。
2. 参数 schema 校验。
3. 映射到 ACP 扩展方法名。
4. 调用 ACP 扩展请求。
5. 结果 schema 校验与错误标准化。

## 3.3 传输层（ACP）

在 ACP 适配层执行真实扩展调用，保持协议细节封装在此层，不上浮给模型工具。

---

## 4. 推荐 operation 设计（v1）

建议不要开放原始 method 透传，先只开放稳定 operation：

- `session_info.get`

### 4.1 输入契约（建议）

```json
{
  "selectorType": "acp_session_id",
  "selectorValue": "sess_..."
}
```

`selectorType` 枚举建议：

- `acp_session_id`
- `nanobot_session_id`
- `channel_session_key`

### 4.2 输出契约（建议）

```json
{
  "found": true,
  "resolvedBy": "acp_session_id",
  "session": {
    "nanobotSessionId": "...",
    "acpSessionId": "...",
    "channelSessionKey": "...",
    "endpoint": {
      "host": "...",
      "port": 18790,
      "url": "http://..."
    }
  }
}
```

失败返回建议统一：

- `preflight_reject`
- `unsupported_operation`
- `remote_invalid_params`
- `unknown_outcome`
- `invalid_remote_result`

---

## 5. 模型自主触发的关键点

要让模型“自动调用”而不是人工触发，必须同时满足：

1. 工具定义足够清晰（description + 参数含义明确）。
2. 系统提示/agent 规则中写明触发条件。示例：
   - “当需要查询跨系统 session 映射或 endpoint 信息时，优先调用 `ask_nanobot_session`。”
3. 工具返回结构稳定且可被模型二次推理利用。

---

## 6. 安全与可靠性要求

1. **最小权限**：只暴露必要 operation，不开放任意 method。
2. **超时控制**：设置明确 timeout，超时标记为 `unknown_outcome`。
3. **重试策略**：默认不自动重试，避免副作用重复。
4. **审计日志**：记录 operation、sessionID、耗时、结果类型（注意脱敏）。
5. **版本化**：对 operation/结果 schema 做版本标识，便于演进。

---

## 7. 与后续 nanobot 改造的衔接

当 Opencode 侧具备上述能力后，nanobot 侧只需实现对应扩展方法处理器（方法名与 schema 对齐），即可完成端到端闭环。

换言之，优先级顺序应为：

1. Opencode：tool + broker + ext transport 适配。  
2. nanobot：扩展方法落地与数据源填充。

---

## 8. 适用范围

本指南只覆盖 Opencode 侧设计，不涉及：

- nanobot 端实现细节
- 生产部署与密钥管理策略
- 多 operation 的权限分级矩阵（可在后续 ADR 补充）
