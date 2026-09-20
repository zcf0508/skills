---
name: agent-friendly-design
description: |
  设计或修改面向 Agent 的契约时应用 Agent 友好约束：API 端点与错误响应、OpenAPI / JSON Schema、function calling tool schema、MCP server 工具、SKILL.md 等。核心规则包括字段语义化、有限集合约束显式化、结构化可恢复错误、禁止静默纠错、写操作幂等、部分成功显式建模、不可逆操作留人工确认、机器可读文件可从已有页面发现。用户要求写接口、定义工具、搭建 MCP、编写 Skill、设计错误响应，或评审这类契约的 Agent 友好性时使用。不用于纯内部实现逻辑、给人看的 UI 文案、纯视觉设计或与 Agent 契约无关的重构。
---

# 面向 Agent 的契约设计

软件的消费方正在从"只有人"变成"人 + Agent"。本 Skill 约束的是后者：Agent 依赖结构和语义，不靠视觉和直觉猜。对人"差不多能用"的设计，对 Agent 往往意味着猜错、重试、局部卡死。

## 触发信号

以下情况说明本 Skill 需要介入：

- 新增或修改对外 API 端点、错误响应、Webhook 负载、OpenAPI / JSON Schema 契约。
- 定义 function calling 的 tool schema，或给 MCP server 添加、修改工具。
- 编写或重构 SKILL.md、AGENTS.md 这类 Agent 指令文件。
- 设计面向 Agent 的文档页、llms.txt、站点可发现性。

纯内部实现逻辑、给人看的 UI 文案、纯视觉设计不适用本 Skill。

## 核心原则

1. **语义化值**：对外枚举和状态用有语义的字符串（`pending`、`shipped`），禁止把内部编码（`status=1`、scene=X12）当契约。内部编码可以留在存储层，在边界上做映射。
2. **约束写进 schema**：enum、必填、取值范围、格式、默认值写进 OpenAPI / JSON Schema / Zod，不藏在文档 prose 里。有限集合必须显式枚举；Agent 基于契约生成参数，靠文档猜就会错，错了就重试，重试放大系统成本。
3. **错误即控制流**：分支逻辑靠机器可读的 `code` / `type`，`message` / `detail` 只服务人和日志。错误响应至少回答：发生了什么、错在哪里、能不能恢复、怎么恢复。附带 `suggestion`、`allowed_values`、`retryable`，禁止只返回一句文案，也禁止让客户端解析会变化的文案。
4. **禁止静默纠错**：模糊匹配到多个候选、替换业务状态、影响计费 / 权限 / 对象范围的参数，显式失败并返回 `did_you_mean` / `suggestions`，把决定权还给调用方。大小写、空白、历史别名这类无害变体可以兼容。"看起来成功但做错了事"比明确失败更危险。
5. **可恢复优先于一次做对**：有副作用的写操作支持幂等键；部分成功显式建模，返回逐项结果而不是只有成功 / 失败两态，否则恢复时可能重复执行已成功的那部分；删除、扣费、对外发送、批量改权限、生产变更这类不可逆操作保留人工确认和回滚路径。
6. **可发现**：机器可读文件（openapi.json、llms.txt、MCP registry 条目）必须从已有人能访问的页面用链接指过去，没有入口的文件等于不存在。站点答案要进初始 HTML，不能藏在客户端 JS 渲染后面；返回真实 HTTP 状态码。

## 领域速查

| 领域 | 必做 | 细则 |
| --- | --- | --- |
| HTTP API | 语义化枚举；约束进 schema；结构化错误；显式版本（如 `/v1/`）；写操作幂等；Agent 能自助完成的认证流 | [references/api-and-schemas.md](references/api-and-schemas.md) |
| tool schema | description 写清做什么、何时用、何时不用；参数语义化命名；副作用显式标注；工具保持原子，一个工具一个动作 | [references/api-and-schemas.md](references/api-and-schemas.md) |
| MCP server | Streamable HTTP 挂 `/mcp`；公开 docs 工具与 OAuth 保护的产品工具分开；注册 MCP Registry；server.json 提供连接前发现 | [references/mcp-servers.md](references/mcp-servers.md) |
| SKILL.md | Goal / Instructions / Examples / Constraints 四段齐全；description 写清做什么和何时触发；示例多于规则；只写决策信息，不写实现细节 | [references/examples.md](references/examples.md) |

## 反模式

### 1. 只返回文案的错误

```json
// 反面：Agent 只能猜或放弃
{ "message": "Invalid parameter" }

// 正面：Agent 可自主分支修复
{
  "code": "INVALID_ENUM",
  "field": "priority",
  "value": "urgent",
  "allowed_values": ["low", "medium", "high", "critical"],
  "suggestions": ["high", "critical"],
  "retryable": false
}
```

### 2. 内部编码直接对外

```json
// 反面：语义不共享，Agent 只能靠历史记忆
{ "status": 2, "scene": "X12" }

// 正面：值语义化，有限集合有显式约束
{ "status": "shipped", "scene": "holiday_promo" }
```

### 3. 静默纠错

```json
// 反面：模糊匹配后擅自替换，"成功"但做错了
{ "status": "shipped", "_note": "auto-corrected 'shpped'" }

// 正面：显式失败，建议交还给调用方决定
{
  "code": "UNKNOWN_STATUS",
  "value": "shpped",
  "did_you_mean": ["shipped", "pending"],
  "retryable": false
}
```

更多领域的完整示例见 [references/examples.md](references/examples.md)。

## 交付前自检

- 对外枚举无语义编码，有限集合有显式 schema 约束。
- 每条错误路径有机器可读 `code` 和恢复信号（`suggestion` / `retryable` / `allowed_values`）。
- 写操作支持幂等；部分成功有逐项状态；不可逆操作有人工确认和回滚路径。
- 没有替调用方做模糊决策的静默纠错。
- 命名、版本、结构全站一致。
- 机器可读文件有入口链接；站点答案在初始 HTML；状态码真实。
- 改动后先跑 typecheck，再跑相关测试；错误路径至少有一个失败用例。

## 边界

- 不追求每个接口绝对最小返回，避免为每个场景新建 VO；判断标准是不把给人看的全量对象当默认契约。
- 高风险、强判断、不可逆的流程，Agent 只做辅助不当主执行者，契约设计要留人工接管点。
- 存量系统不追求一次改完：新写的契约遵守本 Skill，旧契约在每次触及处逐步迁移。

## 参考资料

- API 与 tool schema 细则：[references/api-and-schemas.md](references/api-and-schemas.md)
- MCP server 细则：[references/mcp-servers.md](references/mcp-servers.md)
- before / after 示例库：[references/examples.md](references/examples.md)
- 评估用例位于 `evals/evals.json`
- 方法论来源：<https://www.agentready.org/>、<https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices>
