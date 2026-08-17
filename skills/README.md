# Skills

本目录是 Agent Skills 的统一入口，每个子目录代表一个可独立安装的技能。

## 已收录

- [`testing-frontend-code`](testing-frontend-code/SKILL.md)：前端业务代码的可测试性设计、测试层选择、回归测试与 E2E 治理。

## 目录约定

```text
skills/
└── <skill-name>/
    ├── SKILL.md
    ├── references/  # 可选：按需读取的参考资料
    ├── scripts/     # 可选：可执行辅助脚本
    └── evals/       # 可选：评估用例
```

新增技能时，目录名使用小写 kebab-case，并确保其中存在带 YAML frontmatter 的 `SKILL.md`。具体要求见仓库根目录的 [`AGENTS.md`](../AGENTS.md)。

## 安装示例

```sh
npx skills add <owner>/<repository> --skill <skill-name>
```

安装本仓库当前技能：

```sh
npx skills add <owner>/<repository> --skill testing-frontend-code
```

`<owner>/<repository>` 替换为本项目公开 Git 仓库地址。
