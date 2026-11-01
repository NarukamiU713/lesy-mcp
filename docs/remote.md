# 连接到服务器

Lesy 部署在服务器上时，agent 的配置只有地址不同：

| 方式        | 配置                                                       |
| ----------- | ---------------------------------------------------------- |
| Claude Code | `https://<你的域名>/api/mcp`                               |
| stdio 桥    | `LESY_URL=https://<你的域名>`                              |
| 命令行      | `lesy auth login --url https://<你的域名>`，之后会记住地址 |

**只用 HTTPS**，不要让 Token 走明文的 HTTP。

## 服务器配置

- **`FRONTEND_ORIGIN=https://<你的域名>`：必须设。** `/api` 的 Host 检查只接受 `localhost`、IP 地址、`HOST` 和 `FRONTEND_ORIGIN` 的主机名。不设的话，经过域名的每个 `/api` 请求（包括 `/api/mcp`）都是 403。
- **`TRUST_PROXY`**：在反向代理后面时要设，否则设备授权给出的链接不是 `https`。
- **`LESY_AUTH`**：保持默认的 `true`，否则没有 `/api/mcp`。

## 迁移之后

本机的 Token 随账户数据库一起迁到服务器后仍然有效，但地址变了：

1. 把 Claude Code 和 Claude Desktop 的配置改成服务器的地址。
2. 给每个客户端发一个新 Token，吊销旧的。

## OAuth 2.1（还没有实现）

现在的 API Token 够 Claude Code 和 Claude Desktop 这类客户端用。claude.ai 网页版和手机上的远程 connector 要走 MCP 授权规范的 OAuth 2.1，**目前连不上**。

要支持它们，Lesy 还需要做这些：

- Protected Resource Metadata（RFC 9728）：`/.well-known/oauth-protected-resource`，401 的 `WWW-Authenticate` 加上 `resource_metadata`。
- 授权服务器元数据（RFC 8414）：`/.well-known/oauth-authorization-server`。
- 动态客户端注册（RFC 7591）：`POST /register`。
- 授权页 `/authorize`：用现有的登录会话，显示客户端和权限；必须 PKCE（S256），校验 `redirect_uri` 和 `resource`（RFC 8707）。
- Token 端点 `/token`：授权码换短期的 access token 和轮换的 refresh token，可以在 Profil 里看到和吊销。
- 端点同时接受 API Token 和 OAuth access token，并且 token 的 audience 必须是这个 MCP 端点。
