# Craft Agents 第一个简历点：MCP 协议 + OAuth 2.0 + AES-256-GCM 加密存储

## 一、技术架构概述

这个简历点实现了一个完整的**外部工具集成系统**，包含三个核心模块：

1. **MCP 连接池** (`mcp-pool.ts`)：统一管理所有 MCP 服务的连接和工具调用
2. **OAuth 2.0 授权** (`google-oauth.ts`)：通过 PKCE 流程安全获取用户凭证
3. **加密存储** (`secure-storage.ts`)：使用硬件绑定的 AES-256-GCM 加密保存 token

---

## 二、MCP 协议与连接池

### 2.1 什么是 MCP？

**MCP (Model Context Protocol)** 是一个统一的协议，让 AI 能调用外部工具。

**为什么需要 MCP？**
- 没有 MCP 之前：每个服务（Linear、Jira、GitHub）都有自己的 API，AI 需要适配每一个
- 有了 MCP 之后：所有服务都暴露统一格式的工具，AI 只需调用 MCP 协议

**类比**：就像 USB 接口，所有设备（鼠标、键盘、U盘）都用同样的接口，电脑不需要为每个设备写驱动。

### 2.2 MCP 连接池架构

**核心数据结构** (`mcp-pool.ts:102-131`)：

```typescript
export class MCPClientPool {
  // 1. 所有 MCP 客户端的连接映射：slug → MCP Client
  private clients = new Map<string, PoolClient>();
  
  // 2. 每个服务暴露的工具缓存：slug → Tool[]
  private toolCache = new Map<string, Tool[]>();
  
  // 3. 代理工具名反向映射：proxyName → {slug, originalName}
  private proxyTools = new Map<string, { slug: string; originalName: string }>();
}
```

**为什么需要代理工具名？**

假设 Linear 和 Jira 都有一个叫 `createIssue` 的工具，如果不加前缀就会冲突。所以：
- Linear 的 `createIssue` → 代理名 `mcp__linear__createIssue`
- Jira 的 `createIssue` → 代理名 `mcp__jira__createIssue`

### 2.3 工具调用流程

**代码** (`mcp-pool.ts:387-436`)：

```typescript
async callTool(proxyName: string, args: Record<string, unknown>) {
  // 1. 从代理名拿到 slug 和原始工具名
  const info = this.proxyTools.get(proxyName);
  // 例如: proxyName = "mcp__linear__createIssue"
  //      → info = { slug: "linear", originalName: "createIssue" }
  
  const { slug, originalName } = info;
  
  // 2. 根据 slug 找到对应的 MCP 客户端
  const client = this.clients.get(slug);
  // 例如: slug = "linear" → 拿到 Linear 的 MCP 连接
  
  // 3. 用真实工具名调用远程 MCP 服务
  const result = await client.callTool(originalName, args);
  // 例如: callTool("createIssue", { title: "Bug fix", ... })
  
  return result;
}
```

**完整流程**：
```
AI 发起调用: mcp__linear__createIssue({title: "修复登录 bug"})
    ↓
连接池查表: proxyTools.get("mcp__linear__createIssue")
    ↓
拿到映射: {slug: "linear", originalName: "createIssue"}
    ↓
找到客户端: clients.get("linear") → Linear 的 MCP Client
    ↓
真实调用: client.callTool("createIssue", {title: "修复登录 bug"})
    ↓
Linear API 执行，返回结果
```

---

## 三、OAuth 2.0 PKCE 授权流程

### 3.1 为什么需要 OAuth？

Craft Agents 需要访问用户的 Google Drive、Gmail 等服务，直接要密码不安全。OAuth 让用户在 Google 官网授权，Craft Agents 只拿到一个 **access token**（钥匙），不知道密码。

### 3.2 什么是 PKCE？

**PKCE (Proof Key for Code Exchange)** 是 OAuth 的加强版，防止授权码被中途拦截。

**核心机制**：
1. 客户端生成一个随机字符串 `code_verifier`（43-128 字符）
2. 对它做 SHA-256 哈希得到 `code_challenge`
3. 授权时只发 `code_challenge`（哈希值）
4. 换 token 时发 `code_verifier`（原始值）
5. Google 验证：`SHA256(code_verifier) == code_challenge`

