# Craft Agents 第三个简历点：Session 持久化（JSONL）+ 状态管理（Jotai）

## 一、技术架构概述

Session 是 Craft Agents 的核心数据结构，包含一次完整的对话历史。需要解决两个问题：
1. **持久化**：对话数据如何保存到磁盘，重启后如何恢复？
2. **状态管理**：多个 Session 同时存在，如何高效管理状态，避免不必要的重渲染？

**解决方案**：
- **JSONL (JSON Lines)**：每个 Session 一个 `.jsonl` 文件，第一行是 header（元数据），后面每行一条消息
- **Jotai atomFamily**：每个 Session 独立的 atom，更新 Session A 不会触发 Session B 的重渲染

---

## 二、JSONL 持久化格式

### 2.1 为什么用 JSONL？

**对比其他格式**：

| 格式 | 优点 | 缺点 |
|------|------|------|
| **JSON** | 简单，易读 | 每次保存需要重写整个文件，大对话慢 |
| **SQLite** | 查询快，事务安全 | 需要额外的数据库引擎，跨平台兼容性问题 |
| **JSONL** | **增量写入**，每条消息独立一行，易追加 | 查询需要全文件扫描 |

**JSONL 的优势**：
- **增量写入**：新消息只需追加一行，不用重写整个文件
- **故障恢复**：即使写入中断，已写入的行仍然有效（不会整个文件损坏）
- **懒加载**：只读第一行 header 就能显示 Session 列表，不用加载所有消息

### 2.2 文件格式

**位置**：`~/.craft-agent/workspaces/{workspaceId}/sessions/{sessionId}/session.jsonl`

**格式** (`packages/shared/src/sessions/jsonl.ts:1-6`)：

```
Line 1: SessionHeader (元数据)
Line 2: StoredMessage (第一条消息)
Line 3: StoredMessage (第二条消息)
...
Line N: StoredMessage (第 N-1 条消息)
```

**示例文件内容**：

```jsonl
{"id":"sess-abc","workspaceId":"ws-1","name":"修复登录 bug","lastUsedAt":1736950800000,"messageCount":10,"preview":"帮我修复登录页面的 bug","lastMessageRole":"assistant","tokenUsage":{"inputTokens":1234,"outputTokens":5678,"totalTokens":6912,"costUsd":0.05},"permissionMode":"ask"}
{"id":"msg-1","type":"user","role":"user","content":"帮我修复登录页面的 bug","timestamp":1736950800000}
{"id":"msg-2","type":"assistant","role":"assistant","content":"我来看一下登录代码","timestamp":1736950805000}
{"id":"msg-3","type":"tool","role":"tool","toolName":"Read","toolInput":{"file_path":"/src/login.ts"},"timestamp":1736950806000}
{"id":"msg-4","type":"tool_result","role":"tool","content":"export function login() { ... }","timestamp":1736950807000}
...
```

### 2.3 SessionHeader（第一行元数据）

**定义位置**：`packages/shared/src/sessions/types.ts`

**核心字段**：

```typescript
interface SessionHeader {
  // 基础标识
  id: string;
  workspaceId: string;
  name?: string;                    // Session 名称（用户可编辑）
  
  // 时间戳
  lastUsedAt: number;               // 最后活跃时间（用于排序）
  createdAt?: number;               // 创建时间
  
  // 预计算字段（快速列表加载，不用解析所有消息）
  messageCount: number;             // 消息总数
  preview?: string;                 // 第一条用户消息的前 150 字符
  lastMessageRole?: 'user' | 'assistant' | 'plan' | 'tool' | 'error';  // 最后一条消息的类型
  lastFinalMessageId?: string;      // 最后一条非中间消息的 ID（用于未读检测）
  
  // Token 统计
  tokenUsage?: {
    inputTokens: number;
    outputTokens: number;
    totalTokens: number;
    costUsd: number;
    contextTokens: number;
  };
  
  // 权限和配置
  permissionMode?: 'safe' | 'ask' | 'allow-all';  // 权限模式
  workingDirectory?: string;        // 工作目录
  model?: string;                   // 使用的模型
  
  // 状态标记
  isFlagged?: boolean;              // 是否标记
  hasUnread?: boolean;              // 是否有未读消息
  hidden?: boolean;                 // 是否隐藏（如 mini edit sessions）
  isArchived?: boolean;             // 是否归档
}
```

**为什么预计算这些字段？**

