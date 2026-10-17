# Craft Agents 第二个简历点：WebSocket RPC 协议 + 三层权限系统

## 一、技术架构概述

Craft Agents 是 Electron 桌面应用，分两个进程：
- **主进程** (Main Process)：Node.js 环境，能访问文件系统、执行 shell 命令
- **渲染进程** (Renderer Process)：浏览器环境，显示 UI，受限于浏览器沙箱

UI 需要调用文件读写、命令执行等功能，但浏览器环境做不了。所以需要：
1. **WebSocket RPC 协议**：渲染进程通过 WebSocket 调用主进程的功能
2. **三层权限系统**：主进程在执行危险操作前检查权限模式

---

## 二、WebSocket RPC 协议

### 2.1 为什么用 WebSocket？

**Electron 的 IPC 限制**：
- Electron 自带的 `ipcRenderer` / `ipcMain` 只能用于本地进程间通信
- 无法支持远程访问（比如从浏览器访问本地运行的 Craft Agents）

**WebSocket 的优势**：
- **双向通信**：服务器可以主动推送事件到客户端（比如任务进度更新）
- **跨平台**：CLI、Web UI、Electron 都能用同一套协议
- **可扩展**：支持远程模式（通过 TLS 和 token 认证）

### 2.2 消息格式（MessageEnvelope）

**核心数据结构** (`packages/shared/src/protocol/types.ts:20-63`)：

```typescript
interface MessageEnvelope {
  id: string;              // 消息 ID (UUID)
  type: MessageType;       // 消息类型
  channel?: string;        // 调用的方法名（如 "tool:bash"）
  args?: unknown[];        // 请求参数
  result?: unknown;        // 响应结果
  error?: WireError;       // 错误信息
  seq?: number;            // 序列号（用于可靠投递）
  // ... 其他字段
}
```

**7 种消息类型** (`types.ts:11-18`)：

| 类型 | 方向 | 作用 | 举例 |
|------|------|------|------|
| `handshake` | 客户端 → 服务器 | 连接时握手 | `{type: "handshake", workspaceId: "abc"}` |
| `handshake_ack` | 服务器 → 客户端 | 确认握手 | `{type: "handshake_ack", clientId: "xyz"}` |
| `request` | 客户端 → 服务器 | 调用方法 | `{type: "request", channel: "tool:bash", args: [{command: "ls"}]}` |
| `response` | 服务器 → 客户端 | 返回结果 | `{type: "response", result: "file1\nfile2"}` |
| `event` | 服务器 → 客户端 | 推送事件 | `{type: "event", channel: "session:update", args: [...]}` |
| `error` | 服务器 → 客户端 | 错误响应 | `{type: "error", error: {code: "...", message: "..."}}` |
| `sequence_ack` | 客户端 → 服务器 | 确认收到事件 | `{type: "sequence_ack", lastSeq: 123}` |

### 2.3 完整调用流程

**代码位置**：`packages/server-core/src/transport/server.ts`

#### 步骤 1：客户端发起请求

```typescript
// 渲染进程
const ws = new WebSocket('ws://localhost:8080');

ws.send(JSON.stringify({
  id: "uuid-1234",
  type: "request",
  channel: "tool:bash",
  args: [{ command: "ls -la" }]
}));
```

#### 步骤 2：服务器接收消息

**代码** (`server.ts:609-628`)：

```typescript
ws.on('message', async (data) => {
  const envelope = deserializeEnvelope(data);
  
  if (envelope.type === 'request') {
    await this.onRequest(client, envelope);
  } else if (envelope.type === 'response') {
    this.onClientResponse(envelope);
  } else if (envelope.type === 'sequence_ack') {
    // 客户端确认收到事件，清理缓存
    client.lastAckedSeq = envelope.lastSeq;
  }
});
```

#### 步骤 3：路由到处理函数

**代码** (`server.ts:642-683`)：

