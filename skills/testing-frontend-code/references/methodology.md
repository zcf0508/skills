# 前端可测试性参考

## 核心模型

前端模块包含两类工作：

| 类型 | 形态 | 示例 | 首选验证方式 |
| --- | --- | --- | --- |
| 决策 | 输入到输出、有分支、无副作用 | 权限结果、错误映射、队列下一状态 | 快速单元测试 |
| 执行 | 读取或改变外部世界 | 请求、导航、通知、写入存储 | 边界注入或更高测试层 |

当一个函数同时负责决策与执行时，抽出带类型的决策，让执行层只负责解释决策结果。

```ts
type GrantResult
  = | { ok: true }
    | { ok: false, code: 'FILE_DELETED' | 'NO_PERMISSION' };

type UiAction
  = | { type: 'toast', level: 'success' | 'error', text: string }
    | { type: 'exit-and-toast', text: string };

function decideGrantOutcome(result: GrantResult): UiAction {
  if (result.ok) { return { type: 'toast', level: 'success', text: '授权成功' }; }
  if (result.code === 'FILE_DELETED') { return { type: 'exit-and-toast', text: '文件已被删除' }; }
  return { type: 'toast', level: 'error', text: '无权限' };
}
```

请求包装层只获取结果并执行 `UiAction`，不要重复判断错误码。

## 三条设计判据

1. **分离决策与执行。** 业务 `if` 与 API、路由、状态管理、通知、时钟或存储访问出现在一起，就是需要调整结构的信号。
2. **显式表示状态机。** 对多步骤异步流程，识别状态、事件、合法迁移、重试和终态。不要把契约藏在回调和监听器中。
3. **把副作用移到边界。** 在最外层注入执行器、时钟、网络或存储；内部协作模块保持真实实现。

纯函数本身不是目的，低成本且稳定地验证业务决策才是目的。无业务分支或状态的简单包装函数不必抽取。

## 识别可替换边界

“不可控因素”是指相同业务输入下，仍可能让结果变化的外部条件，例如当前时间、随机数、生成的 ID、网络响应、存储内容或进程通信结果。

“集中在可替换边界”是指业务判断不直接、分散地访问这些条件，而是通过最外层参数、项目已有适配器或执行器获取。生产环境连接真实实现，测试只在这个边界提供可控结果，内部业务模块保持真实运行。

```ts
const sender = createMessageSender({
  generateId,
  sendRequest,
  saveMessage,
});
```

测试可以传入固定 ID、预设网络结果和内存存储，从而让同一输入稳定产生同一可观察结果。不要为了测试而模拟 `createMessageSender` 内部的业务判断。

若唯一需要控制的因素是 `setTimeout`、`setInterval` 或 `Date`，优先使用 Vitest 等当前测试运行器的 fake timers。它们已经在测试环境替换全局时间边界，不要仅为此新增 `Clock`、`sleep` 包装或依赖注入。只有现有工具无法控制相关时间来源，或项目本身已有时钟抽象时，才沿用或补充最小边界。

## 决策清单

实施前，为本次改动范围建立一张精简表格：

| 输入或事件 | 当前状态 | 决策或下一状态 | 可观察结果 | 边界副作用 |
| --- | --- | --- | --- | --- |
| API 返回 `FILE_DELETED` | 正在授权 | `exit-and-toast` | 用户离开页面并看到错误 | 路由和通知 |

只有会改变这张表的未知业务规则，才值得暂停并向用户确认。

## 状态模块测试装置

对队列、上传、审批和重试，使用真实模块配合可控的外部依赖：

```ts
function createHarness() {
  const executed: string[] = [];
  let active = false;

  const queue = createQueue({
    execute: async text => executed.push(text),
    hasActiveTask: () => active,
  });

  return {
    queue,
    executed,
    setActive: (value: boolean) => { active = value; },
  };
}
```

根据真实产品契约断言先进先出、失败保留、重试、取消和恢复。不要模拟 `enqueue`、`drain`、状态迁移或其他内部行为。

## 评审清单

- 每个改变的业务分支都有明确输入和可观察输出。
- 时间、随机数、ID、网络、存储和进程通信等外部可变条件已被识别。
- 外部可变条件集中在测试可控制或替换的边界，没有散落在业务判断中。
- 仅需控制时间时优先使用当前测试运行器的 fake timers，不新增无必要的时钟抽象。
- 业务判断没有与边界副作用混在一起。
- 异步状态和迁移可以穷举。
- 集成测试中的内部模块使用真实实现。
- 测试断言输出、状态或用户可见结果，而非实现细节。
- E2E 场景通过 `SKILL.md` 中的三项筛选，并且只覆盖黄金路径。
- 存量代码重构严格限制在需求触及的决策范围内。
- Snapshot 由测试运行器生成，且差异经过审查。
- 所有验证结论都有真实命令结果支持。

## 常见反模式及恢复方式

| 反模式 | 恢复方式 |
| --- | --- |
| 为一个错误映射编写完整 E2E | 抽出错误映射，用单元测试验证决策 |
| 模拟每个内部协作者 | 恢复真实内部模块，只控制最外层边界 |
| 断言辅助函数被调用三次 | 断言最终状态、输出或可见行为 |
| 补测试前先重构整个旧功能 | 先复现当前分支，再只抽取该分支 |
| 追求 100% 行覆盖率 | 覆盖有意义的决策分支，删除空洞断言 |
| E2E 依赖真实账号或可变服务 | 使用隔离的有状态模拟后端 |

来源：<https://huali.cafe/post/frontend-testing-methodology/>。