**为什么安全？**
- 攻击者即使拦截到授权码 `code`，也不知道 `code_verifier`
- 没有 `code_verifier` 就换不到 token

### 3.3 完整授权流程

**代码位置** (`google-oauth.ts:382-478`)

#### 步骤 1：生成 PKCE 和 state

```typescript
const pkce = generatePKCE();
// 返回: { verifier: "随机字符串", challenge: "SHA256(verifier)" }

const state = generateState();
// 返回: 随机字符串（防 CSRF 攻击）
```

#### 步骤 2：启动回调服务器

```typescript
const callbackServer = await createCallbackServer({ appType, deeplinkUrl });
// 监听 http://localhost:随机端口，等待 Google 重定向回来
```

#### 步骤 3：构造授权 URL

```typescript
const authUrl = new URL('https://accounts.google.com/o/oauth2/v2/auth');
authUrl.searchParams.set('client_id', GOOGLE_CLIENT_ID);
authUrl.searchParams.set('redirect_uri', 'http://localhost:端口');
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('scope', 'https://www.googleapis.com/auth/drive');
authUrl.searchParams.set('code_challenge', pkce.challenge);  // PKCE 哈希
authUrl.searchParams.set('code_challenge_method', 'S256');
authUrl.searchParams.set('state', state);  // CSRF 防护

await openUrl(authUrl.toString());
// 打开浏览器，跳转到 Google 授权页面
```

#### 步骤 4：用户授权后回调

```typescript
const callback = await callbackServer.promise;
// Google 授权后重定向: http://localhost:端口?code=授权码&state=...

// 验证 state 防止 CSRF 攻击
if (callback.query.state !== state) {
  return { success: false, error: 'CSRF attack detected' };
}

const code = callback.query.code;  // 拿到授权码
```

**state 参数的作用**：
- 攻击者可能伪造一个恶意的授权回调链接发给用户
- 如果用户点击，攻击者的授权码会被发到用户的客户端
- state 参数让客户端能验证："这个回调是我刚才发起的授权吗？"
- state 不匹配 → 拒绝，防止被攻击

#### 步骤 5：换取 token

**代码** (`google-oauth.ts:126-162`)

```typescript
async function exchangeCodeForTokens(code, codeVerifier, redirectUri, clientId, clientSecret) {
  const params = new URLSearchParams({
    client_id: clientId,
    client_secret: clientSecret,
    code: code,                    // 授权码
    code_verifier: codeVerifier,   // PKCE 原始值
    grant_type: 'authorization_code',
    redirect_uri: redirectUri,
  });
  
  const response = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: params.toString(),
  });
  
  const data = await response.json();
  
  return {
    accessToken: data.access_token,    // 短期令牌（1 小时）
    refreshToken: data.refresh_token,  // 长期令牌（永久）
    expiresIn: data.expires_in,        // 3600 秒
  };
}
```

### 3.4 Access Token vs Refresh Token

| 类型 | 有效期 | 用途 | 安全性 |
|------|--------|------|--------|
| **Access Token** | 1 小时 | 直接调用 API | 泄露影响小（很快过期） |
| **Refresh Token** | 永久 | 换新的 access token | 必须加密存储 |

**Token 刷新机制** (`google-oauth.ts:189-226`)：

```typescript
async function refreshAccessToken(refreshToken, clientId, clientSecret) {
  const params = new URLSearchParams({
    client_id: clientId,
    client_secret: clientSecret,
    refresh_token: refreshToken,
    grant_type: 'refresh_token',
  });
  
  const response = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    body: params.toString(),
  });
  
  const data = await response.json();
  
  return {
    accessToken: data.access_token,  // 新的 access token
    expiresIn: data.expires_in,
  };
}
```

**为什么这样设计？**
- Access token 过期了不需要用户重新授权，用 refresh token 自动换新的
- 即使 access token 被偷，1 小时后就失效
- Refresh token 必须妥善保管（见下一节加密存储）

