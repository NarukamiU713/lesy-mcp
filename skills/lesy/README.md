# Lesy Skill

[`SKILL.md`](SKILL.md) 是给 Claude Code、Codex 等能运行 shell 的 agent 用的 Skill。用户问自己的德语学习时（“我今天学了什么”“考考我”“我在哪篇文章里见过 X”“查一下这个词”“把这个词的释义记下来”），agent 用命令行工具 `lesy` 读写 Lesy 的数据，并遵守这些规则：

- 在浏览器里登录，agent 自己不批准，也不要密码或 Token。
- 只在用户明确要求时写入，写之前说明要写什么。
- 一题一题地出复习题，不提前给答案。
- 不打印 Token，不读配置文件。

## 前提

- 在 Lesy 主仓库根目录执行过 `npm install` 和 `npm link`，`lesy` 全局可用（`lesy --help`）。见[命令行 `lesy`](../../docs/cli.md#安装)。
- Lesy 在运行（agent 连的是正在运行的后端）。
- 第一次使用时 agent 会运行 `lesy auth login --no-wait`，把链接和码给你：在浏览器里打开、核对码、点 “Zulassen”，然后告诉 agent “好了”。

## 安装

Skill 就是这个目录（`SKILL.md` 和本文件），复制整个 `skills/lesy/` 目录即可。

**Claude Code**：全局安装复制到 `~/.claude/skills/lesy/`（Windows 上是 `%USERPROFILE%\.claude\skills\lesy\`），只在某个项目中使用时复制到那个项目的 `.claude/skills/lesy/`。重新启动 Claude Code 后生效。

```powershell
New-Item -ItemType Directory -Force $env:USERPROFILE\.claude\skills | Out-Null
Copy-Item -Recurse skills\lesy $env:USERPROFILE\.claude\skills\lesy
```

**Codex**：放到 Codex 的 skills 目录，例如用户级的 `~/.codex/skills/lesy/`（Windows 上是 `%USERPROFILE%\.codex\skills\lesy\`）。较新的 Codex 版本也读取 `~/.agents/skills/` 和仓库里的 `.agents/skills/`，以你的 Codex 版本的文档为准。重新启动 Codex 后生效。

更新 Skill 时重新复制一遍。
