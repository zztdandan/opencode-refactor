# OpenCode 本次执行踩坑复盘（完整）

日期：2026-03-11  
范围：`opencode-refactor` 根仓 + `opencode` 子仓（代码改动、环境升级、安装替换、分支推送）

## 目标回顾

1. 确认 `packages/opencode/src/acp/agent.ts` 的截断逻辑是否满足要求（两个超长字段、10KB、带提示）。
2. 补充英文注解并确保通过检查。
3. 升级 Bun；将当前源码版本作为本用户默认 `opencode`；备份旧版本。
4. 创建 `bugfix` 分支、提交并推送到 `origin`。

---

## 踩坑总览（按时间顺序）

## 1) 一开始在错误仓库层级看 diff/log

- **现象**：最早在维护仓根目录执行 `git diff -- opencode/packages/opencode/src/acp/agent.ts`，输出为空。
- **根因**：`opencode` 是子仓，真正变更在 `opencode/` 目录内的独立 git 仓库。
- **修复**：切换 `workdir` 到 `.../opencode` 后重新执行 `git status/diff/log`。
- **经验**：多仓/子模块场景，先确认“当前 git 根 + 文件实际归属仓”。

## 2) AST 搜索误判为“无匹配”

- **现象**：`ast_grep_search` 查 `sessionUpdate: "tool_call_update"` 返回空。
- **根因**：AST 模式必须是完整节点，单字段匹配有时会失败；且文件中是对象字面量里的属性，文本 grep 更稳。
- **修复**：回退到 `grep` + `read` 做精确定位。
- **经验**：AST 适合结构替换，文本常量检索先用 grep/ripgrep。

## 3) LSP 诊断不可用（TS LSP 未安装）

- **现象**：`lsp_diagnostics` 报 `typescript-language-server` 未安装。
- **根因**：环境缺少 LSP 服务，不是代码问题。
- **修复**：改用 `typecheck` 作为权威静态检查。
- **经验**：LSP 失败时立刻切换到项目原生校验链（`tsgo/tsc`）。

## 4) 测试初次失败：依赖未安装

- **现象**：ACP 测试报 `Cannot find package 'glob'`、`Cannot find module '@agentclientprotocol/sdk'`。
- **根因**：工作区依赖未就绪。
- **修复**：在 `opencode/` 执行 `bun install` 后重跑测试通过。
- **经验**：任何测试失败前先判断是“业务失败”还是“环境未初始化”。

## 5) `bun install` 与 `bun upgrade` 多次网络中断

- **现象**：`bun install` 超时，`bun upgrade` / 官方安装脚本多次 `ConnectionClosed` / `unexpected eof`。
- **根因**：网络下载大文件不稳定。
- **修复**：
  - `bun install` 提高超时并重试。
  - Bun 升级改为手工下载发布包（`curl --retry --retry-all-errors`）+ 解压覆盖 `~/.bun/bin/bun`。
- **经验**：弱网下优先“可重试、可续传、可校验”的手工安装路径。

## 6) Typecheck 暴露一个真实 TS 类型坑

- **现象**：`TS2638`: `in` 运算符右侧可能为 primitive。
- **根因**：`update.rawOutput` 联合类型不足以直接用于 `"output" in rawOutput`。
- **修复**：在判断前补 `typeof update.rawOutput !== "object"` 的守卫。
- **经验**：类型系统报错不要绕过，做最小语义守卫修复即可。

## 7) build 因 Bun 版本门槛失败

- **现象**：构建脚本要求 `bun@^1.3.10`，环境是 `1.3.9`。
- **根因**：仓库脚本有显式版本校验。
- **修复**：先升级 Bun，再重跑构建。
- **经验**：遇到构建门槛先满足 toolchain 版本，再判断业务问题。

## 8) build 跨平台目标下载失败（arm64 工具链下载中断）

- **现象**：`build` 在构建 `opencode-linux-arm64` 时下载失败。
- **根因**：默认构建多目标，下载体量大，网络波动放大失败概率。
- **修复**：改用 `build --single` 仅构建本机目标验证。
- **经验**：本地验证优先单目标；发版前再跑全矩阵。

## 9) `bun add -g <local package>` 长时间卡在 resolving

