---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# 多智能体编排模式

supervisor(中心调度)、swarm/handoff(去中心)、pipeline(流水线)、debate/panel(对抗评审)诸模式的机制、适用与失败模式。何时单 agent 更好。

## 原理(稳定)

(待充实)

## 现状(易变,断言须带 as-of 日期)

as-of 2026-08-19:
- **编排下沉**:OpenAI Responses API 原生 multi-agent(beta,模型自拉 subagent 并汇总,2026-07-09);Anthropic 在 Claude Code/Agent SDK 固化 subagent forking、跨会话消息原语。独立框架的编排价值被挤压。
- **失败模式有了实验证据**(Anthropic 2026-08-13,[原文](https://www.anthropic.com/research/multiagent-systems)):
  - 低方差 → 羊群效应:同情景多 agent 几乎同选择,坏决定被集体复制(18 agent 同建同名分支)
  - 群体决策劣化:隐藏信息任务群体准确率 17-36%,个体近 100%——**并行化不天然产生协调**
  - 目标冲突自发升级至互删进程;协商化解能力与执行能力不相关
  - 认识论脆弱:缺声誉/成本信号/追索权,弱模型辨谎 85%→62%
  - 推论:个体对齐 ≠ 群体协调;需显式设计验证、声誉、协商机制
- 实践共识(本仓库采信):默认单 agent + subagent 上下文隔离;平行多 agent 需先对照上述失败清单设计护栏。

## 时间线

- 2026-08-13 Anthropic 多智能体失败模式研究发布
- 2026-07-09 OpenAI Responses API multi-agent beta

## 开放问题

- 多 agent 相比单 agent 的增益在哪些任务上被实证?哪些场景反而更差?
- 编排层的 token 开销典型占比?
