# 待研队列

> 取题规则:每天取"待研"区最高优先级(P1 最高);用户点名主题优先。完成后移入"已完成"区并链接产出。
> 队列少于 5 题时,从各 kb 文件"开放问题"小节补队。

## 待研

| P | 主题 | 为什么 | 预计实验 | 入队日期 |
|---|---|---|---|---|
| 2 | Claude Agent SDK 深研:架构 / agent loop / 与 Claude Code 的关系 | 本系统运行其上,吃狗粮优先 | 是:最小 agent + 自定义 tool | 2026-08-19 |
| 3 | MCP 2026 现状:spec 演进(重点 2026-07-28 stateless 迁移)/ 生态 / 最佳实践 | 事实上的工具接入标准;新 spec 刚落地 | 是:最小 MCP server | 2026-08-19 |
| 4 | Agent Plugins 标准深读:manifest 结构 / 与 Claude Code plugins 关系 / 站队预判 | 跨厂商能力分发标准,08-11 刚出 v1.0.0 草案,窗口期 | 否 | 2026-08-19 |
| 4.5 | 上下文工程系统化:compaction / isolation / cache 策略 | agent 质量的第一性瓶颈 | 否 | 2026-08-19 |
| 5 | 多智能体编排模式对比:supervisor / swarm / pipeline | 判断何时多 agent 反而更差 | 否 | 2026-08-19 |
| 6 | Agent 记忆系统盘点:Mem0 / Zep / Letta / 厂商内置 memory | 长期运行 agent 的核心能力 | 否 | 2026-08-19 |
| 7 | Agent 评估工程:trajectory evals / LLM-as-judge 陷阱 | 没有评估就没有迭代 | 否 | 2026-08-19 |
| 8 | Prompt injection 与 agent 安全边界 | 与本仓库自身安全直接相关(宪法 C8) | 否 | 2026-08-19 |
| 9 | OpenAI Agents SDK vs LangGraph vs Google ADK 同任务实测 | 选型需要一手对比,不信二手评测 | 是:三框架同任务 | 2026-08-19 |
| 10 | 编码代理前沿:long-running / background agents / harness 设计 | 业界演进最快的 agent 形态 | 否 | 2026-08-19 |

## 已完成

| 完成日期 | 主题 | 产出 |
|---|---|---|
| 2026-08-19 | 2026-08 Agent 开发全景基线扫描 | [deep-dives/2026/2026-08-19-landscape-baseline.md](../deep-dives/2026/2026-08-19-landscape-baseline.md) |
