# 面试题库总目录

> **基础库已建成**(2026-08-19):**12 组 74 题**,覆盖 agent 开发、AI 编码工具与大模型工程方向的核心考点。
> 每题讲透五件套(本质/机制/实例/边界/检验),答案分 **30 秒版**(电梯答案)与 **深入版**(展示深度),每题附 **追问预判**。
> 用法:先自己作答 → 对照深入版找差距 → 重点背追问预判。
> 维护方式:此后**不再新增题组**,由每日运行在 P4 做增量维护——新事实改变答案则修订并标 as-of;值得成为新考点则在既有组追加 1 题;仅为素材则补进实例与追问弹药(协议 v1.2)。

## 题组

| # | 题组 | 覆盖考点 | 题数 |
|---|---|---|---|
| 01 | [Agent 基础](01-agent-fundamentals.md) | agent/workflow 判据、agent loop 数据流、tool use 底层机制、ReAct 的内化、长任务上下文四层方案、设计范式全景与 generator-verifier gap | 6 |
| 02 | [上下文工程](02-context-engineering.md) | 注意力退化机制、compaction 保留策略、prompt caching 回本模型、agentic search vs RAG、结构化输出、上下文隔离权衡、Prompt 工程 2026 现状 | 7 |
| 03 | [MCP 与协议](03-mcp-protocols.md) | MCP 架构与原语、2026-07-28 stateless 改版与迁移、MCP/A2A/Agent Plugins 分工、安全风险、transport 选型、server 设计 | 6 |
| 04 | [多智能体](04-multi-agent.md) | 单/多 agent 判据、四种编排模式、Anthropic 失败模式研究与对策、subagent 隔离契约、成本延迟模型、handoff 防循环 | 6 |
| 05 | [记忆系统](05-memory.md) | 记忆分层与上下文的关系、文件/向量/图式三路线取舍、写入策略防污染、检索失败模式、持久化与断点续跑 | 5 |
| 06 | [评估与可观测](06-evals-observability.md) | 轨迹级 vs 结果级评估、LLM-as-judge 偏差与校准、基准饱和与污染、trace/span 与生产指标、从零搭 eval 闭环 | 5 |
| 07 | [安全与注入](07-security-injection.md) | 注入机制三类、lethal trifecta 与分层防御、Wiz-Snowflake 事件复盘、权限沙箱与 HITL、供应链治理、端到端安全设计 | 6 |
| 08 | [生产化与可靠性](08-production.md) | agent 重试幂等、durable execution、成本延迟优化与缓存回本、SLO 与降级、prompt/协议发布工程 | 5 |
| 09 | [框架与选型](09-frameworks.md) | 四大框架抽象对比、何时不用框架、编排下沉的影响、durable execution 边界、厂商绑定评估、LangChain Runnable/LCEL、LangGraph 建图与 checkpointer/HIL | 7 |
| 10 | [编码代理实战](10-coding-agents.md) | harness 构成、CLAUDE.md/AGENTS.md 注入机制、验证闭环四档、fan-out 与并行开发、三家工具对比、fresh context 评审 | 6 |
| 11 | [RAG 检索增强](11-rag.md) | 流程与漏斗式误差归因、chunking 策略、embedding 与向量检索、HNSW/IVF 索引原理、混合检索与 rerank、评估与幻觉、RAG vs 长上下文 vs agentic search | 7 |
| 12 | [大模型工程](12-llm-engineering.md) | Transformer 与 attention、KV Cache 与 MQA/GQA/MLA、Flash Attention、TTFT/TPOT 与 prefill-decode、continuous batching 与 PagedAttention、量化、LoRA/QLoRA、RLHF vs DPO | 8 |

**合计 74 题**。配套实战技能手册见 [skills/](../skills/_index.md)。

> **覆盖面说明**:01–10 组是 agent 应用工程与 2026 前沿(本库的原生优势);11–12 组补齐国内 AI 岗位面试的基本盘(RAG 与大模型底层工程),两块合起来才是完整的准备面。

## 面试观(为什么这样出题)

2026 年的 agent 岗位面试,考察重心已从"会调 API"转向:

1. **机制理解**:tool use 底层怎么工作、上下文为什么退化、CLAUDE.md 为什么是软约束——能从第一性原理推导,而不是背名词
2. **工程判断**:何时单 agent 何时多 agent、何时上框架何时裸写 loop、缓存什么时候反而亏——知道边界比知道方法更重要
3. **一手时效**:MCP 2026-07-28 的 stateless 改版与 sampling/logging 弃用、多智能体失败模式研究、基准饱和——跟没跟前沿一问便知
4. **工具实战**:Claude Code / Codex 用得深不深,直接反映日常生产力(vibe coding 已是岗位默认技能)

## 答题结构建议

- **先给判据,再给细节**:面试官要的是"你怎么决策",不是术语背诵
- **主动标 as-of**:讲版本敏感事实时说明时间口径("截至 2026-08…"),显示你知道它会变
- **主动说边界**:"这个方案在 X 情况下不成立"——比滔滔不绝更能体现深度
- **不确定就说不确定**:编造具体数字是最快的出局方式
