# HTTP 协议

用 MCP 客户端或 SDK 时不用关心这些；这一页给想直接调用端点的人。

Lesy 用 [MCP](https://modelcontextprotocol.io) 的 Streamable HTTP 传输，端点是 `POST /api/mcp`。

## 请求

| 项目            | 要求                                                   |
| --------------- | ------------------------------------------------------ |
| `Authorization` | `Bearer lesy_pat_…`                                    |
| `Content-Type`  | `application/json`                                     |
| `Accept`        | 必须同时包含 `application/json` 和 `text/event-stream` |
| 请求体          | JSON-RPC 消息，最大 1 MiB                              |

```bash
curl -X POST http://127.0.0.1:8787/api/mcp \
  -H 'Authorization: Bearer lesy_pat_…' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

## 无状态

- 每个请求单独处理，请求之间什么也不保留，也没有 `Mcp-Session-Id`。
- 不先 `initialize` 也能直接 `tools/call`。
- 响应总是一个 JSON，从不开 SSE 流；只有通知时返回 202。

## 状态码

| 状态码 | 情况                                                                               |
| ------ | ---------------------------------------------------------------------------------- |
| 200    | 正常的 JSON-RPC 响应（工具失败也是 200，结果里有 `isError: true`）                 |
| 202    | 只有通知                                                                           |
| 401    | 没有或无效的 Token（只带 cookie 也是），带 `WWW-Authenticate: Bearer realm="lesy"` |
| 403    | 请求的 Host 不被接受（见[连接到服务器](remote.md#服务器配置)）                     |
| 404    | 账户没有开启（`LESY_AUTH=false`）                                                  |
| 405    | `GET` 或 `DELETE`，带 `Allow: POST`                                                |
| 406    | `Accept` 不符合要求                                                                |
| 415    | `Content-Type` 不是 JSON                                                           |
| 503    | `storage_unavailable`：账户的数据库打不开                                          |

## 权限

服务器只注册 Token 有权限的工具：

- `tools/list` 只列出这些工具。
- 调用其他工具时得到错误 `Tool … not found`，什么也不执行。
- `initialize` 返回的 `instructions` 说明这是哪个账户的数据，以及这个 Token 被授予了哪些权限。
- 只有 `sync` 权限的 Token，`tools/list` 为空。
