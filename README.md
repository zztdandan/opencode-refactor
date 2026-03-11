# OpenCode Refactor 维护库

本仓库是 OpenCode 项目的 Agent 专用维护库，用于管理文档、配置、日志和临时文件。

## 仓库结构

```
opencode-refactor/
├── AGENTS.md           # Agent 配置和项目说明
├── README.md           # 本文件
├── opencode/           # OpenCode 核心代码库 (子项目)
├── docs/               # 项目文档
│   ├── architecture/   # 架构文档
│   ├── decisions/      # 技术决策记录
│   └── guides/         # 操作指南
├── agents/             # Agent 配置文件
│   ├── prompts/        # 提示词模板
│   └── configs/        # Agent 行为配置
├── logs/               # 运行日志
├── temp/               # 临时文件
└── scripts/            # 辅助脚本
```

## Git 工作流

### 主项目配置

```bash
origin  https://github.com/zztdandan/opencode-refactor.git
```

### 子项目 (opencode) 配置

```bash
origin   https://github.com/zztdandan/opencode.git      (你的fork)
upstream https://github.com/anomalyco/opencode.git      (主库)
```

## 工作流程

1. 在 `opencode/` 目录进行代码开发
2. 提交推送到 `origin` (你的fork)
3. 通过 Pull Request 合并到主库

## 同步上游变更

```bash
cd opencode
git fetch upstream
git checkout main
git merge upstream/main
```

## 相关链接

- [OpenCode 主库](https://github.com/anomalyco/opencode)
- [OpenCode Fork](https://github.com/zztdandan/opencode)
