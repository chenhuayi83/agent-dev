---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# Prompt Injection 攻防

agent 最大的安全威胁:直接/间接注入、工具结果注入、"lethal trifecta"(私有数据+不可信内容+外发通道)、防御分层(隔离/标记/策略/人审)。本仓库宪法 C8 即为此设。

## 原理(稳定)

(待充实)

## 现状(易变,断言须带 as-of 日期)

as-of 2026-08-19:
- **AI 写码 + AI 攻击双向闭环已成现实**:Wiz 事件(08-17)中,Copilot 生成的 workflow 代码引入 shell 注入,而利用漏洞的也是自主 AI(Wiz Red Agent 根据报错自主调整 payload)。两点工程教训:自动扫描器对着漏洞代码未报警;AI 生成的安全敏感代码必须人审([Wiz](https://www.wiz.io/blog/red-agent-snowflake-copilot-cicd-bug))
- 防御产品化仍早期;OpenAI 面向授权安全工作推出 GPT-5.6-Cyber/Daybreak 计划(2026-08,二手来源待核)

## 时间线

- 2026-08-17 Wiz 披露 Copilot 生成代码 → Snowflake Jira 凭证外泄事件(AI 攻击 AI 首个大厂实例)

## 开放问题

- 2026 年注入防御的 SOTA 与实际拦截率?
- CaMeL 类形式化防御的落地情况?