假设有 1000 个 Session，每个 Session 100 条消息。如果不预计算：
- 加载 Session 列表 → 需要读取 1000 个文件 × 100 行 = 10 万行 JSON
- 用户只是想看列表，不需要消息内容

有了预计算：
- 加载 Session 列表 → 只读取 1000 个文件的第一行 = 1000 行 JSON
- **快 100 倍**

### 2.4 路径可移植性（Portable Paths）

**问题**：Session 文件可能被复制到其他机器，绝对路径会失效。

**解决方案**：用 `{{SESSION_PATH}}` 占位符代替绝对路径。

**代码** (`jsonl.ts:23-50`)：

```typescript
const SESSION_PATH_TOKEN = '{{SESSION_PATH}}';

// 写入时：替换绝对路径为占位符
function makeSessionPathPortable(jsonLine: string, sessionDir: string): string {
  // 例如: "/Users/user/workspace/sessions/sess-1/plans/plan.md"
  //   → "{{SESSION_PATH}}/plans/plan.md"
  return jsonLine.replaceAll(sessionDir, SESSION_PATH_TOKEN);
}

// 读取时：展开占位符为当前的绝对路径
function expandSessionPath(jsonLine: string, sessionDir: string): string {
  // 例如: "{{SESSION_PATH}}/plans/plan.md"
  //   → "/home/newuser/workspace/sessions/sess-1/plans/plan.md"
  return jsonLine.replaceAll(SESSION_PATH_TOKEN, sessionDir);
}
```

**使用场景**：
- 附件路径：`storedPath: "{{SESSION_PATH}}/attachments/image.png"`
- Plan 文件路径：`planPath: "{{SESSION_PATH}}/plans/plan.md"`
- Datatable 路径：`src: "{{SESSION_PATH}}/data/results.csv"`

### 2.5 读取 Session

**两种读取方式**：

#### 方式 1：只读 Header（快速列表加载）

**代码** (`jsonl.ts:80-97`)：

```typescript
function readSessionHeader(sessionFile: string): SessionHeader | null {
  // 1. 打开文件
  const fd = openSync(sessionFile, 'r');
  
  // 2. 只读前 8KB（header 足够了）
  const buffer = Buffer.alloc(8192);
  const bytesRead = readSync(fd, buffer, 0, 8192, 0);
  closeSync(fd);
  
  // 3. 找到第一行（第一个 \n 之前）
  const content = buffer.toString('utf-8', 0, bytesRead);
  const firstNewline = content.indexOf('\n');
  const firstLine = content.slice(0, firstNewline);
  
  // 4. 解析 JSON
  const header = JSON.parse(expandSessionPath(firstLine, dirname(sessionFile)));
  
  return header;
}
```

**性能优化**：
- 不读取整个文件，只读前 8KB
- 对于 1000 个 Session，只需要读 1000 × 8KB = 8MB（而不是整个对话历史）

#### 方式 2：读取完整 Session（包含所有消息）

**代码** (`jsonl.ts:103-140`)：

```typescript
function readSessionJsonl(sessionFile: string): StoredSession | null {
  // 1. 读取整个文件
  const content = readFileSync(sessionFile, 'utf-8');
  const lines = content.split('\n').filter(Boolean);
  
  // 2. 解析 header（第一行）
  const firstLine = lines[0];
  const header = JSON.parse(expandSessionPath(firstLine, dirname(sessionFile)));
  
  // 3. 解析消息（剩余行）
  const messages = lines.slice(1).map(line => {
    return JSON.parse(expandSessionPath(line, dirname(sessionFile)));
  });
  
  return {
    ...header,
    messages,
  };
}
```

**容错性**：
- 用 `filter(Boolean)` 过滤空行
- 如果某一行 JSON 解析失败，跳过该行，不影响其他消息（而不是整个 Session 损坏）

### 2.6 写入 Session（原子写入）

**问题**：直接写文件，如果进程在写到一半时崩溃，文件会损坏。

**解决方案**：原子写入（Atomic Write）

**代码** (`jsonl.ts:150-164`)：

