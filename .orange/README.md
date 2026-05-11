# DeerFlow 项目分析文档

本目录包含 DeerFlow 项目的详细分析文档。

## 文档目录

| 文件 | 内容 |
|------|------|
| [01-产品介绍.md](./01-产品介绍.md) | 项目背景、定位、目标用户 |
| [02-功能结构.md](./02-功能结构.md) | 核心功能模块详解 |
| [03-技术架构.md](./03-技术架构.md) | 技术栈、整体架构设计 |
| [04-代码结构.md](./04-代码结构.md) | 目录结构、核心文件说明 |
| [05-数据流.md](./05-数据流.md) | 请求流转、消息流、文件流 |
| [06-Agent设计.md](./06-Agent设计.md) | Middleware 链、子 Agent、记忆系统 |
| [07-部署运行.md](./07-部署运行.md) | 配置、部署方式、安全注意事项 |
| [08-记忆架构.md](./08-记忆架构.md) | 三项目记忆形态对比：Facts JSON vs MEMORY.md vs Dream 两阶段 |
| [09-多轮会话管理.md](./09-多轮会话管理.md) | LangGraph Checkpointer vs SQLite WAL vs JSON 文件会话 |
| [10-Lead-Sub-Agent.md](./10-Lead-Sub-Agent.md) | task 工具 / delegate_task / spawn 三种子 Agent 派生方式 |
| [11-AgentLoop.md](./11-AgentLoop.md) | 中间件链 vs 手写 while 循环 vs 异步事件总线 |
| [12-Thinking模式.md](./12-Thinking模式.md) | Extended/Adaptive Thinking 在三项目中的实现差异 |
| [13-SKILL管理.md](./13-SKILL管理.md) | SKILL.md 格式、加载策略、进化机制横向对比 |
| [14-如何评测.md](./14-如何评测.md) | Terminal-Bench 2 / 组件测试 / 基础测试三级评测体系 |

## 项目一句话

**DeerFlow** 是字节跳动开源的 Super Agent Harness（超级 Agent 运行时），基于 LangGraph，具备中间件驱动架构、沙箱执行、Skills 系统、长期记忆和 IM 渠道接入能力。
