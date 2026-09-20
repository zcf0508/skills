# before / after 示例库

每个示例给出反面写法、正面写法和简短理由。仿照正面写法的结构和信息密度，不要照抄字段名。

## 目录

- [错误响应](#错误响应)
- [枚举语义化](#枚举语义化)
- [静默纠错与建议](#静默纠错与建议)
- [幂等键](#幂等键)
- [部分成功](#部分成功)
- [tool schema description](#tool-schema-description)
- [MCP 工具定义](#mcp-工具定义)
- [SKILL.md description](#skillmd-description)
- [SKILL.md 结构](#skillmd-结构)

## 错误响应

反面：

```json
{ "message": "Oops, something went wrong!" }
```

正面：

```json
{
  "code": "EMAIL_ALREADY_REGISTERED",
  "message": "This email is already registered.",
  "suggestion": "Use the /login endpoint if this is your account.",
  "retryable": false
}
```

理由：反面让 Agent 无法分支，只能重试原请求或放弃。正面给了机器可读 code 和下一步路径，Agent 可以自主改走 `/login`。

## 枚举语义化

反面：

```json
{ "status": 2, "priority": "P1", "flag": "Y" }
```

正面：

```yaml
# OpenAPI schema 片段
status:
  type: string
  enum: [pending, processing, shipped, delivered]
priority:
  type: string
  enum: [low, medium, high, critical]
```

理由：内部编码靠组织记忆存活，Agent 不共享这层记忆。语义值 + schema enum 让 Agent 直接生成合法参数。

## 静默纠错与建议

反面：收到 `priority=urgent` 后自动按 `high` 处理并返回 200。

正面：

```json
{
  "code": "UNKNOWN_PRIORITY",
  "field": "priority",
  "value": "urgent",
  "did_you_mean": ["high", "critical"],
  "retryable": false
}
```

理由：业务状态替换、计费 / 权限 / 范围相关的参数不允许静默改。系统把候选项还给调用方，由它自己改或上抛给人确认。大小写和空白这类无害变体才允许兼容。

## 幂等键

反面：`POST /orders` 重复提交产生两笔订单。

正面：

```http
POST /orders
Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7
```

同一 key 的重试返回首次创建的结果，带 `Idempotency-Key-Replayed: true` 头。理由：Agent 的工具层和模型层都可能自动重试，写操作不幂等会把一次意图放大成多次副作用。

## 部分成功

反面：批量导入 10 条，3 条失败时整体返回 500，成功的那 7 条无法确认。

正面：

```json
{
  "summary": { "total": 10, "succeeded": 7, "failed": 3 },
  "results": [
    { "index": 2, "status": "failed", "code": "INVALID_EMAIL", "retryable": true, "suggestion": "Fix the email format and resubmit this item." }
  ]
}
```

理由：只有成功 / 失败两态时，Agent 恢复时无法区分哪些已生效，容易重复创建。逐项状态让重试可以只针对失败项。

## tool schema description

反面：

```json
{
  "name": "handle_content",
  "description": "Handles content operations."
}
```

正面：

```json
{
  "name": "publish_article",
  "description": "Publish a draft article so it becomes publicly visible. Use when the user asks to publish, go live, or launch an article. Do not use for scheduling (use schedule_article) or for drafts the user has not confirmed. This action is externally visible and not reversible without unpublishing.",
  "inputSchema": {
    "type": "object",
    "required": ["article_id"],
    "properties": {
      "article_id": { "type": "string", "description": "ID of the draft article, e.g. art_123" },
      "confirm": { "type": "boolean", "description": "Must be true; the action is externally visible." }
    }
  }
}
```

理由：description 决定模型是否选对工具，要写做什么、何时用、何时不用，并标注副作用和不可逆性。

## MCP 工具定义

反面：把公开文档搜索也放在 OAuth 后面；所有工具挤在一个 server；只提供只读工具。

正面：

- `https://example.com/mcp/docs`：匿名可用，`search_docs`、`get_page`，供 Agent 和潜在用户发现能力。
- `https://example.com/mcp/app`：OAuth 保护（RFC 9728 metadata + PKCE），工具按 scope 分组，写操作要求 `confirm` 参数。

理由：公开 docs server 是产品能力的展示入口；产品 server 的鉴权边界清晰，Agent 在授权范围内行动。

## SKILL.md description

反面：

```yaml
description: A helper for working with the platform.
```

正面：

```yaml
description: |
  在 GoodBarber 平台上发布、排期和查询内容时使用：创建和发布文章、为文章排期推送通知、查询昨日活跃数据。用户要求发布内容、推送通知或查看运营数据时触发。不用于修改应用主题、管理用户账号或账单操作。
```

理由：description 是 Agent 从上百个 Skill 里选中这一个的依据，写清做什么、何时触发、何时不触发。中英文按团队约定，技术标识保留英文。

## SKILL.md 结构

反面：一个 800 行的 SKILL.md，塞满背景知识解释和实现细节，没有示例。

正面：

```markdown
---
name: processing-pdfs
description: |
  从 PDF 提取、填写或校验内容时使用……
---

# Goal
一句话说明目标和价值。

# Instructions
按真实执行顺序写步骤，写清缺信息时怎么问、默认值是什么。

# Examples
2-3 个输入 / 输出对，覆盖典型场景和边界场景。

# Constraints
格式、权限、安全和不可做事项。
```

要点：SKILL.md 主体控制在 500 行以内，超长内容拆到 `references/` 并从主文件一级链接；示例传达风格的效果好于抽象规则；只写 Agent 做决策需要的信息，不解释 Agent 已经知道的东西。
