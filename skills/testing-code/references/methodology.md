# 代码可测试性参考

## 目录

- 核心模型
- 测不变量，不测动作
- 三条设计判据
- 识别可替换边界
- 决策清单
- 状态模块测试装置
- 评审清单
- 常见反模式及恢复方式

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

类型系统本身是一层廉价验证。TS 里用 `never` 分支穷举状态，配合联合类型收窄让非法状态不可表示。类型能约束状态形状，但不能自动保证外部输入有效、异步结果属于当前操作或并发写入合法；这些仍需边界校验与行为测试。数据库唯一索引可以约束存储层唯一性，但不替代幂等业务契约的验证。只对静态机制实际保证的不变量免除重复的运行时断言。

## 测不变量，不测动作

行为测试验证用户可观察的结果，默认首选。核心逻辑还需要白盒不变量测试补位，因为幂等键生成、缓存一致性、状态机合法迁移、并发去重这类契约，黑盒行为测试不够用。

| 类型 | 验证什么 | 示例 |
| --- | --- | --- |
| 行为测试 | 用户可观察的结果 | 重复下单只扣一次款 |
| 不变量测试 | 状态约束、输入输出契约、合法迁移、幂等性 | 幂等键唯一、缓存与源一致、非法迁移被拒 |

- 核心逻辑的契约测试成本低于行为测试，值得单独覆盖。
- 实现不符合预期但行为恰好正确，是 bug 跑在 bug 上，凑巧的正确会让后续边界问题难排查。
- 不要滑向测动作序列：方法调了几次、用了什么中间对象这类断言会把重构锁死。

## 三条设计判据

1. **分离决策与执行。** 业务 `if` 与 API、数据库、缓存、消息队列、路由、通知、时钟或文件系统访问出现在一起，就是需要调整结构的信号。
2. **显式表示状态机。** 同一事件的处理依赖历史操作，或多个标记、回调与持久化字段共同决定下一步时，在本次改动路径梳理状态、事件、迁移条件、状态归属和终态。明确谁有权改变状态、哪些结果已失效；沿用现有结构或最小状态表示，不强制引入状态机库。
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

测试可以传入固定 ID、预设网络结果和内存存储，从而让同一输入稳定产生同一可观察结果。不要为了测试而模拟被测模块内部的业务判断。注入 fake 时要校验它和真实系统的契约（字段、错误语义、时序），否则 fake 通过而真实系统失败，等于没测。

若唯一需要控制的因素是 `setTimeout`、`setInterval` 或 `Date` 这类时钟，优先使用当前测试运行器（如 Vitest、Jest）或语言内置的测试时钟控制。它们已经在测试环境替换全局时间边界，不要仅为此新增 `Clock`、`sleep` 包装或依赖注入。只有现有工具无法控制相关时间来源，或项目本身已有时钟抽象时，才沿用或补充最小边界。

## 决策清单

读取实现与现有测试后，在对话中展示本次改动的决策结果；简单改动用一句话，复杂改动可用以下表格：

| 行为与代码位置 | 风险或不变量 | 结构决策 | 测试决策与依据 |
| --- | --- | --- | --- |
| 当前异步流程的相关实现 | 旧结果不得覆盖当前操作 | 明确状态归属与迁移条件，保留真实执行模块 | 新增状态模块测试，用可控调度覆盖中断、重试与旧结果晚到的序列 |

结构决策可以是提取逻辑、明确状态或保持原状；测试决策可以是新增、修改、复用或豁免，均需给出代码或既有覆盖依据。记录应包含相关输入或事件、关键迁移、可观察结果及边界副作用，但无需为每个字段另建表格。

展示后直接继续已授权工作，不等待用户批准。只有实质影响正确性且无法从现有资料确定的业务规则才需询问，其余工作继续。实施中调整决策时说明依据，交付时关联实际改动、测试用例和运行结果逐项核对；不以清单已经写出代替验证。

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

单次迁移通过不代表事件组合正确。围绕本次风险，用可控调度运行重复、乱序、中断或恢复序列，在关键迁移后检查不变量及可观察结果；一个序列测试可以跨越多个分支，但只验证一个行为契约。例如取消后重试、旧请求晚到时，验证旧结果不会覆盖当前操作的状态或产生过期副作用。预期来自业务契约，不从当前实现反推；恢复策略不明时先确认，不自行发明。

## 评审清单

- 每个改变的业务分支都有明确输入和可观察输出。
- 时间、随机数、ID、网络、数据库、缓存、消息队列、文件系统和进程通信等外部可变条件已被识别。
- 外部可变条件集中在测试可控制或替换的边界，没有散落在业务判断中。
- 仅需控制时间时优先使用当前测试运行器的 fake timers / 测试时钟，不新增无必要的时钟抽象。
- 业务判断没有与边界副作用混在一起。
- 本次路径中的隐式状态、迁移条件和状态归属已明确，用户可见状态、可执行动作与恢复入口一致。
- 按实际风险验证事件序列与全过程不变量，未将类型安全或单步迁移覆盖当作时序正确的证明。
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
| 为机械对齐性修复搭建多组件 e2e 复现设施 | 指出对齐的既有分支与其覆盖，按测试豁免披露；或把断言下沉到该分支 |

来源：<https://huali.cafe/post/frontend-testing-methodology/>。
