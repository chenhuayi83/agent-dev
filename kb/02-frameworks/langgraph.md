---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# LangGraph

图编排框架:节点/边/状态机模型,checkpointing、human-in-the-loop、durable execution,LangSmith 配套。企业采用最广的编排层之一。

## 原理(稳定)

(待充实)

## 现状(易变,断言须带 as-of 日期)

as-of 2026-08-19:
- 1.0 GA(2025-10-22,与 LangChain 1.0 同日):核心图原语(state/nodes/edges)零 breaking change,semver 承诺到 2.0;`langgraph.prebuilt` 弃用、能力并入 `langchain.agents`([changelog](https://changelog.langchain.com/announcements/langgraph-1-0-is-now-generally-available));注意 1.0.2 等 patch 曾在子包引入意外 breaking([issue #6363](https://github.com/langchain-ai/langgraph/issues/6363))
- 生产背书:Uber/LinkedIn/Klarna 等(官方口径)
- 压力面:编排能力下沉到 API/harness 层后,LangGraph 的独特价值集中在 durable execution、checkpointing、LangSmith 观测闭环与企业治理

## 时间线

- 2025-10-22 LangGraph 1.0 GA

## 开放问题

- LangGraph 1.0 之后 API 稳定性与向后兼容如何?
- graph 式 vs 代码式(纯 loop)编排的选型边界?
