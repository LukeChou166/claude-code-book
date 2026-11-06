# 第9章 MCP 协议集成：开放的工具生态

> **前置知识**：第4章 工具系统、第1章 整体架构与启动  
> **关键文件**：`src/services/mcp/client.ts`（约3400行）、`src/services/mcp/MCPConnectionManager.tsx`、`src/tools/MCPTool/MCPTool.ts`

---

## 9.1 解决什么问题

Claude Code 内置了约 40 个工具（BashTool、FileReadTool 等），但这些工具无法覆盖所有场景：

- **访问内部系统**：公司的内部 API、代码审查平台、CI/CD 系统
- **专业领域工具**：数据库查询、图表生成、CAD 工具
- **外部服务集成**：GitHub、Jira、Notion、Figma
- **私有数据源**：公司内网文档、私有 npm registry

不可能把每个可能的集成都内置到 Claude Code 里。需要一个**标准化的扩展协议**，让第三方开发者无需修改 Claude Code 源码就能提供新工具。

MCP（Model Context Protocol）就是这个协议。

---

## 9.2 MCP 是什么

**Model Context Protocol** 是 Anthropic 开源的标准化协议，定义了 AI 助手与外部工具/服务之间的通信接口。

可以把它类比为 HTTP 协议：
- HTTP 让浏览器和服务器可以互相通信，而不需要浏览器了解每个网站的内部实现
- MCP 让 AI 助手和工具服务器可以互相通信，而不需要 AI 助手了解每个工具的实现细节

**MCP Server**：实现 MCP 协议、提供工具或资源的服务
**MCP Client**（Claude Code）：连接到 MCP Server，使用其提供的工具

---

## 9.3 传输层：三种连接方式

```typescript
// src/services/mcp/client.ts 第 8-17 行
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'
import { WebSocketTransport } from '../../utils/mcpWebSocketTransport.js'
```

Claude Code 支持四种传输方式：

### 方式一：Stdio（本地子进程）

最常见的方式。Claude Code 启动一个子进程，通过 stdin/stdout 通信：

```json
// .claude/settings.json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/projects"],
      "env": {
        "NODE_ENV": "production"
      }
    }
  }
}
```

**优点**：简单可靠，进程隔离，Claude Code 崩溃不影响 MCP Server（反之亦然）  
**缺点**：只能在本地运行，不能跨机器访问  
**适用**：大多数本地工具（数据库驱动、文件系统扩展、自定义脚本）

### 方式二：SSE（HTTP 服务端发送事件）

```json
{
  "mcpServers": {
    "remote-tool": {
      "type": "sse",
      "url": "https://your-mcp-server.example.com/sse"
    }
  }
}
```

**优点**：可以跨网络访问，适合部署在服务器上的工具  
**缺点**：SSE 是单向（服务端→客户端），请求通过普通 HTTP POST 发出  
**适用**：云服务工具（GitHub 集成、Jira 等 SaaS 服务）

### 方式三：StreamableHTTP（新版协议）

```json
{
  "mcpServers": {
    "advanced-tool": {
      "type": "http",
      "url": "https://mcp-server.example.com/mcp"
    }
  }
}
```

MCP 协议的新版本，支持双向流式通信，比 SSE 更高效。

### 方式四：WebSocket

通过 WebSocket 长连接实时双向通信，适合需要服务器主动推送更新的场景。

---

## 9.4 MCP 工具的发现与注册

MCP Server 在启动时声明它提供哪些工具：

```typescript
// MCP 工具发现流程（简化）
async function connectToMCPServer(connection: MCPServerConfig): Promise<MCPToolList> {
  const client = new Client(/* ... */)
  await client.connect(transport)
  
  // 调用 MCP 标准接口获取工具列表
  const { tools } = await client.listTools()
  
  // 把 MCP 工具包装成 Claude Code 的 Tool 接口
  return tools.map(mcpTool => createMCPToolWrapper(mcpTool, client))
}
```

每个 MCP 工具被包装为一个 `MCPTool` 对象，实现了 Claude Code 的 `Tool` 接口：

```typescript
// src/tools/MCPTool/MCPTool.ts（简化）
export class MCPTool implements Tool {
  constructor(
    private mcpToolDef: MCPToolDefinition,
    private mcpClient: Client,
    private serverName: string,
  ) {}

  get name() {
    // 命名格式：mcp__<服务器名>__<工具名>
    // 例：mcp__filesystem__read_file
    return `mcp__${this.serverName}__${this.mcpToolDef.name}`
  }

  async call(input: unknown) {
    // 转发调用给 MCP Server
    const result = await this.mcpClient.callTool({
      name: this.mcpToolDef.name,
      arguments: input,
    })
    return { data: result }
  }

  // 把 MCP 的 JSON Schema 转换为 Claude Code 的 Zod schema
  get inputSchema() {
    return zodSchemaFromJsonSchema(this.mcpToolDef.inputSchema)
  }
}
```

