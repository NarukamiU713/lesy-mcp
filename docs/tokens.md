# Token 和权限

agent 用个人 API Token（`lesy_pat_…`）访问一个账户的数据。有两种方式拿到它：

1. **用命令行 `lesy auth login`，在浏览器里批准（推荐）**：见[设备授权](#设备授权)。
2. 在网页上手动创建，复制到客户端的配置里：见[手动创建](#手动创建)。

两种方式得到的是同一种 Token，都出现在 “Profil” → “API-Tokens” 的列表里，都可以吊销。

## 设备授权

命令行工具 `lesy` 不经过登录表单，也不需要手动复制 Token，流程和 GitHub CLI 一样（[RFC 8628](https://www.rfc-editor.org/rfc/rfc8628)）：

1. `lesy auth login` 打印一个链接和一个码（`XXXX-XXXX`，10 分钟内有效）。
2. 在已经登录 Lesy 的浏览器里打开链接。授权页 `/device` 显示申请的名称和权限。也可以在 Profil 里点 “Gerät mit Code verbinden”，手动输入码。
3. 核对码和终端里的一样，点 “Zulassen”。
4. 命令行拿到一个普通的 API Token，保存在本机。

默认权限是 `articles`、`records`、`dictionary`；加 `--write` 时再加上 `write`，用 `--scope` 可以改选。

agent 不应该自己批准这个请求，也不应该向你要密码或 Token。Skill 里写明了这些规则，见 [`skills/lesy/SKILL.md`](../skills/lesy/SKILL.md)。

## 手动创建

1. 在 “Profil” → “API-Tokens” 中输入一个名称（1–64 个字符，例如 `Claude Desktop`）。
2. 勾选权限，至少选一个。
3. 点 “Token erstellen”。Token **只显示这一次**，马上复制；之后列表里只有名称、权限、创建时间和最后使用时间。

## 权限

| 权限         | 页面上的名称        | 允许                                     | 工具                                                                                     |
| ------------ | ------------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| `articles`   | `Artikel lesen`     | 读文章列表和正文                         | `list_articles`、`get_article`                                                           |
| `records`    | `Lernverlauf lesen` | 读记录的表达、学习历史、卡片和自己的释义 | `learning_history`、`search_records`、`review_quiz`、`list_flashcards`、`get_definition` |
| `dictionary` | `Wörterbuch`        | 查词典                                   | `lookup_word`                                                                            |
| `write`      | `Ändern`            | 改释义和备注、新建卡片组，从不删除       | `set_definition`、`set_expression_note`、`save_flashcards`                               |
| `sync`       | `Synchronisieren`   | 只用于电脑和服务器之间的同步             | 无                                                                                       |

- agent 只看得到有权限的工具。调用其他工具时得到 `Tool … not found`，什么也不执行。
- 只想让 agent 读时不要加 `--write`，手动创建时不要勾选 `Ändern`。
- `write` 本身不包含读的权限。`lesy auth login --write` 会连默认的读权限一起申请；手动创建时要一起勾选。
- 给 agent 用的 Token 不要勾选 `sync`：它没有任何工具权限，只用于同步。
- 权限创建后不能修改。需要更多权限时重新运行 `lesy auth login --write`，或者手动新建一个 Token。

## 使用范围

- Token 属于创建它的账户。
- 它只对 `/api/mcp` 有效（带 `sync` 权限时还有同步接口），不能用来登录网页，也不能访问其他 `/api` 接口。
- 反过来，`/api/mcp` 只认 `Authorization: Bearer <token>`，不认浏览器的 cookie。

## 管理

- 每个 agent 用一个自己的 Token。
- 不用时在 “Profil” → “API-Tokens” 的列表中点 “Widerrufen”，**立即生效**。`lesy auth logout` 只从本机删掉 Token，不会让它失效。
- 修改密码不会吊销 Token。
- 每个账户最多 20 个未吊销的 Token。

服务器端只保存 Token 的 SHA-256 散列，所以丢了只能新建，无法找回。
