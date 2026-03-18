# OpenCode 安全工作范围约束说明（基于源码）

## 1. 目标

本文说明如何在项目目录的 `.opencode/opencode.jsonc` 中，尽量实现以下控制目标：

- 工作目录内可正常读写和分析。
- 超出工作目录后，默认拒绝高风险操作。
- 禁止通过 `bash`/脚本执行绕过工具级限制。
- 按 agent 维度限制可用模型、技能、工具。

> 说明：本文结论来自 `packages/opencode/src` 的实现逻辑，不是文档推断。

## 2. 源码里真正的控制面

### 2.1 配置与权限模型

- 配置与权限 schema：`opencode/packages/opencode/src/config/config.ts`
  - `permission` 支持：`read/edit/glob/grep/list/bash/task/external_directory/skill/...`
  - 动作是 `allow | ask | deny`
- 权限求值：`opencode/packages/opencode/src/permission/next.ts`
  - `evaluate()` 采用“最后匹配规则生效”
  - `disabled()` 会把 `pattern="*" 且 action="deny"` 的工具直接从可用工具集移除

### 2.2 工具执行时的权限检查

- 工具调用统一经过：`opencode/packages/opencode/src/session/prompt.ts`
  - `ctx.ask()` 最终走 `PermissionNext.ask()`
  - 实际规则集是 `agent.permission + session.permission`
- 典型工具检查点：
  - `bash`：`opencode/packages/opencode/src/tool/bash.ts`
  - `read`：`opencode/packages/opencode/src/tool/read.ts`
  - `write`：`opencode/packages/opencode/src/tool/write.ts`
  - `edit`：`opencode/packages/opencode/src/tool/edit.ts`
  - `apply_patch`：`opencode/packages/opencode/src/tool/apply_patch.ts`

### 2.3 工作目录边界

- 越界目录检查：`opencode/packages/opencode/src/tool/external-directory.ts`
  - 不在实例目录/工作树内时触发 `external_directory` 权限
- 边界判定：`opencode/packages/opencode/src/project/instance.ts`
  - `Instance.containsPath()` 判断是否属于 `directory` 或 `worktree`

### 2.4 工具/技能/模型可见性

- 工具按权限裁剪：`opencode/packages/opencode/src/session/llm.ts` (`resolveTools`)
- 技能按权限裁剪：`opencode/packages/opencode/src/skill/skill.ts` (`available`)
- Provider/模型过滤：`opencode/packages/opencode/src/provider/provider.ts`
  - `enabled_providers` / `disabled_providers`
  - provider 下 `whitelist` / `blacklist`

## 3. 可行性矩阵（Oracle 复核）

| 目标 | 结论 | 说明 |
| --- | --- | --- |
| 仅工作区可写 | B（部分可控） | `write/edit/apply_patch` 可被 `external_directory + edit` 约束；但若 `bash` 放开，仍可能绕过。 |
| 工作区外只读 | B（部分可控） | 可通过规则逼近；但一旦放开外部目录，写工具与 `bash` 组合会扩大风险面。 |
| 禁止脚本绕过 | C（仅配置无法在“允许 bash”前提下达成） | `bash` 是 shell 执行器，不是 OS 沙箱；保留 `bash` 时无法给出硬隔离承诺。 |
| 限制模型可用技能/工具 | A（可强约束） | 工具会在 `session/llm.ts` 前置裁剪，技能会在 `skill/skill.ts` 过滤。 |
| 限制 provider/模型可见性 | A（可强约束） | `enabled_providers` / `disabled_providers` + provider `whitelist/blacklist` 生效。 |

## 4. 可落地策略（推荐）

下面给一套“高安全默认”策略：

1. **禁用 `bash`**（这是防脚本绕过的核心）
2. **禁用 `task`**（防止调用其他子 agent 间接获得更宽权限）
3. **`external_directory` 默认 deny**（工作区外拒绝）
4. **只开放必要工具**（`read/glob/grep/list`，编辑相关按需开）
5. **provider + model 白名单**（从源头限制模型范围）
6. **按 agent 绑定模型 + 权限**（实现“某模型只能做某事”的近似效果）

## 5. 示例配置（高安全基线）

