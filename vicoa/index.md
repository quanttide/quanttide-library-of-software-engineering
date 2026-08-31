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

## 源码架构

仓库约 28 万行，核心在 `backend/`（Python）：包含云端 API 与实时服务器、CLI 与本地 daemon、各智能体的无头运行适配器。`apps/web` 为 Next.js 仪表盘，`apps/desktop` 为 Electron 桌面壳（内置 daemon），`apps/mobile` 为 Flutter 移动端。

运行链路为：客户端 → 云端 FastAPI 与 WebSocket → 本地 `vicoa daemon` → 拉起各智能体 CLI。智能体接入平台有两种方式：

| 接入方式 | 适用智能体 | 实现 |
|---------|-----------|------|
| 原生 SDK | Claude Code、Codex | Claude 直连 Agent SDK，Codex 直连其 app-server |
| ACP 协议 | OpenCode、Cursor、Gemini、Copilot、Kimi | 统一封装 Agent Client Protocol，会话、权限、中断等通用逻辑复用 |

daemon 以硬件标识与 API key 确定性派生机器 ID，注册后经 REST 轮询与 WebSocket 双通道接收任务。实时协议中，连接先发 hello 帧声明作用域，按机器、会话、用户、终端四类通道隔离数据。每个智能体以独立 git worktree 与分支并行工作，互不干扰。智能体侧通过 MCP 工具（log_step、ask_question、end_session）与平台交互，自动化调度依赖 cron 任务。

## 参考资料

- [README（英文）](https://github.com/vicoa-ai/vicoa/blob/main/README.md)
- [README（中文）](https://github.com/vicoa-ai/vicoa/blob/main/README.zh.md)
- [Vicoa 文档](https://vicoa.ai/docs)
- [自托管指南](https://github.com/vicoa-ai/vicoa/blob/main/SELF_HOSTING.md)
