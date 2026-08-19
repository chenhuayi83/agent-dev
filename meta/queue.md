# 待研队列

> 取题规则:每天取"待研"区最高优先级(P1 最高);用户点名主题优先。完成后移入"已完成"区并链接产出。
> 类型:前沿(deep-dives/)/ 技能(skills/)。周元复盘检查配比:近 7 个 P2 深内容中 技能 ≥3、前沿 ≥2。
> **面试题不再入队**:基础库 01–10 已于 2026-08-19 建成,日常在 P4 做增量维护(协议 v1.2)。
> 队列少于 5 题时,从各 kb 文件"开放问题"小节补队。

## 待研

| P | 类型 | 主题 | 为什么 | 预计实验 | 入队日期 |
|---|---|---|---|---|---|
| 1 | 技能 | Codex 上手与 Claude Code 对照:CLI/IDE/cloud 形态、AGENTS.md、profiles/sandbox、skills 与 plugins | 用户双工具目标;Claude Code 侧已有首篇,Codex 侧需一手调研讲透 | 视环境:装 CLI 跑最小任务 | 2026-08-19 |
| 2 | 前沿 | Claude Agent SDK 深研:架构 / agent loop / 与 Claude Code 的关系 | 本系统运行其上,吃狗粮;兼具技能价值 | 是:最小 agent + 自定义 tool | 2026-08-19 |
| 3 | 技能 | CLAUDE.md / AGENTS.md 工程学 + auto memory:写好项目记忆 | 官方单列主题,一次配置长期受益 | 否 | 2026-08-19 |
| 4 | 技能 | 上下文管理实战:/clear /compact /btw /rewind、subagent 隔离调查 | 上下文是第一性约束,官方五大反模式半数与此相关 | 否 | 2026-08-19 |
| 5 | 前沿 | MCP 2026 现状:07-28 stateless 迁移 / 生态 / 最佳实践 | 新 spec 刚落地,迁移窗口一年 | 是:最小 MCP server | 2026-08-19 |
| 6 | 技能 | 验证闭环设计:/goal、Stop hooks、对抗评审 subagent | "让 agent 自己收敛"是无人值守的关键 | 是:hook 示例 | 2026-08-19 |
| 7 | 技能 | subagents / agent teams / 并行 worktree 实战 | 规模化 vibe coding 的核心 | 是 | 2026-08-19 |
| 8 | 前沿 | Agent Plugins 标准深读:manifest / 与 Claude Code plugins 关系 / 站队预判 | 08-11 v1.0.0 草案,窗口期 | 否 | 2026-08-19 |
| 9 | 技能 | headless 与 CI 自动化:claude -p / --allowedTools / fan-out / GitHub Actions | 从工具用户到流水线设计者 | 是 | 2026-08-19 |
| 10 | 前沿 | 上下文工程系统化:compaction / isolation / cache 策略 | agent 质量的第一性瓶颈 | 否 | 2026-08-19 |
| 11 | 前沿 | 多智能体编排模式对比:supervisor / swarm / pipeline | 判断何时多 agent 反而更差 | 否 | 2026-08-19 |
| 12 | 前沿 | Agent 记忆系统盘点:Mem0 / Zep / Letta / 厂商内置 | 长期运行 agent 的核心能力 | 否 | 2026-08-19 |
| 13 | 前沿 | Agent 评估工程:trajectory evals / LLM-as-judge 陷阱 / Tuned Evaluators | 没有评估就没有迭代 | 否 | 2026-08-19 |
| 14 | 前沿 | Prompt injection 与 agent 安全边界:2026 攻防现状 | 与本仓库自身安全直接相关(宪法 C8) | 否 | 2026-08-19 |
| 15 | 前沿 | OpenAI Agents SDK vs LangGraph vs Google ADK 同任务实测 | 选型需要一手对比 | 是:三框架同任务 | 2026-08-19 |
| 16 | 前沿 | 编码代理前沿:long-running / background agents / harness 设计(《Making of Claude Code》待读) | 业界演进最快的 agent 形态 | 否 | 2026-08-19 |

## 已完成

| 完成日期 | 类型 | 主题 | 产出 |
|---|---|---|---|
| 2026-08-19 | 前沿 | 2026-08 Agent 开发全景基线扫描 | [deep-dives/2026/2026-08-19-landscape-baseline.md](../deep-dives/2026/2026-08-19-landscape-baseline.md) |
| 2026-08-19 | 技能 | Claude Code 核心心智模型与高效工作流 | [skills/claude-code/01-core-workflow.md](../skills/claude-code/01-core-workflow.md) |
| 2026-08-19 | 面试题 | **基础库 01–10 十组一次性建成**(此后转增量维护) | [interview/_index.md](../interview/_index.md) |