```typescript
function writeSessionJsonl(sessionFile: string, session: StoredSession): void {
  // 1. 创建 header
  const header = createSessionHeader(session);
  const sessionDir = dirname(sessionFile);
  
  // 2. 生成所有行（header + messages）
  const lines = [
    makeSessionPathPortable(JSON.stringify(header), sessionDir),
    ...session.messages.map(m => makeSessionPathPortable(JSON.stringify(m), sessionDir)),
  ];
  
  // 3. 先写入临时文件
  const tmpFile = sessionFile + '.tmp';
  writeFileSync(tmpFile, lines.join('\n') + '\n');
  
  // 4. 原子性地重命名（替换旧文件）
  try { unlinkSync(sessionFile); } catch { /* 忽略文件不存在的错误 */ }
  renameSync(tmpFile, sessionFile);
  //         ^^^^^^^^^^^^^^^^^^^^^^^^
  //         重命名是原子操作：要么成功，要么失败，不会出现半个文件
}
```

**原子写入的保证**：
- 写入过程中崩溃 → 旧文件完好无损，临时文件可能损坏（但不影响）
- 写入完成后崩溃 → 新文件已完整写入
- **永远不会出现半个文件**

### 2.7 预计算字段的生成

**代码** (`jsonl.ts:171-215`)：

```typescript
function createSessionHeader(session: StoredSession): SessionHeader {
  return {
    ...session,
    // 更新时间戳
    lastUsedAt: Date.now(),
    
    // 预计算字段
    messageCount: session.messages.length,
    lastMessageRole: extractLastMessageRole(session.messages),
    preview: extractPreview(session.messages),
    lastFinalMessageId: extractLastFinalMessageId(session.messages),
    tokenUsage: session.tokenUsage,
  };
}

// 提取最后一条消息的角色
function extractLastMessageRole(messages: StoredMessage[]): 'user' | 'assistant' | ... {
  const lastMessage = messages[messages.length - 1];
  return lastMessage?.type;
}

// 提取第一条用户消息的前 150 字符
function extractPreview(messages: StoredMessage[]): string {
  const firstUserMessage = messages.find(m => m.type === 'user');
  if (!firstUserMessage?.content) return undefined;
  
  // 清理特殊标记和 HTML 标签
  const sanitized = firstUserMessage.content
    .replace(/<edit_request>[\s\S]*?<\/edit_request>/g, '')  // 移除编辑请求块
    .replace(/<[^>]+>/g, '')  // 移除 HTML 标签
    .replace(/\s+/g, ' ')     // 规范化空白
    .trim();
  
  return sanitized.slice(0, 150);
}

// 提取最后一条非中间消息的 ID（用于未读检测）
function extractLastFinalMessageId(messages: StoredMessage[]): string | undefined {
  // 倒序查找最后一条 assistant 消息且 isIntermediate = false
  for (let i = messages.length - 1; i >= 0; i--) {
    const msg = messages[i];
    if (msg?.type === 'assistant' && !msg.isIntermediate) {
      return msg.id;
    }
  }
  return undefined;
}
```

---

## 三、Jotai 状态管理

### 3.1 为什么用 Jotai？

**问题**：用 React Context 或 Redux 管理所有 Session 的状态会怎样？

```typescript
// 全局 state（反例）
const allSessions = [
  { id: 'sess-1', messages: [...1000 条消息] },
  { id: 'sess-2', messages: [...1000 条消息] },
  { id: 'sess-3', messages: [...1000 条消息] },
  ...
];

// Session 1 新增一条消息
allSessions[0].messages.push(newMessage);

// 问题：所有订阅 allSessions 的组件都会重渲染！
// 即使 Session 2 和 Session 3 的用户界面也会重渲染
```

**Jotai 的解决方案**：**atomFamily** —— 每个 Session 独立的 atom

```typescript
// 每个 Session 独立的 atom
const sessionAtomFamily = atomFamily(
  (sessionId: string) => atom<Session | null>(null)
);

// Session 1 的组件订阅
const session1 = useAtomValue(sessionAtomFamily('sess-1'));

// Session 2 的组件订阅
const session2 = useAtomValue(sessionAtomFamily('sess-2'));

// 更新 Session 1 → 只有订阅 Session 1 的组件重渲染
// Session 2 的组件不受影响
```

### 3.2 Jotai atomFamily 核心概念

**atomFamily** 是一个工厂函数，根据参数（如 sessionId）创建独立的 atom。

**代码** (`apps/electron/src/renderer/atoms/sessions.ts:142-145`)：

