# 命令行 `lesy`

这是推荐的接入方式：`lesy` 给能运行 shell 的 agent 和你自己用，在浏览器里批准就能登录，不用复制 Token。配上 [Skill](../skills/lesy/README.md)，agent 就知道什么时候用哪个命令。

它的所有数据命令都是对 `/api/mcp` 的调用，所以工具、权限和上限都和 MCP 相同；参数校验和“从不删除”的规则都在后端。

## 安装

`lesy` 在 Lesy 主仓库里。在主仓库根目录（执行过 `npm install` 之后）执行一次：

```powershell
npm link
```

之后 `lesy` 和 `lesy-mcp` 在任何目录都可用。仓库移到别处后重新执行。

## 登录

```powershell
lesy auth login           # 默认权限 articles、records、dictionary
lesy auth login --write   # 再加上写入（释义、笔记、卡片）
```

命令打印一个链接（`http://127.0.0.1:8787/device?user_code=…`）和一个码，然后等你批准，步骤见[设备授权](tokens.md#设备授权)。

| 选项            | 作用                                                                    |
| --------------- | ----------------------------------------------------------------------- |
| `--url URL`     | 后端地址。没有时依次取已保存的配置、`LESY_URL`、`http://127.0.0.1:8787` |
| `--scope a,b,c` | 改选权限                                                                |
| `--write`       | 再加上 `write` 权限                                                     |
| `--name NAME`   | Token 的名称                                                            |
| `--open`        | 自动用浏览器打开链接                                                    |
| `--no-wait`     | 只申请、打印链接和码，不等待（给 agent 用）                             |

**agent 的流程**：运行 `lesy auth login --no-wait`，把链接和码交给你；你批准后告诉它，它再运行 `lesy auth complete`。`complete` 默认最多等 120 秒（`--timeout`），还没批准时退出码是 4，10 分钟内可以再运行。

其他登录命令：

- `lesy auth status`：显示地址、Token 前缀、权限和可用的工具数。
- `lesy auth logout`：从本机删掉 Token。要让它失效，还要在 “Profil” → “API-Tokens” 里吊销。

## 命令

完整的列表和选项见 `lesy --help`。

| 命令                                                                         | 工具                        |
| ---------------------------------------------------------------------------- | --------------------------- |
| `lesy articles [--limit N] [--offset N]`                                     | `list_articles`             |
| `lesy article <id> [--offset N] [--max-chars N]`                             | `get_article`               |
| `lesy history [--period P \| --from YYYY-MM-DD [--to YYYY-MM-DD]]`           | `learning_history`          |
| `lesy search <query...>`                                                     | `search_records`            |
| `lesy quiz [--period P \| --from … [--to …]] [--count N]`                    | `review_quiz`               |
| `lesy flashcards [--limit N] [--offset N]`                                   | `list_flashcards`           |
| `lesy lookup <word> [--context TEXT]`                                        | `lookup_word`               |
| `lesy definition <word>`                                                     | `get_definition`            |
| `lesy set-definition <word> --meaning TEXT [--part-of-speech …] [--notes …]` | `set_definition`（写）      |
| `lesy set-note <articleId> <expressionId> <note>`                            | `set_expression_note`（写） |
| `lesy save-flashcards <articleId> --file <cards.json\|->`                    | `save_flashcards`（写）     |
| `lesy tools`                                                                 | 列出 Token 可用的工具       |
| `lesy call <tool> [<json-args>] [--args-file PATH\|-]`                       | 直接调用任意工具            |

时间段 `P`：`today`、`yesterday`、`this_week`、`last_week`、`last_7_days`、`last_30_days`、`this_month`、`last_month`、`this_year`。

例子：

```powershell
lesy history --period this_week          # 这周记录的表达
lesy search umsetzen                     # 在记录里找一个词，以及它出现在哪篇文章
lesy lookup Umsetzung                    # 查词典
lesy quiz --period last_7_days --count 6 # 复习题（带答案）
lesy articles --limit 5 --json           # 结构化 JSON，给脚本和 agent
```

写入的命令需要用 `--write` 登录。没有 `write` 权限时得到 `Tool … not found`（退出码 1）。

## 输出和退出码

每个命令都可以加这些选项：

- `--json`：输出结构化的 JSON。
- `--url URL`：这一次使用别的后端。
- `-h`、`--help`：显示帮助。

stdout 只有结果，提示、进度和错误都写到 stderr，所以结果可以直接交给管道或 agent 解析。任何输出都不含完整的 Token。

| 退出码 | 含义                                           |
| ------ | ---------------------------------------------- |
| 0      | 成功                                           |
| 1      | 工具错误                                       |
| 2      | 用法或配置错误                                 |
| 3      | 没有登录、Token 被拒绝或授权被拒绝（包括过期） |
| 4      | `lesy auth complete` 超时                      |
| 5      | 后端连不上                                     |

## 配置文件

| 系统    | 位置                                                                |
| ------- | ------------------------------------------------------------------- |
| Windows | `%APPDATA%\lesy\config.json`                                        |
| 其他    | `~/.config/lesy/config.json`（设置了 `XDG_CONFIG_HOME` 时在它下面） |

也可以用环境变量 `LESY_CONFIG` 指定别的路径。文件里有 Token，像密码一样保管。

环境变量 `LESY_URL` 和 `LESY_TOKEN` 优先于配置文件。
