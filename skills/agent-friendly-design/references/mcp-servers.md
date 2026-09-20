# MCP server 细则

## 目录

- [传输与端点](#传输与端点)
- [工具设计](#工具设计)
- [鉴权与作用域](#鉴权与作用域)
- [发现与注册](#发现与注册)
- [页面侧工具（WebMCP）](#页面侧工具webmcp)
- [常见反模式](#常见反模式)

## 传输与端点

- 使用 Streamable HTTP 传输，端点挂在 `/mcp`。
- 旧的 HTTP + SSE 传输逐步淘汰，新 server 不要用它起步。
- 一个产品可以挂多个 server：公开 docs server（匿名可用，提供文档搜索和只读能力）和产品 server（OAuth 保护，提供真实操作）。把两类工具混在一个 server 里会让鉴权边界模糊。

## 工具设计

- 工具即契约：name、description、inputSchema 三件套质量决定 Agent 能否正确使用。命名用动词开头的小写 snake_case（`publish_article`、`schedule_push`）。
- description 遵循三要素（做什么、何时用、何时不用），并标注副作用和不可逆性。
- inputSchema 用 JSON Schema 显式表达：必填、enum、格式、取值范围。用 Zod 时可以用 `z.toJSONSchema` 从同一 schema 生成 DOM 暴露和执行校验，避免两份契约漂移。
- 工具保持原子。读取和写入分开，批量能力用"逐项结果"表达部分成功，不要造"一键完成整个流程"的复合工具。
- 工具列表本身也要可发现：server 首页或 `server.json` 里列出可用工具和一句话说明。

## 鉴权与作用域

- 产品 server 走 OAuth 2.0：protected-resource metadata 挂在 `/.well-known/oauth-protected-resource`（RFC 9728），authorization server metadata 用 RFC 8414 的可发现格式，授权码流配 PKCE。
- Token 的作用域按最小权限划分，和工具能力对应；文档里写清每个 scope 能调哪些工具。
- 支持自助式的 key 生成和 scope 管理，Agent 代表的所有者必须能限制 Agent 能做什么、不能做什么。
- 高风险工具（删除、扣费、对外发送、批量变更）要求显式确认参数，并把审批 / 确认环节设计进系统，不是"相信 Agent 会小心"。

## 发现与注册

- 注册到 MCP Registry，并维护 server.json，让客户端在连接前就能发现能力和端点。
- Server cards（连接前发现 server 元信息的约定，如 SEP-2127 草案）是新兴方向：目标宿主支持就提供，路径规范可能变动，不要硬编码假设。
- 产品文档页要有一节"如何从 Agent 连接"，给出 server URL、认证方式和工具概览；这一页同时是 Agent 发现 server 的入口。

## 页面侧工具（WebMCP）

- 如果产品有人工操作的 Web UI 且希望 Agent 操作它，可以在页面上按 WebMCP 约定把工具注册到 `document.modelContext`，工具直接在已登录会话里执行。这是 W3C Community Group 草案，能力检测后再注册。
- WebMCP 与服务端 MCP 互补，不互替：页面工具跑在实时页面和登录态里，服务端工具跑在 API 层。只注册产品真实提供的操作。
- 页面也可以用 sr-only DOM 向浏览器 Agent 暴露当前可用能力和调用契约，DOM 是主要暴露通道，内容要含参数要求和调用示例。

## 常见反模式

- 只读工具一大堆、写操作一个没有：Agent 能读不能动，"action gap"。
- 一个 `manage_everything` 工具，参数里塞 mode 开关。
- 工具描述复制 API 文档的笼统摘要，没写何时用、何时不用。
- 把公开文档搜索也放在 OAuth 后面，Agent 连"这个 server 能干什么"都查不到。
- 静默重试失败操作而不返回结构化错误，掩盖了部分失败。