```typescript
export const sessionAtomFamily = atomFamily(
  (_sessionId: string) => atom<Session | null>(null),
  //                     ^^^^^^^^^^^^^^^^^^^^^^^^
  //                     每个 sessionId 创建一个新的 atom
  (a, b) => a === b
  //        ^^^^^^^
  //        相等判断：相同 sessionId 返回同一个 atom
);

// 使用
const sessionAtom = sessionAtomFamily('sess-1');  // 拿到 Session 1 的 atom
const session = useAtomValue(sessionAtom);         // 订阅这个 atom
```

**工作原理**：

```
sessionAtomFamily 内部维护一个 Map：

Map {
  'sess-1' => atom<Session | null>(session1Data),
  'sess-2' => atom<Session | null>(session2Data),
  'sess-3' => atom<Session | null>(session3Data),
}

调用 sessionAtomFamily('sess-1')：
  ├─ 检查 Map 是否已有 'sess-1'
  ├─ 有 → 返回已存在的 atom
  └─ 没有 → 创建新 atom，加入 Map，返回
```

### 3.3 完整的状态管理架构

**代码位置**：`apps/electron/src/renderer/atoms/sessions.ts`

#### 3.3.1 核心 Atoms

```typescript
// 1. 每个 Session 独立的 atom（完整数据，包含消息）
export const sessionAtomFamily = atomFamily(
  (_sessionId: string) => atom<Session | null>(null)
);

// 2. Session 元数据 Map（轻量级，用于列表显示，不包含消息）
export const sessionMetaMapAtom = atom<Map<string, SessionMeta>>(new Map());
//                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                                     只有基本信息：id, name, preview, lastMessageAt, ...

// 3. Session ID 列表（用于列表排序）
export const sessionIdsAtom = atom<string[]>([]);

// 4. 已加载消息的 Session 集合（用于懒加载）
export const loadedSessionsAtom = atom<Set<string>>(new Set());

// 5. 当前活跃的 Session ID
export const activeSessionIdAtom = atom<string | null>(null);
```

#### 3.3.2 SessionMeta（轻量级元数据）

**为什么需要 SessionMeta？**

Session 列表只需要显示：
- Session 名称
- 最后活跃时间
- 是否有未读消息
- Token 使用统计

**不需要**：
- 所有消息内容（可能有几千条）

**定义** (`sessions.ts:20-94`)：

```typescript
interface SessionMeta {
  id: string;
  name?: string;
  preview?: string;              // 第一条消息预览
  workspaceId: string;
  lastMessageAt?: number;        // 最后活跃时间（用于排序）
  isProcessing?: boolean;        // 是否正在处理
  hasUnread?: boolean;           // 是否有未读消息
  messageCount?: number;         // 消息总数
  lastFinalMessageId?: string;   // 最后一条非中间消息 ID
  permissionMode?: string;       // 权限模式
  tokenUsage?: { ... };          // Token 统计
  // ... 其他轻量级字段
}

function extractSessionMeta(session: Session): SessionMeta {
  const { messages, sessionFolderPath, ...sessionFields } = session;
  //      ^^^^^^^^  ^^^^^^^^^^^^^^^^^^ 
  //      排除消息和路径（不需要在列表显示）
  
  return {
    ...sessionFields,
    messageCount: messages.length,
    lastFinalMessageId: findLastFinalMessageId(messages),
  };
}
```

#### 3.3.3 更新操作（Action Atoms）

**1. 更新完整 Session**

```typescript
export const updateSessionAtom = atom(
  null,  // 没有 read 函数（纯 action）
  (get, set, sessionId: string, updater: (prev: Session | null) => Session | null) => {
    // 1. 更新 Session atom
    const sessionAtom = sessionAtomFamily(sessionId);
    const currentSession = get(sessionAtom);
    const newSession = updater(currentSession);
    set(sessionAtom, newSession);
    
    // 2. 同步更新 metadata
    if (newSession) {
      const metaMap = get(sessionMetaMapAtom);
      const newMetaMap = new Map(metaMap);
      newMetaMap.set(sessionId, extractSessionMeta(newSession));
      set(sessionMetaMapAtom, newMetaMap);
    }
  }
);

// 使用
setUpdateSession('sess-1', (prev) => ({
  ...prev,
  name: '新名称',
}));
```

**2. 追加消息（流式输出）**