```typescript
private async onRequest(client: ClientConnection, envelope: MessageEnvelope) {
  const { channel, id, args } = envelope;
  
  // 1. 检查 channel 是否存在
  if (!channel) {
    return this.sendResponseError(id, 'CHANNEL_NOT_FOUND', 'Missing channel');
  }
  
  // 2. 查找对应的处理函数
  const handler = this.handlers.get(channel);
  //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  //     例如: channel = "tool:bash" → 查找 bashToolHandler
  
  if (!handler) {
    return this.sendResponseError(id, 'CHANNEL_NOT_FOUND', `No handler for: ${channel}`);
  }
  
  // 3. 构造请求上下文
  const ctx: RequestContext = {
    clientId: client.id,
    workspaceId: client.workspaceId,
    webContentsId: client.webContentsId,
  };
  
  // 4. 调用处理函数（60 秒超时）
  try {
    const result = await Promise.race([
      handler(ctx, ...args),
      //     ^^^^^^^^^^^ 在这里执行实际的工具调用
      new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Handler timeout')), 60_000)
      ),
    ]);
    
    // 5. 返回成功响应
    const response: MessageEnvelope = {
      id,
      type: 'response',
      channel,
      result,
    };
    this.safeSend(client.ws, serializeEnvelope(response));
    
  } catch (err) {
    // 6. 返回错误响应
    const code = isErrorCode(err.code) ? err.code : 'HANDLER_ERROR';
    this.sendResponseError(id, channel, code, err.message);
  }
}
```

#### 步骤 4：返回结果

```typescript
// 服务器发送响应
ws.send(JSON.stringify({
  id: "uuid-1234",
  type: "response",
  channel: "tool:bash",
  result: "total 48\ndrwxr-xr-x  12 user  staff  384 Jan 15 10:30 .\n..."
}));

// 客户端收到响应
ws.onmessage = (event) => {
  const envelope = JSON.parse(event.data);
  if (envelope.type === 'response') {
    console.log(envelope.result);  // 输出命令结果
  }
};
```

### 2.4 事件推送（服务器 → 客户端）

WebSocket 是双向的，服务器可以主动推送事件：

**代码** (`server.ts:180-202`)：

```typescript
broadcast(channel: string, ...args: any[]): void {
  const timestamp = Date.now();
  
  for (const [clientId, client] of this.clients) {
    // 给每个连接的客户端发送事件
    this.bufferAndMaybeSendEvent(client, channel, args, timestamp);
  }
}
```

**使用场景**：
- 任务进度更新：`broadcast('task:progress', { taskId, progress: 0.5 })`
- Session 状态变化：`broadcast('session:update', { sessionId, messages: [...] })`
- 权限模式切换：`broadcast('permission:mode-changed', { mode: 'ask' })`

---

## 三、三层权限系统

### 3.1 三种权限模式

**定义位置**：`packages/shared/src/agent/mode-types.ts:24-43`

```typescript
type PermissionMode = 'safe' | 'ask' | 'allow-all';

// 用户看到的名字（映射）
'safe'      → 'Explore'  (探索模式)
'ask'       → 'Ask'      (询问模式)
'allow-all' → 'Execute'  (执行模式)
```

**三种模式的行为** (`mode-types.ts:304-340`)：

| 内部名称 | 用户看到的名字 | 行为描述 | 图标 | 用途 |
|---------|---------------|----------|------|------|
| `safe` | **Explore** (探索) | 只读探索，阻止写操作，**从不提示** | 🧭 指南针 | 安全探索代码，AI 只能读取、搜索、查看 |
| `ask` | **Ask** (询问) | 执行前**弹窗询问**用户 | ℹ️ 信息 | 默认模式，危险操作需要用户确认 |
| `allow-all` | **Execute** (执行) | 自动执行，**不提示** | 🔁 循环 | 完全信任 AI，适合自动化任务 |

**模式切换**：用户按 `SHIFT+TAB` 循环切换三种模式。

### 3.2 Safe 模式的硬编码阻止列表

**代码** (`mode-types.ts:265-283`)：

```typescript
const SAFE_MODE_CONFIG: ModeConfig = {
  // 永远阻止的工具（无读-only 变体）
  blockedTools: new Set([
    'Write',        // 写文件
    'Edit',         // 编辑文件
    'MultiEdit',    // 批量编辑
    'NotebookEdit', // Jupyter 笔记本编辑
  ]),
  
  // Bash 只读命令白名单（从 ~/.craft-agent/permissions/default.json 加载）
  readOnlyBashPatterns: [],
  
  // MCP 工具白名单（从配置加载）
  readOnlyMcpPatterns: [],
  
  displayName: 'Explore',
  shortcutHint: 'SHIFT+TAB',
};
```