---

## 四、AES-256-GCM 加密存储

### 4.1 为什么需要加密？

OAuth 拿到的 **refresh_token** 是永久有效的，如果明文存储：
- 攻击者拿到文件 → 可以随时换 access token → 访问用户所有数据

所以必须用 **硬件绑定的加密** 保护。

### 4.2 文件格式

**存储位置**：`~/.craft-agent/credentials.enc`

**文件结构** (`secure-storage.ts:15-24`)：

```
[Header - 64 字节]
├── Magic: "CRAFT01\0" (8 字节) - 文件标识
├── Flags: 4 字节 - 预留配置位
├── Salt: 32 字节 - PBKDF2 盐值（文件创建时随机生成）
├── Reserved: 20 字节 - 预留扩展

[Encrypted Payload - 可变长度]
├── IV: 12 字节 - AES-GCM 初始化向量（每次写入随机生成）
├── Auth Tag: 16 字节 - GCM 认证标签（防篡改）
└── Ciphertext: 可变 - 加密后的凭证 JSON
```

### 4.3 硬件 UUID 获取

**核心思想**：密钥从机器唯一的硬件 ID 派生，换机器就解不开。

**代码** (`secure-storage.ts:65-99`)：

```typescript
function getStableMachineId(): string {
  try {
    if (process.platform === 'darwin') {
      // macOS: IOPlatformUUID - 绑定主板，永远不变
      const output = execSync(
        'ioreg -rd1 -c IOPlatformExpertDevice | grep IOPlatformUUID',
        { encoding: 'utf-8' }
      );
      const match = output.match(/"IOPlatformUUID"\s*=\s*"([^"]+)"/);
      if (match?.[1]) return match[1];
      
    } else if (process.platform === 'win32') {
      // Windows: MachineGuid - 系统安装时生成
      const output = execSync(
        'reg query HKEY_LOCAL_MACHINE\\SOFTWARE\\Microsoft\\Cryptography /v MachineGuid',
        { encoding: 'utf-8' }
      );
      const match = output.match(/MachineGuid\s+REG_SZ\s+(\S+)/);
      if (match?.[1]) return match[1];
      
    } else {
      // Linux: machine-id - 系统安装时生成
      const machineIdPath = '/var/lib/dbus/machine-id';
      const altPath = '/etc/machine-id';
      if (existsSync(machineIdPath)) {
        return readFileSync(machineIdPath, 'utf-8').trim();
      } else if (existsSync(altPath)) {
        return readFileSync(altPath, 'utf-8').trim();
      }
    }
  } catch {
    // 失败降级：用户名 + home 目录
  }
  
  return `${userInfo().username}:${homedir()}`;
}
```

**硬件 UUID 来源**：

| 平台 | 来源 | 特点 |
|------|------|------|
| **macOS** | IOPlatformUUID | 绑定主板，除非换主板否则永不改变 |
| **Windows** | MachineGuid 注册表 | Windows 安装时生成，重装系统会变 |
| **Linux** | /var/lib/dbus/machine-id | 系统安装时生成，重装系统会变 |
| **Fallback** | 用户名 + home 目录 | 最弱的绑定，但总比硬编码强 |

### 4.4 密钥派生（PBKDF2）

**核心代码** (`secure-storage.ts:329-330`)：

```typescript
const PBKDF2_ITERATIONS = 100000;  // 10 万次迭代
const KEY_SIZE = 32;  // 256 位 = 32 字节

this.encryptionKey = pbkdf2Sync(
  stableMachineId,      // 输入：硬件 UUID
  salt,                 // 盐：从文件头读取（32 字节随机值）
  PBKDF2_ITERATIONS,    // 迭代次数：100000
  KEY_SIZE,             // 输出长度：32 字节
  'sha256'              // 哈希算法
);
```

**PBKDF2 参数解析**：

1. **输入材料**：`stableMachineId` (硬件 UUID)
2. **盐值**：文件头的 32 字节随机数，每个文件不同
3. **迭代次数**：10 万次，让暴力破解变慢
4. **输出长度**：32 字节 = AES-256 需要的密钥长度
5. **哈希算法**：SHA-256

