---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# 其他编码代理

Codex(CLI/云)、Cursor(IDE+background)、Copilot(agent mode)、Gemini CLI、OpenHands、Devin、Amp 等的形态与差异化。

## 原理(稳定)

(待充实)

## 现状(易变,断言须带 as-of 日期)

as-of 2026-08-19:
- 第一梯队:Claude Code、**Codex**(#2,Terminal-Bench 公开纪录)、**Cursor**(Cursor 3 于 04-02 转 agent-first,IDE 降为备用面板;Composer 2.5 模型 05-18)、Copilot Agent Mode、Cline
- Artificial Analysis Coding Agent Index(05-18 口径):Claude Code+Opus 4.7 66 > Codex+GPT-5.5 65 > Composer 2.5 62 —— **harness 差距成为主要可操作变量,模型趋同**([firecrawl 综述](https://www.firecrawl.dev/blog/best-ai-coding-agents))
- 分流:终端派(Claude Code/Codex CLI)vs IDE 派(Cursor/Windsurf);两派都在补对方形态
- 新秀涌现:Kiro、Muse Code、Antigravity 2.0、Hax(C 极简终端 agent,08-12)、MathCode(数学专用,08-16)——极简与垂直两方向并行生长

## 时间线

- 2026-08-12/16 Hax、MathCode 等新形态亮相(HN 百分级热度)
- 2026-04-02 Cursor 3:agent-first 界面重构

## 开放问题

- 各家在 SWE-bench 类基准与真实采用上的位置(as-of)?
- 开源 vs 闭源 coding agent 的差距趋势?
