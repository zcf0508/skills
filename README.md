# zcf0508's Skills

一组面向日常前端研发的 Agent Skills，沉淀可复用的工程判断、工作流和验证方式。

## 安装

从 GitHub 安装全部技能：

```sh
pnpx skills add zcf0508/skills --skill='*'
```

全局安装：

```sh
pnpx skills add zcf0508/skills --skill='*' -g
```

更多 CLI 用法见 [vercel-labs/skills](https://github.com/vercel-labs/skills)。

## Skills

### 手写技能

由仓库维护者根据实际工程经验编写和持续迭代。

| Skill | 说明 |
| --- | --- |
| [`testing-frontend-code`](skills/testing-frontend-code) | 用可测试性约束设计、实现、重构或评审前端业务代码；根据风险选择测试层，控制 E2E 成本。 |

## 新增或维护 Skill

每个 Skill 使用 `skills/<skill-name>/SKILL.md` 作为入口，目录名使用全小写 kebab-case。

创建或修改前阅读 [AGENTS.md](AGENTS.md)，其中包含目录约定、内容要求、验证方式，以及 Anthropic 官方的 [Skill 编写最佳实践](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)。

## License

MIT

