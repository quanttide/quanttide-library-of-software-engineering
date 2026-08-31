# Vicoa 资料

Vicoa 是开源的 AI 编排器，用于在任意设备上运行一支编程智能体（coding agent）团队。本文记录其基本资料与参考资料。

## 基本资料

- 项目名称：Vicoa
- 定位：AI 编排器，统一调度多个编程智能体
- 许可证：AGPLv3
- 官网：[vicoa.ai](https://vicoa.ai)
- 代码仓库：[github.com/vicoa-ai/vicoa](https://github.com/vicoa-ai/vicoa)
- 社区：[Discord](https://discord.gg/mqz4qRPV4j)、[X](https://x.com/vicoaai)

## 关键特性

Vicoa 在同一工作区中并行运行多个编程智能体，支持 Claude Code、Codex、OpenCode、Gemini、Cursor、GitHub Copilot、Kimi、Hermes 等。每个智能体使用独立的 git worktree 与分支，可在同一仓库并行工作互不干扰。

Vicoa 支持多端协同，桌面端、CLI 与 iOS/Android 移动端同步同一会话，可从手机发起或继续编程智能体会话，并接收决策与完成通知。它提供内置的 git 差异审查、文件浏览器、终端、任务看板、自动化调度（cron）与技能管理等功能。

## 参考资料

- [README（英文）](https://github.com/vicoa-ai/vicoa/blob/main/README.md)
- [README（中文）](https://github.com/vicoa-ai/vicoa/blob/main/README.zh.md)
- [Vicoa 文档](https://vicoa.ai/docs)
- [自托管指南](https://github.com/vicoa-ai/vicoa/blob/main/SELF_HOSTING.md)
