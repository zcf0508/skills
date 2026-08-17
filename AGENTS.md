# 仓库说明

## 用途

本仓库用于维护可复用的 Agent Skill。每个 Skill 必须能够被 `skills` CLI 独立发现，并且只专注于一个工作流。

## 编写参考

创建或修改 Skill 时，遵循 Anthropic 官方的[Skill 编写最佳实践](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)。重点保持内容精简、描述可发现、按需加载参考资料，并使用真实场景持续评估和迭代。

## 目录结构

- `skills/<skill-name>/SKILL.md`：单个 Skill 的必需入口文件。
- `skills/<skill-name>/references/`：可选的参考资料，按需读取。
- `skills/<skill-name>/scripts/`：可选的可执行辅助脚本。必须说明使用接口；使用前先通过 `--help` 确认用法。

Skill 目录名使用全小写 kebab-case。不要直接在 `skills/` 下新增 `SKILL.md`；`skills/` 是技能目录容器，不是一个 Skill。

## Skill 要求

每个 `SKILL.md` 必须：

- 以 YAML frontmatter 开头，并包含非空的 `name` 和 `description` 字段。
- 明确一个目标、执行步骤、实际示例和约束条件。
- 准确说明触发语句和不适用范围，避免误触发。
- 保持操作说明简洁；过长的参考内容放在 `references/`。
- 不包含密钥、凭证或个人信息；存在破坏性操作时必须先取得明确确认。

## 验证

编辑 Markdown Skill 时，不要仅为此安装依赖。发布前执行：

```sh
npm pack --dry-run --json
```

确认输出只包含预期的 `skills/<skill-name>/SKILL.md` 等文件，不包含生成物、本地文件或敏感内容。

## 分发

`skills` CLI 支持从本地路径、Git 仓库、Git URL 和直接下载的压缩包安装。已发布的 npm 包不是 `skills add` 的标准来源。若要让他人远程安装，请将本仓库推送至可访问的 Git 平台，并通过仓库 URL 或 `owner/repository` 简写安装。

