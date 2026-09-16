# ZCode Search MCP 配置教程（web-search-prime）

> 给 ZCode / Codex / MiniMax Code 接入智谱官方联网搜索 MCP `web-search-prime` 的完整配置方案。
> 所有步骤均在本机（Windows 11 + Git Bash）实测通过，配置写法均已对照各客户端实际代码验证。

## 这是什么

`web-search-prime` 是智谱 BigModel 官方提供的联网搜索 MCP 服务（Streamable HTTP 传输），注册后给 Agent 提供一个 `webSearchPrime` 工具：返回网页标题、URL、摘要、站名、图标等搜索结果。

- 官方文档：<https://docs.bigmodel.cn/cn/coding-plan/mcp/search-mcp-server>
- 端点：`https://open.bigmodel.cn/api/mcp/web_search_prime/mcp`
- 旧客户端 SSE 端点：`https://open.bigmodel.cn/api/mcp/web_search_prime/sse?Authorization=YOUR_API_KEY`
- 认证：`Authorization: Bearer <API_KEY>` 请求头
- 工具：仅 `webSearchPrime` 一个

## 前置条件：拿到 API Key

Key 必须是 **GLM Coding Plan 的 API Key**（个人版在 bigmodel.cn 的套餐概览里新建；团队版用团队套餐 Key）。注意：**团队套餐 Key 与平台其他 API Key 不通用**。

> 如果你在用 ZCode 桌面端并登录了 GLM Coding Plan，key 通常已经存在本地：
> `~/.zcode/v2/config.json` → `provider["builtin:bigmodel-coding-plan"].options.apiKey`

### 验证 Key 是否可用

向端点发一个 MCP `initialize` 请求（替换 `YOUR_API_KEY`）：

```bash
curl -s -X POST https://open.bigmodel.cn/api/mcp/web_search_prime/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}'
```

返回 `HTTP 200` 且 `serverInfo` 为 `mcp-web-search-prime` 即可用。

---

## 1. ZCode

**配置文件**（用户级）：`~/.zcode/cli/config.json` → `mcp.servers`，与现有条目并列添加：

```json
{
  "mcp": {
    "servers": {
      "web-search-prime": {
        "type": "http",
        "url": "https://open.bigmodel.cn/api/mcp/web_search_prime/mcp",
        "headers": {
          "Authorization": "Bearer YOUR_API_KEY"
        }
      }
    }
  }
}
```

要点（来自官方 MCP 配置规范）：

- 用标准字段名：`type` / `url` / `headers`；`type` 省略时会按 `url` 自动推断为 `http`
- schema 是**严格校验**的：多余未知键会导致整个 server 被静默丢弃
- 不要把 OpenCode 风格的 `command: ["npx", ...]` 数组粘进来
- 改完**重启会话/重启 ZCode**，然后到 **设置 → MCP** 确认状态为 `connected`

**验证**（可选，程序化方式）：ZCode 的 app-server 提供 `mcp/list` RPC（注意协议不是 JSON-RPC，消息体不要带 `jsonrpc` 字段，`workspaceKey` 直接填 workspacePath）：

```json
{"id":1,"method":"mcp/list","params":{"workspace":{"workspacePath":"<你的工作目录>","workspaceKey":"<同 workspacePath>"}}}
```

预期输出：`"web-search-prime": {"status": "connected", "toolCount": 1}`

---

## 2. Codex（CLI ≥ 0.44，本教程在 0.154.0 实测）

**方式一：CLI 添加 + 手动补 header（推荐）**

```bash
codex mcp add web-search-prime --url https://open.bigmodel.cn/api/mcp/web_search_prime/mcp
```

然后编辑 `~/.codex/config.toml`，在该条目下补 `http_headers`：

```toml
[mcp_servers.web-search-prime]
url = "https://open.bigmodel.cn/api/mcp/web_search_prime/mcp"

[mcp_servers.web-search-prime.http_headers]
Authorization = "Bearer YOUR_API_KEY"
```

**方式二：bearer token 走环境变量**（`codex mcp add` 支持 `--bearer-token-env-var`，适合不想把 key 写进配置文件的场景）。

**验证**：

```bash
codex mcp get web-search-prime
# 预期：enabled: true / transport: streamable_http / Auth: Bearer token
codex mcp list
```

---

## 3. MiniMax Code

**配置文件**：`~/.minimax/mcp/mcp.json` → `mcpServers`，与现有条目并列添加：

```json
{
  "mcpServers": {
    "web-search-prime": {
      "url": "https://open.bigmodel.cn/api/mcp/web_search_prime/mcp",
      "type": "streamable-http",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      },
      "enabled": true,
      "configured": true,
      "description": "BigModel web-search-prime MCP: live web search"
    }
  }
}
```

要点：

- MiniMax Code 的配置校验器对 `streamable-http` 类型只接受 `transport`/`type`、`url`、`headers`、`enabled`、`timeoutMs`、`description` 这几个字段，非 stdio 传输会读取 `headers` 做认证
- 没有可热测的 MCP CLI，**下次启动 MiniMax Code 生效**，在应用内的 MCP 管理界面确认连接状态

---

## 故障排查

| 症状 | 原因与处理 |
|---|---|
| ZCode 设置里 server 不出现 | JSON 语法错误或 schema 严格校验把条目丢了——检查多余字段、字段拼写 |
| 状态 `failed` / 401 | Key 错误或过期；重新用上面的 curl 探针验证 key |
| 能连上但调用报错 | GLM Coding Plan 套餐过期，或用的是平台通用 Key 而非 Coding Plan Key |
| 改了配置没生效 | 重启对应客户端会话；MiniMax Code 需重启应用 |
| Key 轮换后部分客户端失效 | 同一个 key 被写进了多个客户端，**三个地方要一起改** |

## 安全提醒

- Key 不要写进任何会公开的文档/仓库（本教程所有配置均为 `YOUR_API_KEY` 占位符）
- HTTP 型 MCP 的 key 明文存在配置文件里，注意备份文件的权限与同步范围

## 参考

- BigModel 搜索 MCP 官方文档：<https://docs.bigmodel.cn/cn/coding-plan/mcp/search-mcp-server>
- ZCode MCP 配置文档：<https://zcode.z.ai/cn/docs/mcp-services>
- Codex MCP 子命令：`codex mcp add --help`