```typescript
export const appendMessageAtom = atom(
  null,
  (get, set, sessionId: string, message: Message) => {
    const sessionAtom = sessionAtomFamily(sessionId);
    const session = get(sessionAtom);
    
    if (session) {
      set(sessionAtom, {
        ...session,
        messages: [...session.messages, message],
        //        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //        创建新数组，不修改原数组（React 不可变更新）
      });
    }
  }
);

// 使用
setAppendMessage('sess-1', {
  id: 'msg-new',
  type: 'assistant',
  content: 'Hello',
  timestamp: Date.now(),
});
```

**3. 更新流式内容（逐字输出）**

```typescript
export const updateStreamingContentAtom = atom(
  null,
  (get, set, sessionId: string, content: string, turnId?: string) => {
    const sessionAtom = sessionAtomFamily(sessionId);
    const session = get(sessionAtom);
    if (!session) return;
    
    const messages = [...session.messages];
    const lastMsg = messages[messages.length - 1];
    
    // 追加到最后一条流式消息
    if (lastMsg?.role === 'assistant' && lastMsg.isStreaming) {
      messages[messages.length - 1] = {
        ...lastMsg,
        content: lastMsg.content + content,
        //       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //       逐字追加
      };
      set(sessionAtom, { ...session, messages });
    }
  }
);

// 使用（AI 流式输出时调用）
setUpdateStreamingContent('sess-1', 'H');
setUpdateStreamingContent('sess-1', 'e');
setUpdateStreamingContent('sess-1', 'l');
setUpdateStreamingContent('sess-1', 'l');
setUpdateStreamingContent('sess-1', 'o');
```

### 3.4 懒加载机制

**问题**：启动时加载 1000 个 Session，每个 100 条消息 = 10 万条消息 → 内存爆炸

**解决方案**：
1. **启动时**：只加载 SessionMeta（从 header），不加载消息
2. **用户打开 Session 时**：才加载该 Session 的完整消息

**代码流程**：

```typescript
// 1. 启动时：只加载元数据
async function initializeSessions() {
  const sessionFiles = listSessionFiles();
  
  const metaMap = new Map();
  for (const file of sessionFiles) {
    // 只读 header，不读消息
    const header = readSessionHeader(file);
    metaMap.set(header.id, extractSessionMeta(header));
  }
  
  set(sessionMetaMapAtom, metaMap);
  // 此时 sessionAtomFamily 中没有任何 Session 的完整数据
}

// 2. 用户点击 Session 时：加载完整数据
async function loadSessionMessages(sessionId: string) {
  // 检查是否已加载
  const loadedSessions = get(loadedSessionsAtom);
  if (loadedSessions.has(sessionId)) {
    return;  // 已加载，不重复加载
  }
  
  // 读取完整 Session（包含所有消息）
  const session = readSessionJsonl(`/path/to/${sessionId}/session.jsonl`);
  
  // 更新 atom
  set(sessionAtomFamily(sessionId), session);
  
  // 标记为已加载
  const newLoadedSessions = new Set(loadedSessions);
  newLoadedSessions.add(sessionId);
  set(loadedSessionsAtom, newLoadedSessions);
}

// 3. React 组件
function ChatPanel({ sessionId }) {
  const session = useAtomValue(sessionAtomFamily(sessionId));
  const loadedSessions = useAtomValue(loadedSessionsAtom);
  
  useEffect(() => {
    if (!loadedSessions.has(sessionId)) {
      loadSessionMessages(sessionId);
    }
  }, [sessionId]);
  
  if (!session) {
    return <div>Loading messages...</div>;
  }
  
  return (
    <div>
      {session.messages.map(msg => <MessageBubble key={msg.id} message={msg} />)}
    </div>
  );
}
```

### 3.5 性能优化总结

**优化 1：atomFamily 隔离更新**
- Session A 新增消息 → 只有订阅 Session A 的组件重渲染
- Session B、C、D 的组件不受影响

**优化 2：SessionMeta 分离**
- Session 列表只订阅 `sessionMetaMapAtom`（轻量级）
- 不订阅 `sessionAtomFamily`（包含所有消息）
- 消息更新不会触发列表重渲染

**优化 3：懒加载**
- 启动时只加载 header（1KB），不加载消息（可能 1MB）
- 用户打开时才加载完整消息

**性能对比**：

| 场景 | 不优化 | 优化后 |
|------|--------|--------|
| 启动加载 1000 个 Session | 读取 1000MB（所有消息） | 读取 8MB（只读 header） |
| Session A 流式输出 | 所有 Session 组件重渲染 | 只有 Session A 组件重渲染 |
| Session 列表滚动 | 重渲染所有消息组件 | 只渲染可见的列表项 |