命名约定 `mcp__<服务器名>__<工具名>` 有两个好处：
1. **避免命名冲突**：同名工具来自不同 MCP Server 不会冲突
2. **便于权限配置**：可以用 `mcp__filesystem__*` 这样的 glob 规则控制整个 MCP Server 的权限

---

## 9.5 MCP Resources：数据访问标准

MCP 除了工具（Tools），还有 Resources 概念：

```typescript
// src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts
export const ListMcpResourcesTool = buildTool({
  name: 'ListMcpResources',
  description: 'List available resources from MCP servers',
  // ...
})

// src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts
export const ReadMcpResourceTool = buildTool({
  name: 'ReadMcpResource',
  description: 'Read a resource from an MCP server by URI',
  // 支持的 URI 格式：mcp://server-name/resource-path
  // 例：mcp://filesystem/projects/myapp/README.md
  // ...
})
```

**Resource 和 Tool 的区别**：
- **Tool**：执行操作，有副作用（类比 HTTP POST）
- **Resource**：读取数据，无副作用（类比 HTTP GET）

Resource 的好处是 AI 可以更安全地使用它们（不需要严格的权限检查），类似于只读文件系统访问。

---

## 9.6 OAuth 认证

MCP Server 可能需要 OAuth 认证（比如访问 GitHub API 的 MCP Server 需要用户授权）：

```typescript
// src/services/mcp/auth.ts（简化）
export async function handleMCPOAuth(
  serverUrl: string,
  scope: string[],
): Promise<OAuthTokens> {
  // 1. 在本地启动一个临时 HTTP 服务器监听回调
  const { callbackUrl, port } = await startOAuthCallbackServer()
  
  // 2. 打开浏览器让用户授权
  const authUrl = buildOAuthAuthorizationUrl({
    serverUrl,
    redirectUri: callbackUrl,
    scope,
  })
  await openBrowser(authUrl)
  
  // 3. 等待用户授权完成，接收 callback
  const { code } = await waitForOAuthCallback(port)
  
  // 4. 用授权码换取 access token
  const tokens = await exchangeCodeForTokens({ code, ... })
  
  // 5. 存储 token（Keychain 或配置文件）
  await storeOAuthTokens(serverUrl, tokens)
  
  return tokens
}
```

后续请求时，token 会自动附加到 HTTP 请求头中，支持 token 刷新（OAuth refresh token 流程）。

---

## 9.7 MCPConnectionManager：连接生命周期管理

`MCPConnectionManager.tsx` 负责管理所有 MCP 连接的生命周期：

```typescript
// src/services/mcp/MCPConnectionManager.tsx（简化）
export function MCPConnectionManager({
  mcpConfig,
  onConnectionsUpdated,
}: MCPConnectionManagerProps) {
  
  const [connections, setConnections] = useState<MCPConnection[]>(
    mcpConfig.servers.map(s => ({ ...s, type: 'pending' }))
  )

  useEffect(() => {
    // 并行连接所有 MCP Server
    const connectPromises = mcpConfig.servers.map(async server => {
      try {
        const client = await connectToServer(server)
        const tools = await client.listTools()
        setConnections(prev => updateConnection(prev, server.name, { type: 'connected', tools }))
      } catch (error) {
        // 连接失败不是致命错误，只标记为失败状态
        setConnections(prev => updateConnection(prev, server.name, {
          type: 'failed',
          error: error.message
        }))
      }
    })
    
    // 不等待所有连接完成——先显示已连接的
    // 连接状态会流式更新到 UI
  }, [mcpConfig])
  
  return null  // 纯逻辑组件，不渲染 UI
}
```

连接状态有三种：
- `pending`：正在连接
- `connected`：连接成功，工具已注册
- `failed`：连接失败（MCP Server 不可用）

在 UI 中，`pending` 状态的 MCP Server 的工具不可用；`failed` 状态会显示错误信息，但不阻止 Claude Code 正常使用。

---

## 9.8 工具结果的大小处理

MCP Server 返回的工具结果可能非常大（比如数据库查询返回几 MB 的数据）：

```typescript
// src/utils/mcpValidation.ts（简化）
const MAX_MCP_CONTENT_CHARS = 20_000

export function mcpContentNeedsTruncation(content: MCPContent[]): boolean {
  return getContentSizeEstimate(content) > MAX_MCP_CONTENT_CHARS
}

export function truncateMcpContentIfNeeded(
  content: MCPContent[],
  maxChars: number = MAX_MCP_CONTENT_CHARS,
): { content: MCPContent[]; wasTruncated: boolean } {
  if (!mcpContentNeedsTruncation(content)) {
    return { content, wasTruncated: false }
  }
  // 截断文本块，保留最重要的前 N 字符
  // 并添加截断提示："[结果已截断，原始大小：X 字符]"
  return { content: truncated, wasTruncated: true }
}
```

