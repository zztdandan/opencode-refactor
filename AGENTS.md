# OpenCode Agent 维护库

本仓库是 OpenCode 项目的 Agent 专用维护库，用于管理文档、配置、日志和临时文件。

## 项目结构

```
opencode-refactor/
├── AGENTS.md           # 本文件 - 项目总览和Agent配置
├── README.md           # 项目说明文档
├── .worktrees/         # Worktree 工作区（临时功能开发）
│   ├── <feature-1>/    # 临时工作区：特定功能开发
│   ├── <bugfix-xxx>/   # 临时工作区：修复官方 bug
│   └── <pr-contribution>/  # 临时工作区：准备提交 PR 的改动
├── opencode/           # 子项目 - OpenCode 核心代码库（主库，仅同步）
│   ├── origin -> https://github.com/zztdandan/opencode.git (你的 fork)
│   └── upstream -> https://github.com/anomalyco/opencode.git (主库)
├── docs/               # 项目文档
│   ├── architecture/   # 架构文档
│   ├── decisions/      # 技术决策记录 (ADRs)
│   └── guides/         # 操作指南
├── agents/             # Agent 配置文件
│   ├── prompts/        # 提示词模板
│   └── configs/        # Agent 行为配置
├── logs/               # 运行日志和调试信息
├── temp/               # 临时文件
└── scripts/            # 辅助脚本
```

## Git Topology & Collaboration Rules

### 2.1 超级项目 (opencode-refactor)

```bash
# 远端配置
origin  https://github.com/zztdandan/opencode-refactor.git (私有远端 - 仅用于私有提交)
```

- 当前分支：`main`
- `origin`（GitHub）仅用于私有提交
- 日常 upstream/fetch/fork 同步，不依赖私有远端

### 2.2 子项目 (opencode)

```bash
# 远端配置
origin   https://github.com/zztdandan/opencode.git      (你的 fork - 用于推送)
upstream https://github.com/anomalyco/opencode.git      (主库 - 用于同步)
```

**分支说明**：
- `origin/main`：你的 fork 的主分支，用于推送个人改动
- `upstream/main`：官方主库的主分支，用于同步最新代码

**工作流程:**
1. 所有代码修改提交到 `origin` (你的 fork)
2. 通过 Pull Request 从 fork 合并到 upstream (主库), PR 由用户手动提出
3. 定期同步 upstream 到本地

```bash
# 同步上游变更（在主库 opencode/ 下操作）
cd /path/to/opencode-refactor/opencode
git fetch upstream
git checkout main
git merge upstream/main
```

### 2.3 Worktree 策略（核心开发模式）

采用 **Git Worktree 多工作区开发模式**，严格分离"同步库"与"开发库"：

```
opencode-refactor/
├── opencode/                   # 主工作区：仅用于同步，永不修改
│   ├── origin -> https://github.com/zztdandan/opencode.git (你的 fork)
│   ├── upstream -> https://github.com/anomalyco/opencode.git (主库)
│   └── 永远保持 main 分支与 upstream 一致
└── .worktrees/
    ├── <feature-1>/            # 临时工作区：特定功能开发
    │   └── 基于 main 或特定分支创建
    ├── <bugfix-xxx>/           # 临时工作区：修复官方 bug
    └── <pr-contribution>/      # 临时工作区：准备提交 PR 的改动
```

**各工作区能力分配与修改模式**：

| 工作区 | 路径 | 用途 | 修改策略 |
|--------|------|------|----------|
| **主库** | `opencode/` | 仅用于与 GitHub fork/官方同步 | 🚫 **禁止修改** - 永远维持 main 分支一致性，不做任何改动 |
| **临时 worktree** | `.worktrees/<名称>/` | 特定任务开发 | ✅ 根据功能需要创建，完成后删除 |

**工作流规则**：

1. **主库永不修改**：`opencode/` 目录仅作为同步锚点，任何情况下不直接修改代码
2. **开发在临时 worktree**：所有代码编写、调试、验证在按需创建的临时 worktree 中进行
3. **功能导向命名**：worktree 根据具体功能/任务命名（如 `acp-stdio-fix`、`feature-x`），不预先创建固定区域
4. **任务完成即清理**：功能开发完成、PR 合并后，删除对应的临时 worktree
5. **贡献官方流程**：
   - 需要开发功能 → 在 `opencode/` 下基于 main 新建临时 worktree
   - 完成开发 → 提交到 origin (个人 fork) → 发起 PR 到 upstream (官方)
   - 合并后删除临时 worktree → 同步主库

**Worktree 常用命令**：