将以下内容放入目标项目的 `.opencode/opencode.jsonc`（按需改路径与模型）：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",

  // 仅启用需要的 provider
  "enabled_providers": ["opencode"],

  "provider": {
    "opencode": {
      // 仅允许这几个模型出现
      "whitelist": ["gpt-5", "gpt-5-mini"]
    }
  },

  "default_agent": "build",

  // 全局权限基线
  "permission": {
    "*": "deny",

    // 工作区内常用只读工具
    "read": "allow",
    "glob": "allow",
    "grep": "allow",
    "list": "allow",

    // 如需改文件可放开 edit；否则保持 deny
    "edit": "allow",

    // 关键：禁用脚本执行与子任务代理
    "bash": "deny",
    "task": "deny",

    // 关键：拒绝工作区外目录访问
    "external_directory": {
      "*": "deny"
    },

    // 示例：只允许部分技能
    "skill": {
      "*": "deny",
      "git-master": "allow",
      "frontend-ui-ux": "allow"
    }
  },

  // 可进一步给 agent 覆盖权限与模型
  "agent": {
    "build": {
      "model": "opencode/gpt-5",
      "permission": {
        "bash": "deny",
        "task": "deny",
        "external_directory": {
          "*": "deny"
        }
      }
    },
    "explore": {
      "disable": true
    },
    "general": {
      "disable": true
    }
  }
}
```

## 6. 关键细节与常见误区

### 6.1 “工作区外只读”是可配置但不绝对安全

理论上可以尝试：

- `external_directory` 放宽为 `ask/allow`
- `read` 允许外部
- `edit` 对外部模式设 `deny`

但这会显著增加绕过面（尤其当 `bash` 未禁用时）。实践上更推荐“工作区外全部 deny”。

### 6.2 只靠 `edit` 限制不够

`write/edit/apply_patch` 受 `edit` 权限控制，但若 `bash` 允许，仍可通过 shell 改文件。因此“禁脚本绕过”的前提是 `bash: deny`。

### 6.3 模型-工具-技能的三维精细矩阵没有独立一键开关

源码当前是：

- 模型可见性：provider 白名单/黑名单
- 工具/技能权限：按 agent 权限规则

即“某模型只能用某工具/技能”通常需要通过“**把模型绑定到 agent，再给该 agent 绑定权限**”间接实现。

## 7. 剩余风险与硬边界

以下是 `.opencode` 配置本身无法完全解决的边界：

- **不是 OS 级沙箱**：没有内核级隔离（如容器、seccomp、macOS sandbox）
- **同进程权限模型**：若开放高风险工具（特别是 `bash`），能力边界由权限规则而非系统隔离保证
- **路径判定以应用层为主**：建议把高敏环境再放进容器/受限用户执行
- **用户显式附件读取属于特例路径**：`session/prompt.ts` 在处理部分文件附件时会构造内部 `ReadTool` 上下文并使用 `bypassCwdCheck`，这是会话输入预处理链路，不等同于常规工具调用权限门禁

## 8. 实施建议（落地顺序）

1. 先把 `bash` 和 `task` 设为 `deny`
2. 再把 `external_directory` 设为 `{"*":"deny"}`
3. 验证必须工具链是否仍可完成日常任务
4. 仅按最小权限逐项放开
5. 同时收紧 provider/model 白名单

## 9. 折衷方案（保留 bash 能力）

你提到不能禁用 `bash`，这个场景建议采用“**bash 保留但强约束**”方案，而不是全开：

### 9.1 设计思路

1. **默认拒绝**：先 `"*": "deny"`，其余全部白名单放开。
2. **bash 命令分层**：
   - 低风险命令（`pwd/ls/git status`）直接 `allow`
   - 中风险命令（`git diff/bun test`）设为 `ask`
   - 高风险命令（`rm/curl/wget/node/python/bash/sh`）设为 `deny`
3. **目录双闸门**：`external_directory` 默认 `deny`，防止 bash 在工作区外落盘。
4. **禁子代理升级**：`task: deny`，防止通过子 agent 拿到更宽工具集。
5. **技能最小集**：`skill` 默认 deny，只放行业务必需技能。
6. **模型收敛**：`enabled_providers + whitelist` 锁死模型范围，避免高能力模型在宽权限配置下漂移。

### 9.2 推荐配置（保留 bash 的“可用且可控”版本）

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "enabled_providers": ["opencode"],
  "provider": {
    "opencode": {
      "whitelist": ["gpt-5", "gpt-5-mini"]
    }
  },
  "permission": {
    "*": "deny",

    "read": "allow",
    "glob": "allow",
    "grep": "allow",
    "list": "allow",
    "edit": "ask",

    "external_directory": {
      "*": "deny"
    },

    "task": "deny",
    "skill": {
      "*": "deny",
      "git-master": "allow"
    },

    "bash": {
      "pwd *": "allow",
      "ls *": "allow",
      "git status *": "allow",
      "git diff *": "ask",
      "git log *": "ask",
      "bun test *": "ask",
      "bun typecheck *": "ask",

      "rm *": "deny",
      "curl *": "deny",
      "wget *": "deny",
      "node *": "deny",
      "python *": "deny",
      "bash *": "deny",
      "sh *": "deny"
    }
  }
}
```

### 9.3 方案边界（必须接受）

- 该方案是“**显著降风险**”，不是“绝对防绕过”。
- 只要 `bash` 仍可用，就不是硬隔离；命令模式匹配无法替代 OS 级沙箱。
- 若后续要再提安全等级，优先路线不是继续堆规则，而是叠加容器/受限用户执行。

---

如果需要更强隔离（例如“工作区外永远只读且不可执行任何脚本”），建议在 `.opencode` 配置之外，再叠加运行时沙箱（容器/受限用户/文件系统挂载策略）。