**为什么这四个工具永远被阻止？**
- 它们是**根本性的写操作**，没有"只读版本"
- 即使用户自定义配置文件，也无法在 Explore 模式下解除这四个工具的阻止

### 3.3 权限检查核心逻辑

**代码位置**：`packages/shared/src/agent/mode-manager.ts:1803-1932`

```typescript
function shouldAllowToolInMode(
  toolName: string,
  toolInput: unknown,
  mode: PermissionMode,
  options?: {
    plansFolderPath?: string;
    dataFolderPath?: string;
    permissionsContext?: PermissionsContext;
  }
): ToolCheckResult {
  
  // ============================================================
  // 1. allow-all 模式：一切通过
  // ============================================================
  if (mode === 'allow-all') {
    return { allowed: true };
  }
  
  // ============================================================
  // 2. ask 模式：一切通过（但会弹窗询问用户）
  // ============================================================
  if (mode === 'ask') {
    return { allowed: true };
  }
  
  // ============================================================
  // 3. safe 模式：白名单检查
  // ============================================================
  
  // 3.1 永远允许的只读工具
  const ALWAYS_ALLOWED_TOOLS = new Set([
    'Read', 'Glob', 'Grep', 'WebFetch', 'WebSearch',
    'TaskOutput', 'ListAgents', 'Skill', ...
  ]);
  
  if (ALWAYS_ALLOWED_TOOLS.has(toolName)) {
    return { allowed: true };
  }
  
  // 3.2 检查 MCP 工具变体（如 mcp__plan__SubmitPlan）
  for (const allowedTool of ALWAYS_ALLOWED_TOOLS) {
    if (toolName.endsWith(`__${allowedTool}`)) {
      return { allowed: true };
    }
  }
  
  // 3.3 浏览器工具别名（browser_open/browser_snapshot/...）
  if (isBrowserToolNameOrAlias(toolName)) {
    return { allowed: true };
  }
  
  // 3.4 Bash 命令检查
  if (toolName === 'Bash') {
    const command = toolInput.command;
    
    // 调用 getBashRejectionReason 检查命令是否只读
    const rejection = getBashRejectionReason(command, config);
    
    if (!rejection) {
      // 命令安全（如 ls/cat/grep），通过
      return { allowed: true };
    }
    
    // 特殊例外：写入 plans 文件夹或 data 文件夹
    const targetPath = extractBashWriteTarget(command);
    
    if (targetPath && options?.plansFolderPath) {
      if (isPathWithinDirectory(targetPath, options.plansFolderPath)) {
        // 写入 plans 文件夹，允许
        return { allowed: true };
      }
    }
    
    if (targetPath && options?.dataFolderPath) {
      if (isPathWithinDirectory(targetPath, options.dataFolderPath)) {
        // 写入 data 文件夹，允许
        return { allowed: true };
      }
    }
    
    // 否则阻止，返回详细错误原因
    return {
      allowed: false,
      reason: formatBashRejectionMessage(rejection, config),
    };
  }
  
  // 3.5 Write/Edit/MultiEdit/NotebookEdit 永远阻止
  if (SAFE_MODE_CONFIG.blockedTools.has(toolName)) {
    return {
      allowed: false,
      reason: `${toolName} is blocked in Explore mode. Switch to Ask or Execute mode (SHIFT+TAB).`,
    };
  }
  
  // 3.6 其他工具默认阻止
  return {
    allowed: false,
    reason: `Tool ${toolName} is not allowed in Explore mode.`,
  };
}
```

### 3.4 Bash 命令的只读检查

**核心逻辑**：用正则表达式匹配命令，判断是否只读。

**白名单示例** (`~/.craft-agent/permissions/default.json`)：

```json
{
  "allowedBashPatterns": [
    "^(ls|dir)( .*)? *$",
    "^cat( .*)? *$",
    "^grep( .*)? *$",
    "^git status *$",
    "^git log( .*)? *$",
    "^find( .*)? *$",
    "^pwd *$",
    "^which( .*)? *$"
  ]
}
```

**检查流程**：