```bash
# 创建功能 worktree
cd /path/to/opencode-refactor/opencode
git worktree add ../.worktrees/acp-stdio-fix -b fix/acp-stdio

# 创建 feature worktree
git worktree add ../.worktrees/feature-x -b feature/x main

# 列出所有 worktrees
git worktree list

# 删除 worktree（完成后清理）
git worktree remove ../.worktrees/acp-stdio-fix
```

## 快速开始

### 首次设置

1. **克隆主仓库**：
```bash
git clone https://github.com/zztdandan/opencode-refactor.git
cd opencode-refactor
```

2. **初始化 opencode 子模块**（主库 - 仅同步）：
```bash
cd opencode
# 确认远端配置正确
git remote -v
# 应显示：
# origin    https://github.com/zztdandan/opencode.git (你的 fork)
# upstream  https://github.com/anomalyco/opencode.git (主库)

# 如果缺少 upstream，添加它：
git remote add upstream https://github.com/anomalyco/opencode.git

# 同步到最新
git fetch upstream
git checkout main
git merge upstream/main
```

3. **创建临时 worktree**（根据需要）：
```bash
cd /path/to/opencode-refactor/opencode
# 创建功能开发 worktree（注意相对路径）
git worktree add ../.worktrees/my-feature -b feature/my-feature main
```

4. **安装依赖**（在 worktree 中）：
```bash
cd /path/to/opencode-refactor/.worktrees/my-feature
# 根据项目类型执行相应安装命令（如 bun install）
```

### 日常工作流程

1. **同步主库**（仅更新，不修改）：
```bash
cd /path/to/opencode-refactor/opencode
git fetch upstream
git merge upstream/main
```

2. **创建功能 worktree**（根据需要）：
```bash
cd /path/to/opencode-refactor/opencode
git worktree add ../.worktrees/my-feature -b feature/my-feature main
```

3. **开发工作**（在 worktree 中）：
```bash
cd /path/to/opencode-refactor/.worktrees/my-feature
# 进行代码修改、调试、验证
```

4. **提交并推送**（到个人 fork）：
```bash
git add .
git commit -m "feat: your feature description"
git push origin feature/my-feature  # 推送到你的 fork
```

5. **贡献官方**（通过 PR）：
   - 在 GitHub 上从 `zztdandan/opencode` 发起 PR 到 `anomalyco/opencode`

6. **清理 worktree**（PR 合并后）：
```bash
cd /path/to/opencode-refactor/opencode
git worktree remove ../.worktrees/my-feature
# 同步主库到最新
git fetch upstream
git merge upstream/main
```

## 相关链接

- [主库](https://github.com/anomalyco/opencode.git)
- [Fork库](https://github.com/zztdandan/opencode.git)
- [维护库](https://github.com/zztdandan/opencode-refactor.git)

## 子模块管理注意事项

### Git 配置

**`.gitignore` 配置**：

项目根目录的 `.gitignore` 已配置忽略：
- `.worktrees/` - 所有 worktree 工作区
- `temp/` - 临时文件
- `logs/` - 日志文件
- `.vscode/`, `.idea/` - IDE 配置

**重要**：`opencode/` 子模块的 `.gitignore` 由子模块自己管理，不会被主仓库忽略。

### 子模块状态

本项目使用 Git Submodule 管理 `opencode` 子项目。请注意：

1. **子模块指向 commit，而非分支**
   - 主仓库记录的是子模块的特定 commit SHA
   - 更新子模块时需要明确提交新的 commit SHA

2. **更新子模块到最新 upstream**
```bash
cd opencode
git fetch upstream
git merge upstream/main
cd ..  # 回到主仓库
git add opencode
git commit -m "chore: update opencode submodule to latest"
```

3. **Worktree 中的子模块**
   - 每个 worktree 都共享同一个 `.git` 目录
   - 子模块在各 worktree 中独立管理
   - 开发时在 worktree 中更新子模块即可

4. **避免在主库 `opencode/` 中直接修改**
   - 严格遵循 Worktree 策略
   - 主库仅用于同步和查看官方最新状态

## 项目特定说明

### ACP 集成开发

当前的 ACP（Agent Client Protocol）相关工作：

- ACP stdio 传输优化
- Python SDK 互操作性改进
- 大型消息 truncation 处理

开发时在 `.worktrees` 下创建相应的功能 worktree 进行。

### PR 管理流程

1. 在 `opencode/` 子模块下创建功能 worktree
2. 在 `.worktrees/<名称>/` 中完成开发
3. 提交到 origin (个人 fork: `zztdandan/opencode`)
4. 在 GitHub 上创建 PR 到 upstream (官方: `anomalyco/opencode`)
5. 等待 review 和合并
6. 合并后清理 worktree 并同步主库：

```bash
cd /path/to/opencode-refactor/opencode
git worktree remove ../.worktrees/my-feature
git fetch upstream
git merge upstream/main
```