---

## 四、完整数据流

### 4.1 创建新 Session

```
1. 用户输入消息："帮我写一个排序算法"
   ↓
2. 创建 Session 对象：
   const session = {
     id: 'sess-new',
     workspaceId: 'ws-1',
     messages: [
       { id: 'msg-1', type: 'user', content: '帮我写一个排序算法', ... }
     ],
     lastUsedAt: Date.now(),
     permissionMode: 'ask',
   };
   ↓
3. 写入 JSONL 文件：
   writeSessionJsonl('/path/to/sess-new/session.jsonl', session);
   
   生成文件：
   {"id":"sess-new","messageCount":1,"preview":"帮我写一个排序算法",...}
   {"id":"msg-1","type":"user","content":"帮我写一个排序算法",...}
   ↓
4. 更新 Jotai 状态：
   set(sessionAtomFamily('sess-new'), session);
   set(sessionMetaMapAtom, metaMap.set('sess-new', extractSessionMeta(session)));
   set(sessionIdsAtom, [...sessionIds, 'sess-new']);
   ↓
5. UI 自动更新（React 重渲染）
```

### 4.2 AI 流式输出

```
1. AI 开始回复，逐字推送内容：
   WebSocket 推送事件: { type: 'text_delta', content: 'H' }
   WebSocket 推送事件: { type: 'text_delta', content: 'e' }
   WebSocket 推送事件: { type: 'text_delta', content: 'l' }
   ...
   ↓
2. 渲染进程接收事件，更新状态：
   setUpdateStreamingContent('sess-1', 'H');
   setUpdateStreamingContent('sess-1', 'e');
   setUpdateStreamingContent('sess-1', 'l');
   ...
   ↓
3. Jotai 更新：
   sessionAtomFamily('sess-1') 的最后一条消息的 content 逐字追加
   ↓
4. React 重渲染（只重渲染 Session 1 的聊天面板，其他 Session 不受影响）
   ↓
5. AI 回复完成后，写入 JSONL：
   追加一行到文件：
   {"id":"msg-2","type":"assistant","content":"完整的回复内容",...}
```

### 4.3 懒加载 Session 消息

```
1. 用户启动应用：
   ├─ 加载所有 Session 的 header（只读第一行）
   ├─ 构建 sessionMetaMapAtom
   └─ 显示 Session 列表（不包含消息）
   ↓
2. 用户点击 Session A：
   ├─ 检查 loadedSessionsAtom：Session A 未加载
   ├─ 读取完整 JSONL 文件（包含所有消息）
   ├─ 更新 sessionAtomFamily('sess-A')
   ├─ 标记 Session A 为已加载
   └─ 显示聊天面板（包含所有消息）
   ↓
3. 用户切换到 Session B：
   ├─ 检查 loadedSessionsAtom：Session B 未加载
   ├─ 读取 Session B 的 JSONL
   ├─ 更新 sessionAtomFamily('sess-B')
   └─ 显示 Session B 的聊天
   ↓
4. 用户切回 Session A：
   ├─ 检查 loadedSessionsAtom：Session A 已加载
   ├─ 直接从 sessionAtomFamily('sess-A') 读取
   └─ 不重新加载文件
```

---

## 五、关键技术点总结

### 5.1 JSONL 持久化

- **问题**：如何高效保存对话历史？
- **方案**：JSONL 格式，第一行 header（元数据），后续每行一条消息
- **优势**：
  - 增量写入（追加新消息不用重写整个文件）
  - 懒加载（只读 header 就能显示列表）
  - 故障恢复（即使写入中断，已写入的行仍然有效）
  - 原子写入（先写临时文件再重命名，防止文件损坏）

### 5.2 Jotai atomFamily

- **问题**：多个 Session 同时存在，如何避免不必要的重渲染？
- **方案**：每个 Session 独立的 atom，更新隔离
- **优势**：
  - Session A 更新 → 只有订阅 Session A 的组件重渲染
  - Session B、C、D 不受影响
  - 解决了"Session A 流式输出导致 Session B 输入框失焦"的问题

### 5.3 懒加载机制

- **问题**：1000 个 Session × 100 条消息 = 10 万条消息，内存爆炸
- **方案**：启动时只加载 header，用户打开时才加载完整消息
- **优势**：
  - 启动快（只读 8MB header，而不是 1GB 消息）
  - 内存省（只保留当前打开的 Session 的消息）

