---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# 生态地图

agent 生态分层地图:模型层/框架层/协议层/记忆层/评估观测层/安全层/应用层,各层玩家与集中度。

## 原理(稳定)

分层框架本身稳定:**模型 → 协议 → 框架/harness → 记忆/评估/安全(横切)→ 应用**。玩家名单和集中度是易变信息,放"现状"。

## 现状(as-of 2026-08-19)

| 层 | 主要玩家 | 集中度/趋势 |
|---|---|---|
| 模型 | Anthropic Claude 5 / OpenAI GPT-5.6 / Google Gemini 3 / Kimi、Grok 等 | 旗舰能力趋同,价格分档竞争 |
| 协议 | **MCP**(工具接入,07-28 转 stateless)/ **A2A**(agent 互通,LF 治理 150+ 组织)/ **Agent Plugins**(打包分发,08 月新出) | 三协议分工互补,均为开放治理;标准竞争暂告段落 |
| 框架/编排 | LangGraph 1.0、Google ADK、OpenAI Agents SDK、Claude Agent SDK、CrewAI/AG2 等 | **编排能力下沉**:OpenAI 进 API 层、Anthropic 进 harness 层;独立框架价值被挤向 durable execution/企业治理 |
| 托管运行时 | Gemini Enterprise Agent Platform、Anthropic Managed Agents、LangChain 平台 | "managed agents" 叙事 2026-08 升温,自建 loop → 托管迁移开始 |
| 编码代理 | Claude Code(#1)、Codex、Cursor、Copilot Agent Mode、Cline;新秀 Kiro/Muse/Hax 等 | 终端派 vs IDE 派;harness 是主要差异点;插件市场化 |
| 评估/观测 | LangSmith(Tuned Evaluators)、Langfuse、Braintrust、vals.ai、HAL | 评估器开始"可训练化";第三方复测机构影响力上升 |
| 安全 | Wiz 等云安全厂商入场 agent 攻防;OpenAI Daybreak/GPT-5.6-Cyber | AI 攻击 AI 已是现实(Wiz Red Agent);防御产品化早期 |
| 记忆 | Mem0 / Zep / Letta / 厂商内置 | 待深研(队列 P6),本次未扫描 |

## 时间线

(事件统一记在 [timeline.md](timeline.md))

## 开放问题

- 生态整合(收购/停更)动态?
- 记忆层与安全层的格局需要专题扫描补齐(队列 P6/P8)
