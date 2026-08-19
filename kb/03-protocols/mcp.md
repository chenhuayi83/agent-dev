---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# MCP(Model Context Protocol)

工具/资源接入的事实标准:server/client 架构、tools/resources/prompts 原语、transport 演进、auth、registry 生态。

## 原理(稳定)

(待充实)

## 现状(易变,断言须带 as-of 日期)

as-of 2026-08-19:
- 当前 spec 版本 **2026-07-28**:协议核心转 **stateless**(移除 session/握手/`Mcp-Session-Id`),新增 `Mcp-Method`/`Mcp-Name` 头与 `ttlMs`/`cacheScope` 缓存元数据;serverless/多副本部署成为一等公民([官方博客](https://blog.modelcontextprotocol.io/posts/2026-07-28/))
- **Extensions 框架**正式化:MCP Apps(服务端渲染交互 UI)与 Tasks(长任务)以版本化扩展形式存在,核心协议保持精简
- auth 对齐 OAuth 2.0/OIDC,可直连 Entra/Okta 等企业身份系统
- 首个正式**弃用政策**:07-28 提议弃用的特性最早 2027-07-28 移除,迁移有一年窗口
- 客户端覆盖:Claude、ChatGPT、VS Code、Cursor 等全主流([modelcontextprotocol.io](https://modelcontextprotocol.io/));Claude 侧适配公告见 [claude.com/blog](https://claude.com/blog/bringing-mcp-2026-07-28-to-claude)
- 生态定位:与 A2A(agent 互通)、Agent Plugins(打包分发)三分工,MCP 占运行时工具调用

## 时间线

- 2026-07-28 spec 新版:stateless 核心 + Apps/Tasks Extensions + OAuth 对齐 + 弃用政策

## 开放问题

- MCP spec 最新版本引入了什么(elicitation/auth/registry)?
- MCP server 的安全审计现状与供应链风险?
