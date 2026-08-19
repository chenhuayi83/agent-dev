---
last_updated: 2026-08-19
maturity: growing
deep_dives: []
---

# 沙箱与隔离

agent 执行环境隔离:容器/microVM/gVisor、文件系统与网络边界、代码执行沙箱、浏览器隔离。

## 原理(稳定)

(待充实)

## 原理(稳定)

**保护路径(protected paths)是沙箱的一个关键设计**:禁止 agent 写入会改变自身行为的文件——`.claude/**` 配置、`.mcp.json`、`.git/hooks` 等。原理是切断"agent 通过修改自身配置来提权"的路径,与本仓库宪法 C1(不得自改宪法与接口层)是同一思路的不同实现层。

**凭证代理注入模式**:与其把密钥交给 agent,不如由沙箱层持有——配置中声明凭证的 `mask`(在上下文中脱敏)+ `injectHosts`(仅对指定 host 由代理注入真实值)。agent 能用凭证却看不到凭证,注入攻击即使得手也窃不走明文。

## 现状(as-of 2026-08-19,核实于面试题 07 组撰写,详见 [interview/07-security-injection.md](../../interview/07-security-injection.md))

- Claude Code 沙箱实现:macOS 用 Seatbelt、Linux 用 bubblewrap;网络侧 `allowedDomains` / `strictAllowlist`;权限侧 auto mode 由分类器模型拦截 scope escalation 与 hostile-content-driven actions
- **官方自陈的沙箱局限**(这是设计时必须知道的边界):不终止 TLS 时可被 domain fronting 绕过;挂载 `docker.sock` 等价于交出宿主;**subagent 与父会话共享沙箱**(隔离的是上下文,不是权限边界)
- 推论:沙箱降低损害面但不消除;必须与出口控制、短 TTL 凭证、审计三者叠加

## 时间线

## 开放问题

- 主流 coding agent 的沙箱实现对比?
- 沙箱逃逸的已知案例?
