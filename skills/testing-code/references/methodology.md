# 代码可测试性参考

## 核心模型

业务代码通常包含两类工作：

| 类型 | 形态 | 示例 | 首选验证方式 |
| --- | --- | --- | --- |
| 决策 | 输入到输出、有分支、无副作用 | 权限结果、错误映射、队列下一状态、请求参数校验 | 快速单元测试 |
| 执行 | 读取或改变外部世界 | 请求、数据库操作、消息发送、文件写入、导航 | 边界注入或更高测试层 |

当一个函数同时负责决策与执行时，抽出带类型的决策，让执行层只负责解释决策结果。例如 TypeScript 里用联合类型把决策表达为不可变值：

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

async function executeGrantAction(action: UiAction, router: Router, notifier: Notifier) {
  if (action.type === 'exit-and-toast') {
    await notifier.show(action.text);
    await router.back();
  } else {
    await notifier.show(action.text);
  }
}
```

Java 可用 `record` 返回决策，Go 可用 `struct` + `error`，外层解释后再触发数据库/网络副作用。核心不是纯函数，而是把业务判断从副作用执行中分离，让判断可以低成本稳定验证。

## 三条设计判据

1. **分离决策与执行。** 业务 `if` 与 API、数据库、缓存、消息队列、路由、通知、时钟或文件系统访问出现在一起，就是需要调整结构的信号。
2. **显式表示状态机。** 对多步骤异步流程，识别状态、事件、合法迁移、重试和终态。不要把契约藏在回调、监听器或隐式线程里。
3. **把副作用移到边界。** 在最外层注入执行器、时钟、网络、数据库或存储；内部协作模块保持真实实现。

纯函数本身不是目的，低成本且稳定地验证业务决策才是目的。无业务分支或状态的简单包装函数不必抽取。

## 识别可替换边界

"不可控因素"是指相同业务输入下，仍可能让结果变化的外部条件，例如当前时间、随机数、生成的 ID、网络响应、数据库内容、缓存内容、消息队列结果、文件系统状态或进程通信结果。

"集中在可替换边界"是指业务判断不直接、分散地访问这些条件，而是通过最外层参数、项目已有适配器或执行器获取。生产环境连接真实实现，测试只在这个边界提供可控结果，内部业务模块保持真实运行。

不同语言的惯用边界注入方式：

- **TypeScript**：工厂 options 或构造函数注入边界函数/对象。
  ```ts
  const sender = createMessageSender({
    generateId,
    sendRequest,
    saveMessage,
  });
  ```

- **Go**：通过 `interface` 与 `context.Context` 把边界抽象出来，测试用 fake 实现替换。
  ```go
  type MessageStore interface {
    Save(context.Context, Message) error
  }

  func NewSender(store MessageStore, sender RequestSender) *Sender {
    return &Sender{store: store, sender: sender}
  }
  ```

- **Java**：通过接口 + 依赖注入框架（如 Spring）把 Repository、Gateway 等边界外置。
  ```java
  public interface MessageStore { void save(Message m); }
  public interface RequestSender { void send(Request r); }

  @Service
  public class MessageService {
    public MessageService(MessageStore store, RequestSender sender) { /* ... */ }
  }
  ```

测试可以传入固定 ID、预设网络结果和内存存储，从而让同一输入稳定产生同一可观察结果。不要为了测试而模拟被测模块内部的业务判断。

若唯一需要控制的因素是 `setTimeout`、`setInterval` 或 `Date` 这类时钟，优先使用当前测试运行器（如 Vitest、Jest）或语言内置的测试时钟控制。它们已经在测试环境替换全局时间边界，不要仅为此新增 `Clock`、`sleep` 包装或依赖注入。只有现有工具无法控制相关时间来源，或项目本身已有时钟抽象时，才沿用或补充最小边界。

## 决策清单

实施前，为本次改动范围建立一张精简表格：

| 输入或事件 | 当前状态 | 决策或下一状态 | 可观察结果 | 边界副作用 |
| --- | --- | --- | --- | --- |
| API 返回 `FILE_DELETED` | 正在授权 | `exit-and-toast` | 用户离开页面并看到错误 | 路由和通知 |
| 订单金额超过阈值 | 待扣款 | 需要审批 | 状态变为 PENDING_APPROVAL | 写入数据库 |
| 消息队列消费失败 | 处理中 | 重试计数 +1 | 下一次延迟重试 | 消息确认/重入队 |

只有会改变这张表的未知业务规则，才值得暂停并向用户确认。

## 状态模块测试装置

对队列、上传、审批、重试和并发控制，使用真实模块配合可控的外部依赖。例如 TypeScript 里用 options 对象注入可观察的执行器：

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

Go 里则用 interface + 闭包控制同样的边界，同时展示它特有的 `context` 传播：

```go
func createHarness() *Harness {
  var executed []string
  active := false
  queue := NewQueue(QueueDeps{
    Execute: func(ctx context.Context, text string) error {
      executed = append(executed, text)
      return nil
    },
    HasActiveTask: func() bool { return active },
  })
  return &Harness{queue: queue, executed: &executed, setActive: func(v bool) { active = v }}
}
```

根据真实产品契约断言先进先出、失败保留、重试、取消和恢复。不要模拟 `enqueue`、`drain`、状态迁移或其他内部行为。

## 评审清单

- 每个改变的业务分支都有明确输入和可观察输出。
- 时间、随机数、ID、网络、数据库、缓存、消息队列、文件系统和进程通信等外部可变条件已被识别。
- 外部可变条件集中在测试可控制或替换的边界，没有散落在业务判断中。
- 仅需控制时间时优先使用当前测试运行器的 fake timers / 测试时钟，不新增无必要的时钟抽象。
- 业务判断没有与边界副作用混在一起。
- 异步状态和迁移可以穷举。
- 集成测试中的内部模块使用真实实现。
- 测试断言输出、状态或用户可见结果，而非实现细节。
- E2E / 全链路场景通过 `SKILL.md` 中的三项筛选，并且只覆盖黄金路径。
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
| E2E / 全链路测试依赖真实账号或可变服务 | 使用隔离的有状态模拟后端或测试环境 |

来源：<https://huali.cafe/post/frontend-testing-methodology/>。