**为什么安全？**

| 安全机制 | 作用 |
|----------|------|
| **硬件绑定** | 密钥依赖机器特定的 UUID，文件拷到别的机器解不开 |
| **加盐** | 即使两台机器 UUID 相同（概率极低），salt 不同导致密钥不同 |
| **慢哈希** | 10 万次迭代让攻击者暴力破解每次都要花大量时间 |
| **认证加密** | GCM 模式的 auth tag 防止文件被篡改（改了就解密失败） |

### 4.5 加密/解密流程

**加密写入**：

```
OAuth 拿到 refresh_token
    ↓
准备明文: JSON.stringify({ refresh_token, access_token, ... })
    ↓
获取密钥:
  - 读取硬件 UUID: getStableMachineId()
  - 生成随机 salt (32 字节)
  - PBKDF2 派生: pbkdf2Sync(uuid, salt, 100000, 32, 'sha256')
    ↓
AES-GCM 加密:
  - 生成随机 IV (12 字节)
  - cipher = createCipheriv('aes-256-gcm', key, iv)
  - ciphertext = cipher.update(plaintext) + cipher.final()
  - authTag = cipher.getAuthTag() (16 字节)
    ↓
写入文件:
  [Header: CRAFT01\0 + flags + salt + reserved]
  [Payload: iv + authTag + ciphertext]
```

**解密读取**：

```
读取文件: ~/.craft-agent/credentials.enc
    ↓
解析 Header:
  - 验证 Magic: "CRAFT01\0"
  - 提取 salt (32 字节)
    ↓
派生密钥:
  - 读取硬件 UUID: getStableMachineId()
  - PBKDF2: pbkdf2Sync(uuid, salt, 100000, 32, 'sha256')
    ↓
解析 Payload:
  - 提取 IV (12 字节)
  - 提取 authTag (16 字节)
  - 提取 ciphertext
    ↓
AES-GCM 解密:
  - decipher = createDecipheriv('aes-256-gcm', key, iv)
  - decipher.setAuthTag(authTag)
  - plaintext = decipher.update(ciphertext) + decipher.final()
    ↓
解析 JSON: JSON.parse(plaintext) → 拿到 refresh_token
```

---

## 五、完整数据流

把三个模块串起来：

```
1. 用户发起授权
   ↓
2. OAuth PKCE 流程:
   - 生成 code_verifier 和 code_challenge
   - 打开 Google 授权页面
   - 用户同意授权
   - Google 重定向回来，带上 code 和 state
   - 验证 state 防止 CSRF
   - 用 code + code_verifier 换 token
   ↓
3. 拿到 access_token 和 refresh_token
   ↓
4. 加密存储:
   - 获取硬件 UUID (IOPlatformUUID / MachineGuid / machine-id)
   - 生成随机 salt
   - PBKDF2 派生 AES-256 密钥 (10 万次迭代)
   - AES-GCM 加密 token
   - 写入 ~/.craft-agent/credentials.enc
   ↓
5. 下次启动:
   - 读取文件，提取 salt
   - 用同样的硬件 UUID + salt 派生密钥
   - 解密拿到 refresh_token
   ↓
6. 配置 MCP 服务 (例如 Google Drive MCP):
   - 传入 refresh_token
   - MCP 服务自动刷新 access_token (每小时)
   ↓
7. MCP 连接池管理:
   - 连接到 Google Drive MCP 服务
   - 注册工具: listFiles, createFile, ...
   - 代理工具名: mcp__google_drive__listFiles
   ↓
8. AI 调用工具:
   - AI: 调用 mcp__google_drive__listFiles()
   - 连接池: 查表 → {slug: "google_drive", originalName: "listFiles"}
   - 找到客户端: clients.get("google_drive")
   - 真实调用: client.callTool("listFiles", {})
   - Google Drive API 执行，返回文件列表
```

---

## 六、关键技术点总结

### 6.1 MCP 协议

