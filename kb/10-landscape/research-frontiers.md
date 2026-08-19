---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# 研究前沿

值得跟踪的研究方向:agent RL 训练、self-improvement、世界模型、多 agent 涌现、形式化安全。论文向观察哨。

## 原理(稳定)

收录标准:方向性研究(非单篇灌水),且对 agent 工程实践有 12 个月内的潜在影响。

## 现状(as-of 2026-08-19)

- **多智能体系统性失败模式**:Anthropic 08-13 研究给出实验证据——低方差羊群效应、群体决策劣化(17-36% vs 个体近 100%)、冲突自发升级、认识论脆弱(辨谎 85%→62%);结论"个体对齐 ≠ 群体协调",提出需要声誉/验证/协商等社会机制([原文](https://www.anthropic.com/research/multiagent-systems))。基础设施(Responses API multi-agent)跑在失败模式研究前面,落差即风险。
- **评估器可训练化**:LangSmith Tuned Evaluators(08-18)把 LLM-as-judge 从"提示词工程"推进到"按团队标准微调"——评估的评估问题(judge 对齐)进入产品化([LangChain](https://www.langchain.com/blog/introducing-langsmith-tuned-evaluators-starting-with-perceived-error))。
- **agent 互操作治理**:arXiv 出现对 MCP/A2A/ACP 表达力边界的分析(如 [Governance Gaps in Agent Interoperability Protocols](https://arxiv.org/pdf/2606.31498)),协议研究从"怎么连"转向"治理表达力"。
- agentic RL / self-improvement:本次基线未覆盖,待专题扫描。

## 时间线

(事件统一记在 [timeline.md](timeline.md))

## 开放问题

- agentic RL(端到端训练 agent)的最新进展?
- 多 agent 失败模式研究有无后续复现/反驳?