- **现象**：全局安装本地包两次超时。
- **根因**：Bun 全局安装解析工作区依赖时耗时/网络不稳定。
- **修复**：切换 `npm install -g <local path>` 完成安装尝试。
- **经验**：同一目的可多工具兜底（bun/npm/mise）。

## 10) npm 全局装完，`which opencode` 仍指向旧版

- **现象**：仍命中 `~/.local/share/mise/.../1.2.10/opencode`。
- **根因**：`PATH` 优先级中 mise 路径在 npm global bin 前。
- **修复**：在 `~/.bun/bin` 放置同名 wrapper（PATH 更靠前），强制指向源码入口。
- **经验**：安装成功 ≠ 命令已切换，必须验证 `which` + `--version`。

## 11) npm 全局 bin 的 `opencode` 直接跑报 ESM/CJS 冲突

- **现象**：`require is not defined in ES module scope`。
- **根因**：包内 `bin/opencode` 使用 `require`，在当前安装上下文被当作 ESM 执行。
- **修复**：绕过该入口，使用 `bun run .../src/index.ts` wrapper。
- **经验**：本地源码替换场景，最稳是显式调用源码入口，而不是赌全局 bin 解释器行为。

## 12) Bun 命令参数顺序踩坑

- **现象**：`bun run --cwd packages/opencode typecheck` 输出帮助信息而不是执行。
- **根因**：`--cwd` 与 `run` 子命令组合顺序不当。
- **修复**：改为 `bun --cwd packages/opencode run typecheck`（或进目录后执行）。
- **经验**：Bun CLI 子命令参数要严格按文档顺序。

## 13) 推送时被 pre-push hook 触发全量 Turbo 校验，且污染 `bun.lock`

- **现象**：`git push` 触发 `bun turbo typecheck`/build，过程中改写 `bun.lock`（大量 registry URL 变化）。
- **根因**：仓库 pre-push hook 会执行重量级校验并产生锁文件副作用。
- **修复**：
  - 每次 push 后检查 `git status`。
  - 明确恢复 `bun.lock`（`git restore --worktree -- bun.lock`）避免脏改动混入。
- **经验**：有 hook 的仓库，push 之后也要看一次工作区是否被 hook 改写。

## 14) 远程跟踪分支无法正常 set-upstream

- **现象**：`git branch --set-upstream-to=origin/bugfix bugfix` 报“不是一个分支”。
- **根因**：`remote.origin.fetch` 只配置了 `dev`（`+refs/heads/dev:refs/remotes/origin/dev`），非通配分支映射。
- **修复**：
  - 分支已成功推送（`git ls-remote --heads origin bugfix` 可见）。
  - 保持现状，不改全局/仓库 fetch 规则，避免引入额外配置变更。
- **经验**：遇到受限 fetch refspec，先确认“已推送成功”再处理 tracking 体验问题。

---

## 关键结果（最终状态）

- Bun 版本：`1.3.10`（已生效）。
- 旧版 opencode 备份：`/home/base/opencode-backup/20260311-165955`。
- 当前默认 opencode：`/home/base/.bun/bin/opencode`，`opencode --version = local`。
- 代码提交：`cc045bf`（`bugfix` 分支）已推送到 `origin`。
- 远端分支存在：`refs/heads/bugfix`（已验证）。

---

## 可复用 SOP（下次直接照做）

1. **先确认仓库层级**：子仓就进子仓再看 git 状态。  
2. **先装依赖再测**：避免把环境问题误判为代码问题。  
3. **弱网安装策略**：失败即切手工下载 + 重试参数。  
4. **命令切换验证**：每次替换 CLI 后必须 `which + --version`。  
5. **push 后二次检查**：防 hook 污染 lockfile。  
6. **受限 fetch refspec 场景**：先确认远端分支已存在，再决定是否需要额外配置。

---

## 本次最有价值的三条经验

1. **“执行成功”不等于“目标达成”**：安装成功后仍可能命中旧 PATH。  
2. **把失败分层**：环境失败、网络失败、类型失败、业务失败要分开处理。  
3. **在有 hook 的工程里，push 也是一次“可能改文件”的构建动作**：必须收尾清理。