- **问题**：每个服务 API 不同，AI 需要适配每一个
- **方案**：统一 MCP 协议，所有服务暴露标准格式的工具
- **核心**：连接池 + 工具代理名映射（避免冲突）

### 6.2 OAuth 2.0 PKCE

- **问题**：需要访问用户数据，直接要密码不安全
- **方案**：用户在官网授权，只给 AI 一个临时钥匙（token）
- **PKCE**：用哈希验证防止授权码被中途拦截
- **state 参数**：防止 CSRF 攻击（伪造的授权回调）

### 6.3 AES-256-GCM 加密

- **问题**：refresh_token 永久有效，必须安全存储
- **方案**：硬件绑定的 AES-256-GCM 加密
- **硬件绑定**：密钥从主板 UUID 派生，换机器解不开
- **PBKDF2**：10 万次迭代让暴力破解变慢
- **GCM 模式**：认证加密，防篡改

---

## 七、面试可能的问题

### Q1: 为什么用 MCP 协议而不是直接调 API？

**答案**：
- 统一接口：AI 不需要学每个服务的 API 文档
- 插件化：新增服务只需实现 MCP 规范，不用改 AI 代码
- 工具发现：MCP 服务启动时自动告诉 AI 有哪些工具可用

### Q2: OAuth 的 PKCE 流程为什么比传统 OAuth 更安全？

**答案**：
- 传统 OAuth：授权码被拦截 → 攻击者能换 token
- PKCE：授权时发哈希，换 token 时发原值 → 攻击者即使拦截授权码，没有原值也换不到 token

### Q3: 为什么 access token 只有 1 小时，而 refresh token 永久？

**答案**：
- Access token 用来调 API，万一泄露影响大 → 短期失效降低风险
- Refresh token 只用来换新 token，不直接调 API → 可以长期有效，加密存储即可

### Q4: AES-256-GCM 的 "GCM" 是什么？

**答案**：
- **GCM (Galois/Counter Mode)**：一种认证加密模式
- 除了加密，还生成 **auth tag**（认证标签）
- 解密时验证 auth tag，如果文件被篡改，auth tag 不匹配，解密失败

### Q5: 为什么密钥派生要迭代 10 万次？

**答案**：
- 让暴力破解变慢：攻击者每猜一个密码都要算 10 万次 PBKDF2
- 正常用户只在启动时算一次，不影响体验
- 平衡安全性和性能（10 万次大约几十毫秒）

### Q6: 硬件 UUID 如果换了怎么办？

**答案**：
- macOS 的 IOPlatformUUID 绑定主板，除非换主板否则不变
- Windows/Linux 的 machine-id 在重装系统时会变
- 如果换了 → 解密失败 → 用户需要重新授权 OAuth（重新登录 Google）
- 这是安全和便利的权衡：要么永久存储不安全，要么硬件绑定但换机器要重新授权

---

## 八、代码位置索引

| 模块 | 文件 | 关键行数 | 功能 |
|------|------|----------|------|
| MCP 连接池 | `packages/shared/src/mcp/mcp-pool.ts` | 102-131 | 三个核心 Map 定义 |
| 工具调用 | `packages/shared/src/mcp/mcp-pool.ts` | 387-436 | callTool 路由逻辑 |
| OAuth 授权 | `packages/shared/src/auth/google-oauth.ts` | 382-478 | PKCE 完整流程 |
| Token 交换 | `packages/shared/src/auth/google-oauth.ts` | 126-162 | 授权码换 token |
| Token 刷新 | `packages/shared/src/auth/google-oauth.ts` | 189-226 | refresh_token 换新 access_token |
| 文件格式 | `packages/shared/src/credentials/backends/secure-storage.ts` | 15-24 | Header + Payload 结构 |
| 硬件 UUID | `packages/shared/src/credentials/backends/secure-storage.ts` | 65-99 | 跨平台获取硬件标识 |
| 密钥派生 | `packages/shared/src/credentials/backends/secure-storage.ts` | 329-330 | PBKDF2 派生 AES-256 密钥 |

---

**第一个简历点完成！** 下一步继续第二个简历点：WebSocket RPC 协议 + 三层权限系统。
