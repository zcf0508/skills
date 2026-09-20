# API 与 tool schema 细则

## 目录

- [HTTP API](#http-api)
  - [语义化与枚举](#语义化与枚举)
  - [命名与版本](#命名与版本)
  - [错误结构](#错误结构)
  - [幂等与部分成功](#幂等与部分成功)
  - [认证与限流](#认证与限流)
  - [站点可发现性](#站点可发现性)
- [tool schema](#tool-schema)
  - [description 三要素](#description-三要素)
  - [参数设计](#参数设计)
  - [副作用与原子性](#副作用与原子性)
  - [错误返回](#错误返回)

## HTTP API

### 语义化与枚举

- 对外值语义化：`pending` / `processing` / `shipped` 优于 `LOW` / `MEDIUM` / `HIGH`，`P1` / `P2` / `P3` 和 `01` / `02` / `03` 不可用。
- 有限集合显式写 enum，同时把语义和约束放进 schema。
- 大小写、合法空白、分隔符等无害变体可以兼容；历史字段名到新字段名的映射放在边界层。

```yaml
priority:
  type: string
  enum: [low, medium, high, critical]
  default: medium
```

### 命名与版本

- 全站统一命名风格（snake_case 或 camelCase 二选一），不混用 `userId` 和 `user_id`。
- URL 结构可预测，资源名用复数名词。
- 显式版本（`/v1/` 或 `X-API-Version` header），禁止静默 breaking change；废弃端点返回结构化告警，并允许通过自省查询支持的版本。
- OpenAPI 契约中每个 operation 有语义化的 operationId。

### 错误结构

- 错误响应用固定结构：机器信号（`code` / `type`）+ 人读信息（`message` / `detail`）+ 恢复信号（`suggestion` / `retryable` / `allowed_values` / `did_you_mean`）。
- `detail` 会随版本、本地化、润色变化，客户端禁止解析它做分支。
- 报错时给纠正路径：能改参数重试的给建议值，要找人的说明找谁，不可逆的明确标出。

```json
{
  "code": "QUOTA_EXCEEDED",
  "message": "Monthly quota of 10000 calls reached.",
  "retryable": true,
  "retry_after_seconds": 86400,
  "upgrade_url": "https://example.com/pricing"
}
```

### 幂等与部分成功

- 所有有副作用的写操作支持幂等键（如 `Idempotency-Key` header 或请求字段），Agent 比人更容易重试，重复请求不能产生重复效果。
- 批量操作返回逐项结果，标明每项的成功 / 失败和失败原因，让调用方只重试失败项：

```json
{
  "results": [
    { "item": "a1", "status": "created", "id": "res_123" },
    { "item": "a2", "status": "failed", "code": "DUPLICATE_SKU", "retryable": false }
  ]
}
```

- 状态机有细粒度状态（已创建 / 待审批 / 部分完成），只有成功 / 失败两态会让 Agent 在恢复时重复执行已完成的部分。

### 认证与限流

- 优先 API Key 自助生成或 OAuth2 Client Credentials 这类 Agent 能自动完成的流。
- 用户委托场景用 OAuth 2.0，认证元数据挂在 `/.well-known`（RFC 8414、RFC 9728），配合 PKCE；不要用 CAPTCHA、邮件确认这类交互式门槛挡 Agent。
- 限流返回结构化配额错误（剩余额度、重置时间），人机流量可以分开限流。

### 站点可发现性

站点要能被 Agent 找到和读到，面向站点的接口同样适用：

- 答案放进初始 HTML（服务端渲染或预渲染），客户端 JS 注入的内容对 fetch-only 的 Agent 不可见。
- 返回真实 HTTP 状态码，2xx / 3xx / 4xx 语义正确。
- llms.txt、openapi.json、JSON-LD 等机器可读文件，从首页、页脚、`<link rel="alternate">` 或文档页用链接指过去；不被链接指向的文件很少被Agent发现。
- JSON-LD 的关键事实在可见文本里重复一遍，markdown 转换器可能丢掉 script 块。
- 代码块加围栏和语言标记，Agent 经常原样引用抓到的代码片段。

## tool schema

### description 三要素

工具的 description 决定模型会不会选对工具，写清三件事：

- 做什么：动作 + 对象 + 效果（`Create a draft push notification for the given app`）。
- 何时用：典型触发场景。
- 何时不用：边界和易混淆的相邻工具（`Do not use for scheduled pushes; use schedule_push`）。

避免空泛描述（`Handles notifications`）。相邻工具的能力边界写进各自 description，不靠模型猜。

### 参数设计

- 参数名语义化（`user_id` 优于 `uid`），与 API 字段命名一致。
- 有限集合用 enum 显式约束；必填、格式、取值范围都写进 schema。
- 默认值语义明确；允许为空的情况说明什么条件可以为空。
- 参数含义会改变计费、权限、对象范围时，在 description 里显式警告。

### 副作用与原子性

- description 显式标注副作用：只读、创建、更新、删除、不可逆、触发外部发送。
- 一个工具只做一个动作。需要"发布三篇文章并各排期一条推送"时，提供 `publish_article` 和 `schedule_push` 两个工具，由 Agent 组合，而不是造一个巨型工具；巨型工具参数多、分支多、错误难定位。
- 不可逆操作（删除、扣费、对外发送）在 schema 或执行流程中要求显式确认参数（如 `confirm: true` 或 dry-run 模式），并支持 human-in-the-loop。

### 错误返回

- 工具执行失败返回结构化错误：机器可读 code、人读 message、恢复信号（suggestion、retryable）。
- 已卸载或不可用的工具返回结构化可恢复错误（说明当前不可用和替代路径），不要让调用方猜测。
