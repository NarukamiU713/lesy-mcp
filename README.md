# Lesy MCP：接入 agent 的文档

Lesy 是一个德语阅读助手：读文章时记录表达和备注，写自己的释义，做卡片。这份文档说明怎样让 Claude Code、Claude Desktop、Codex 等 agent 读写**你自己账户**里的这些数据，比如：

- “我这周学了什么？”
- “考考我。”
- “我在哪篇文章里见过 _umsetzen_？”
- “把这个词的释义记下来。”

## 选择接入方式

三种方式用的是同一个端点（`POST /api/mcp`）、同样的工具和权限。

| 方式                      | 适合                                           | 登录方式                 | 需要 Lesy 主仓库      |
| ------------------------- | ---------------------------------------------- | ------------------------ | --------------------- |
| **MCP over HTTP**         | 能直接连 HTTP 的客户端，例如 Claude Code       | 手动复制 Token           | 不需要                |
| **MCP over stdio**        | 只能启动本地进程的客户端，例如 Claude Desktop  | 手动复制 Token           | 需要（桥 `lesy-mcp`） |
| **命令行 `lesy` + Skill** | 能运行 shell 的 agent，例如 Claude Code、Codex | 在浏览器里批准，不用复制 | 需要（命令行 `lesy`） |

不确定时：用 Claude Code 选第一种，用 Claude Desktop 选第二种，用 Codex 选第三种。

## 快速开始（Claude Code）

前提：Lesy 正在运行，并且开启了账户（`LESY_AUTH`，默认开启）。

1. 打开 Lesy，进入 “Profil” → “API-Tokens”，输入名称，勾选权限，点 “Token erstellen”。Token（`lesy_pat_…`）**只显示这一次**，马上复制。
2. 添加服务器。本机默认地址是 `http://127.0.0.1:8787`，部署在服务器上时换成 `https://<你的域名>`：

   ```powershell
   claude mcp add --transport http lesy http://127.0.0.1:8787/api/mcp --header "Authorization: Bearer lesy_pat_…"
   ```

3. 运行 `claude mcp list`，确认连接正常，然后问 agent：“我这周学了什么？”

其他客户端见[客户端配置](docs/clients.md)。

## 工具一览

服务器只提供 Token 有权限的工具。

| 权限         | 工具                                                                                     | 用途                       |
| ------------ | ---------------------------------------------------------------------------------------- | -------------------------- |
| `articles`   | `list_articles`、`get_article`                                                           | 读文章                     |
| `records`    | `learning_history`、`search_records`、`review_quiz`、`list_flashcards`、`get_definition` | 读记录、复习题、卡片、释义 |
| `dictionary` | `lookup_word`                                                                            | 查词典                     |
| `write`      | `set_definition`、`set_expression_note`、`save_flashcards`                               | 写释义、改备注、新建卡片组 |

写工具只新增或修改，**从不删除**任何东西；会覆盖已有值的工具在结果里返回旧值（`previous`），可以用同一个工具改回去。参数和返回值见[工具参考](docs/tools.md)。

## 使用前要知道

- **Lesy 要在运行**：agent 连的是正在运行的后端。后端没启动时，Claude Code 显示连接失败，stdio 桥对每个请求回答一个说明原因的错误。
- **必须开启账户**：`LESY_AUTH=false` 时没有账户，也就没有 `/api/mcp`（404）。
- **Token 就是它勾选的那些权限**：谁拿到它，谁就能用这些权限读写这个账户的数据。只勾选 agent 需要的，像密码一样保管，不要提交到仓库或贴到聊天里；泄露了就立即 “Widerrufen”。
- **只有记录过的表达**：MCP 只能看到 “Aufgenommen”，看不到浏览器里的查词记录（“Nachgeschlagen”）。
- **浏览器里没保存的笔记优先**：agent 用 `set_expression_note` 改了一条备注，而浏览器面板里这条备注还有没保存的修改时，下次 “Aufnahme abschließen” 会用浏览器的版本覆盖它。
- **暂不支持 claude.ai 网页版和手机 connector**：它们需要 OAuth 2.1，还没有实现。见[连接到服务器](docs/remote.md#oauth-21还没有实现)。

## 文档

| 文档                           | 内容                                                      |
| ------------------------------ | --------------------------------------------------------- |
| [客户端配置](docs/clients.md)  | Claude Code、Claude Desktop 和其他 stdio 客户端           |
| [Token 和权限](docs/tokens.md) | 创建和吊销 Token、各权限的含义、设备授权                  |
| [命令行 `lesy`](docs/cli.md)   | 安装、登录、全部命令、退出码、配置文件                    |
| [工具参考](docs/tools.md)      | 每个工具的参数、返回值、上限和错误码                      |
| [HTTP 协议](docs/protocol.md)  | 不用 SDK 直接调用端点：请求格式、状态码、错误             |
| [连接到服务器](docs/remote.md) | HTTPS、服务器配置、迁移后的 Token、OAuth 的计划           |
| [Skill](skills/lesy/README.md) | 给 Claude Code、Codex 的 Skill 怎样安装（`skills/lesy/`） |

## 这个仓库里有什么

这里只有文档和 Skill，没有可运行的代码。

- HTTP 接入只需要一个正在运行的 Lesy 和一个 Token，不需要别的东西。
- 命令行 `lesy` 和 stdio 桥 `lesy-mcp` 在 Lesy 主仓库的 `scripts/` 里，主仓库目前没有公开，所以这两种方式需要你手上有一份主仓库。

文档对应 Lesy `0.1.2`。