```typescript
function getBashRejectionReason(command: string, config: ModeConfig) {
  // 1. 检查是否有危险操作符（>, >>, |, &&, ;）
  if (command.includes('>') || command.includes('>>')) {
    return { type: 'dangerous_operator', operatorType: 'redirect' };
  }
  
  // 2. 提取基础命令（第一个词）
  const baseCommand = command.trim().split(/\s+/)[0];
  
  // 3. 检查是否在危险命令列表
  const DANGEROUS_COMMANDS = ['rm', 'mv', 'chmod', 'sudo', 'git push', ...];
  if (DANGEROUS_COMMANDS.has(baseCommand)) {
    return { type: 'dangerous_command', command: baseCommand };
  }
  
  // 4. 检查是否匹配白名单正则
  for (const pattern of config.readOnlyBashPatterns) {
    if (pattern.regex.test(command)) {
      return null;  // 匹配白名单，允许
    }
  }
  
  // 5. 没有匹配，阻止
  return { type: 'not_in_allowlist' };
}
```

**特殊例外**：写入 plans 文件夹

```typescript
// AI 可以在 Explore 模式下写入 plans 文件夹（用于规划）
if (command.includes('> /path/to/session/plans/file.md')) {
  // 提取目标路径
  const targetPath = '/path/to/session/plans/file.md';
  
  // 检查是否在 plans 文件夹内
  if (isPathWithinDirectory(targetPath, plansFolderPath)) {
    return { allowed: true };  // 允许
  }
}
```

---

## 四、完整数据流

把 WebSocket RPC 和权限系统串起来：

```
1. 用户在 UI 输入消息："帮我删除 temp.txt"
   ↓
   
2. AI (Claude) 决定调用工具：
   {
     tool: "Bash",
     input: { command: "rm temp.txt" }
   }
   ↓
   
3. 渲染进程通过 WebSocket 发送请求：
   ws.send({
     id: "uuid-1234",
     type: "request",
     channel: "tool:bash",
     args: [{ command: "rm temp.txt" }]
   })
   ↓
   
4. 主进程接收请求：
   WebSocket.onMessage → deserializeEnvelope → onRequest
   ↓
   
5. 路由到 Bash 工具处理器：
   handlers.get("tool:bash") → bashToolHandler(ctx, { command: "rm temp.txt" })
   ↓
   
6. 权限检查（在 handler 内部）：
   permissionManager.evaluateToolCall("Bash", { command: "rm temp.txt" })
   
   ├─ 当前模式：safe (Explore)
   │  ├─ 检查命令："rm" 是危险命令
   │  ├─ 不在白名单
   │  └─ 阻止，返回错误
   │
   ├─ 当前模式：ask (Ask)
   │  ├─ 命令允许（但需要用户确认）
   │  ├─ 通过 WebSocket 推送权限请求事件：
   │  │   ws.send({
   │  │     type: "event",
   │  │     channel: "permission:request",
   │  │     args: [{
   │  │       requestId: "perm-5678",
   │  │       toolName: "Bash",
   │  │       command: "rm temp.txt",
   │  │       description: "Delete temp.txt"
   │  │     }]
   │  │   })
   │  ├─ 渲染进程弹窗：
   │  │   "AI wants to run: rm temp.txt"
   │  │   [Allow] [Deny]
   │  ├─ 用户点击 [Allow]
   │  ├─ 渲染进程发送批准：
   │  │   ws.send({
   │  │     type: "request",
   │  │     channel: "permission:approve",
   │  │     args: [{ requestId: "perm-5678" }]
   │  │   })
   │  └─ 继续执行
   │
   └─ 当前模式：allow-all (Execute)
      └─ 直接执行，不检查
   ↓
   
7. 执行命令（如果通过权限检查）：
   execSync("rm temp.txt")
   ↓
   
8. 返回结果：
   ws.send({
     id: "uuid-1234",
     type: "response",
     channel: "tool:bash",
     result: ""  // rm 命令没有输出
   })
   ↓
   
9. 渲染进程收到响应，更新 UI
```

---

## 五、关键技术点总结

### 5.1 WebSocket RPC 协议

- **问题**：浏览器环境无法直接访问文件系统和执行命令
- **方案**：通过 WebSocket RPC 让渲染进程调用主进程的功能
- **优势**：
  - 双向通信（服务器可推送事件）
  - 跨平台（CLI、Web UI、Electron 统一协议）
  - 支持远程访问（TLS + token 认证）

### 5.2 三层权限系统