对于二进制内容（如图片），会持久化到临时文件：

```typescript
// src/utils/mcpOutputStorage.ts（简化）
export async function persistBinaryContent(
  content: BinaryMCPContent,
): Promise<string> {
  const fileName = `mcp_output_${Date.now()}.${getExtension(content.mimeType)}`
  const filePath = path.join(os.tmpdir(), fileName)
  await writeFile(filePath, Buffer.from(content.data, 'base64'))
  return filePath
}
```

---

## 9.9 Elicitation（信息收集）

MCP 协议新增了 Elicitation 功能，允许 MCP Server 向用户**请求额外信息**：

```typescript
// src/services/mcp/elicitationHandler.ts
// 当 MCP Server 发送 elicit 请求时（比如"需要用户输入 API Key"）
// Claude Code 弹出一个 UI 对话框收集信息，然后把结果返回给 MCP Server
```

这使得 MCP Server 可以实现交互式工作流，比如：
- "你想导出为 PDF 还是 DOCX 格式？"
- "请输入数据库连接字符串："

---

## 9.10 官方 MCP 服务器目录

```typescript
// src/services/mcp/officialRegistry.ts
// Anthropic 维护一个官方认可的 MCP Server 目录
// 用户可以通过 /mcp install <server-name> 命令快速安装

export async function fetchOfficialMCPServers(): Promise<OfficialMCPServer[]> {
  // 从 Anthropic 的官方 API 获取经过验证的 MCP Server 列表
  const response = await fetch(OFFICIAL_MCP_REGISTRY_URL)
  return response.json()
}
```

Claude.ai 官网上的 MCP Server 目录和 Claude Code 的安装流程共享同一套数据，保持生态一致性。

---

## 9.11 多数据来源的环境变量展开

MCP Server 配置中的环境变量支持引用外部来源：

```typescript
// src/services/mcp/envExpansion.ts（简化）
export async function expandEnvVars(
  env: Record<string, string>,
): Promise<Record<string, string>> {
  const expanded: Record<string, string> = {}
  for (const [key, value] of Object.entries(env)) {
    // 支持三种格式：
    // 普通环境变量：$HOME, $CUSTOM_VAR
    expanded[key] = value.replace(/\$(\w+)/g, (_, name) => process.env[name] ?? '')
    
    // Keychain 引用（macOS）：${KEYCHAIN:my-secret}
    if (value.startsWith('${KEYCHAIN:')) {
      const secretName = value.slice(11, -1)
      expanded[key] = await getFromKeychain(secretName)
    }
    
    // 1Password 引用：op://Vault/Item/Field
    if (value.startsWith('op://')) {
      expanded[key] = await getFromOnePassword(value)
    }
  }
  return expanded
}
```

这让用户可以把 API Key 安全存储在 macOS Keychain 或 1Password 中，而不是明文写在 settings.json 里。

---

## 9.12 优势与不足

### 优势

**标准化生态**：任何人都可以按照 MCP 规范开发工具服务，与 Claude Code 的配合和与其他支持 MCP 的 AI 助手（VS Code Copilot、Zed 等）是一样的——一次实现，多处使用。

**多传输层支持**：本地进程（stdio）、HTTP SSE、WebSocket 都支持，覆盖了从"我的个人脚本"到"团队共享的云服务"的所有场景。

**连接失败不阻塞**：MCP Server 连接失败不影响 Claude Code 正常使用，只是该服务器的工具不可用。

**工具结果自动截断**：防止 MCP Server 返回超大结果把上下文窗口填满。

**OAuth 集成**：内置 OAuth 流程让访问需要授权的 SaaS 服务（GitHub、Google Drive 等）变得无缝。

### 不足

**`client.ts` 约 3400 行**：是整个项目最大的单个文件，包含了传输层、认证、工具调用、结果处理、重连逻辑等大量职责，极难阅读和维护。

**MCP Server 的工具 schema 必须是 JSON Schema 格式**：Claude Code 内部用 Zod，需要在运行时把 JSON Schema 转换为 Zod schema（`zodSchemaFromJsonSchema` 函数）。这个转换不完整，某些高级 JSON Schema 特性（比如 `$ref`、`allOf`）可能无法正确转换。

**没有 MCP Server 版本管理**：如果 MCP Server 更新了工具的 schema（比如新增了必填参数），Claude Code 不会自动检测，只有在调用失败时才会报错。没有类似 npm package.json 的版本约束机制。

**性能开销**：每次工具调用都需要跨进程（stdio）或跨网络（SSE/HTTP）通信，相比内置工具（直接函数调用）有额外的延迟。对于高频工具调用的场景，这个开销会累积。
