# OpenCode 使用 Bun 的 Worktree 源码安装/更新/回退导览（实测版）

> 本文按一次完整实操沉淀：在 `.worktrees/sourceinstall` 中合并 `acp-bugfix`，完成源码构建与命令替换，并验证 web/CLI 界面是否可用。

## 0. 超短 SOP（10 行版）

```bash
cd <repo-root>/.worktrees/sourceinstall
git merge bugfix/acp-bugfix
bun install
./packages/opencode/script/build.ts --single
hash -r
which opencode
opencode --version
opencode --help
opencode web --port 4096 --hostname 127.0.0.1 --print-logs
curl -i http://127.0.0.1:4096/
```

> 若 `which opencode` 不是 `~/.bun/bin/opencode`，先修正 wrapper 再继续。

## 1. 目标与原则

目标：

- 在 worktree 中安装并构建最新源码版本。
- 将默认 `opencode` 入口切到该 worktree 构建产物。
- 保留可回退路径，避免污染主同步仓。

原则：

- 主库 `opencode/` 只同步，不直接开发；开发与验证在 `.worktrees/<name>/`。
- 先 `which`/`--version` 再认定切换成功。
- 合并分支前先看工作区是否干净，若冲突立即停止处理冲突。

---

## 2. 前置检查

```bash
# 当前命中的 opencode / bun
which opencode
opencode --version
which bun
bun --version

# 查看 opencode 子仓 worktree 与分支
git -C opencode worktree list
git -C opencode branch -vv
```

确认点：

- `bun` 版本满足仓库要求（实测为 `1.3.10`）。
- `sourceinstall` worktree 是否已存在。
- `bugfix/acp-bugfix` 分支是否可见。

---

## 3. 创建/进入 worktree（sourceinstall）

若尚未创建：

```bash
cd <repo-root>/opencode
git worktree add ../.worktrees/sourceinstall -b sourceinstall dev
```

进入工作区：

```bash
cd <repo-root>/.worktrees/sourceinstall
git status --short --branch
```

---

## 4. 在 worktree 合并 acp-bugfix（冲突即停）

```bash
cd <repo-root>/.worktrees/sourceinstall
git merge bugfix/acp-bugfix
```

处理规则：

- 无冲突：继续后续安装步骤。
- 有冲突：**立即停止**，先解决冲突再继续。

实测记录：

- 本次为无冲突合并，产生合并提交。
- 合并前发现 `bun.lock` 已是脏状态（非本次引入），需注意不要误还原用户本地改动。

---

## 5. 安装依赖与构建（worktree 内）

```bash
cd <repo-root>/.worktrees/sourceinstall
bun install
./packages/opencode/script/build.ts --single
```

关键说明：

- 推荐使用 `./packages/opencode/script/build.ts --single`（来自仓库 `CONTRIBUTING.md`）。
- `--single` 只构建本机目标，适合本地验证。
- 构建成功后应出现：
  - `packages/opencode/dist/opencode-linux-x64/bin/opencode`

验证产物：

```bash
test -x packages/opencode/dist/opencode-linux-x64/bin/opencode
packages/opencode/dist/opencode-linux-x64/bin/opencode --version
```

---

## 6. 固定入口：`~/.bun/bin/opencode` wrapper

建议 wrapper 直接指向 worktree：

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="/home/base/repo/github-refactor/opencode-refactor/.worktrees/sourceinstall/packages/opencode"
BIN="$ROOT/dist/opencode-linux-x64/bin/opencode"

if [ -x "$BIN" ]; then
  exec "$BIN" "$@"
fi

exec bun run --conditions=browser "$ROOT/src/index.ts" "$@"
```

```bash
chmod 755 ~/.bun/bin/opencode
hash -r
```

---

## 7. 切换后强制验证（必须）

```bash
which opencode
opencode --version
opencode --help
```

通过标准：

- `which opencode` 命中 `~/.bun/bin/opencode`。
- `opencode --version` 与 sourceinstall 构建版本一致（示例：`0.0.0-sourceinstall-...`）。
- `--help` 能正常显示命令与 Banner（说明 CLI 可正常渲染）。

---

## 8. Web 与 CLI 界面实测方法

### 8.1 Web（本机 smoke test）

```bash
opencode web --port 4096 --hostname 127.0.0.1 --print-logs
```

另一个终端验证：

```bash
curl -i http://127.0.0.1:4096/
```

通过标准：

- 返回 `HTTP/1.1 200 OK`。
- 日志出现 `Web interface: http://127.0.0.1:4096/`。

### 8.2 CLI（TUI）

```bash
opencode .
```

通过标准：

- 终端能看到 TUI 初始界面（ANSI 画面正常，不是纯错误堆栈）。

---

## 9. 后续更新 SOP（worktree 复用）

```bash
cd <repo-root>/.worktrees/sourceinstall
git fetch origin
git pull
bun install
./packages/opencode/script/build.ts --single
opencode --version
```

只要 wrapper 的 `ROOT` 不变，后续无需重复改 PATH 或软链。

---

## 10. 回退与卸载

### A) 回退旧版本（推荐）

1. 备份并移走 `~/.bun/bin/opencode` wrapper。  
2. 恢复旧来源（mise/npm/brew 等）在 PATH 中的优先级。  
3. 验证：

```bash
which opencode
opencode --version
```

### B) 停用源码入口

```bash
rm -f ~/.bun/bin/opencode
hash -r
```

---

## 11. 本次新增踩坑（重点）

### 坑 1：`bun --cwd packages/opencode run build --single` 可能误用

在当前仓库，官方文档给出的本地构建命令是：

```bash
./packages/opencode/script/build.ts --single
```

如果仍要走 `bun run`，要确认参数透传行为（通常需 `--`）。

### 坑 2：`opencode web` 会触发项目级 `.opencode` 依赖安装

实测日志出现：

- 尝试安装 `@opencode-ai/plugin@0.0.0-sourceinstall-...`
- 该版本在 registry 不存在，导致安装告警/失败

影响：

- `web` 页面主入口仍可返回 `200`，但项目插件安装流程会报错。

建议：

- 测试时先关注 `web` 基本可达性（`HTTP 200` + health）。
- 若要彻底无告警，需额外处理 sourceinstall 通道的插件版本来源（本地 link/私有源/版本策略）。

### 坑 3：shell 命令缓存导致版本看起来没切换

```bash
hash -r
which opencode
opencode --version
```

---

## 12. 最小可复制流程（worktree 版）

```bash
# 0) 进入 sourceinstall worktree
cd <repo-root>/.worktrees/sourceinstall

# 1) 合并 bugfix
git merge bugfix/acp-bugfix

# 2) 安装 + 构建
bun install
./packages/opencode/script/build.ts --single

# 3) 激活入口并校验
chmod 755 ~/.bun/bin/opencode
hash -r
which opencode
opencode --version
opencode --help

# 4) web smoke
opencode web --port 4096 --hostname 127.0.0.1 --print-logs
curl -i http://127.0.0.1:4096/
```

---

## 13. 结论

采用“worktree 开发 + `~/.bun/bin/opencode` 固定入口 + `--single` 本机构建”后：

- 版本切换可控，可随时回退。
- 主同步仓不被污染。
- web 与 CLI 都能快速做回归验证，同时更容易暴露 sourceinstall 特有坑点。