- **问题**：AI 可能执行危险操作（删除文件、执行恶意命令）
- **方案**：三层权限模式 + 工具白名单
- **三种模式**：
  - **safe (Explore)**：只读探索，阻止所有写操作
  - **ask (Ask)**：执行前询问用户
  - **allow-all (Execute)**：自动执行，不询问

### 5.3 Bash 命令检查

- **白名单机制**：只有匹配正则的只读命令才能在 Explore 模式执行
- **危险命令阻止**：`rm`, `sudo`, `chmod`, `git push` 等永远需要权限
- **特殊例外**：写入 plans 文件夹或 data 文件夹允许（用于 AI 规划）

### 5.4 事件推送

- **权限请求**：ask 模式下，主进程推送 `permission:request` 事件到渲染进程
- **任务进度**：后台任务推送 `task:progress` 事件
- **可靠投递**：每个事件有序列号 `seq`，客户端用 `sequence_ack` 确认收到

---

## 六、面试可能的问题

### Q1: 为什么用 WebSocket 而不是 Electron 的 IPC？

**答案**：
- IPC 只能本地进程间通信，无法支持远程访问
- WebSocket 支持双向通信，服务器可主动推送事件
- 统一协议，CLI、Web UI、Electron 都能用

### Q2: 三层权限系统怎么实现的？

**答案**：
- 每个 Session 有一个 `permissionMode` 字段（safe/ask/allow-all）
- 工具执行前调用 `shouldAllowToolInMode` 检查
- safe 模式：白名单检查，只允许只读工具和命令
- ask 模式：推送 `permission:request` 事件，等待用户确认
- allow-all 模式：直接通过，不检查

### Q3: Explore 模式下哪些工具被阻止？

**答案**：
- **永远阻止**：Write, Edit, MultiEdit, NotebookEdit（硬编码）
- **Bash 命令**：只允许白名单内的只读命令（ls/cat/grep/git status）
- **特殊例外**：写入 plans 文件夹或 data 文件夹允许

### Q4: ask 模式下权限请求的流程是什么？

**答案**：
1. AI 尝试调用工具（如 `Bash: rm file.txt`）
2. 权限检查通过（ask 模式允许，但需要确认）
3. 主进程推送 `permission:request` 事件到渲染进程
4. 渲染进程弹窗："AI wants to run: rm file.txt"
5. 用户点击 [Allow] 或 [Deny]
6. 渲染进程发送 `permission:approve` 或 `permission:deny`
7. 主进程继续执行或中止

### Q5: WebSocket 的 `seq` 和 `sequence_ack` 有什么用？

**答案**：
- **seq**：服务器给每个事件分配的序列号，单调递增
- **sequence_ack**：客户端确认收到某个 seq 之前的所有事件
- **作用**：可靠投递，网络断开重连后，服务器可以重发客户端没确认的事件

### Q6: 为什么 Explore 模式允许写入 plans 文件夹？

**答案**：
- AI 需要在 plans 文件夹写入规划文档（plan.md）
- plans 文件夹是 Session 专属的，不会影响用户代码
- 这是一个合理的例外：只读探索 + 规划写入

---

## 七、代码位置索引

| 模块 | 文件 | 关键行数 | 功能 |
|------|------|----------|------|
| 消息格式 | `packages/shared/src/protocol/types.ts` | 20-63 | MessageEnvelope 定义 |
| 消息类型 | `packages/shared/src/protocol/types.ts` | 11-18 | 7 种 MessageType |
| WebSocket 服务器 | `packages/server-core/src/transport/server.ts` | 642-683 | onRequest 请求处理 |
| 事件推送 | `packages/server-core/src/transport/server.ts` | 180-202 | broadcast 推送事件 |
| 权限模式定义 | `packages/shared/src/agent/mode-types.ts` | 24-43 | PermissionMode 类型 |
| 权限模式配置 | `packages/shared/src/agent/mode-types.ts` | 304-340 | 三种模式的行为描述 |
| Safe 模式配置 | `packages/shared/src/agent/mode-types.ts` | 265-283 | 阻止的工具列表 |
| 权限检查核心 | `packages/shared/src/agent/mode-manager.ts` | 1803-1932 | shouldAllowToolInMode |
| Bash 命令检查 | `packages/shared/src/agent/mode-manager.ts` | 1856-1932 | Bash 只读检查逻辑 |

---

**第二个简历点完成！** 下一步继续第三个简历点：Session 持久化（JSONL）+ 状态管理（Jotai）。
