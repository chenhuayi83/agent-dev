---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# Claude Code

终端/IDE/云端的编码 agent:harness 结构(系统提示/工具集/hooks/skills/subagents)、CLAUDE.md 机制、权限模型、无头与 SDK 化。

## 原理(稳定)

(待充实)

## 现状(易变,断言须带 as-of 日期)

as-of 2026-08-19:
- 市场位置:多家 2026-07 口径评为编码代理第一,凭 Opus 5 + per-subagent model control;46% 开发者首选(社区调查,二手)([redlinesoft](https://blog.redlinesoft.net/posts/agentic-coding-agent-comparison-2026/))
- 形态:CLI + 桌面 + Web + IDE 插件;spring 2026 插件市场一等公民化
- 近两周 CHANGELOG(2.1.231→2.1.235,as-of 08-19):**subagent forking 默认开启**(fork 继承完整会话与 cache)、`@` 提及其他会话(SendMessage 跨会话协作)、GitLab MR 支持、用量恢复自动续跑、spellcheck([CHANGELOG](https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md))
- 解读:多会话/多 agent 协作原语在 harness 层快速固化,与"编排下沉"趋势一致
- 实战技能手册(用法向,与本文的观察向互补):[skills/claude-code/](../../skills/claude-code/01-core-workflow.md)

## 时间线

- 2026-08(2.1.232)subagent forking 默认开启;跨会话 @ 提及
- 2026-07-06 Anthropic 发布《The Making of Claude Code》(harness 设计一手资料,待深研)

## 开放问题

- Claude Code 近期版本的关键演进(看 CHANGELOG)?
- skills 生态的现状?
