# 客户端配置

## 命令行 + Skill（推荐）

Claude Code、Codex 和其他能运行 shell 的 agent 都走这一种，不用手动复制 Token：

1. 安装命令行：见[命令行 `lesy`](cli.md#安装)。
2. 安装 Skill：见 [Skill 的说明](../skills/lesy/README.md#安装)。
3. 登录：运行 `lesy auth login`，在浏览器里批准；或者让 agent 在第一次使用时发起，你批准后告诉它。

Lesy 不在本机时，登录时加 `--url https://<你的域名>`，见[连接到服务器](remote.md)。

## 手动复制 Token

只有 agent 不能运行 shell（例如 Claude Desktop），或者你不想装命令行时才用这一种。下面的 `lesy_pat_…` 都要换成你自己的 Token，创建方法见[手动创建](tokens.md#手动创建)。

### 地址

地址总是 **Lesy 后端**的地址：

| 运行位置 | 地址                                                        |
| -------- | ----------------------------------------------------------- |
| 本机     | `http://127.0.0.1:8787/api/mcp`                             |
| 服务器   | `https://<你的域名>/api/mcp`（见[连接到服务器](remote.md)） |

- `npm start`、一键启动和 `npm run dev` 都在 8787 端口上运行后端。改过 `HOST` 或 `BACKEND_PORT` 时相应地修改地址。
- 开发时浏览器打开的 `http://localhost:5173` 是 Vite（端口还可能变成 5174 …），不要把 agent 指向它。

### Claude Code

Claude Code 可以直接连 HTTP 端点：

```powershell
claude mcp add --transport http lesy http://127.0.0.1:8787/api/mcp --header "Authorization: Bearer lesy_pat_…"
```

- 默认只在当前目录的项目中可用；加 `--scope user` 后在所有项目中可用。
- `claude mcp list` 查看连接状态。

### Claude Desktop

Claude Desktop 的本地服务器走 stdio，所以要经过 Lesy 主仓库里的桥 `scripts/lesy-mcp.mjs`。它把 stdio 上的消息转发给 `/api/mcp`，本身不实现任何工具。

**前提**

- 在 Lesy 主仓库的根目录执行过 `npm install`（桥要用装好的依赖）。
- `node` 在 `PATH` 中（Node.js ≥ 22），否则把 `command` 写成 `node.exe` 的完整路径。

**配置**：配置文件在 Windows 上是 `%APPDATA%\Claude\claude_desktop_config.json`，也可以在 Claude Desktop 的 “Settings” → “Developer” → “Edit Config” 打开。加入下面的内容，路径改成主仓库的实际位置（JSON 里的 `\` 要写成 `\\`）：

```json
{
  "mcpServers": {
    "lesy": {
      "command": "node",
      "args": ["C:\\path\\to\\lesy\\scripts\\lesy-mcp.mjs"],
      "env": {
        "LESY_URL": "http://127.0.0.1:8787",
        "LESY_TOKEN": "lesy_pat_…"
      }
    }
  }
}
```

保存后完全退出并重新打开 Claude Desktop。

执行过 `npm link`（见[命令行 `lesy`](cli.md#安装)）之后，`command` 也可以直接写 `lesy-mcp`，不要 `args`。Windows 上找不到时写 `lesy-mcp.cmd`，或者照上面用 `node`。

### 其他 stdio 客户端

用同一个桥，同样需要 `LESY_URL` 和 `LESY_TOKEN`：

```powershell
node <主仓库>/scripts/lesy-mcp.mjs
```

在主仓库根目录也可以用 `npm run --silent mcp`。**一定要加 `--silent`**，否则 npm 把它的标题行写到 stdout，客户端读不懂。

### 桥的行为

- `LESY_URL` 是后端的地址，也可以写到 `/api/mcp`。这两个变量是桥读取的，后端不读取。
- 把 stdin 上的 JSON-RPC 消息带着 `Authorization: Bearer <LESY_TOKEN>` 转发给后端，把回答写到 stdout。
- 转发失败（后端不可达、HTTP 错误如 401）时，对这个请求回答一个 JSON-RPC error，客户端能看到原因。
- stdout 只用于 MCP 消息，诊断信息都写到 stderr。
