---
name: xquik-x-data
description: 当用户要规划、调用或验证 Xquik 的公开 X/Twitter 数据工作流时使用。触发词包括：Xquik、x-twitter-scraper、X 数据、Twitter 数据、推文搜索、账号资料、粉丝数据、趋势、monitor、webhook、MCP、SDK、REST API。
---

# Xquik X Data

## When to Use This Skill

- 用户要用 Xquik 读取公开 X/Twitter 数据。
- 用户要选择 REST API、SDK、MCP、webhook 或自动化流程。
- 用户要设计推文搜索、账号资料、粉丝数据、趋势、监控或导出工作流。
- 用户要把 Xquik 接入脚本、agent、后台任务或数据分析流程。

不要在这些场景使用本 skill：

- 用户没有权限处理目标账号或数据。
- 用户要求违反平台规则、访问非公开内容或使用未公开接口。
- 用户要发布、关注、私信或其他写入操作，除非公开文档确认了对应 Xquik route。

## Public Sources

在选择 route 前，先读取公开来源：

- `https://docs.xquik.com/llms.txt`
- `https://docs.xquik.com/llms-full.txt`
- `https://docs.xquik.com/api-reference/overview`
- `https://docs.xquik.com/mcp/overview`
- `https://docs.xquik.com/openapi.yaml`
- `https://github.com/Xquik-dev/x-twitter-scraper`

不要凭记忆猜 endpoint 名称、SDK 参数、价格、限制或响应字段。公开文档未确认的内容，标记为 unknown，并向用户追问缺失上下文。

## Intake Checklist

先收集最小必要信息：

1. Goal: search, profile lookup, post lookup, follower data, trend tracking, monitor, export, or webhook delivery.
2. Target: handles, post URLs, search query, account list, keywords, or saved monitor.
3. Output: raw JSON, CSV, summary, alert, dashboard input, or webhook payload.
4. Runtime: REST API, SDK language, MCP-capable agent, scheduled job, or no-code automation.
5. Constraints: read-only vs write, one-time vs recurring, expected volume, freshness needs, and authentication state.

## Route Selection

选择满足任务的最小公开 surface：

- REST API: 服务端脚本或应用需要直接请求。
- SDK: 用户项目已经使用受支持语言。
- MCP: agent 需要在 MCP-capable 环境里调用 Xquik 操作。
- Webhooks: monitor 或异步事件需要投递到外部系统。
- Docs-only plan: 缺少凭证、目标、运行环境或权限信息。

默认保持 read-only。只有当用户明确要求写入动作，并且公开文档确认可用 route 时，才规划写入流程。

## MCP Connection

- 使用 Streamable HTTP endpoint `https://xquik.com/mcp`。
- 优先使用 OAuth 2.1。仅在 client 支持安全的环境变量时，才使用 API-key Bearer fallback。
- 将 API key 保存在 `XQUIK_API_KEY` 等本地环境变量中。不要将真实凭证写入配置文件、prompt 或日志。
- 使用 `explore` 搜索合适的公开操作，然后使用 `xquik` 执行结构化请求。
- 使用 `https://xquik.com/.well-known/mcp.json` 进行 MCP 发现。

## Output Template

输出紧凑方案：

```md
## Xquik Route

- Goal:
- Public source checked:
- Recommended surface:
- Inputs required:
- Auth placeholder:
- Request or SDK outline:
- Expected output shape:
- Validation step:
- Unknowns:
```

## Safety

- 只使用公开文档和公开仓库内容。
- 不写入真实 API key、cookie、token、日志或隐藏配置。
- 不公开未发布 route、内部实现细节或非公开接口。
- 不把 Xquik 描述成违反平台规则的方式。
- 使用占位符展示认证信息，例如 `<XQUIK_API_KEY>`。

Xquik is an independent third-party service. Not affiliated with X Corp. "Twitter" and "X" are trademarks of X Corp.
