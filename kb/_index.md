# 知识库总目录

> 每文件一行:简介 + 成熟度(stub 骨架 / growing 生长中 / mature 成熟)。P4 沉淀后更新本表。


## 00-fundamentals 基础概念

- [Agent 的定义与边界](00-fundamentals/agent-definitions.md) — 什么是 agent(stub)
- [Tool Use 工具调用](00-fundamentals/tool-use.md) — 工具调用是 agent 的手脚(stub)
- [规划与推理](00-fundamentals/planning-reasoning.md) — agent 如何决定做什么(growing)

## 01-context-engineering 上下文工程

- [上下文工程总览](01-context-engineering/overview.md) — 上下文是 agent 最稀缺的资源(growing)
- [Compaction 与摘要](01-context-engineering/compaction-summarization.md) — 长会话的生存机制(stub)
- [检索与 Agentic Search](01-context-engineering/retrieval-rag.md) — 从静态 RAG 到 agentic search(stub)
- [结构化输出](01-context-engineering/structured-outputs.md) — 让模型可靠输出 JSON/schema(stub)

## 02-frameworks 框架

- [Claude Agent SDK](02-frameworks/claude-agent-sdk.md) — Anthropic 官方 agent 框架,Claude Code 同源内核(stub)
- [OpenAI Agents SDK](02-frameworks/openai-agents-sdk.md) — OpenAI 官方 agent 框架(Swarm 后继)(stub)
- [LangGraph](02-frameworks/langgraph.md) — 图编排框架(growing)
- [Google ADK](02-frameworks/google-adk.md) — Google Agent Development Kit(growing)
- [其他框架盘点](02-frameworks/other-frameworks.md) — CrewAI(角色协作)、AG2/AutoGen(对话式多 agent)、smolagents(代码即行动)、PydanticAI(类型安全)、Vercel AI SDK(前端全栈)等的定位与活跃度(stub)
- [框架选型](02-frameworks/framework-selection.md) — 选型决策树(stub)

## 03-protocols 协议

- [MCP(Model Context Protocol)](03-protocols/mcp.md) — 工具/资源接入的事实标准(growing)
- [A2A(Agent2Agent)](03-protocols/a2a.md) — agent 间互操作协议(Google 发起,Linux Foundation 托管)(growing)
- [新兴协议](03-protocols/emerging-protocols.md) — 支付(AP2、x402)、身份、发现(AGNTCY)、站点声明(llms.txt、agents.md)等围绕 agent 互操作的新协议观察哨(stub)

## 04-multi-agent 多智能体

- [多智能体编排模式](04-multi-agent/orchestration-patterns.md) — supervisor(中心调度)、swarm/handoff(去中心)、pipeline(流水线)、debate/panel(对抗评审)诸模式的机制、适用与失败模式(growing)
- [Subagents 子代理](04-multi-agent/subagents.md) — 上下文隔离的核心手段(stub)
- [Handoffs 与通信](04-multi-agent/handoffs-communication.md) — agent 间转交与消息(stub)

## 05-memory 记忆

- [记忆架构](05-memory/memory-architectures.md) — 短期(会话内)/长期(跨会话)记忆分层;文件式、向量式、图式(时序知识图谱)记忆的机制与取舍;写入策略(何时记、记什么)(growing)
- [记忆工具与产品](05-memory/memory-tools.md) — Mem0、Zep(Graphiti)、Letta(MemGPT)、LangMem 及厂商内置 memory(OpenAI/Anthropic/Google)的能力对比与集成方式(stub)
- [状态持久化](05-memory/state-persistence.md) — agent 状态的落盘与恢复(stub)

## 06-evals 评估

- [Agent 评估方法](06-evals/agent-evals.md) — 轨迹级评估(trajectory)、结果级评估、LLM-as-judge 的偏差与校准、在线 A/B、eval 驱动开发闭环(stub)
- [基准测试](06-evals/benchmarks.md) — SWE-bench(Verified/Pro)、GAIA、tau-bench、terminal-bench、OSWorld 等 agent 基准的构成、饱和度与污染问题(growing)
- [可观测性与 Tracing](06-evals/observability-tracing.md) — trace/span 模型、LangSmith/Langfuse/Braintrust/W&B Weave 等平台、OpenTelemetry GenAI 语义约定、生产监控指标设计(stub)

## 07-safety-security 安全

- [Prompt Injection 攻防](07-safety-security/prompt-injection.md) — agent 最大的安全威胁(growing)
- [沙箱与隔离](07-safety-security/sandboxing.md) — agent 执行环境隔离(growing)
- [权限与 Human-in-the-Loop](07-safety-security/permissions-hitl.md) — 最小权限、行动分级(只读/可逆/不可逆)、审批流设计、durable authorization 的边界(stub)

## 08-production 生产化

- [部署模式](08-production/deployment-patterns.md) — agent 的运行形态(stub)
- [可靠性工程](08-production/reliability.md) — 重试与幂等、超时与熔断、durable execution、失败恢复(checkpoint 续跑)、优雅降级(stub)
- [成本与延迟优化](08-production/cost-latency.md) — prompt caching、模型分级(路由)、batch API、并行化、token 预算护栏;成本可观测(growing)

## 09-coding-agents 编码代理

- [Claude Code](09-coding-agents/claude-code.md) — 终端/IDE/云端的编码 agent(growing)
- [其他编码代理](09-coding-agents/other-coding-agents.md) — Codex(CLI/云)、Cursor(IDE+background)、Copilot(agent mode)、Gemini CLI、OpenHands、Devin、Amp 等的形态与差异化(growing)
- [自主软件工程](09-coding-agents/autonomous-swe.md) — long-running/background agents(stub)
- [Harness 设计](09-coding-agents/harness-design.md) — 编码 agent 的"机箱"设计学(stub)

## 10-landscape 全景

- [大事年表](10-landscape/timeline.md) — agent 领域大事记,按月倒序,每条一行(日期+事实+来源)(growing)
- [模型能力快照](10-landscape/model-capabilities.md) — 主流模型的 agent 相关能力对比(as-of 快照,过时即整段重写)(growing)
- [生态地图](10-landscape/ecosystem-map.md) — agent 生态分层地图(growing)
- [研究前沿](10-landscape/research-frontiers.md) — 值得跟踪的研究方向(growing)
