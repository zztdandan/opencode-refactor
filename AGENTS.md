# OpenCode Agent 维护库

本仓库是 OpenCode 项目的 Agent 专用维护库，用于管理文档、配置、日志和临时文件。

## 项目结构

```
opencode-refactor/
├── AGENTS.md           # 本文件 - 项目总览和Agent配置
├── README.md           # 项目说明文档
├── opencode/           # 子项目 - OpenCode 核心代码库
│   ├── origin -> https://github.com/zztdandan/opencode.git (你的fork)
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

## Git 工作流

### 主项目 (opencode-refactor)

```bash
# 远端配置
origin  https://github.com/zztdandan/opencode-refactor.git (主远端)
```

### 子项目 (opencode)

```bash
# 远端配置
origin   https://github.com/zztdandan/opencode.git      (你的fork - 默认推送)
upstream https://github.com/anomalyco/opencode.git      (主库 - 用于同步)
```

**工作流程:**
1. 所有代码修改提交到 `origin` (你的fork)
2. 通过 Pull Request 从 fork 合并到 upstream (主库)
3. 定期同步 upstream 到本地

```bash
# 同步上游变更
git fetch upstream
git checkout main
git merge upstream/main
```

## Agent 配置

### 核心代理

| Agent | 职责 | 触发条件 |
|-------|------|----------|
| Explore | 代码库探索，模式发现 | "Find X", "How does Y work" |
| Librarian | 外部文档查询 | 不熟悉库/框架时使用 |
| Oracle | 架构咨询，复杂决策 | 重大架构变更 |
| Metis | 需求澄清 | 任务不明确时 |
| Momus | 计划审查 | 工作计划完成后 |

### 文件命名规范

- Agent 配置: `agents/configs/{agent-name}.md`
- 提示词模板: `agents/prompts/{task-type}.md`
- 决策记录: `docs/decisions/YYYY-MM-DD-{decision-title}.md`
- 架构文档: `docs/architecture/{component}.md`

## 快速开始

1. 初始化子模块:
```bash
cd opencode
git fetch upstream
git checkout main
```

2. 安装依赖 (根据 opencode 项目要求):
```bash
cd opencode
# 根据项目类型执行相应安装命令
```

## 相关链接

- [主库](https://github.com/anomalyco/opencode.git)
- [Fork库](https://github.com/zztdandan/opencode.git)
- [维护库](https://github.com/zztdandan/opencode-refactor.git)
