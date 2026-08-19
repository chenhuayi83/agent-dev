---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# Claude Code

终端/IDE/云端的编码 agent:harness 结构(系统提示/工具集/hooks/skills/subagents)、CLAUDE.md 机制、权限模型、无头与 SDK 化。

## 原理(稳定)

**CLAUDE.md 的注入机制**(核实于 2026-08-19 官方 memory 文档):它不是系统提示的一部分,而是**以「系统提示之后的一条 user message」注入**。这一条机制解释了三件事:
1. 为什么它是**软约束**——模型可以在权衡后不遵守;要硬约束必须走 **hooks** 或 managed settings(确定性执行,非建议);
2. 为什么**过长会失效**——它与任务指令竞争同一注意力,越长越稀释;官方给出约 200 行的目标线;
3. 为什么 `@path` 导入**不省上下文**——被导入内容照样进上下文(最多 4 跳),它省的是维护成本而非 token。

其他机制要点:四层作用域**拼接而非覆盖**;子目录 CLAUDE.md 按需加载;`.claude/rules/` 支持 `paths:` frontmatter 做路径条件加载;compaction 之后**只有项目根 CLAUDE.md 会被重新注入**(所以关键约束应放根文件);`/doctor` 可帮助裁剪。

**与 AGENTS.md 的语义差异**(跨工具复用的真实坑):AGENTS.md 是**就近文件胜出**(nearest-wins),CLAUDE.md 是**全链拼接**——同一套目录结构下两者的生效内容并不等价,迁移时必须重新验证。

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
