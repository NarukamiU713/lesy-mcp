# Lesy MCP：接入 agent 的文档

Lesy 是一个德语阅读助手：读文章时记录表达和备注，写自己的释义，做卡片。这份文档说明怎样让 Claude Code、Codex、Claude Desktop 等 agent 读写**你自己账户**里的这些数据，比如：

- “我这周学了什么？”
- “考考我。”
- “我在哪篇文章里见过 _umsetzen_？”
- “把这个词的释义记下来。”

## 选择接入方式

**优先用命令行 `lesy` 加 Skill**：在浏览器里点一下就登录好，不用复制 Token。只有 agent 不能运行 shell（例如 Claude Desktop），或者你不想装命令行时，才手动复制 Token。

| 方式                              | 适合                                           | 登录方式       |
| --------------------------------- | ---------------------------------------------- | -------------- |
| **命令行 `lesy` + Skill（推荐）** | 能运行 shell 的 agent，例如 Claude Code、Codex | 在浏览器里批准 |
| 手动复制 Token，走 HTTP           | 能直接连 HTTP 的客户端，例如 Claude Code       | 手动复制 Token |
| 手动复制 Token，走 stdio          | 只能启动本地进程的客户端，例如 Claude Desktop  | 手动复制 Token |

三种方式用的是同一个端点（`POST /api/mcp`）、同样的工具和权限。

## 快速开始：命令行 + Skill

前提：

- 手上有一份 Lesy 主仓库（见[这个仓库里有什么](#这个仓库里有什么)），Node.js ≥ 22。
- Lesy 正在运行，并且开启了账户（`LESY_AUTH`，默认开启）。

1. 在主仓库根目录安装命令行：

   ```powershell
   npm install
   npm link
   ```

2. 安装 Skill。下面是 Claude Code 的全局安装，Codex 和其他位置见 [Skill 的说明](skills/lesy/README.md)：

   ```powershell
   New-Item -ItemType Directory -Force $env:USERPROFILE\.claude\skills | Out-Null
   Copy-Item -Recurse skills\lesy $env:USERPROFILE\.claude\skills\lesy
   ```

3. 登录。Lesy 在服务器上时加 `--url https://<你的域名>`：

   ```powershell
   lesy auth login           # 只读：文章、记录、词典
   lesy auth login --write   # 另外允许写入：释义、备注、卡片
   ```

   命令打印一个链接和一个码。在已经登录 Lesy 的浏览器里打开链接，核对码和终端里的一样，点 “Zulassen”。

   这一步也可以交给 agent：第一次使用时它会把链接和码发给你，你批准后告诉它就行。

4. 运行 `lesy auth status` 确认已登录，重新启动 agent，然后问它：“我这周学了什么？”

命令的完整列表见[命令行 `lesy`](docs/cli.md)。

## 其次：手动复制 Token

不用命令行时，在网页上创建 Token，再写进客户端的配置。这种方式走 HTTP 时不需要主仓库。

1. 打开 Lesy，进入 “Profil” → “API-Tokens”，输入名称，勾选权限，点 “Token erstellen”。Token（`lesy_pat_…`）**只显示这一次**，马上复制。
2. 添加服务器。以 Claude Code 为例，本机默认地址是 `http://127.0.0.1:8787`，部署在服务器上时换成 `https://<你的域名>`：

   ```powershell
   claude mcp add --transport http lesy http://127.0.0.1:8787/api/mcp --header "Authorization: Bearer lesy_pat_…"
   ```

3. 运行 `claude mcp list`，确认连接正常。

Claude Desktop 和其他 stdio 客户端见[客户端配置](docs/clients.md#手动复制-token)。

## 工具一览

服务器只提供 Token 有权限的工具。

| 权限         | 工具                                                                                     | 用途                       |
| ------------ | ---------------------------------------------------------------------------------------- | -------------------------- |
| `articles`   | `list_articles`、`get_article`                                                           | 读文章                     |
| `records`    | `learning_history`、`search_records`、`review_quiz`、`list_flashcards`、`get_definition` | 读记录、复习题、卡片、释义 |
| `dictionary` | `lookup_word`                                                                            | 查词典                     |
| `write`      | `set_definition`、`set_expression_note`、`save_flashcards`                               | 写释义、改备注、新建卡片组 |

写工具只新增或修改，**从不删除**任何东西；会覆盖已有值的工具在结果里返回旧值（`previous`），可以用同一个工具改回去。参数和返回值见[工具参考](docs/tools.md)，每个工具对应的命令也在那里。

## 使用前要知道

- **Lesy 要在运行**：agent 连的是正在运行的后端。后端没启动时，`lesy` 的退出码是 5，Claude Code 显示连接失败，stdio 桥对每个请求回答一个说明原因的错误。
- **必须开启账户**：`LESY_AUTH=false` 时没有账户，也就没有 `/api/mcp`（404）。
- **只给 agent 需要的权限**：`lesy auth login` 默认只读，需要写入时才加 `--write`。谁拿到 Token，谁就能用它的权限读写这个账户的数据，所以要像密码一样保管，不要提交到仓库或贴到聊天里；泄露了就在 “Profil” → “API-Tokens” 里立即 “Widerrufen”。
- **只有记录过的表达**：agent 只能看到 “Aufgenommen”，看不到浏览器里的查词记录（“Nachgeschlagen”）。
- **浏览器里没保存的笔记优先**：agent 改了一条备注，而浏览器面板里这条备注还有没保存的修改时，下次 “Aufnahme abschließen” 会用浏览器的版本覆盖它。
- **暂不支持 claude.ai 网页版和手机 connector**：它们需要 OAuth 2.1，还没有实现。见[连接到服务器](docs/remote.md#oauth-21还没有实现)。

## 文档

| 文档                           | 内容                                                      |
| ------------------------------ | --------------------------------------------------------- |
| [命令行 `lesy`](docs/cli.md)   | 安装、登录、全部命令、退出码、配置文件                    |
| [Skill](skills/lesy/README.md) | 给 Claude Code、Codex 的 Skill 怎样安装（`skills/lesy/`） |
| [客户端配置](docs/clients.md)  | 各客户端怎样接入：先是命令行，然后是手动复制 Token        |
| [Token 和权限](docs/tokens.md) | 设备授权、手动创建和吊销 Token、各权限的含义              |
| [工具参考](docs/tools.md)      | 每个工具的命令、参数、返回值、上限和错误码                |
| [连接到服务器](docs/remote.md) | HTTPS、服务器配置、迁移后的登录、OAuth 的计划             |
| [HTTP 协议](docs/protocol.md)  | 不用 SDK 直接调用端点：请求格式、状态码、错误             |

## 这个仓库里有什么

这里只有文档和 Skill，没有可运行的代码。

- 命令行 `lesy` 和 stdio 桥 `lesy-mcp` 在 Lesy 主仓库的 `scripts/` 里。主仓库目前没有公开，所以这两种方式需要你手上有一份主仓库。
- 手动复制 Token 走 HTTP 只需要一个正在运行的 Lesy 和一个 Token，不需要别的东西。

文档对应 Lesy `0.1.2`。