### 5.4 路径可移植性

- **问题**：Session 文件被复制到其他机器，绝对路径失效
- **方案**：用 `{{SESSION_PATH}}` 占位符代替绝对路径
- **优势**：Session 文件可以在不同机器间迁移

---

## 六、面试可能的问题

### Q1: 为什么用 JSONL 而不是 JSON？

**答案**：
- JSON 需要重写整个文件，JSONL 可以增量追加
- JSONL 第一行是 header，可以快速加载列表（不用解析所有消息）
- JSONL 即使写入中断，已写入的行仍然有效（JSON 会整个文件损坏）

### Q2: 什么是原子写入？为什么需要？

**答案**：
- 先写临时文件（`.tmp`），写完后重命名覆盖旧文件
- 重命名是原子操作：要么成功，要么失败，不会出现半个文件
- 防止写入过程中崩溃导致文件损坏

### Q3: Jotai 的 atomFamily 是什么？

**答案**：
- 工厂函数，根据参数（如 sessionId）创建独立的 atom
- 内部维护一个 Map，相同参数返回同一个 atom
- 每个 Session 独立的 atom，更新 Session A 不会触发 Session B 的重渲染

### Q4: SessionMeta 和 Session 的区别？

**答案**：
- **SessionMeta**：轻量级元数据，只有 id、name、preview、messageCount 等
- **Session**：完整数据，包含所有消息（可能几千条）
- **用途**：Session 列表只订阅 SessionMeta，不订阅 Session，避免消息更新触发列表重渲染

### Q5: 懒加载是怎么实现的？

**答案**：
1. 启动时只读 header（`readSessionHeader`），不读消息
2. 用 `loadedSessionsAtom` 记录哪些 Session 已加载完整消息
3. 用户打开 Session 时，检查是否已加载，未加载则调用 `readSessionJsonl` 读取完整消息
4. 标记为已加载，下次直接从内存读取

### Q6: 路径可移植性是怎么实现的？

**答案**：
- 写入时：`makeSessionPathPortable` 把绝对路径替换为 `{{SESSION_PATH}}`
- 读取时：`expandSessionPath` 把 `{{SESSION_PATH}}` 展开为当前的绝对路径
- 例如：`/Users/user/sessions/sess-1/plans/plan.md` → `{{SESSION_PATH}}/plans/plan.md`

### Q7: 为什么 header 要预计算字段？

**答案**：
- 如果不预计算，加载 Session 列表需要读取所有消息来计算 `messageCount`、`preview` 等
- 1000 个 Session × 100 条消息 = 10 万条消息要解析
- 预计算后，只需读取 1000 个 header（第一行），快 100 倍

---

## 七、代码位置索引

| 模块 | 文件 | 关键行数 | 功能 |
|------|------|----------|------|
| JSONL 读写 | `packages/shared/src/sessions/jsonl.ts` | 1-6 | JSONL 格式说明 |
| 路径可移植 | `packages/shared/src/sessions/jsonl.ts` | 23-50 | makeSessionPathPortable / expandSessionPath |
| 读取 Header | `packages/shared/src/sessions/jsonl.ts` | 80-97 | readSessionHeader（只读第一行） |
| 读取完整 Session | `packages/shared/src/sessions/jsonl.ts` | 103-140 | readSessionJsonl（读所有消息） |
| 写入 Session | `packages/shared/src/sessions/jsonl.ts` | 150-164 | writeSessionJsonl（原子写入） |
| 预计算字段 | `packages/shared/src/sessions/jsonl.ts` | 171-215 | createSessionHeader（生成 header） |
| Jotai atomFamily | `apps/electron/src/renderer/atoms/sessions.ts` | 142-145 | sessionAtomFamily 定义 |
| SessionMeta | `apps/electron/src/renderer/atoms/sessions.ts` | 20-94 | SessionMeta 接口定义 |
| 更新 Session | `apps/electron/src/renderer/atoms/sessions.ts` | 190-206 | updateSessionAtom |
| 追加消息 | `apps/electron/src/renderer/atoms/sessions.ts` | 258-271 | appendMessageAtom |
| 流式更新 | `apps/electron/src/renderer/atoms/sessions.ts` | 277-297 | updateStreamingContentAtom |

---

**第三个简历点完成！** 下一步继续第四个简历点：Bun monorepo + Electron 跨平台构建。
